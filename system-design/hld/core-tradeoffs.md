# Latency, Throughput, Availability, and Consistency

## Table of Contents

- [Overview](#overview)
- [Latency](#latency)
- [Throughput](#throughput)
- [Latency vs Throughput](#latency-vs-throughput)
- [Availability](#availability)
- [Consistency](#consistency)
- [Interviewer Angles](#interviewer-angles)
- [References](#references)

---

## Overview

These four properties are the vocabulary of system design tradeoffs. Every architecture decision — caching strategy, database choice, replication model, service topology — ultimately trades one of these against another. Understanding them precisely, not just definitionally, separates strong candidates from weak ones.

---

## Latency

### What It Is

The time elapsed from when a request is sent to when its response is fully received. End-to-end delay for a single operation.

### Why It Exists

Latency is a physical constraint. The speed of light limits cross-continental round trips to ~150–200ms. Every additional component in the path — network hops, load balancers, application servers, databases — adds its own queuing time, processing time, and I/O wait.

### Why It Is Called Latency

From Latin *latere* ("to lie hidden"). The delay is hidden — invisible between cause and effect.

### Key Latency Numbers

Memorise these for interviews. They anchor any "back-of-envelope" discussion.

| Operation                    | Approximate Latency |
|------------------------------|---------------------|
| L1 cache reference           | ~1 ns               |
| L2 cache reference           | ~4 ns               |
| RAM read                     | ~100 ns             |
| SSD random read              | ~0.1 ms             |
| Network RTT (same datacenter)| ~0.5 ms             |
| HDD random seek              | ~10 ms              |
| Network RTT (cross-country)  | ~100 ms             |
| Network RTT (cross-continent)| ~200 ms             |

### Percentiles, Not Averages

Average latency is misleading. If 99 requests complete in 1 ms and 1 request takes 5,000 ms, the average is ~51 ms — looks acceptable, but 1% of users suffer a 5-second wait.

Use **p50, p95, p99, p999** (percentile thresholds):
- **p99 = 100 ms** means 1 in every 100 requests is slower than 100 ms
- In a system at 10,000 RPS, that is 100 users/second experiencing the slow path

**Tail latency compounds in microservice fans-out.** If one request calls 10 downstream services each at p99 = 100 ms:

```
P(at least one slow call) = 1 - (0.99)^10 ≈ 9.6%
```

With 50 services, nearly half of all requests will hit a slow tail call. This is a core argument for **hedging requests** (send duplicate requests, take the first response) and **timeout budgets**.

### Types of Latency

- **Network latency** — RTT over the wire
- **Disk I/O latency** — seek time + transfer time; SSDs 100x faster than HDDs
- **Processing latency** — CPU computation, GC pauses
- **Queuing latency** — waiting behind other requests in a queue or thread pool

---

## Throughput

### What It Is

The number of operations completed per unit of time. Measured as RPS (requests per second), QPS (queries per second), TPS (transactions per second), or MB/s depending on context.

### Why It Exists

Throughput measures system *capacity*. You need to know the upper bound of what your system can handle before degrading under load.

### Little's Law

The most important formula for reasoning about throughput:

```
L = λ × W
```

- `L` — average number of in-flight requests in the system at any instant
- `λ` — arrival rate / throughput (requests/sec)
- `W` — average time each request spends in the system (latency)

Rearranged:

```
λ = L / W    ⟹    Throughput = Concurrency / Latency
```

**What this means:** If you double your latency but also double your concurrency (more goroutines, more replicas), throughput stays constant. Horizontal scaling works because you increase `L` proportionally to increases in `W`.

Example: system with 100 concurrent goroutines and 50 ms average latency:
```
λ = 100 / 0.05 = 2,000 RPS
```

If latency rises to 100 ms (say, due to a slow downstream):
```
λ = 100 / 0.10 = 1,000 RPS   ← throughput halved
```

To restore throughput, double concurrency to 200 goroutines.

---

## Latency vs Throughput

They frequently trade against each other. Understanding when and why is an interviewer signal.

**Batching** — the canonical tradeoff:
- Write operations individually: latency low, throughput limited by per-request overhead
- Batch writes: throughput improves (fewer round trips, better compression, amortised fixed costs) but per-item latency increases (each item waits for the batch to fill)
- Used in: Kafka producer batching, database write-ahead logs, HTTP/2 multiplexing

**Replication:**
- More read replicas → higher read throughput, but replication lag means stale reads → worse read consistency (latency of consistency, not request latency)

**Caching:**
- Rare case that improves both simultaneously — cache hits avoid the slow path entirely

**Interviewer prompt:** When asked "how would you improve performance?" — always clarify: do you mean reduce latency for individual requests, or increase maximum throughput (capacity)? The solutions are different.

```
                  ┌────────────────────────────────────┐
                  │         Batching Tradeoff           │
                  │                                     │
  Throughput ↑    │  ●●●●●●●●●●●   batch writes        │
                  │                                     │
                  │  ●            individual writes     │
  Throughput ↓    │                                     │
                  └────────────────────────────────────┘
                     Latency ↓           Latency ↑
```

---

## Availability

### What It Is

The percentage of time a system is operational and correctly serving requests. Availability = uptime / (uptime + downtime).

### Why It Exists

Failures are inevitable in distributed systems — hardware dies, software crashes, networks partition. Availability quantifies how well the system tolerates and recovers from those failures.

### The Nines

| Availability       | Downtime per Year | Downtime per Month |
|--------------------|-------------------|--------------------|
| 99% (2 nines)      | ~3.65 days        | ~7.3 hours         |
| 99.9% (3 nines)    | ~8.76 hours       | ~43.8 minutes      |
| 99.99% (4 nines)   | ~52.6 minutes     | ~4.38 minutes      |
| 99.999% (5 nines)  | ~5.26 minutes     | ~26 seconds        |

### Calculating System Availability

**Serial dependencies (A calls B calls C):**

```
Availability_total = A × B × C
```

Three 99.9% services in series:
```
0.999 × 0.999 × 0.999 = 0.997   →   99.7%   (lost a nine)
```

Every dependency you add erodes overall availability. This is why tight coupling is dangerous.

**Parallel / redundant components:**

```
Availability = 1 - (1 - A)^n
```

Two 99% services in parallel:
```
1 - (1 - 0.99)^2 = 1 - 0.0001 = 99.99%   (gained two nines)
```

Redundancy is the primary mechanism for achieving high availability.

### MTBF and MTTR

```
Availability = MTBF / (MTBF + MTTR)
```

- **MTBF** — Mean Time Between Failures (how often it breaks)
- **MTTR** — Mean Time To Recover (how fast you fix it)

You can improve availability in two directions:
- Increase MTBF → more reliable hardware, fewer bugs, chaos engineering
- Reduce MTTR → fast deploys, health checks + auto-restart, circuit breakers, good runbooks, observability

Fast recovery is often easier and cheaper than preventing failure.

### SLA, SLO, SLI (the vocabulary)

- **SLI** (Service Level Indicator) — the raw metric being measured (e.g., request success rate)
- **SLO** (Service Level Objective) — internal target (e.g., 99.9% success rate over 30 days)
- **SLA** (Service Level Agreement) — contractual commitment to the customer; breach triggers penalties

Error budget = 100% - SLO. If SLO is 99.9%, you have 8.76 hours/year of allowed downtime. Teams spend error budget on risky deployments and features; if budget is exhausted, deployments freeze.

---

## Consistency

### What It Is

All nodes in a distributed system see the same data at the same time, and reads always reflect the most recent committed write.

### Why It Is Hard

When you replicate data (for fault tolerance or throughput), writes go to one node and must propagate to replicas. Network partitions can delay or prevent that propagation. Replicas diverge — consistency is the problem of controlling that divergence.

### The Consistency Spectrum

Ranked from strongest (most guarantees, highest latency) to weakest (fewest guarantees, lowest latency):

```
  Strongest                                              Weakest
  ────────────────────────────────────────────────────────────────►
  Linearizable  Sequential  Causal  Read-your-writes  Eventual
```

- **Linearizable (Strong)** — every read reflects the latest write; operations appear globally instantaneous and ordered. Hardest to implement; highest latency. All replicas must coordinate on every write. Example: Google Spanner, etcd.

- **Sequential** — all nodes observe operations in the same order, but that order may lag behind real time. No global "now."

- **Causal** — causally related operations appear in order across all nodes ("if you saw my write, you see everything I saw before writing"). Unrelated operations may diverge. Tracks happens-before relationships.

- **Read-your-writes** — a client always sees its own writes immediately, even if other clients don't yet. (e.g., you post a tweet and see it instantly, others may not yet.)

- **Monotonic Read** — once a client reads a value, it never observes an older value. Prevents "going back in time."

- **Eventual** — replicas will converge to the same state eventually, with no timing guarantee. Lowest latency, highest availability. Examples: DNS propagation, Cassandra, DynamoDB (default mode).

---

## Interviewer Angles

- **"How would you improve the latency of this system?"** — Immediately ask: "Do you mean p99 or p50? And are we optimising latency at the expense of throughput, or do we need both?" Then: caching, connection pooling, reducing network hops, async processing, CDN for static assets.

- **"What does 99.99% availability mean?"** — "About 52 minutes of downtime per year. If this service calls two other services each at 99.99%, the composite is 99.97% — roughly 2.5 hours. Redundancy is how we recover the lost nines."

- **"How do you choose between eventual and strong consistency?"** — Ask about the use case: financial transactions need strong; social media feeds are fine with eventual. Then discuss the latency and availability cost of strong consistency.

- **"What's the difference between latency and throughput?"** — Latency is per-request delay; throughput is system capacity. Then give Little's Law. Then describe when they trade against each other (batching) vs when you can improve both (caching).

---

## References

- [Latency Numbers Every Programmer Should Know — Colin Scott](https://colin-scott.github.io/personal_website/research/interactive_latency.html)
- [Designing Data-Intensive Applications — Martin Kleppmann, Ch. 5 (Replication) and Ch. 9 (Consistency and Consensus)](https://dataintensive.net/)
- [Little's Law — Wikipedia](https://en.wikipedia.org/wiki/Little%27s_law)
- [Google SRE Book — Availability Table](https://sre.google/sre-book/availability-table/)