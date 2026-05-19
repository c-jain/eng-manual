# Caching

## Table of Contents

- [What Is a Cache?](#what-is-a-cache)
- [Write Strategies](#write-strategies)
  - [Write-Through](#write-through)
  - [Write-Back (Write-Behind)](#write-back-write-behind)
  - [Write-Around](#write-around)
  - [Strategy Comparison](#strategy-comparison)
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
Cache ──── (synchronous write) ────► DB
  │
  │  ACK only after DB confirms
  ▼
Client gets response
```

**What it does:** Every write is sent to both the cache and the DB, synchronously. The client receives an acknowledgement only after both succeed.

**Why it exists:** Guarantees the cache and DB are always in sync. There is no dirty data in the cache. Simple to reason about.

**Problems it brings:**
- Write latency is DB-bound — the cache offers no write performance benefit
- Cache fills with data that may never be re-read (write-heavy workloads waste cache space)
- If the DB is slow, every write is slow regardless of the cache

**Good for:** Read-heavy workloads where read consistency matters — user profiles, product catalogues, permission tables.

**NOTE:** In Write-Through, writes synchronously propagate through the cache layer to the database, but internally the system usually commits to the DB first and then updates/invalidate the cache before acknowledging the client.

### Write-Back (Write-Behind)

```
Client
  │
  ▼
Cache ──── ACK immediately to client
  │
  │  (async flush, on timer or eviction)
  ▼
  DB
```

**What it does:** Writes go to the cache only. The client gets an immediate ACK. The cache flushes dirty entries to the DB asynchronously — on eviction, on a timer, or in batches.

**Why it exists:** Dramatically improves write throughput. Absorbs write bursts. Useful when many overwrites happen to the same key before it ever needs to be persisted (e.g., a counter incremented 100 times per second).

**Problems it brings:**
- **Durability risk:** if the cache node crashes before flushing, writes are lost permanently
- Complex recovery logic — you need to track which entries are "dirty" (not yet flushed)
- "Eventually" consistent with the DB, but the lag is hard to bound

**Good for:** High write throughput scenarios — gaming leaderboards, analytics counters, shopping cart updates — where losing a few seconds of writes is acceptable.

### Write-Around

```
Client
  │
  ├──────────────► DB  (all writes go directly to DB)
  │
  └──── Cache is untouched on writes
              │
              │ (cache is only populated on cache-miss reads)
              ▼
```

**What it does:** Writes bypass the cache entirely and go straight to the DB. The cache is only populated lazily, on a cache miss during a subsequent read.

**Why it exists:** Avoids polluting the cache with write-once/read-never data. Example: bulk log ingestion — you write millions of log lines that may never be queried. Putting them in cache evicts actually-hot data.

**Problems it brings:**
- The first read after a write is always a cache miss — you always hit the DB at least once for freshly written data
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