---
Status: 🌳 Evergreen
Created: 2026-05-19
Last Updated: 2026-06-24
---

# Caching

## Table of Contents

- [What Is a Cache?](#what-is-a-cache)
- [Write Strategies](#write-strategies)
  - [Write-Through](#write-through)
  - [Write-Back (Write-Behind)](#write-back-write-behind)
  - [Write-Around](#write-around)
  - [Strategy Comparison](#strategy-comparison)
- [Read Strategies](#read-strategies)
  - [Cache-Aside (Lazy Loading)](#cache-aside-lazy-loading)
  - [Read-Through](#read-through)
- [Eviction Policies](#eviction-policies)
  - [LRU — Least Recently Used](#lru--least-recently-used)
  - [LFU — Least Frequently Used](#lfu--least-frequently-used)
  - [LRU vs LFU](#lru-vs-lfu)
- [Common Problems in Caching](#common-problems-in-caching)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)


## What Is a Cache?

A **cache** (from the French *cacher*, "to hide") is a fast, small, temporary storage layer placed between a slow data source (a database, a disk, a remote API) and a consumer (your application). It "hides" the expensive origin behind a cheap lookup.

**Why it exists:** A fundamental hardware asymmetry:

| Storage | Typical Access Latency |
|---|---|
| L1 CPU cache | ~1 ns |
| RAM | ~100 ns |
| SSD | ~100 µs |
| HDD | ~10 ms |
| Network (same DC) | ~1 ms |
| Network (cross-region) | ~100–200 ms |

A cache exploits this by storing recently or frequently accessed data closer to the reader.

**The core tension:** A cache is a *copy* — always potentially stale. Every caching strategy is about deciding when to synchronise with the source of truth and accepting what the lag costs you (stale reads, lost writes).

**Two fundamental operations every cache must answer:**

- **Read path:** Is the key in cache? (cache hit → return; cache miss → fetch from origin, optionally populate cache, return)
- **Write path:** When a write arrives, what gets written, where, and in what order?


## Write Strategies

Write strategies answer: **when a write comes in, what do we write to, and in what order?**

### Write-Through

```
Client
  │
  ▼
  DB ──── (success) ────► Cache ──── (success) ────► ACK client
  │                           │
  └── fails: return error     └── fails: delete cache key, return error
      (cache untouched)           (next read repopulates from DB)
```

**What it does:** Every write goes to the DB first, then updates the cache. The client receives an acknowledgement only after both succeed. The order is DB → cache, not cache → DB.

**Why DB first?** The DB is the source of truth; the cache is a derived copy. If the cache were written first, reads during the window between the cache write and the DB write would observe data that was never durably committed — a correctness violation. A missing cache entry causes a temporary performance issue (one extra DB read); a stale cache entry causes a correctness bug. Correctness always wins.

**Implementation:**

```go
func UpdateUser(user User) error {
    // Step 1: write to the durable source of truth first.
    if err := db.Update(user); err != nil {
        return err // cache is untouched; state is consistent
    }

    // Step 2: update the cache.
    if err := cache.Set(user.ID, user); err != nil {
        // Do not return the stale entry — delete it so the next read
        // falls through to the DB and repopulates the cache correctly.
        cache.Delete(user.ID)
        return err
    }

    return nil
}
```

**Failure scenarios:**

| Scenario | State After | Handling |
|---|---|---|
| DB write fails | DB = old, cache = old | Return error. Nothing to undo. Clean. |
| DB succeeds, cache update fails | DB = new, cache = stale | Delete cache key. Next read repopulates from DB. Return error. |
| Cache delete also fails | DB = new, cache = stale | Log critical alert. Push to retry queue. Background repair worker reconciles. |
| App crashes between DB commit and cache update | DB = new, cache = stale | TTL expiry eventually restores consistency. For stronger guarantees, use the transactional outbox pattern — write an outbox event atomically with the DB commit; a background worker updates the cache from the outbox. |
| Concurrent writes arrive out of order | DB = v2, cache = v1 | Store a version number or `updated_at` timestamp alongside cached values. Cache only accepts writes with a newer version than what it currently holds. |

**Why it exists:** Guarantees the cache and DB are always in sync for reads. There is no dirty data in the cache. Simple to reason about under normal operation.

**Problems it brings:**
- Write latency is DB-bound — the cache offers no write performance benefit
- Cache fills with data that may never be re-read (write-heavy workloads waste cache space)
- If the DB is slow, every write is slow regardless of the cache

**Good for:** Read-heavy workloads where read consistency matters — user profiles, product catalogues, permission tables.

### Write-Back (Write-Behind)

```
Client
  │
  ▼
Cache ──── ACK immediately to client
  │
  │  (async flush — on timer, eviction, or batch threshold)
  ▼
  DB
```

**What it does:** Writes go to the cache only. The client gets an immediate ACK. The cache flushes dirty entries to the DB asynchronously — on eviction, on a timer, or in batches.

Write-back is the **one strategy where cache-first is correct by design** — the cache is the primary write target. The DB is a deferred, async destination.

**Why it exists:** Dramatically improves write throughput. Absorbs write bursts. Useful when many overwrites happen to the same key before it ever needs to be persisted (e.g., a counter incremented 100 times per second — only the final value needs to reach the DB).

**Implementation:**

```go
func UpdateUser(user User) error {
    // Write to cache only; mark entry dirty. Client is ACKed here.
    if err := cache.Set(user.ID, user, MarkDirty); err != nil {
        return err // cache write failed; nothing inconsistent
    }
    return nil
    // No db.Update() call — flusher handles this asynchronously.
}

// FlushDirtyEntries runs in background — on timer, eviction, or size threshold.
func FlushDirtyEntries() {
    for _, entry := range cache.DirtyEntries() {
        if err := db.Update(entry); err != nil {
            retryQueue.Push(entry) // do not drop; retry with backoff
            continue
        }
        cache.ClearDirty(entry.ID) // mark clean ONLY after DB confirms
    }
}
```

**Failure scenarios:**

| Scenario | State After | Handling |
|---|---|---|
| Cache write fails | Cache = old, DB = old | Return error. Nothing inconsistent. |
| Async flush to DB fails | Cache = new (dirty), DB = old | Push to retry queue. Retry with backoff. Alert if retry budget exhausted. |
| Cache node crashes before flush | Cache = gone, DB = old | **Data permanently lost** — the fundamental durability trade-off of write-back. Mitigate with cache persistence (Redis AOF/RDB) or a write-ahead log. |
| Partial flush — some keys flushed, some not | Cache = mixed dirty/clean, DB = partial | Per-entry dirty flag ensures only unflushed entries are retried. |
| Concurrent writes to same key, flush out of order | DB = older value overwrites newer | Clear dirty flag only after DB confirms. Use per-key versioned writes so an older flush cannot overwrite a newer DB state. |
| DB permanently unavailable | Dirty entries accumulate indefinitely | Cap dirty set size. Once full, reject new writes or fall back to write-through. Alert immediately. |

**Critical discipline:** mark an entry dirty on cache write; clear it dirty **only after** the DB flush succeeds. Clearing before confirmation means a subsequent crash silently loses the entry with no retry.

**Flush Trigger Mechanisms**

The three triggers mentioned above are complementary — all three run simultaneously in a real system:

*On a timer* — a background goroutine wakes on a fixed interval and flushes all dirty entries. Not an OS cron job; an in-process ticker is standard (lower overhead, no external dependency, interval is seconds not minutes).

```go
func StartFlusher(interval time.Duration) {
    go func() {
        ticker := time.NewTicker(interval)
        defer ticker.Stop()
        for range ticker.C {
            FlushDirtyEntries()
        }
    }()
}
```

*On eviction* — most cache libraries expose an eviction callback that fires whenever a key is removed (by LRU pressure, TTL expiry, or explicit delete). The callback flushes the dirty entry before it disappears. Critical: once evicted, the entry is gone from the cache — if the DB write inside the callback fails, it must be pushed to a retry queue immediately; it will not be available for the next timer tick.

```go
cache := lru.NewWithEvict(capacity, func(key, value any) {
    entry := value.(CacheEntry)
    if entry.IsDirty {
        if err := db.Update(entry); err != nil {
            retryQueue.Push(entry) // evicted — must not lose it
        }
    }
})
```

Go libraries supporting eviction callbacks: `hashicorp/golang-lru` (`NewWithEvict`), `dgraph-io/ristretto`, `bluele/gcache`.

*On size threshold* — every `Set` checks whether the dirty entry count has crossed a configured limit. If yes, it triggers an immediate flush without waiting for the timer. Prevents a write burst from accumulating thousands of dirty entries that all hit the DB at once when the timer fires (a self-inflicted thundering herd).

```go
func (c *WriteBackCache) Set(key string, entry CacheEntry) error {
    c.mu.Lock()
    defer c.mu.Unlock()

    entry.IsDirty = true
    c.store[key] = entry
    c.dirtyKeys = append(c.dirtyKeys, key)

    if len(c.dirtyKeys) >= c.dirtyThreshold {
        go c.FlushDirtyEntries() // async so Set() returns immediately
    }
    return nil
}
```

How all three interact:

```
Write hits cache → dirty count crosses threshold? → YES → immediate batch flush
                                                  → NO  → wait
Timer fires (every N seconds) ──────────────────────────► flush all dirty entries
Entry evicted by LRU / TTL ─────────────────────────────► flush that entry now
                                                           or push to retry queue
```

The timer is the safety net ensuring nothing sits dirty indefinitely. The threshold caps the dirty set size. The eviction callback is the last line of defence before an entry leaves the cache entirely.

**Problems it brings:**
- **Durability risk:** if the cache node crashes before flushing, writes are lost permanently
- Complex recovery logic — must track which entries are dirty and unflushed
- DB lag is hard to bound — "eventually consistent" with an unbounded window

**Good for:** High write throughput scenarios — gaming leaderboards, analytics counters, shopping cart updates — where losing a few seconds of writes is acceptable.

### Write-Around

```
Client
  │
  ├──────────────────────────────► DB (all writes go here directly)
  │                                    │
  │                               success / fail
  │
  └── Cache is never touched on writes.
      Populated only on subsequent read misses.
```

**What it does:** Writes bypass the cache entirely and go straight to the DB. The cache is only populated lazily on a cache miss during a subsequent read.

**Why it exists:** Avoids polluting the cache with write-once/read-never data. Example: bulk log ingestion — you write millions of log lines that may never be queried. Putting them in cache evicts actually-hot data.

Write-around has the simplest failure surface of the three strategies — the cache is never involved in the write path.

**Implementation:**

```go
func UpdateUser(user User) error {
    // Write directly to DB. Cache is not touched — intentional.
    if err := db.Update(user); err != nil {
        return err
    }
    // Optional: evict stale entry so the next read gets fresh data
    // rather than serving the old cached value until TTL expires.
    cache.Delete(user.ID) // failure here is non-fatal; TTL handles it anyway
    return nil
}

func GetUser(id string) (User, error) {
    // Standard read-through layered on top of write-around.
    if user, ok := cache.Get(id); ok {
        return user, nil // cache hit
    }
    // Cache miss — fetch from DB and populate cache for future reads.
    user, err := db.Get(id)
    if err != nil {
        return User{}, err
    }
    cache.Set(id, user) // non-fatal if this fails; next read retries
    return user, nil
}
```

**Failure scenarios:**

| Scenario | State After | Handling |
|---|---|---|
| DB write fails | DB = old, cache = old | Return error. Nothing inconsistent. Simplest case. |
| DB write succeeds | DB = new, cache = old or absent | Intentional. First subsequent read is a cache miss → DB fetch → cache populate. |
| Cache population on read fails | DB = new, cache = absent | Non-fatal. Next read retries. DB is always correct. |
| Concurrent read during write window | Reader may get old cached value | Acceptable — write-around makes no consistency promise on cache during writes. If unacceptable for a specific key, explicitly `cache.Delete(key)` after the DB write (see implementation above). |
| Repeated reads of recently written data | All misses until cache warms | Expected cost of keeping the cache clean. Mitigate by explicitly populating the cache after a write if immediate re-reads are anticipated. |

**Problems it brings:**
- The first read after a write is always a cache miss — you pay DB latency at least once for freshly written data
- Higher read latency for recently written data during the cold period

**Good for:** Workloads where you write large amounts of data that won't be immediately re-read — video uploads, bulk imports, log pipelines.

### Strategy Comparison

| | Write-Through | Write-Back | Write-Around |
|---|---|---|---|
| Write target | Cache + DB (sync) | Cache only (async to DB) | DB only |
| Write latency | High (DB-bound) | Low (cache-speed) | Medium |
| Read after write | Always a cache hit | Cache hit (until eviction) | Cache miss on first read |
| Durability risk | None | Yes (crash = lost data) | None |
| Cache pollution | Possible | Possible | Avoided |
| Best for | Read-heavy, consistency matters | Write-heavy, some loss OK | Write-once, rarely re-read data |


## Read Strategies

The write strategies (write-through, write-back, write-around) answer the write path. Read strategies answer the read path: **how does data get into the cache, and who is responsible for it?**

There are two read strategies worth knowing. They are not alternatives to the write strategies — a system uses one read strategy and one write strategy together.

### Cache-Aside (Lazy Loading)

**What it is:** The application code manually handles all three steps on every read: check cache → on a miss, fetch from DB → write the result into cache. The application is fully in control of when and what gets cached.

**Why it's called "cache-aside":** The cache sits *beside* the application-to-DB path rather than *in* it. The application makes two separate calls — one to the cache, one to the DB on a miss — rather than all reads being intercepted by a single proxy. The alternative name, **lazy loading**, captures the timing: data enters the cache only on the first actual request, not proactively.

**Why it exists:** The cache only ever holds data that has genuinely been requested. Nothing is pre-populated speculatively. This avoids cache pollution and gives the application full control over TTL and key structure per entry.

### How It Works — Step By Step

```
Read request arrives
        │
        ▼
  Check cache
        │
   ┌────┴────┐
   │         │
  HIT       MISS
   │         │
   │         ▼
   │    Fetch from DB
   │         │
   │         ▼
   │    Write to cache (with TTL)
   │         │
   └────┬────┘
        │
        ▼
  Return value to client
```

```go
// GetUser demonstrates the cache-aside read pattern.
// The application explicitly handles all three steps.
func GetUser(id string) (User, error) {
    // Step 1: check cache.
    if user, ok := cache.Get(id); ok {
        return user, nil // cache hit — return immediately
    }

    // Step 2: cache miss — fetch from the source of truth.
    user, err := db.Get(id)
    if err != nil {
        return User{}, err
    }

    // Step 3: populate cache for future reads.
    // TTL prevents stale data from sitting indefinitely.
    cache.Set(id, user, ttl)

    return user, nil
}
```

### Problems It Brings

**Cold start penalty.** The first request for any key always misses and pays full DB latency. A cache restart or flush resets every key to cold, causing a latency spike until the working set warms again.

**Thundering herd (cache stampede).** When a popular key expires, many concurrent requests all miss simultaneously and race to the DB to reload it. This is the most common operational problem with cache-aside. See the Common Problems section for solutions.

**Stale reads.** If the DB is updated by an external process — a migration, a direct SQL update, a different service — the cache has no way to know and keeps serving the old value until TTL expires. The application must explicitly call `cache.Delete(key)` after any write it controls; external writes are always a staleness risk.

### Pairing Cache-Aside With Write Strategies

Cache-aside is a read strategy and is independent of the write strategy. The write strategy you pair it with determines the staleness risk on reads:

| Write Strategy Paired | Effect On Cache-Aside Reads |
|---|---|
| Write-Through | Low stale risk — cache is updated synchronously on every write |
| Write-Back | Low stale risk for in-flight data — cache has the latest write; risk only on crash before flush |
| Write-Around | Highest stale risk — writes bypass the cache entirely; stale until TTL expires or explicit `cache.Delete` |

**The most common real-world combination:** cache-aside (reads) + write-around (writes) + explicit `cache.Delete` after each write. Reads are lazy; writes go directly to the DB; the delete keeps staleness bounded by preventing a stale entry from surviving until TTL. This is the pattern used in most system design case studies (Twitter timelines, Facebook TAO, Amazon product pages).

---

### Read-Through

**What it is:** The cache acts as a transparent proxy between the application and the DB. The application only ever calls `cache.Get(key)`. On a miss, the cache itself fetches from the DB, populates itself, and returns the result — the application never writes DB-fetch logic.

```
Cache-Aside — app does all three steps:    Read-Through — app does one step:

  app → cache.Get()                          app → cache.Get()
  app → db.Get()       (on miss)                      ↓ (on miss, internal to cache)
  app → cache.Set()                              cache → db.Get()
                                                 cache → cache.Set()
                                                 cache → return to app
```

**Why it exists:** Simplifies application code — no miss-handling logic scattered across the codebase. The caching behaviour is centralised in one place (the cache client or middleware).

**Problems it brings:**
- The cache client or library must know how to fetch from the DB — this couples them, which can be awkward if the DB access pattern is complex
- Less flexible: the application cannot easily customise TTL or key structure per entry
- Same cold start and thundering herd risks as cache-aside

**When to prefer read-through over cache-aside:**
- When simplicity matters more than flexibility — one fewer step for the application to manage
- When using a cache library or proxy (e.g., AWS DAX for DynamoDB, read-through Redis clients) that natively supports it

**Refresh-Ahead (brief mention):** A third read strategy where the cache proactively refreshes entries *before* they expire, based on predicted access patterns. Eliminates cold start for known-hot keys but wastes resources if predictions are wrong. Rarely discussed in interviews but worth knowing the name.


## Eviction Policies

When the cache is full and a new entry must be stored, something must be evicted. Eviction policies answer: **what do we throw out?**

### LRU — Least Recently Used

**What it is:** Evict the entry that was accessed least recently — i.e., the one that has gone untouched for the longest time.

**Why it's called that:** Literally, the item used *least recently*.

**Intuition:** Temporal locality — if you accessed something recently, you're likely to access it again soon. If you haven't touched it in a while, you probably won't.

**Implementation — O(1) get and put:**

The standard approach is a **doubly linked list + hash map**:

```
HashMap: key → pointer to node in the list

Doubly Linked List (head = MRU, tail = LRU):

 [dummy head] ←→ [node A] ←→ [node B] ←→ [node C] ←→ [dummy tail]
  (MRU side)                                             (LRU side)
```

- **Get:** move the accessed node to the head (it's now MRU)
- **Put:** insert at head; if over capacity, evict the tail node

```go
// LRUCache implements a fixed-capacity Least Recently Used cache.
// Get and Put are both O(1).
type LRUCache struct {
    cap   int
    cache map[int]*Node
    head  *Node // dummy head — MRU side
    tail  *Node // dummy tail — LRU side
}

type Node struct {
    key, val   int
    prev, next *Node
}

func NewLRUCache(cap int) *LRUCache {
    head := &Node{}
    tail := &Node{}
    head.next = tail
    tail.prev = head
    return &LRUCache{
        cap:   cap,
        cache: make(map[int]*Node),
        head:  head,
        tail:  tail,
    }
}

// removeNode detaches a node from its current position in the list.
func (c *LRUCache) removeNode(n *Node) {
    n.prev.next = n.next
    n.next.prev = n.prev
}

// insertFront places a node immediately after the dummy head (MRU position).
func (c *LRUCache) insertFront(n *Node) {
    n.next = c.head.next
    n.prev = c.head
    c.head.next.prev = n
    c.head.next = n
}

// Get returns the value for key, or -1 if absent.
// Promotes the accessed node to MRU position.
func (c *LRUCache) Get(key int) int {
    if n, ok := c.cache[key]; ok {
        c.removeNode(n)
        c.insertFront(n)
        return n.val
    }
    return -1
}

// Put inserts or updates a key-value pair.
// If at capacity, evicts the LRU entry first.
func (c *LRUCache) Put(key, val int) {
    if n, ok := c.cache[key]; ok {
        c.removeNode(n)
        n.val = val
        c.insertFront(n)
        return
    }
    if len(c.cache) == c.cap {
        lru := c.tail.prev // node just before dummy tail is LRU
        c.removeNode(lru)
        delete(c.cache, lru.key)
    }
    n := &Node{key: key, val: val}
    c.insertFront(n)
    c.cache[key] = n
}
```

**Problems LRU brings:**
- **Scan resistance failure:** a single sequential scan (reading 1M items once) evicts all genuinely hot items — called LRU pollution or cache churn
- Ignores frequency — a page accessed once yesterday evicts a page accessed 999 times last week if the latter was accessed slightly less recently

### LFU — Least Frequently Used

**What it is:** Evict the entry that has been accessed the fewest total times.

**Why it's called that:** The item used *least frequently*.

**Intuition:** Frequency locality — a homepage accessed 10,000 times is more valuable than a product page accessed twice, regardless of when they were last accessed.

**Implementation — O(1) get and put:**

The approach: a **hash map of frequencies, each pointing to a doubly linked list of keys at that frequency**, plus a `minFreq` pointer.

```
keyMap:    key → node (stores key, val, freq)
freqMap:   freq → doubly linked list of nodes at that frequency

Example state (minFreq = 1):

  freq 1 → [D] ←→ [E]    (D and E accessed once)
  freq 3 → [A] ←→ [B]    (A and B accessed 3 times)
  freq 7 → [C]            (C accessed 7 times)

On eviction: remove from freqMap[minFreq]'s tail (LRU tiebreak within same freq)
```

On a `Get(key)`:
1. Find the node via `keyMap`
2. Remove it from `freqMap[node.freq]`
3. Increment `node.freq`; insert into `freqMap[node.freq]`
4. If `freqMap[minFreq]` is now empty, increment `minFreq`

On a `Put(key, val)`:
1. If at capacity, evict tail of `freqMap[minFreq]`
2. Insert new node with `freq = 1`; reset `minFreq = 1`

**Problems LFU brings:**
- **Historical frequency bias:** an item accessed 1,000 times last month but never this week will never be evicted — LFU has no time decay
- **Cold-start problem:** new items always start at freq=1 and are at immediate risk of eviction, even if they're becoming popular
- More complex to implement correctly than LRU

### LRU vs LFU

| | LRU | LFU |
|---|---|---|
| Eviction criterion | Least recently accessed | Least frequently accessed |
| Strength | Works well for temporal locality | Works well for popularity-based access |
| Weakness | Vulnerable to sequential scans; ignores frequency | Ignores recency; new items cold-start disadvantaged |
| Implementation complexity | Moderate (HashMap + DLL) | Higher (HashMap + per-freq DLL + minFreq pointer) |
| Use case | General caches, session stores, web server caches | Query result caches, media popularity caches, recommendation feeds |

**Note on real-world systems:** Redis and Memcached use *approximated* LRU — sampling N random keys and evicting the oldest among them — because maintaining an exact ordered list is expensive in distributed settings. Redis 4.0+ also ships an LFU approximation mode.


## Common Problems in Caching

### Cache Stampede (Thundering Herd)

A cache key expires. At that exact moment, 1,000 requests simultaneously get a cache miss and all rush to the DB to recompute the value. The DB gets a spike it may not handle.

**Solutions:**
- **Mutex / lock on the key:** only one request is allowed to recompute; others wait and then read the freshly cached value
- **Probabilistic early expiration:** start refreshing the cache slightly before TTL expires, using a random probability that increases as TTL approaches zero — so one request races ahead to refresh while others still serve the stale value
- **Stale-while-revalidate:** serve the stale cached value to all current requesters while one background process refreshes it

### Cache Penetration

Requests for keys that **don't exist** in the cache (and don't exist in the DB either) always miss the cache and hammer the DB.

**Example:** an attacker floods requests for non-existent user IDs.

**Solutions:**
- Cache negative results: cache a sentinel value (e.g., `null`) for missing keys with a short TTL
- **Bloom filter:** a probabilistic data structure placed in front of the cache that answers "definitely not in DB" with no false negatives — non-existent keys are rejected without ever reaching the cache or DB

### Cache Avalanche

Many cache keys expire at the same time (e.g., all set with the same TTL). A wave of cache misses hits the DB simultaneously.

**Solutions:**
- Add random jitter to TTL: `TTL = base_TTL + rand(0, jitter_range)` so expiries are spread out
- Use a layered cache (L1 in-process + L2 Redis) so L2 still absorbs many misses even if L1 expires

### Hot Key Problem

A single cache key receives a disproportionate number of requests (e.g., a viral tweet, a flash sale product). A single cache node becomes a bottleneck.

**Solutions:**
- **Local in-process caching** (L1 cache): cache hot keys in each application server's memory — eliminates network round-trips entirely
- **Key sharding:** replicate the hot key under multiple names (`hot_key_0`, `hot_key_1`, ...) and round-robin requests across them


## Interview Cheat Sheet

**Core trade-off framing:**
> Every caching decision trades consistency for performance. The question is always: how much staleness can this system tolerate, and what is the cost of a write being lost?

**Write strategy signals:**
- "User profile reads" → write-through (consistency on reads, writes are infrequent)
- "Leaderboard / counter" → write-back (high write throughput, some loss OK)
- "Log ingestion / bulk import" → write-around (don't pollute cache with write-once data)

**Read strategy signals:**
- "Most common pattern", "case study", "Twitter / Facebook / Amazon" → cache-aside (lazy loading); pair with write-around + explicit `cache.Delete` on write
- "Strong consistency after every write" → write-through (cache always synchronised, no stale reads)

**Eviction policy signals:**
- "General purpose, session data" → LRU (temporal locality)
- "Recommendation feed, popular content" → LFU (popularity/frequency)
- "You mentioned sequential scan risk" → mention LRU pollution; propose LFU or 2Q/ARC variants

**Common follow-up questions to expect:**
- "What's a cache stampede and how do you prevent it?" — mutex, probabilistic early expiry, stale-while-revalidate
- "What's the difference between cache penetration and cache avalanche?" — penetration = missing keys always miss; avalanche = many keys expire simultaneously
- "How does Redis implement LRU?" — approximated LRU via random sampling, not exact; Redis 4.0+ also supports LFU approximation
- "What's a hot key problem?" — single key overwhelmed; solutions: L1 in-process cache, key replication/sharding


## References

- [Redis eviction policies documentation](https://redis.io/docs/manual/eviction/)
- [An Analysis of Facebook's Memcache Architecture (NSDI 2013)](https://www.usenix.org/conference/nsdi13/technical-sessions/presentation/nishtala)
- [Caching Strategies — AWS Architecture Blog](https://aws.amazon.com/caching/best-practices/)