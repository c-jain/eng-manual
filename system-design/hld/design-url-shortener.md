---
Status: 🌳 Evergreen
Created: 2026-06-21
Last Updated: 2026-07-02
---

# URL Shortener (TinyURL)

## Table Of Contents

1. [What It Is And Why It Exists](#what-it-is-and-why-it-exists)
2. [Why It Is Called "URL Shortener"](#why-it-is-called-url-shortener)
3. [Step 1 — Clarify Requirements](#step-1--clarify-requirements)
4. [Step 2 — Estimate Scale](#step-2--estimate-scale)
5. [Step 3 — High-Level Design](#step-3--high-level-design)
6. [Step 4 — Deep Dive: Short Code Generation](#step-4--deep-dive-short-code-generation)
7. [Step 4 — Deep Dive: Database And Storage](#step-4--deep-dive-database-and-storage)
8. [Step 4 — Deep Dive: Redirection](#step-4--deep-dive-redirection)
9. [Step 4 — Deep Dive: Caching, Expiration, And Abuse Prevention](#step-4--deep-dive-caching-expiration-and-abuse-prevention)
10. [Step 5 — Trade-Offs](#step-5--trade-offs)
11. [Step 6 — Wrap-Up](#step-6--wrap-up)
12. [Edge Cases And Extensions](#edge-cases-and-extensions)
13. [Problems This Design Brings](#problems-this-design-brings)
14. [How To Remember This](#how-to-remember-this)
15. [What Interviewers Actually Look For](#what-interviewers-actually-look-for)
16. [Interview Cheat Sheet](#interview-cheat-sheet)
17. [References](#references)

## What It Is And Why It Exists

A URL shortener takes an arbitrarily long URL and returns a short alias that, when visited, redirects the browser to the original URL. `https://example.com/products/category/electronics?ref=newsletter&utm_source=...` becomes `short.ly/aZ3kX9`.

**The problems it solves:**

- **Shareability** — short links fit in places long ones don't: SMS, print media, verbal communication, QR codes, social posts with character limits.
- **Link management** — the short link is a stable pointer. The destination behind it can be updated without reprinting business cards or re-sending the link.
- **Click analytics** — every redirect is a server hit, which means every click can be counted, timestamped, and geo-tagged, something a raw `<a href>` can never offer the link owner.
- **Aesthetics and trust** — a clean, branded short domain reads better than a URL with five query parameters.

This is a deceptively small-sounding problem that interviewers love precisely because it forces a candidate through every layer of the standard HLD toolkit in a tightly scoped space: encoding scheme, database choice, caching, redirection semantics, and abuse handling all show up in a system simple enough to fully design in 45 minutes.

## Why It Is Called "URL Shortener"

**URL** stands for **Uniform Resource Locator** — the addressing scheme standardized by Tim Berners-Lee in RFC 1738 (1994) to give every resource on the web a single, unambiguous address. **Shortener** is exactly what it sounds like: a service that produces a shorter alias for that address.

**TinyURL**, launched in 2002, was the first service to popularize the category — "tiny" was simply a synonym for "short," chosen as a punchy product name. It became popular enough, especially once Twitter's 140-character limit made every character precious in 2006, that "TinyURL" is now used informally the way "Kleenex" or "Google" are used: a brand name standing in for the whole category, even though Bitly, Rebrandly, and dozens of others now compete in the same space.

## Step 1 — Clarify Requirements

**Functional:**

- Given a long URL, generate a unique short URL that redirects to it.
- Given a short URL, redirect the client to the original long URL.
- Support an optional user-supplied custom alias (`short.ly/my-product`).
- Support an optional expiration time after which the link stops resolving.

**Non-functional:**

- **High availability** — a broken redirect is a dead link everywhere it was shared; this system should never be the reason a link stops working.
- **Low latency on redirect** — the redirect is on the critical path of someone else's user experience; it should resolve in single-digit milliseconds wherever possible.
- **Uniqueness** — two different long URLs must never resolve from the same short code.
- **Read-heavy at scale** — a link is written once and clicked many times; the read path is the one that has to scale.

**Explicitly out of scope** (state this out loud in an interview):

> "I'll focus on shorten-and-redirect, plus expiration and custom aliases. I'll treat user accounts, a full analytics dashboard, and malware/phishing URL scanning as pluggable services I'd wire in later, unless you'd like me to go deeper on one of those."

## Step 2 — Estimate Scale

Working through a concrete example, not because these exact numbers matter, but because the back-of-envelope math is what justifies every design decision that follows.

**Assumptions:**

```
New URLs created per day        : 1,000,000
Read : write ratio              : 100 : 1
Average stored entry size       : ~500 bytes  (long_url + short_code + metadata)
Retention horizon for storage   : 5 years
```

**Write throughput:**

```
1,000,000 writes/day ÷ 100,000 sec/day  ≈ 10 write QPS
(using 1 day ≈ 10^5 seconds — the standard interview approximation; true value is 86,400)
```

**Read throughput:**

```
10 write QPS × 100 (read:write ratio) = 1,000 read QPS
```

**Storage:**

```
1,000,000 × 500 bytes               = 500 MB / day
500 MB/day × 400 days/year × 5 years = 1,000 GB  = 1 TB over 5 years
(using 1 year ≈ 400 days — the standard interview approximation; true value is 365)
```

**Conclusions this forces**, exactly as the back-of-envelope step is supposed to:

- 10 write QPS is trivial for a single primary database — writes are not the bottleneck.
- 1,000 read QPS, overwhelmingly the same kind of query (single-key lookup by `short_code`), makes a **cache in front of the database mandatory**, not optional.
- ~1 TB over 5 years is small enough to fit comfortably on a single well-provisioned node, but the access pattern (point lookups, no joins) still argues for a key-value store over a relational one — more on that in the database deep dive.

**How long should a short code be?**

Over 5 years at this rate, the system issues `1,000,000 × 400 × 5 = 2 billion` short codes. A base62 alphabet (`0–9`, `a–z`, `A–Z` = 62 symbols) gives:

Use the approximation **62 ≈ 64 = 2^6**, so 62^k ≈ 2^(6k) — powers of 2 you already know:

```
62^5 ≈ 2^30 ≈  1 billion   ←  NOT enough (need 2B)
62^6 ≈ 2^36 ≈ 64 billion   ←  enough (32× the 5-year requirement)
62^7 ≈ 2^42 ≈  4 trillion  ←  way more than enough
```

In an interview: *"62 is close to 64 = 2^6, so a 6-char code gives ~2^36 ≈ 64 billion slots — well above the 2 billion needed. 6 characters suffices; industry uses 7 for safety margin."*

Six characters is technically sufficient for this scale. Most real systems still pick **seven** — the cost of one extra character is nothing, and the headroom (3.5 trillion codes) comfortably absorbs a decade of unexpectedly higher growth without a migration. This is a good thing to say out loud in an interview: it shows you did the math instead of guessing "7 because that's what TinyURL uses."

## Step 3 — High-Level Design

```
+-------------------+
|  Client           |  browser or mobile app
+-------------------+
         |
         v
+-------------------+
|  Load Balancer    |  routes traffic across app server instances
+-------------------+
         |
         v
+-------------------+
|  App Servers      |  stateless; horizontally scaled
+-------------------+
    |       |       |          |
    v       v       v          v
+-------+ +----+ +----------+ +---------------+
| Cache | | DB | | ID Range | | Message Queue |
| Redis | | KV | | Allocator| |               |
+-------+ +----+ +----------+ +---------------+
                                    |
                                    v
                             +------------+
                             | Analytics  |
                             | Service    |
                             +------------+
```

Two distinct request paths share the same App Servers:

**Write path (shorten a URL):** App Server asks the ID Range Allocator for the next counter ID, base62-encodes it into a short code, writes the `short_code → long_url` mapping to the DB, and returns the short URL to the client. The ID Range Allocator is a lightweight coordination service (or a DB sequence with range leasing) that hands each App Server a pre-allocated block of IDs — say, 1,000 at a time — so servers rarely need to coordinate and the counter never produces duplicates.

**Read path (redirect):** App Server looks up `short_code` in the Cache first. On a hit, it immediately returns the redirect. On a miss, it fetches from the DB, populates the Cache, then redirects. After redirecting, it drops a lightweight click event onto the Message Queue asynchronously — the client already has its redirect, so analytics recording never adds latency. A background Analytics Service drains the queue and writes to a separate clicks store.

The Cache absorbs the read-heavy traffic (100:1 ratio from the estimate). The Message Queue decouples click recording from the hot redirect path. The ID Range Allocator removes the counter coordination bottleneck from the write path.

**API design:**

```
POST /api/v1/shorten
  body:  { long_url: string, custom_alias?: string, expires_at?: timestamp }
  resp:  { short_url: string, short_code: string, expires_at: timestamp|null }

GET /{short_code}
  resp:  301 or 302, Location: <long_url>
```

**Data model:**

```
urls:
  short_code   (PK)        -- the 7-char base62 code; also the lookup key
  long_url     (text)
  created_at   (timestamp)
  expires_at   (timestamp, nullable)
  user_id      (FK, nullable)

clicks:                    -- written asynchronously, off the hot path
  short_code   (FK)
  clicked_at   (timestamp)
  referrer     (text, nullable)
  geo          (text, nullable)
```

Splitting `clicks` into its own table (or even its own analytics store entirely) matters: it keeps the hot, latency-critical `urls` table narrow and avoids a `click_count` column that needs a write on every single redirect.

## Step 4 — Deep Dive: Short Code Generation

This is the heart of the problem, and the part interviewers probe hardest. There are three standard approaches.

### A. Counter-Based (Base62 Encode An Auto-Increment ID)

A global, ever-increasing counter assigns each new URL the next integer. That integer is base62-encoded into the short code.

```go
// alphabet defines the base62 character set: 0-9, a-z, A-Z (62 symbols).
const alphabet = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
const base = uint64(len(alphabet))

// Encode converts a counter value into a base62 short code.
// Time: O(log62 n) — one division per output digit.
// Space: O(log62 n) — for the output buffer.
func Encode(n uint64) string {
    if n == 0 {
        return string(alphabet[0])
    }
    var b []byte
    for n > 0 {
        b = append(b, alphabet[n%base])
        n /= base
    }
    for i, j := 0, len(b)-1; i < j; i, j = i+1, j-1 {
        b[i], b[j] = b[j], b[i]
    }
    return string(b)
}

// Decode reverses Encode, recovering the original counter value.
// Time: O(k) where k is the code length. Space: O(1).
func Decode(code string) (uint64, error) {
    var n uint64
    for _, c := range code {
        idx := strings.IndexRune(alphabet, c)
        if idx < 0 {
            return 0, errors.New("invalid base62 character")
        }
        n = n*base + uint64(idx)
    }
    return n, nil
}
```

Worth testing here: a round-trip property test across known values, and specifically the 62⁶/62⁷ boundary computed above — confirming the capacity math actually holds at the edge, not just in the middle of the range.

**Dry run — `Encode(125)`:**

The algorithm is base conversion: repeatedly take the remainder mod 62 (that gives the current least-significant digit), then divide to peel it off. Digits come out least-significant first, so reverse at the end.

```
alphabet = "0123456789abc...xyzABC...XYZ"
                ^--- index 10 = 'a'

n=125: 125 % 62 = 1  → alphabet[1] = '1',  append '1',  n = 125/62 = 2
n=2:     2 % 62 = 2  → alphabet[2] = '2',  append '2',  n =   2/62 = 0
loop ends (n == 0)

b = ['1','2']  ← built least-significant digit first
reverse → ['2','1']
Output: "21"
```

Verify: `2×62 + 1 = 125` ✓

**Dry run — `Decode("21")`:**

Decode uses Horner's method: for each character left-to-right, multiply the running total by 62 then add the character's index. Most-significant digit first, so no reversal needed.

```
n = 0
'2': idx=2,  n = 0×62 + 2  = 2
'1': idx=1,  n = 2×62 + 1  = 125

Output: 125  ✓
```

**Complexity:**

`Encode(n)` — Time `O(log₆₂ n)`: the loop runs once per output digit (how many times can you divide n by 62 before reaching 0). For any counter up to 62^7 ≈ 4T, that is at most 7 iterations — effectively O(1). Space `O(log₆₂ n)`: the output buffer holds those digits, at most 7 bytes.

`Decode(code)` — Time `O(k)` where k is code length: one pass, one multiply-add per character, k ≤ 7 in practice, so O(1). Space `O(1)`: only the running accumulator, no buffer.

**Why this approach is attractive:** counters never repeat, so collisions are structurally impossible. No existence check is ever needed before insert.

**The problem it brings:** a single global counter is a single point of write contention — every app server needs the *next* value, and naively that means a synchronous round-trip to whatever holds the counter on every single shorten request.

**The standard fix — range allocation:** each app server, on startup (and whenever it runs low), asks a coordination service (Zookeeper, etcd) for a *block* of IDs, e.g. `[1,000,000, 1,001,000)`. It then hands out IDs from that block locally, in memory, with zero coordination overhead, until the block is exhausted and it asks for the next one. This is the same range-allocation idea used by distributed ID generators like Snowflake, applied here to a simple counter.

**A second problem — predictability:** sequential IDs mean short codes are sequential too (`Encode(1000)`, `Encode(1001)`, ...). Anyone can enumerate the entire URL corpus by incrementing through codes.

The fix is to **scramble the counter before encoding** so outputs look random while preserving two properties: (a) uniqueness — no two counters can scramble to the same value, and (b) reversibility — you can recover the original counter for debugging or auditing.

The simplest interview-safe approach is XOR with a secret constant:

```go
const secret = uint64(0x5a4bcdef12345678)

func scramble(n uint64) uint64   { return n ^ secret }
func unscramble(n uint64) uint64 { return n ^ secret } // XOR is its own inverse
```

XOR flips a fixed set of bits determined by `secret`. Two different inputs have different bit patterns, so their XOR outputs are also different — bijection guaranteed. The scrambled value is then passed to `Encode` as normal; base62 encoding is unchanged.

A stronger version is a **reversible bit-permutation**: instead of flipping bits, physically reorder all 32 (or 64) bits according to a secret mapping. Bit at position 0 moves to position 17, bit at position 1 moves to position 3, and so on. Because no bits are lost — just rearranged — the mapping is still a bijection and trivially reversed by applying the inverse mapping. In production, the standard tool is **format-preserving encryption** (e.g. AES-FFX): encrypt the counter with a secret key, producing an output of the same bit-width. Cryptographically strong, still a bijection, still reversible with the key.

### B. Hash-Based (Hash The Long URL, Truncate)

Hash the long URL (MD5 or SHA-256) and take the first 7 base62 characters of the result.

**Why this is attractive:** it's deterministic — shortening the same long URL twice always returns the same short code, which gives you free deduplication with no extra bookkeeping.

**The Problem It Brings — Collisions**

A 7-character base62 code encodes ~42 bits of information. "Bits of information" means: how many distinct values can this code represent? One base62 character can be one of 62 values, so it holds log₂(62) ≈ 5.95 bits. Seven characters: `7 × 5.95 ≈ 41.7 bits`, rounded to ~42 bits, meaning 62^7 ≈ 3.5 trillion possible codes.

MD5 produces a 128-bit hash. When you truncate it to 7 base62 characters, you squeeze 128 bits into 42 bits — discarding 86 bits. Two completely different long URLs can hash to different MD5 values but still share the same 7-character prefix. That is a collision.

By the birthday paradox, collisions become statistically likely well before the keyspace is half full. Every insert therefore needs a collision check: look up the candidate short code; if it exists, compare `long_url` — same URL means reuse the code (free deduplication); different URL means append a salt, re-hash, and retry.

**The Bloom Filter Optimisation**

Checking "does this short code already exist?" against the DB on every single write is expensive at scale. You want to rule out most candidates without touching the DB. That is what a Bloom filter does.

**What it is:** a bit array of size `m` (all zeros initially) plus `k` independent hash functions. It answers the question "is this item in the set?" with two possible answers: **definitely not** (certain) or **probably yes** (uncertain — confirm with DB).

**Why it's named that:** Burton Howard Bloom invented it in 1970. A "filter" because it filters out expensive DB lookups. The name is just the inventor's surname.

```
Bit array, size m (all zeros at start):
[ 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 ]
  0 1 2 3 4 5 6 7 8 9 ...         m-1

k = 3 hash functions: h1, h2, h3

--- INSERT "aZ3kX9q" (newly issued code) ---

h1("aZ3kX9q") → bit 3  → set bit[3]  = 1
h2("aZ3kX9q") → bit 9  → set bit[9]  = 1
h3("aZ3kX9q") → bit 14 → set bit[14] = 1

[ 0 0 0 1 0 0 0 0 0 1 0 0 0 0 1 0 ]

--- CHECK "aZ3kX9q" (collision candidate: is this code taken?) ---

h1 → bit[3]  = 1
h2 → bit[9]  = 1
h3 → bit[14] = 1
All set → "probably exists" → confirm with DB

--- CHECK "bX7mQ2p" (another candidate) ---

h1("bX7mQ2p") → bit[2] = 0  ← stop immediately
Result: "definitely not in set" → skip DB entirely
```

**The key asymmetry:** a false negative is impossible. If an item was inserted, all its bits were set to 1 and stay 1 (Bloom filters have no delete). So "definitely not present" is always correct. A false positive is possible — a new item's bits might all happen to already be 1 from *other* items' insertions, making the filter wrongly say "probably exists." This causes an unnecessary DB read, but never an incorrect result.

**What problems it brings:**
- No deletes: once a bit is set you cannot know which item set it, so you cannot safely unset it. A Counting Bloom filter supports deletes but costs more memory.
- False positive rate grows as the filter fills — tune `m` and `k` to keep it acceptable (typically < 1%).
- It is a pre-filter, not a replacement for the DB. It eliminates most DB reads; the DB remains the source of truth.

**How To Remember It:** a Bloom filter is a bouncer with imperfect memory. It never lets in someone who was never on the list (no false negatives). But occasionally it wrongly waves through a new person it thinks it recognises (false positives). The DB check behind it is the actual ID verification.

### C. Random Generation + Collision Retry

Generate 7 random base62 characters; check for existence (via the Bloom filter above, then the DB); regenerate on collision.

**Why this is attractive:** simplest to implement, no shared counter, no coordination service needed.

**The problem it brings:** retry rate climbs as the keyspace fills. At the scale estimated above (1.825 billion codes issued against a 3.52 trillion capacity, ~0.05% full), collisions stay rare for a very long time — but this approach degrades gracefully *worse* than the other two as utilization climbs toward saturation, which is worth naming even if it isn't a near-term concern.

### Comparison

| Approach | Collision-Free? | Deterministic? | Coordination Needed | Predictability Risk |
|---|---|---|---|---|
| Counter + base62 | Yes, structurally | No | Yes (range allocator) | Yes, unless permuted |
| Hash + truncate | No — needs retry logic | Yes (same input → same output) | No | No |
| Random + retry | No — needs retry logic | No | No | No |

## Step 4 — Deep Dive: Database And Storage

**Access pattern:** the overwhelming majority of queries are a single-key point lookup — `SELECT long_url WHERE short_code = ?` — with no joins and no range scans. This access pattern is the textbook case for a **key-value store** (DynamoDB, Cassandra): O(1)-ish lookups, horizontal scalability by partitioning on the key itself, and no relational features being paid for and unused.

A relational database (PostgreSQL) is a perfectly reasonable choice too at the scale estimated here — ~1,200 read QPS and ~1 TB of data fit comfortably on a well-indexed single primary with read replicas, and the operational simplicity of a single, well-understood technology is a real advantage for a small team. The justification to give an interviewer either way: *"At this scale either works; I'm choosing X because [simpler ops / proven horizontal scaling], and I'd revisit if read QPS grew another two orders of magnitude."*

**Sharding, if and when needed:** shard by `short_code` (or a hash of it), not by any other field — every hot-path query already arrives with the short code in hand, so this keeps every redirect a single-shard lookup. Sharding by `long_url` would help nothing, since redirects never query by long URL.

**Why click counts live in a separate table, written asynchronously:** incrementing a `click_count` column synchronously on every redirect means every single read also becomes a write — directly undermining the read-heavy optimizations (caching, replicas) the rest of the design depends on. Instead, the redirect handler publishes a `{short_code, timestamp, ...}` event onto a message queue and returns immediately; a separate consumer aggregates click data without ever touching the hot path. (See `hld/message-queues.md` for the queue mechanics this leans on.)

## Step 4 — Deep Dive: Redirection

For the definitions of 301 and 302, see `hld/networking-basics.md` (Status Codes). What matters here is the trade-off between them, which is the single most-asked follow-up question on this problem. Shown as two separate scenarios since the same boxes mean different things in each:

**Scenario A — 301 Permanent Redirect:**

```
Click 1: Browser --request--> App Server --301, Location--> Browser
         Browser CACHES the short_code -> long_url mapping itself.

Click 2: Browser uses its OWN cached mapping and navigates directly.
         App Server is never contacted again for this short_code.
```

**Scenario B — 302 Temporary Redirect:**

```
Click 1: Browser --request--> App Server --302, Location--> Browser navigates (not cached)

Click 2: Browser --request--> App Server --302, Location--> Browser navigates (not cached)
         App Server is contacted on EVERY click, every time.
```

**The trade-off:** 301 is faster for the end user after the first click (zero server round-trips) and cuts server load dramatically for repeat visits — but the link owner loses the ability to see repeat-click analytics, and cannot change the destination after a browser has cached the old mapping.

**302 keeps every click observable and every destination changeable** (useful for time-limited promotions or A/B-tested landing pages) at the cost of hitting the server on every single click.

**In practice, most production URL shorteners use 302**, specifically *because* click analytics is usually a core product requirement, not an afterthought — and that's the answer most interviewers are listening for, paired with the reasoning above rather than a memorized fact.

## Step 4 — Deep Dive: Caching, Expiration, And Abuse Prevention

**Caching:** with a 100:1 read:write ratio, a cache in front of the database (Redis) is not an optimization, it's load-bearing. The read strategy is cache-aside (see `hld/caching.md`) — load on miss, lazy population, never pre-fill on write since most newly created codes may never be clicked. LRU eviction is a sensible default given the extreme popularity skew (a handful of viral URLs absorb most traffic). Risk: thundering herd on popular key expiry — mitigate with TTL jitter or request coalescing.

**Expiration:** an `expires_at` column supports two complementary mechanisms — **lazy checks** (on every read, compare against `expires_at`; if past, return a `410 Gone` response instead of redirecting — `410` signals the resource existed but has been permanently removed, see `hld/networking-basics.md` for the full HTTP status code reference) and a **background sweep** (a periodic batch job that deletes expired rows from the DB and evicts them from the cache). Lazy alone is simplest but leaves dead rows accumulating in storage; sweeping keeps storage tidy at the cost of an extra moving part. Most systems run both — lazy for correctness, sweep for hygiene.

**Abuse prevention:**

- **Rate limit link creation** per IP/user (token bucket, already covered in `hld/rate-limiting.md`) — without this, the shorten endpoint is an open invitation to spam-generate millions of links.
- **Open-redirect / phishing risk:** a trusted, well-known short domain is an attractive disguise for a malicious destination URL — `short.ly/aZ3kX9` reveals nothing about where it actually goes. Production systems screen submitted URLs against a malware/phishing blocklist (e.g. Google Safe Browsing) before issuing a short code. Worth naming explicitly even though implementing the scanner itself is out of scope.

## Step 5 — Trade-Offs

| Decision | Option A | Option B | This Design Chooses |
|---|---|---|---|
| Short code generation | Counter + base62 (collision-free, needs coordination) | Hash + truncate (deterministic, needs collision handling) | Counter + range allocation — simpler invariants at this scale |
| Storage | Key-value store (simple ops at huge scale) | Relational (joins, ACID, familiar tooling) | Either is defensible; justify against actual read QPS |
| Redirect status | 301 (fast, cacheable, no repeat analytics) | 302 (every click observable, slower) | 302 — analytics is a stated requirement |
| Click tracking | Synchronous increment (simple, slow) | Async via queue (fast, eventually consistent) | Async — protects the hot read path |
| Expiration | Lazy-only (simple, storage grows) | Lazy + background sweep (tidy, extra moving part) | Both — correctness now, hygiene later |

## Step 6 — Wrap-Up

We designed a URL shortener using a range-allocated counter for collision-free short code generation, a key-value store optimized for single-key redirect lookups, a cache absorbing the 100:1 read-heavy traffic, and asynchronous click tracking that keeps analytics off the hot path. The biggest production risk is a single short code going viral and creating a hot-key read spike even through the cache — I'd address that with edge/CDN caching for the most-requested redirects. Given more time, I'd add URL safety scanning on creation, multi-region replication for global latency, and a self-serve analytics dashboard.

## Edge Cases And Extensions

Every item named as out of scope in Step 1 gets a concrete answer here — "out of scope" should mean "deferred," not "no plan."

**User accounts:** the current design treats every URL as anonymous. Adding accounts needs a `users` table and a nullable `user_id` foreign key on `urls`, plus a standard session/JWT auth layer in front of the API. This unlocks a "my links" list (query `urls` by `user_id`) and lets rate limiting switch from per-IP to per-user, which is harder to spoof. One design choice worth naming explicitly: whether custom aliases stay globally unique across all users, or become namespaced per-user (`short.ly/alice/promo` vs a flat global `short.ly/promo`) — the current schema assumes global uniqueness, and switching to per-user namespacing would change the uniqueness constraint on `custom_alias` from a single-column unique index to a composite `(user_id, custom_alias)` index.

**Full analytics dashboard:** the design already writes every click asynchronously to a `clicks` store — see Step 4 (Database And Storage) — specifically so a dashboard is a downstream read problem, not a redesign. A periodic aggregation job rolls up raw click events into an `analytics_summary` table (clicks per short_code per hour, top referrers, geo breakdown); the dashboard's read API queries only the summary table, never the raw event firehose. Same "async, off the hot path" principle already used for click tracking itself, applied one layer further downstream.

**Malware/phishing URL scanning:** covered briefly under Abuse Prevention above. Unlike user accounts and the analytics dashboard, this is a small pluggable interface an interviewer could reasonably ask to see written out — HLD interviewers rarely want a full auth system in Go, but a one-method safety check is fair game.

```go
// URLSafetyChecker abstracts screening a destination URL against a
// malware/phishing blocklist before a short code is issued for it.
// Injected as an interface so the real provider (Google Safe Browsing,
// an internal blocklist, etc.) can change without touching the handler.
type URLSafetyChecker interface {
    IsSafe(longURL string) (bool, error)
}

// ShortenRequest is what the shorten handler checks before ever touching
// the counter or the database — reject early, spend nothing.
func ShortenRequest(checker URLSafetyChecker, longURL string) error {
    safe, err := checker.IsSafe(longURL)
    if err != nil {
        return fmt.Errorf("safety check: %w", err)
    }
    if !safe {
        return fmt.Errorf("url rejected: flagged as unsafe")
    }
    return nil
}
```

Worth testing here: a rejection case against a mock blocklist, and a pass-through case for a URL not on it. This runs synchronously at submission time, before a short code is even issued — affordable because it sits on the low-QPS write path (~10 QPS from Step 2), not the high-QPS redirect read path. The same reasoning that keeps click tracking off the read path applies here in reverse: expensive checks belong wherever the traffic is lightest.

## Problems This Design Brings

- **Hot-key / viral link problem** — one extremely popular short code can dominate read traffic even with caching in place if it briefly falls out of cache (thundering herd on repopulation). Mitigation: CDN-level edge caching for top-N hottest redirects, longer TTLs for high-traffic keys.
- **Counter as a coordination bottleneck** if range allocation isn't implemented — solved above, but easy to miss if rushed.
- **Predictable/enumerable IDs** if the counter is exposed without permutation — a real privacy and scraping concern.
- **Liability for hosted destinations** — the shortener becomes infrastructure that bad actors can exploit for phishing, since the destination is hidden behind a trusted-looking domain.
- **Storage growth without sweeping** — expired links left unswept accumulate indefinitely.

## How To Remember This

**The three-step cycle: Generate → Store → Redirect.** Every URL shortener, regardless of implementation details, is this loop. Anchor any deep-dive question to "which of these three steps does this affect?"

**For the generation strategies:** *"Counter for guarantees, Hash for determinism, Random for simplicity."* Counter guarantees no collisions ever; hash guarantees the same input always maps to the same output; random is the simplest to reason about but needs retry logic.

**For 301 vs 302:** *301 = browser remembers, server forgets you. 302 = server remembers, browser forgets nothing.* Whichever side does the remembering is the side that can see/control what happens next.

## What Interviewers Actually Look For

| Signal | Green Flag | Red Flag |
|---|---|---|
| Short code strategy | Names the collision trade-off unprompted, picks a strategy and defends it | Jumps to "just use a hash" without mentioning collisions |
| Scale-driven reasoning | Uses the read:write ratio to justify caching, not just because "caching is good" | Adds caching reflexively without connecting it to the estimation step |
| Redirect semantics | Volunteers the 301 vs 302 trade-off without being asked | Doesn't know the difference, or states one as simply "correct" |
| Hot path discipline | Keeps click tracking off the synchronous read path | Increments a counter column on every redirect without noticing the cost |
| Security awareness | Raises ID predictability or phishing/open-redirect risk | Treats the design as purely a happy-path encoding problem |

## Interview Cheat Sheet

```
Estimate:    write QPS  = new_urls_per_day / 100,000   (1 day ≈ 10^5 sec)
             read QPS   = write QPS × read:write ratio
             code length: pick base62^k > total URLs expected over your horizon

Generation:  Counter+base62   -> collision-free, needs range-allocated IDs
             Hash+truncate    -> deterministic, needs collision retry + Bloom filter
             Random+retry     -> simplest, degrades as keyspace fills

Storage:     Key-value store for single-key short_code -> long_url lookups
             Shard by short_code (the field every hot-path query already has)

Redirect:    301 = cached by browser, fast, NO repeat analytics
             302 = hits server every time, slower, FULL analytics

Caching:     Mandatory at 100:1+ read:write ratio. Cache-aside, LRU eviction.

Click data:  Async via queue — never block the redirect to record analytics.

Abuse:       Rate limit creation. Screen URLs for phishing/malware.
```

## References

- *System Design Interview* Vol. 1 — Alex Xu, Chapter 8 (Design A URL Shortener)
- [RFC 1738 — Uniform Resource Locators](https://www.rfc-editor.org/rfc/rfc1738)
- ByteByteGo Blog — https://blog.bytebytego.com
- `hld/rate-limiting.md`, `hld/caching.md`, `hld/message-queues.md` — referenced strategies, not re-derived here