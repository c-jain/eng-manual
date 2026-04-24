# Horizontal vs Vertical Scaling

## Table of Contents

- [What Is Scaling](#what-is-scaling)
- [Vertical Scaling (Scale Up)](#vertical-scaling-scale-up)
- [Horizontal Scaling (Scale Out)](#horizontal-scaling-scale-out)
- [Side-by-Side Comparison](#side-by-side-comparison)
- [The Stateless Requirement](#the-stateless-requirement)
- [Database Scaling Is a Different Beast](#database-scaling-is-a-different-beast)
- [Load Balancers: The Enabler of Horizontal Scale](#load-balancers-the-enabler-of-horizontal-scale)
- [Cost Curve Intuition](#cost-curve-intuition)
- [Real-World Patterns](#real-world-patterns)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

---

## What Is Scaling

Scaling is the act of increasing a system's capacity to handle more load — more requests per second, more data, more users. There are exactly two axes you can scale along: the machine itself, or the number of machines.

- **Vertical scaling** — make the existing machine bigger (more CPU cores, more RAM, faster disk)
- **Horizontal scaling** — add more machines and distribute the load across them

The terms come from how you'd draw it on a diagram: vertical scaling grows the box upward (taller), horizontal scaling grows the fleet sideways (more boxes at the same level).

Both strategies exist because no single approach is universally correct. Vertical is simple but has a ceiling. Horizontal is powerful but introduces distributed systems complexity. Real production systems almost always use a combination.

---

## Vertical Scaling (Scale Up)

### What It Means

You take one server and replace it with a more powerful one. Instead of a 4-core/16 GB machine, you run a 32-core/256 GB machine.

```
Before                    After

┌─────────────┐           ┌─────────────────────┐
│  4 cores    │  ──────►  │     32 cores        │
│  16 GB RAM  │           │     256 GB RAM      │
│  App + DB   │           │     App + DB        │
└─────────────┘           └─────────────────────┘
```

### Why It Exists

It is the simplest possible response to a capacity problem. No code changes, no network calls, no new failure modes. The app has no idea anything changed — it just wakes up with more resources.

### Advantages

- Zero architectural change — deploy the exact same binary
- No distributed coordination (no load balancers, no inter-node state sync)
- Lower operational complexity — one machine to monitor, patch, and debug
- Latency stays local — no network hops between components
- Works for stateful workloads without any special treatment

### Disadvantages

- **Hard ceiling** — there is a maximum machine size. Cloud providers cap out around 448 vCPUs / 24 TB RAM (AWS `u-24tb1.metal`). Beyond that, vertical scaling is simply not possible
- **Single point of failure** — one machine means one crash takes everything down. There is no redundancy
- **Downtime on upgrade** — resizing typically requires stopping the instance, resizing, and restarting
- **Diminishing returns on cost** — doubling specs often more than doubles the price at the high end
- **Does not improve availability** — a bigger machine is not a more available machine

---

## Horizontal Scaling (Scale Out)

### What It Means

You add more identical machines behind a load balancer. Each machine runs the same application code and shares nothing (or shares state via an external store).

```
                     ┌──────────────┐
            ┌───────►│   Server 1   │
            │        └──────────────┘
Clients ──► LB ─────►│   Server 2   │
            │        └──────────────┘
            └───────►│   Server 3   │
                     └──────────────┘
```

### Why It Exists

It removes the hard ceiling of a single machine and adds fault tolerance. If one node dies, the load balancer stops routing to it and the remaining nodes absorb its traffic. You can also scale down during off-peak hours and pay only for what you use.

### Advantages

- **No theoretical ceiling** — add nodes indefinitely (within budget and coordination limits)
- **Fault tolerance** — one node failure is survivable; the system degrades gracefully instead of going down entirely
- **Geographic distribution** — nodes can be placed in different data centres or regions
- **Rolling deployments** — drain one node, update it, bring it back; zero downtime
- **Pay-as-you-grow** — add capacity incrementally; remove it when demand drops

### Disadvantages

- **Stateless requirement** — the load balancer can route any request to any node, so in-process state (local caches, sessions stored in memory) is unreliable unless pinned (sticky sessions) or externalised
- **Distributed systems complexity** — you now have a network between components. Network partitions, clock skew, and partial failures are real problems
- **Data consistency challenges** — if multiple nodes write to the same data store, you need to reason about consistency
- **Operational overhead** — you now manage a fleet: health checks, auto-scaling groups, load balancer config, service discovery
- **Debugging is harder** — a request may touch different nodes across retries; **distributed tracing** is necessary

---

## Side-by-Side Comparison

| Dimension | Vertical | Horizontal |
|---|---|---|
| Implementation effort | Low (resize and restart) | High (stateless design, LB, orchestration) |
| Maximum capacity | Bounded by largest available machine | Effectively unbounded |
| Fault tolerance | None (SPOF) | High (nodes fail independently) |
| Cost model | Steep curve at high end | Linear / pay-per-node |
| Good fit | Stateful services, DBs (short-term) | Stateless services, web/API tiers |
| Code changes required | No | Often yes (externalise session/cache) |
| Downtime to scale | Usually yes | No (add nodes while live) |

---

## The Stateless Requirement

This is the most important design concept horizontal scaling introduces. A service is **stateless** when a node holds no data between requests that another node would not also have.

The problem:

```
Scenario: Session stored in-process (broken with horizontal scale)

Request 1 (login) ──► Server 1 → stores session in Server 1's RAM
Request 2 (profile) ─► Server 2 → no session found → user is logged out
```

The fix — externalise all shared state:

```
Request 1 (login)   ──► Server 1 ──► writes session to Redis
Request 2 (profile) ──► Server 2 ──► reads session from Redis → OK
```

What needs externalising:

- **Sessions** → Redis, a distributed cache, or JWT (token carries state itself)
- **In-process caches** → Redis or Memcached
- **Uploaded files** → object storage (S3, GCS)
- **Websocket connections** → sticky sessions (pin user to one node) or a pub/sub broker (e.g., Redis Pub/Sub)
- **Rate limiting counters** → Redis atomic increments

The rule of thumb: if you would lose data by killing a node, that data is state and must move out of the node.

---

## Database Scaling Is a Different Beast

Application servers are naturally stateless and scale horizontally with little friction. Databases carry the hard state and are far more constrained.

### Vertical First

For most systems, the database is scaled vertically first. A more powerful machine handles more concurrent queries, more RAM means a larger buffer pool (less disk I/O), and faster CPUs help with query planning.

### Read Replicas (Horizontal for Reads)

Once writes are manageable but reads are the bottleneck, add **read replicas**. The primary accepts all writes; replicas receive the replication stream and serve reads.

```
Writes ──► Primary ──► replication stream ──► Replica 1
                   └──────────────────────► Replica 2

Reads  ──► Replica 1 (or 2, load-balanced)
```

Trade-off: replicas are eventually consistent. There is a replication lag. A write on the primary may not be visible on a replica for milliseconds to seconds.

### Sharding (Horizontal for Writes)

Once writes saturate the primary, you shard: partition data across multiple independent database nodes, each owning a range of keys.

```
User IDs 0–9M   ──► Shard A
User IDs 9M–18M ──► Shard B
User IDs 18M+   ──► Shard C
```

Sharding is operationally expensive. Cross-shard joins are slow or impossible. Re-sharding when a shard gets too large is painful. This is a last resort, not a first move.

### Practical Database Scaling Order

```
1. Vertical scale the primary
        │
        ▼
2. Add a read replica (serves reads, reduces primary load)
        │
        ▼
3. Add a caching layer (Redis in front of DB for hot reads)
        │
        ▼
4. Add more read replicas
        │
        ▼
5. Shard (only when write throughput is the hard limit)
```

---

## Load Balancers: The Enabler of Horizontal Scale

A load balancer is the entry point that makes horizontal scaling possible. Without it, clients cannot discover or balance across multiple nodes.

### What It Does

- Receives incoming traffic and forwards it to one of the backend servers
- Performs health checks; removes unhealthy nodes from rotation
- Can terminate TLS, add request headers (e.g., `X-Forwarded-For`), and handle SSL offloading

### Common Algorithms

- **Round Robin** — cycle through nodes in order. Simple, assumes uniform request cost
- **Least Connections** — route to the node with fewest active connections. Better when requests have variable duration
- **IP Hash / Sticky Sessions** — hash the client IP to always route to the same node. Necessary for stateful workloads that cannot externalise state
- **Weighted** — assign traffic ratios per node. Useful for canary deploys or heterogeneous hardware

### Layer 4 vs Layer 7

- **L4 (Transport)** — operates on TCP/UDP. Fast; cannot inspect HTTP content. Used for raw throughput
- **L7 (Application)** — inspects HTTP headers, URLs, cookies. Can route `/api` to one cluster and `/static` to another. Most web systems use L7

---

## Cost Curve Intuition

A useful mental model for interviews:

```
Cost
 │                               ● Vertical
 │                         ●
 │                   ●                ● ● ● ● Horizontal
 │             ●               ● ●
 │       ● ●         ● ●
 │ ● ●
 └──────────────────────────────────── Capacity

Vertical: cost grows superlinearly at high specs
Horizontal: cost grows roughly linearly with nodes
```

At low capacity, vertical is cheaper (no extra infra overhead). At high capacity, horizontal is more cost-effective. The crossover point is system-dependent but is usually reached well before hitting the vertical ceiling.

---

## Real-World Patterns

**Typical three-tier web application**

```
                        ┌─────────────────────────────┐
                        │     Load Balancer (L7)      │
                        └──────┬──────────────┬───────┘
                               │              │
                    ┌──────────▼──┐      ┌────▼────────┐
                    │  App Node 1 │      │  App Node 2 │   ◄── Horizontal
                    └──────────┬──┘      └────┬────────┘
                               │              │
                    ┌──────────▼──────────────▼────────┐
                    │           Redis Cache            │   ◄── Shared state
                    └──────────────────────────────────┘
                               │
                    ┌──────────▼──────────────────────┐
                    │     Primary DB (Vertical)       │
                    └──────────┬──────────────────────┘
                               │ replication
                    ┌──────────▼──────────────────────┐
                    │        Read Replica             │
                    └─────────────────────────────────┘
```

The app tier is stateless and scales horizontally. The cache and database are shared dependencies; the DB scales vertically first, then gains replicas, then shards.

---

## Interview Cheat Sheet

**When asked "how would you scale X?"**

- Start with the bottleneck. Is it CPU, memory, I/O, or bandwidth? Diagnose before prescribing
- For app servers: horizontal (stateless services are trivially scalable)
- For databases: vertical → read replicas → caching → sharding (in that order)
- Always mention the load balancer as a prerequisite for horizontal
- Mention what state needs to be externalised and where (Redis for sessions/cache)

**Common follow-up questions**

- "What if one node goes down?" — horizontal handles this; vertical does not (SPOF)
- "How do you deploy without downtime?" — rolling deploys only work horizontally
- "Your DB is the bottleneck now — what do you do?" — read replicas, caching layer, then sharding
- "What's the difference between sharding and replication?" — replication copies data (HA + read scale); sharding partitions data (write scale)
- "When would you choose vertical?" — stateful services early in their life, managed databases where sharding is operationally costly, low-traffic systems where simplicity wins

**Phrases that signal depth**

- Mention replication lag and its consistency implications
- Mention the CAP theorem briefly: horizontal scale across regions forces a choice between consistency and availability during a partition
- Mention that sticky sessions are a smell — they couple users to nodes and reduce resilience; externalising state is the right fix

---

## References

- [Designing Data-Intensive Applications — Martin Kleppmann, Ch. 1 (Reliability, Scalability, Maintainability)](https://dataintensive.net/)
- [AWS: Scaling Your Applications (Elastic Load Balancing docs)](https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html)
- [Google Cloud: Horizontal vs Vertical Scaling](https://cloud.google.com/architecture/scalability)
- [High Scalability blog: Scaling patterns](http://highscalability.com/)
- [System Design Primer — Horizontal vs Vertical Scaling](https://github.com/donnemartin/system-design-primer#horizontal-scaling)