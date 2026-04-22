# Back-of-the-Envelope Estimation (Capacity Planning)

## Table of Contents

- [What It Is and Why It Exists](#what-it-is-and-why-it-exists)
- [The Mental Model: Why Numbers Matter](#the-mental-model-why-numbers-matter)
- [Core Numbers to Internalize](#core-numbers-to-internalize)
  - [Powers of Two](#powers-of-two)
  - [Latency Numbers Every Engineer Should Know](#latency-numbers-every-engineer-should-know)
  - [Everyday Size References](#everyday-size-references)
- [The Four Dimensions of Estimation](#the-four-dimensions-of-estimation)
  - [Traffic (Requests per Second)](#traffic-requests-per-second)
  - [Storage](#storage)
  - [Bandwidth](#bandwidth)
  - [Memory (Cache)](#memory-cache)
- [A Repeatable Interview Framework](#a-repeatable-interview-framework)
- [Worked Example: Twitter-Scale Feed System](#worked-example-twitter-scale-feed-system)
- [Common Approximation Tricks](#common-approximation-tricks)
- [What Interviewers Actually Look For](#what-interviewers-actually-look-for)
- [Pitfalls to Avoid](#pitfalls-to-avoid)
- [References](#references)

---

## What It Is and Why It Exists

**Back-of-the-envelope estimation** is rough quantitative reasoning done with simple arithmetic to understand the *order of magnitude* of a system's resource needs — before writing a single line of code or drawing a single architecture box.

The name comes from the literal idea of scribbling quick calculations on whatever is at hand — a napkin, an envelope — rather than pulling out a spreadsheet. The goal is *directionally correct*, not precisely correct.

**Why it exists:** Real systems fail not because engineers chose the wrong framework, but because they underestimated load by 10× or 100×. Estimation forces you to answer: Can a single machine handle this? Do we need sharding? Will our storage bill be $10/month or $10,000/month? These questions must be answered *before* design, not after.

**Why interviewers care:** They use this exercise to check whether you have a realistic mental model of distributed systems — that you know the difference between 1,000 RPS (a single server) and 1,000,000 RPS (a fleet), and that you can use that to drive architectural decisions. It is not a math test. It is a reasoning test.

---

## The Mental Model: Why Numbers Matter

Every architectural decision has a threshold. Estimation tells you which side of the threshold you are on.

```
          1 machine                cluster               distributed system
               |                      |                         |
    0 ─────────┼──────────────────────┼─────────────────────────┼──────────▶  RPS
               |                      |                         |
             ~1K                    ~10K                      ~100K+
```

- Under ~1K RPS → a single well-tuned server can handle most workloads
- ~10K–100K RPS → you need horizontal scaling, load balancers, read replicas
- 1M+ RPS → sharding, CDNs, edge caching, specialized storage become mandatory

Estimation is how you figure out which regime you are in.

---

## Core Numbers to Internalize

### Powers of Two

Data sizes grow in powers of 2. Every engineer must know these cold.

```
Name        Abbrev    Exact value           Approx
──────────────────────────────────────────────────
Kilobyte    KB        1,024 bytes           ~10^3
Megabyte    MB        1,048,576 bytes       ~10^6
Gigabyte    GB        1,073,741,824 bytes   ~10^9
Terabyte    TB        ~1.1 × 10^12          ~10^12
Petabyte    PB        ~1.1 × 10^15          ~10^15
```

**Memory trick:** When multiplying data sizes, convert everything to the same unit first, then compute.

### Latency Numbers Every Engineer Should Know

These are the latency numbers famously compiled by Jeff Dean (circa 2012). The absolute values have shrunk, but the *relative ratios* remain true and are what matter in interviews.

```
Operation                              Latency        Ratio vs L1
─────────────────────────────────────────────────────────────────
L1 cache reference                     ~0.5 ns        1×
Branch misprediction                   ~5 ns          10×
L2 cache reference                     ~7 ns          14×
Mutex lock/unlock                      ~25 ns         50×
Main memory reference (RAM)            ~100 ns        200×
Compress 1KB with Snappy               ~3,000 ns      6,000×
Send 1KB over 1 Gbps network           ~10,000 ns     20,000×
Read 4KB randomly from SSD             ~150,000 ns    300,000×
Read 1MB sequentially from memory      ~250,000 ns    500,000×
Round-trip within same datacenter      ~500,000 ns    1,000,000×
Read 1MB sequentially from SSD         ~1,000,000 ns  2,000,000×
Disk seek                              ~10,000,000 ns 20,000,000×
Read 1MB sequentially from disk        ~20,000,000 ns 40,000,000×
Send packet CA → Netherlands → CA      ~150,000,000 ns ~300,000,000×
```

**The takeaways that drive architecture:**

- Memory is ~200× faster than SSD for random reads → keep hot data in RAM (Redis)
- SSD is ~1000× faster than disk seek → prefer SSDs for random access patterns
- Network within a datacenter (~0.5ms) is acceptable; cross-continent (~150ms) is painful
- Compression is cheap; network I/O is expensive → compress before sending

### Everyday Size References

These help anchor your estimates to real content:

```
Item                          Size
───────────────────────────────────────
ASCII character               1 byte
UTF-8 character (English)     1–2 bytes
UTF-8 character (CJK)         3 bytes
Integer (int64)               8 bytes
UUID (string form)            36 bytes
Short tweet text              ~280 bytes
Typical JSON API response     ~1 KB
Web page (HTML only)          ~30–100 KB
High-res photo (compressed)   ~3–5 MB
4K video (1 minute)           ~350 MB
```

---

## The Four Dimensions of Estimation

Every system design estimation breaks down into four questions.

### Traffic (Requests per Second)

This is the starting point. Everything else is derived from it.

**Two useful constants:**
- Seconds in a day: **86,400** ≈ **10^5** (round to 100,000)
- Seconds in a month: ~2.6 million ≈ **3 × 10^6**

**Read/Write ratio:** Most systems are read-heavy. A typical ratio is 100:1 (social media) or 10:1 (e-commerce). Interviewers expect you to state this assumption.

```
Given:
  DAU (daily active users) = 10 million
  Each user performs 10 reads + 1 write per day

Write RPS = (10M × 1) ÷ 86,400 ≈ 100K ÷ 86K ≈ ~120 writes/sec
Read RPS  = (10M × 10) ÷ 86,400 ≈ 1M ÷ 86K  ≈ ~1,200 reads/sec

Round to: ~100 writes/sec, ~1,000 reads/sec
```

**Peak traffic:** Production traffic is not flat. Assume peak is 2–3× average. State this.

### Storage

Storage = (data written per second) × (retention period)

```
Given:
  Write rate: 100 writes/sec
  Payload per write: 1 KB
  Retention: 5 years

Data/sec  = 100 × 1 KB = 100 KB/sec
Data/day  = 100 KB × 86,400 ≈ 8.64 GB/day ≈ ~10 GB/day
Data/year = 10 GB × 365 ≈ 3.65 TB/year
Data/5yr  = 3.65 × 5 ≈ ~18 TB
```

Add overhead for: replication (typically 3×), indexes (~20–50% of data size), and metadata.

**Storage type decision:** Use latency numbers to justify.
- Hot data (read frequently) → SSD or in-memory
- Warm data (read occasionally) → HDD
- Cold data (archival) → object storage (S3 Glacier)

### Bandwidth

Bandwidth = (data size per request) × RPS

```
Given:
  Read RPS: 1,000/sec
  Response size: 10 KB (e.g., a feed page)

Outbound bandwidth = 1,000 × 10 KB = 10 MB/sec = 80 Mbps
```

A modern server NIC is 1–10 Gbps. If your bandwidth exceeds a few hundred Mbps for a single machine, you need multiple servers or a CDN to offload static content.

### Memory (Cache)

Cache sizing follows the **80/20 rule (Pareto principle):** 20% of your data is responsible for 80% of your reads. Caching that 20% handles most traffic.

```
Given:
  Daily reads: 1,000 RPS × 86,400 sec ≈ 86M read requests/day
  Unique objects read: assume 10M (users/posts)
  Object size: 1 KB

Total data space: 10M × 1 KB = 10 GB
Cache 20%:        10 GB × 0.2 = 2 GB

→ A single Redis instance (tens of GB capacity) is sufficient
```

---

## A Repeatable Interview Framework

Use this structure every time. State each step out loud so the interviewer can follow your reasoning.

```
Step 1: Clarify Scale Assumptions
  └─ DAU ÷ MAU, geographic spread, growth rate

Step 2: Estimate Traffic
  └─ writes/sec, reads/sec, read:write ratio, peak multiplier

Step 3: Estimate Storage
  └─ payload size × write rate × retention + replication factor

Step 4: Estimate Bandwidth
  └─ response size × read RPS (outbound), payload × write RPS (inbound)

Step 5: Estimate Memory (if caching is relevant)
  └─ 20% of working set

Step 6: Draw Conclusions
  └─ "Given these numbers, we need X servers, Y TB storage, a cache layer"
```

---

## Worked Example: Twitter-Scale Feed System

**Step 1 — Scale Assumptions:**

```
MAU: 300M, DAU: 50M
Each user: 1 tweet/day (write), 10 feed views/day (read)
Tweet content: ≤280 chars text + optional media
Retention: 5 years
```

**Step 2 — Traffic:**

```
Write (tweets):
  50M DAU × 1 tweet/day ÷ 86,400 = ~580 writes/sec ≈ ~600 writes/sec

Read (feed views):
  50M × 10 reads/day ÷ 86,400 = ~5,800 reads/sec ≈ ~6,000 reads/sec

Read:Write ratio ≈ 10:1
Peak multiplier: 3× → peak writes ~1,800/sec, peak reads ~18,000/sec
```

**Step 3 — Storage:**

```
DB row per tweet:
  Text + metadata (user_id, timestamp, likes): ~300 bytes
  Media pointer (S3 URL, stored for 30% of tweets): ~100 bytes extra

Weighted average row size:
  (70% × 300) + (30% × 400) = 210 + 120 = 330 bytes → round to ~300 bytes

Data/day  = 600 writes/sec × 300 bytes × 86,400 = ~15.5 GB/day ≈ 16 GB/day
Data/year = 16 GB × 365 ≈ 5.8 TB/year
Data/5yr  = ~29 TB (raw)

With 3× replication: ~87 TB of tweet text/metadata in DB

Media (images/video): stored separately in S3, not in the DB
  30% of 600 writes/sec = 180 media writes/sec
  180 × 1 MB × 86,400 = ~15.5 TB/day of raw media
```

**Step 4 — Bandwidth:**

```
Inbound (writes):
  600 × 300 bytes ≈ 180 KB/sec — negligible

Outbound (reads):
  Each feed page returns ~20 tweets × 300 bytes = 6 KB per read
  6,000 reads/sec × 6 KB = 36 MB/sec = ~288 Mbps
  Peak (3×): ~870 Mbps → approaches a single server NIC limit

→ Conclusion: CDN for media, multiple read replicas for feed API
```

**Step 5 — Memory (Cache):**

```
Hot tweets: top ~1% of tweets get ~90% of reads (power law distribution)
Working set: assume 20M hot tweets × 300 bytes = 6 GB
→ A Redis cluster with even 64 GB of RAM comfortably handles hot tweet cache
```

**Step 6 — Conclusions:**

- Write path can likely be handled by 2–3 write servers with a message queue (Kafka)
- Read path needs multiple servers + aggressive caching (Redis)
- Storage requires a distributed DB (Cassandra fits write-heavy + time-series tweet data)
- Media needs object storage (S3) + CDN
- ~87 TB for text/metadata in DB; ~15 TB/day and petabyte-scale over 5 years for media in S3

---

## Common Approximation Tricks

- **1 million requests/day ≈ 12 requests/sec** (1M ÷ 86,400 ≈ 11.6)
- **1 billion requests/day ≈ 12,000 requests/sec**
- Round to nearest power of 10 for all intermediate steps
- Express storage in "per day" first, then multiply — easier to catch errors
- 1 byte = 8 bits (remember when converting MB/sec to Mbps)
- A server can handle **~10K–50K HTTP requests/sec** for simple reads (varies hugely with payload)
- A PostgreSQL instance can handle **~5K–10K writes/sec** comfortably
- Redis can handle **~100K–1M ops/sec** depending on operation type

---

## What Interviewers Actually Look For

- **Structured thinking over accuracy.** They want to see you break the problem into known dimensions (traffic, storage, bandwidth, memory) and reason step by step. An answer that is 2× off but methodical beats a "right" answer pulled from thin air.

- **Explicit assumptions.** Always state what you are assuming (DAU, payload size, read:write ratio). Interviewers often change one assumption mid-exercise to see how you adapt.

- **Connecting numbers to decisions.** The estimation is not the goal. The goal is saying: "Because we have 18,000 reads/sec at peak, we cannot serve this from a single server — we need a read replica strategy and a cache." This is what separates senior from junior thinking.

- **Order-of-magnitude awareness.** Know the difference between 10 GB and 10 TB in terms of architectural implications. One fits on a laptop SSD; the other requires a distributed storage system.

- **Knowing when to stop.** You do not need to estimate every sub-system. Estimate the bottleneck. State what you are skipping and why.

---

## Pitfalls to Avoid

- **Forgetting replication.** Raw storage is always multiplied by your replication factor (typically 3). Skipping this understates storage needs by 3×.

- **Flat traffic assumption.** Real traffic has peaks (morning rush, events, viral moments). Always apply a peak multiplier (2–3×) to derive provisioning targets.

- **Confusing MB and Mbps.** Storage is in bytes; bandwidth is in bits per second. 1 MB/sec = 8 Mbps. Easy to slip.

- **Ignoring metadata and indexes.** A database table with 1 KB rows might use 1.3–1.5 KB on disk after indexes. For large datasets this matters.

- **Solving the wrong problem.** If the interviewer says "assume images are stored in S3", do not spend time estimating image storage. Focus on what they actually want you to size.

---

## References

- *System Design Interview – An Insider's Guide* — Alex Xu (Chapter 2)
- [Jeff Dean's Latency Numbers](https://github.com/sirupsen/napkin-math) — napkin-math repo (continuously updated)
- [Numbers Every Programmer Should Know](https://colin-scott.github.io/personal_website/research/interactive_latency.html) — interactive latency visualizer
- [Google SRE Book — Chapter 19: Load Balancing at the Frontend](https://sre.google/sre-book/load-balancing-frontend/)