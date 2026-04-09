# HLD Interview Framework

## Table of Contents

1. [The 6-step loop](#1-the-6-step-loop)
2. [Step 1 — Clarify requirements](#2-step-1--clarify-requirements)
3. [Step 2 — Estimate scale](#3-step-2--estimate-scale)
4. [Step 3 — High-level design](#4-step-3--high-level-design)
5. [Step 4 — Deep dive](#5-step-4--deep-dive)
6. [Step 5 — Trade-offs](#6-step-5--trade-offs)
7. [Step 6 — Wrap-up](#7-step-6--wrap-up)
8. [What interviewers actually evaluate](#8-what-interviewers-actually-evaluate)
9. [Interview template (copy-paste)](#9-interview-template-copy-paste)
10. [Quick reference cheat sheet](#10-quick-reference-cheat-sheet)
11. [References](#11-references)

---

## 1. The 6-step loop

```
┌─────────────────────────────────────────────┐  time budget
│  1. Clarify requirements                    │  3–5 min
│     ↓                                       │
│  2. Estimate scale                          │  3–5 min
│     ↓                                       │
│  3. High-level design                       │  10–15 min
│     ↓                                       │
│  4. Deep dive (interviewer-directed)        │  10–15 min
│     ↓                                       │
│  5. Trade-offs                              │  5–8 min
│     ↓                                       │
│  6. Wrap-up                                 │  2–3 min
└─────────────────────────────────────────────┘
                                    total ≈ 35–45 min
```

**Meta-principle:** Interviewers evaluate *structured thinking under ambiguity*, not encyclopedic knowledge. A candidate who clarifies well and voluntarily surfaces trade-offs beats one who memorises architectures but can't adapt.

Spending the right *time* on each step is as important as knowing the content. Most candidates under-invest in steps 1 and 5 and over-invest in step 3.

---

## 2. Step 1 — Clarify requirements

**Never skip this. Interviewers give vague prompts deliberately ("design Twitter", "design YouTube").**

Your goal: narrow scope and surface assumptions before drawing a single box. This step demonstrates systems thinking, not just systems knowledge.

### Functional requirements — what the system does

Ask what features are in scope:

- "What are the core user actions?" (create, read, follow, search, like…)
- "Is the feed chronological or ranked?"
- "Do we need real-time delivery, or is eventual consistency fine?"
- "Should we support multimedia — images, video?"
- "Any geographic constraints — global or single region?"
- "Do we need strong consistency anywhere, or is eventual OK throughout?"

### Non-functional requirements — quality attributes

| Attribute | Example clarifying question |
|-----------|----------------------------|
| Scale | "How many DAU? What's the expected growth rate?" |
| Latency | "P99 read latency target? Any SLA commitments?" |
| Availability | "Five-nines, or can we tolerate brief downtime?" |
| Consistency | "For a social feed, is stale data acceptable?" |
| Durability | "Can we ever lose a write, or must everything persist?" |
| Security | "Any PII handling, encryption at rest/in transit?" |

### What to explicitly state as out-of-scope

Explicitly say what you won't design. This shows disciplined scoping:

> "I'll focus on tweet creation and feed read flow. I'll leave search, recommendations, and ads out of scope — let me know if you'd like to cover any of those."

### Interviewer signal

Do you know *what* to ask? Do you *listen* and update your design accordingly? Jumping to boxes without clarifying is a red flag. Equally bad: stalling with too many questions for 15 minutes.

---

## 3. Step 2 — Estimate scale

Back-of-envelope math isn't a formality — it *drives architectural decisions*. What you calculate here should directly change what you design in step 3.

### Standard estimation chain

```
DAU × active_write_fraction → writes/day
writes/day ÷ 86,400          → write QPS
write QPS × read:write ratio → read QPS
writes/day × avg_payload     → storage/day
storage/day × retention      → total storage
```

### Worked example — "Design Twitter"

```
Inputs (from clarification):
  DAU = 100M
  10% post per day
  Read:write ratio = 100:1
  Avg tweet payload = 300 bytes
  Media multiplier = 10×
  Avg follower count = 200

Writes:
  100M × 10% = 10M tweets/day
  10M ÷ 86,400 ≈ 116 write QPS  → round to ~120

Reads:
  120 × 100 = ~12,000 read QPS

Storage:
  10M × 300B = 3 GB/day (text)
  ×10 for media → ~30 GB/day total

Fan-out (timeline writes):
  10M posts × 200 followers = 2B timeline writes/day
  2B ÷ 86,400 ≈ 23,000 timeline write QPS
```

**Conclusions this forces:**
- 12k read QPS → read replicas + caching are non-negotiable
- 23k timeline write QPS → a single relational DB cannot absorb this → Cassandra or a dedicated timeline store
- Fan-out at this scale is the hard architectural problem (push vs pull vs hybrid)

### Rules of thumb to memorise

| Quantity | Value |
|----------|-------|
| Seconds in a day | 86,400 ≈ 10^5 |
| Bytes in 1 GB | 10^9 |
| L1 cache hit | ~1 ns |
| Memory access | ~100 ns |
| SSD random read | ~100 µs |
| Disk seek | ~10 ms |
| Same-DC round trip | ~0.5 ms |
| Cross-region round trip | ~150 ms |

---

## 4. Step 3 — High-level design

Draw the **minimal viable architecture** first. Get interviewer buy-in before going deep. Over-engineering the first pass is a common mistake.

### Core component template

```
Client (web / mobile)
    │
    ▼
Load Balancer / API Gateway
    │
    ├──► App Servers (stateless, horizontal scale)
    │         │
    │         ├──► Primary DB (PostgreSQL / Cassandra)
    │         ├──► Cache (Redis / Memcached)
    │         ├──► Object Store (S3, GCS — for media)
    │         └──► Message Queue (Kafka, RabbitMQ)
    │                    │
    │                    └──► Async Workers / Consumers
    │
    └──► CDN (static assets, media delivery)
```

### API design

Always sketch 2–3 key endpoints. This defines your system's surface area:

```
POST   /v1/tweets
       body: { text: string, media_ids: []string }
       → { tweet_id: string, created_at: timestamp }

GET    /v1/feed?user_id={id}&cursor={cursor}&limit={n}
       → { tweets: [Tweet], next_cursor: string }

GET    /v1/tweet/{id}
       → Tweet

DELETE /v1/tweet/{id}
       → 204 No Content
```

### Data model

Identify entities and relationships early — this guides your DB choice:

```
users:    user_id (PK), username, email, created_at
tweets:   tweet_id (PK), user_id (FK), content, created_at, like_count
follows:  follower_id, followee_id, created_at      ← composite PK
likes:    user_id, tweet_id, created_at             ← composite PK
timeline: user_id, tweet_id, tweet_created_at       ← pre-computed, Cassandra
```

### DB choice decision tree

| Criteria | Choose |
|----------|--------|
| Complex queries, joins, strong consistency, ACID | PostgreSQL / MySQL |
| High write throughput, global distribution, wide rows | Cassandra / DynamoDB |
| Flexible schema, nested documents | MongoDB |
| Ultra-low latency reads, simple key lookup | DynamoDB / Redis |
| Search, full-text, faceted filtering | Elasticsearch |

Always justify your choice: "I'm using Cassandra for the timeline table because writes are the bottleneck — Cassandra's LSM-tree architecture handles 20k+ writes/sec comfortably, and we only need simple key-range reads."

---

## 5. Step 4 — Deep dive

The interviewer will steer you into one or two components. Know all of these well.

### Feed generation — the canonical deep-dive topic

```
┌─────────────────────────────────────────────────────────────────┐
│  Push (fan-out on write)                                        │
│                                                                 │
│  On post → write tweet_id to every follower's timeline cache   │
│                                                                 │
│  + Fast read: O(1) cache hit on pre-built timeline             │
│  − Write amplification: celebrity with 10M followers =         │
│    10M timeline writes per post                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  Pull (fan-out on read)                                         │
│                                                                 │
│  On read → fetch tweets from all followees, merge, rank        │
│                                                                 │
│  + Simple writes (O(1) — just store the tweet)                 │
│  − Slow reads: merge N sorted lists at read time               │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  Hybrid — used by Twitter/X                                     │
│                                                                 │
│  Push for normal users (≤ N followers)                         │
│  Pull for celebrity accounts (> N followers)                   │
│  At read time: merge pre-built timeline + celebrity tweets      │
│                                                                 │
│  ✓ Bounded write amplification + fast reads                    │
└─────────────────────────────────────────────────────────────────┘
```

### Database deep-dive

**Indexing:**
- Primary index on `tweet_id` (Snowflake ID or ULID — sortable by time)
- Composite index on `(user_id, created_at DESC)` for user's own tweet history
- Avoid `SELECT *` — project only needed columns; covering index avoids table lookup

**Sharding:**
- Shard by `user_id`: user-centric queries stay on one shard — good for profile reads
- Shard by `tweet_id`: write load distributed — but cross-user queries hit all shards
- Hot shard risk: celebrity accounts. Mitigations: shard splitting, application-level routing, dedicated celebrity shards

**Read replicas:**
- Route all reads to replicas; writes to primary
- Accept slightly stale reads (replication lag ~ms)
- Typical: 1 primary + 2–3 replicas per region

### Caching strategy

| Strategy | How it works | Use when |
|----------|-------------|---------|
| Cache-aside (lazy) | Miss → load from DB → populate cache | Read-heavy, cache misses tolerable |
| Write-through | Every write → DB + cache synchronously | Reads must stay fresh |
| Write-behind | Write → cache → async flush to DB | High write throughput, durability risk OK |
| Read-through | Cache handles miss and DB load | Simplify application code |

**Cache invalidation:**
- TTL-based: simple, eventually consistent
- Event-driven: on write, publish invalidation event → cache service deletes key
- Versioned keys: `timeline:{user_id}:v{version}` — never invalidate, just version up

### Rate limiting

```
Token bucket:
  - Each user gets N tokens refilled per second
  - Each request consumes 1 token; allows bursting up to bucket size
  - Implementation: Redis counter + TTL
  - Use for: general APIs, burst-friendly endpoints

Sliding window counter:
  - Track request count in rolling time window
  - Implemented with Redis sorted set (score = timestamp)
  - No boundary spike: more precise than fixed window
  - Use for: strict limits (payment APIs, OTP generation)
```

### Async processing with message queues

```
Producer (app server)
    │
    ▼  event: { type: "new_tweet", tweet_id: X, user_id: Y }
Message Queue (Kafka topic / SQS)
    │
    ▼
Consumer group (async workers)
    ├── Fan-out service     → write to follower timelines
    ├── Notification service → push, email, SMS
    └── Analytics service   → event logging, counters

Design considerations:
  At-least-once delivery → consumers must be idempotent (dedup by tweet_id)
  Dead letter queue (DLQ) → after N retries, route to DLQ for inspection
  Backpressure → consumers signal when overwhelmed; don't blow up the DB
```

---

## 6. Step 5 — Trade-offs

State every decision as a trade-off. This is the senior-engineer signal. Never present one design as "the right answer" — always say what you're optimising for and what you're giving up.

### SQL vs NoSQL

| Dimension | SQL (PostgreSQL, MySQL) | NoSQL (Cassandra, DynamoDB) |
|-----------|------------------------|----------------------------|
| Consistency | Strong (ACID) | Eventual (tunable in Cassandra) |
| Schema | Rigid, relational | Flexible, denormalised |
| Scaling | Vertical + limited horizontal | Horizontal by design |
| Queries | Complex joins, aggregations | Simple key/range lookups |
| Transactions | Native multi-row ACID | Limited (Cassandra LWT) |
| Use when | Financial data, complex relations | High write throughput, global scale |

### CAP theorem

```
CP (Consistency + Partition tolerance):
  Reject reads/writes rather than serve stale/inconsistent data
  → Banking, inventory, booking systems

AP (Availability + Partition tolerance):
  Serve possibly stale data — system stays available
  → Social feeds, DNS, shopping carts, counters
```

**PACELC** (practical extension): even without a partition, systems trade off latency vs consistency. High-consistency reads (quorum reads in Cassandra) are slower than low-consistency reads.

### Synchronous vs asynchronous

| Dimension | Synchronous | Asynchronous |
|-----------|-------------|--------------|
| Caller response | Immediate result | ACK; result arrives later |
| Error handling | Simple — caller sees error directly | Complex — retry, idempotency, DLQ |
| Throughput under load | Degrades — callers block | Queue absorbs spikes |
| Use for | User-facing reads, payment confirmation | Fan-out writes, notifications, analytics |

### Monolith vs microservices

| Dimension | Monolith | Microservices |
|-----------|----------|---------------|
| Operational simplicity | High — single deploy unit | Low — N services, N deployments |
| Independent scaling | Not possible | Scale only the bottleneck service |
| Fault isolation | Single point of failure | One service fails independently |
| Team autonomy | Hard at scale | Each team owns their service |
| Recommendation | Start here | Migrate when team or scale forces it |

---

## 7. Step 6 — Wrap-up

Allocate 2–3 minutes. This is frequently skipped but signals maturity.

Template:

> "We designed a [system] using [key choices]. The core architecture is [1-sentence summary]. The biggest production bottleneck would be [X], which I'd address with [Y]. Given more time, I'd add [A], [B], and [C]. Happy to go deeper on any of these."

Good "with more time" items:
- CDN for media delivery
- Circuit breakers and bulkheads between services
- Distributed tracing (Jaeger, Tempo)
- Full-text search tier (Elasticsearch)
- Multi-region active-active for global latency
- Event sourcing for audit trail

---

## 8. What interviewers actually evaluate

| Signal | Green flag | Red flag |
|--------|-----------|----------|
| Scoping | Asks precise questions; defines out-of-scope | Dives in without clarifying |
| Communication | Thinks out loud; draws while explaining | Silent sketching |
| Depth | Justifies each component choice | Names components without rationale |
| Trade-offs | Voluntarily surfaces trade-offs | Only presents "the best" solution |
| Adaptability | Updates design from new constraints | Defends initial design rigidly |
| Estimation | Does back-of-envelope; uses conclusions | Skips estimation or ignores conclusions |
| Failure modes | Asks "what happens when X goes down?" | Designs only the happy path |
| Self-awareness | Names known gaps | Claims design handles all cases |

---

## 9. Interview template (copy-paste)

```
STEP 1 — CLARIFY (3–5 min)
  Functional:   list 3–5 core features in scope
  Scale:        DAU, read/write ratio
  Non-func:     latency SLA, availability, consistency model
  Out of scope: explicitly state what you won't design

STEP 2 — ESTIMATE (3–5 min)
  Write QPS:    DAU × write_fraction ÷ 86400
  Read QPS:     write_QPS × read:write ratio
  Storage/day:  writes/day × avg_payload
  Fan-out:      write_QPS × avg_followers (if applicable)

STEP 3 — HIGH-LEVEL (10–15 min)
  Components:   client → LB → app servers → [DB, cache, queue, object store]
  APIs:         2–3 key endpoints with request/response shapes
  Data model:   key entities, relations, access patterns
  DB choice:    SQL / NoSQL + one-line justification

STEP 4 — DEEP DIVE (10–15 min)  ← follow interviewer's lead
  The bottleneck: what breaks at scale?
  Caching:      what to cache, strategy, TTL, invalidation
  Async:        which writes should go through a queue?
  Sharding:     shard key, hot spot risks

STEP 5 — TRADE-OFFS (5–8 min)
  Decision 1:   SQL vs NoSQL → chose X because Y
  Decision 2:   push vs pull → chose X because Y
  Decision 3:   sync vs async → chose X because Y

STEP 6 — WRAP-UP (2–3 min)
  Summary:      one sentence of what was built
  Known gaps:   what would break first in production
  Extensions:   what you'd add with more time
```

---

## 10. Quick reference cheat sheet

```
Clarify:    functional + non-functional. Explicitly state out-of-scope.
Estimate:   write QPS = DAU × fraction ÷ 86400
            read QPS  = write QPS × read:write ratio
            storage   = writes/day × avg_payload
Design:     Client → LB → App Servers → [DB, Cache, Queue, Object Store, CDN]
Trade-offs: SQL vs NoSQL | CP vs AP | Push vs Pull | Sync vs Async | Monolith vs Micro
Wrap-up:    gaps + next steps

Scale rules of thumb:
  1M  DAU, 10% write → ~1 write QPS       simple setup
  10M DAU, 10% write → ~12 write QPS      read replicas useful
  100M DAU, 10% write → ~120 write QPS    caching mandatory, sharding likely
  1B  DAU             → multi-region, global data distribution required

Memory: ~100ns | SSD: ~100µs | Disk seek: ~10ms | Same-DC RTT: ~0.5ms
```

---

## 11. References

- [System Design Primer — donnemartin](https://github.com/donnemartin/system-design-primer)
- [Designing Data-Intensive Applications — Martin Kleppmann](https://dataintensive.net/)
- [Grokking the System Design Interview — educative.io](https://www.educative.io/courses/grokking-the-system-design-interview)
- [CAP Theorem revisited — Eric Brewer / InfoQ](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/)
- [The Log: What every software engineer should know — Jay Kreps](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)
- [Twitter's architecture — HighScalability](http://highscalability.com/blog/2013/7/8/the-architecture-twitter-uses-to-deal-with-150m-active-users.html)