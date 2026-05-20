# Database Scaling — Replication, Partitioning, Sharding

## Table of Contents

- [Why Database Scaling Exists](#why-database-scaling-exists)
- [The Three Concepts at a Glance](#the-three-concepts-at-a-glance)
- [Replication](#replication)
  - [Single-Leader Replication](#single-leader-replication)
  - [Replication Lag and Its Anomalies](#replication-lag-and-its-anomalies)
  - [Multi-Leader Replication](#multi-leader-replication)
  - [Leaderless Replication](#leaderless-replication)
- [Partitioning](#partitioning)
  - [Vertical Partitioning](#vertical-partitioning)
  - [Horizontal Partitioning](#horizontal-partitioning)
- [Sharding](#sharding)
  - [Why It Is Called Sharding](#why-it-is-called-sharding)
  - [Sharding Strategies](#sharding-strategies)
  - [Problems Sharding Introduces](#problems-sharding-introduces)
  - [Consistent Hashing](#consistent-hashing)
- [Combining All Three](#combining-all-three)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

---

## Why Database Scaling Exists

A single database node has hard resource ceilings: CPU cores, RAM, disk I/O throughput, and network bandwidth. When your application grows, you hit one or more of these ceilings.

- **Vertical scaling** (buying a bigger machine) offers a quick fix but has a hard upper limit and leaves you with a single point of failure.
- **Horizontal scaling** (adding more machines) removes those limits but requires strategies to distribute data and load across nodes. Replication, partitioning, and sharding are those strategies.

How to remember the distinction between the three:

> **Replication** = same data on more nodes.  
> **Partitioning** = different data in different buckets, within one system.  
> **Sharding** = different data on different machines.

---

## The Three Concepts at a Glance

```
Problem                          Solution
---------------------------------------------------------
Reads are slow / node fails   -> Replication
Dataset too large for one DB  -> Horizontal Partitioning
Write throughput at ceiling   -> Sharding
```

---

## Replication

Replication makes copies of the *same data* on multiple nodes. It serves two distinct goals:

- **Availability** — replicas act as failover targets if the primary crashes.
- **Read scalability** — route read queries to replicas, leaving the primary free for writes.

### Single-Leader Replication

All writes go to one designated node (the **primary** or **leader**). The primary ships a replication log to one or more **replicas** (also called secondaries or followers). Reads can be served from replicas.

```
Writes
  |
  v
[Primary]
  |
  +--[replication log]--> [Replica 1]
  |
  +--[replication log]--> [Replica 2]

Reads -> [Replica 1] or [Replica 2]
```

**Synchronous vs asynchronous replication:**

- **Async (default):** Primary acknowledges the write before replicas confirm. Low write latency, but replicas can be stale — this gap is called **replication lag**.
- **Sync:** Primary waits for at least one replica to confirm before acking the client. Durable and consistent, but write latency is higher and the system stalls if the sync replica is slow or unavailable.
- **Semi-sync:** One replica is designated sync; the rest are async. A pragmatic middle ground used by MySQL.

### Replication Lag and Its Anomalies

Async replication means replicas may not yet have the latest writes. This is **eventual consistency** at the database layer, and it creates observable anomalies:

- **Read-your-writes violation:** You write a record, then immediately read it from a stale replica — your own write is invisible. Mitigation: route reads for your own data to the primary, or use sticky sessions.
- **Monotonic read violation:** Two successive reads from different replicas return data from different points in time — data appears to go *backward*. Mitigation: pin a session to the same replica.
- **Replication lag spikes:** Heavy write bursts or a slow replica can cause lag to grow to seconds or minutes, not milliseconds. Monitor lag as a key metric.

### Multi-Leader Replication

Multiple nodes accept writes. Each leader replicates to the others. Used for:

- **Multi-datacenter setups** — each datacenter has a local leader for low-latency writes.
- **Offline-capable clients** — each device is its own leader that syncs when online.

The new core problem: **write conflicts**. Two leaders can accept conflicting updates to the same record simultaneously. Resolution strategies:

- **Last-write-wins (LWW):** Timestamp-based; silently discards one write. Dangerous — data loss.
- **Merge:** Application-defined merge function; viable for some data types (counters, sets).
- **CRDTs:** Data structures designed to merge automatically without conflicts.
- **Expose to application:** Surface the conflict and let the user or business logic decide.

### Leaderless Replication

Any node accepts reads and writes (Dynamo-style — used by Cassandra, Riak). Consistency is achieved via **quorums**:

```
N = total replica nodes
W = nodes that must confirm a write
R = nodes that must respond to a read

If W + R > N, at least one node overlaps -> you always read the latest write
```

Common config: N=3, W=2, R=2. A single node failure is tolerated on both reads and writes.

Trade-offs:
- Flexible tuning of consistency vs latency via W/R values.
- Concurrent writes still cause conflicts; requires vector clocks or LWW for resolution.
- **Sloppy quorums** accept writes to available nodes even if the quorum of home nodes is unavailable, improving availability at the cost of stronger consistency.

---

## Partitioning

Partitioning divides a dataset into smaller subsets called **partitions**. Unlike sharding, partitions live within a single database system. The database query planner is aware of partition boundaries and can optimise queries accordingly.

### Vertical Partitioning

Split a table by **columns**. Move infrequently accessed or large columns (BLOB fields, long text) to a separate table or storage tier.

```
Before:
products(id, name, price, description_text, image_blob)

After vertical partitioning:
products_core(id, name, price)
products_media(product_id, description_text, image_blob)
```

Benefits: smaller row size improves cache efficiency and reduces I/O for queries that only need core fields.

### Horizontal Partitioning

Split a table by **rows** using a **partition key**. Each partition holds a non-overlapping range of rows.

```
users table, partitioned by signup_year:

Partition 2022  ->  rows where signup_year = 2022
Partition 2023  ->  rows where signup_year = 2023
Partition 2024  ->  rows where signup_year = 2024
```

The query planner applies **partition pruning** — a query with `WHERE signup_year = 2023` never touches the 2022 or 2024 partitions. This is a major win for time-series, event logs, and range-heavy workloads.

PostgreSQL, MySQL, and most RDBMSs support declarative horizontal partitioning.

---

## Sharding

Sharding is **horizontal partitioning across multiple independent database nodes**, each holding a distinct subset of the data. Each node is called a **shard**.

Sharding is what you reach for when:
- Your data is too large for one machine's disk.
- Your write throughput exceeds one machine's capacity.

### Why It Is Called Sharding

The word *shard* means a fragment of a broken whole (as in broken glass). The dataset is conceptually shattered into independent fragments; each fragment — a shard — is complete and functional on its own but holds only part of the full data. The term became mainstream through the early online game *Ultima Online*, which ran its player world across independent server shards.

### Sharding Strategies

**Range-based sharding**

Shard by contiguous ranges of the shard key.

```
Shard A:  user_id  1 – 1,000,000
Shard B:  user_id  1,000,001 – 2,000,000
Shard C:  user_id  2,000,001 – 3,000,000
```

- Range queries are efficient — they hit one shard.
- Hotspot risk: sequential IDs mean all new writes land on the same shard.
- Uneven data growth breaks range assumptions over time.

**Hash-based sharding**

Apply a hash function to the shard key; assign to shard by modulo.

```
shard = hash(user_id) % num_shards
```

- Uniform key distribution by design.
- Range queries become scatter-gather — must fan out to all shards and merge.
- Adding or removing a shard changes `num_shards`, causing most keys to remap and requiring massive data movement. Consistent hashing mitigates this.

**Directory-based sharding**

A dedicated lookup service maps each key (or key range) to a shard.

```
[Client]
   |
   v
[Routing / Lookup Service]
   |
   +-- key 'user:123' -> Shard B
   +-- key 'user:456' -> Shard A
   |
   v
[Correct Shard]
```

- Maximum flexibility; rebalancing is a metadata update.
- Lookup service is a bottleneck and a single point of failure.
- Adds a network hop to every operation.

**Geographic / tenant-based sharding**

Route by user region or customer tenant. Used in multi-tenant SaaS for data residency and isolation.

### Problems Sharding Introduces

Sharding is a significant architectural commitment. Interviewers will probe these:

- **Cross-shard joins:** Data for a query may live on multiple shards. SQL JOINs across shards are not possible natively; you either denormalise, join in application code, or redesign the data model.
- **Cross-shard transactions:** ACID guarantees across shards require distributed transactions (two-phase commit, 2PC). 2PC is slow, introduces coordinator failure risk, and most teams avoid it by designing data access patterns that don't need cross-shard atomicity.
- **Resharding:** When a shard grows too large, you must split it. In naive modulo hashing, changing the number of shards remaps almost every key. Consistent hashing dramatically reduces the remapping.
- **Hotspots / celebrity problem:** A shard key like `user_id` seems uniform until one user (a celebrity with millions of followers) generates a disproportionate write load on one shard. Mitigations: append a random salt to the key, use composite keys, or special-case hot keys.
- **Operational complexity:** Every shard is an independent database. Schema migrations, backups, monitoring, and incident response must be applied across all shards.

### Consistent Hashing

Standard modulo hashing: `shard = hash(key) % N`. Change N → almost all keys remap.

Consistent hashing places both shards and keys on a virtual ring. A key is assigned to the first shard clockwise from it on the ring.

```
Ring (conceptual):

        Key X
         |
    +---------+
    |         |
  Shard C   Shard A
    |         |
    +--Shard B-+

Key X is assigned to Shard A (first shard clockwise).
Adding Shard D only remaps keys between D and Shard A.
```

Adding or removing one shard only remaps `1/N` of keys on average. **Virtual nodes (vnodes)** — each physical shard claims multiple positions on the ring — smooth out uneven distribution.

---

## Combining All Three

Real production systems layer all three:

```
[Application Servers]
         |
         v
[Shard Router]
  shard = hash(user_id) % 4
         |
  +------+------+------+------+
  |      |      |      |      |
[S0]   [S1]   [S2]   [S3]     <- shards (different data per shard)
  |
  +-- [Primary]               <- accepts all writes
  |-- [Replica 1]             <- serves reads
  +-- [Replica 2]             <- standby / DR
```

Each shard is itself a replicated cluster. Sharding distributes writes and data volume; replication provides availability and read scale *within* each shard.

---

## Interview Cheat Sheet

**Common prompts and anchor answers:**

- *"Reads are slow, how do you fix it?"* — Read replicas, read-through cache (Redis/Memcached), index review.
- *"Writes are hitting a ceiling, how do you fix it?"* — Sharding; also consider async write queue, CQRS, write-optimised DB (Cassandra).
- *"How do you choose a shard key?"* — High cardinality, even distribution, aligns with the dominant query pattern, avoids known hotspots.
- *"What breaks when you shard?"* — Cross-shard joins and transactions, resharding pain, hotspots, operational overhead.
- *"What is replication lag?"* — Async replicas may not yet reflect recent writes; causes read-your-writes and monotonic read violations.
- *"Partitioning vs sharding?"* — Partitioning is within one DB node; sharding is across multiple independent nodes.
- *"How do you reduce resharding overhead?"* — Consistent hashing; virtual nodes.
- *"How do you handle a hotspot shard?"* — Salting the shard key, special-casing hot keys, composite keys.

**Signal phrases that show depth:**

- "replication lag" and its anomalies (read-your-writes, monotonic reads)
- "consistent hashing to minimise key remapping on topology change"
- "shard key selection is the hardest and most consequential decision"
- "cross-shard transactions require distributed coordination — we prefer to design around them"
- "quorum reads and writes give tunable consistency/availability trade-offs"
- "each shard needs its own replica set — sharding and replication are complementary"

---

## References

- Kleppmann, M. *Designing Data-Intensive Applications* — Chapters 5 (Replication), 6 (Partitioning). O'Reilly, 2017.
- Amazon DynamoDB paper — Dynamo: Amazon's Highly Available Key-Value Store (2007).
- Google Bigtable paper — Bigtable: A Distributed Storage System for Structured Data (2006).
- [PostgreSQL documentation — Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)
- [Cassandra documentation — Data partitioning and replication](https://cassandra.apache.org/doc/latest/)