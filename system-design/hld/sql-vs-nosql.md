# SQL vs NoSQL — When to Use Which, Trade-offs

## Table of Contents

- [What These Are](#what-these-are)
- [Why NoSQL Exists](#why-nosql-exists)
- [The Four NoSQL Families](#the-four-nosql-families)
- [ACID vs BASE](#acid-vs-base)
- [CAP Theorem](#cap-theorem)
- [Scaling](#scaling)
- [Schema and Query Flexibility](#schema-and-query-flexibility)
- [When to Use Which](#when-to-use-which)
- [The Hybrid Reality](#the-hybrid-reality)
- [Trade-off Summary](#trade-off-summary)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

---

## What These Are

**SQL databases** (relational databases) store data in tables of rows and columns with a predefined schema. They use SQL (Structured Query Language) for queries and enforce ACID guarantees for transactions.

The relational model was invented by E.F. Codd at IBM in 1970. The key insight was that data with natural relationships could be described with relational algebra, and a single query language (SEQUEL → SQL) could answer any question over that data. ACID guarantees became essential as banking and inventory systems required correctness above all else.

**NoSQL** = "Not Only SQL" — coined around 2009 to describe a family of non-relational databases that do not rely on a fixed schema or SQL, and often relax ACID guarantees in exchange for availability and horizontal scale.

---

## Why NoSQL Exists

In the early 2000s, internet-scale companies ran into hard ceilings with relational databases:

- **Scale ceiling** — adding more hardware to a single node has a physical limit. Horizontal sharding with foreign-key constraints was enormously painful.
- **Schema rigidity** — altering a column on a billion-row live table could lock the table for hours.
- **Not all data is tabular** — user activity streams, social graphs, and product catalogs do not map cleanly to rows and columns.
- **Write throughput** — relational engines with strict isolation struggle under millions of concurrent writes.

Google built BigTable (2004), Amazon built Dynamo (2007), Facebook built Cassandra (2008). These were later open-sourced or inspired open-source equivalents. The common theme: **trade some correctness for horizontal scale and availability**.

---

## The Four NoSQL Families

### Document Stores

Data is stored as self-contained JSON-like documents. Each document can have a different structure.

- **Examples:** MongoDB, Firestore, CouchDB
- **Sweet spot:** user profiles, product catalogs, CMS content — anything naturally document-shaped
- **Strength:** schema flexibility, rich intra-document queries, easy to evolve
- **Weakness:** poor cross-document JOINs, no built-in referential integrity

### Key-Value Stores

Data is a hash map at scale: every record is a value reachable by a unique key. No structure assumed on the value.

- **Examples:** Redis, DynamoDB (primarily), Memcached
- **Sweet spot:** sessions, caching, rate limiting, feature flags
- **Strength:** fastest reads and writes of any NoSQL type
- **Weakness:** no query flexibility — only get/put/delete by key

### Wide-Column Stores

Data is in rows, but each row can have a different set of columns. Data is grouped into column families and sorted by clustering keys.

- **Examples:** Apache Cassandra, HBase, ScyllaDB
- **Sweet spot:** time-series data, event logs, IoT telemetry, analytics write paths
- **Strength:** extremely high write throughput, scales horizontally with consistent hashing
- **Weakness:** queries must align with partition key + clustering column design — schema must be designed around access patterns

### Graph Databases

Data is stored as nodes (entities) and edges (relationships), each with properties.

- **Examples:** Neo4j, Amazon Neptune
- **Sweet spot:** social networks, recommendation engines, fraud detection (relationship traversals)
- **Strength:** multi-hop relationship queries that would require many JOINs in SQL
- **Weakness:** not suited for bulk analytics or simple CRUD at scale

---

## ACID vs BASE

These are the two competing consistency philosophies.

### ACID (relational databases)

- **Atomicity** — a transaction is all-or-nothing; partial updates never persist
- **Consistency** — a transaction moves the database from one valid state to another (constraints, triggers enforced)
- **Isolation** — concurrent transactions execute as if they were sequential; intermediate states are invisible
- **Durability** — once committed, a transaction survives crashes (written to disk/WAL)

### BASE (most NoSQL systems)

- **Basically Available** — the system remains operational; it will respond to every request
- **Soft state** — the state of the system may change over time even without new input (replicas converging)
- **Eventual Consistency** — given no new updates and enough time, all replicas will converge to the same value

**Memory hook:** ACID = hard guarantees, harder to scale. BASE = soft guarantees, easier to scale.

### What "eventual consistency" means in practice

If you write a record to Cassandra and then immediately read it from a different replica, you may get a stale value. Given no further writes and after a short replication lag, all replicas will return the same value. For many applications (social feeds, analytics, recommendations) this is acceptable. For banking balances or inventory counts, it is not.

---

## CAP Theorem

In a distributed system, you can only guarantee two of the following three properties simultaneously:

- **Consistency (C)** — every read sees the most recent write (or returns an error)
- **Availability (A)** — every request receives a response (not necessarily the latest data)
- **Partition Tolerance (P)** — the system continues operating despite network partitions between nodes

Network partitions are unavoidable in real distributed systems, so **P is non-negotiable**. The real choice is between **CP** (sacrifice availability on partition) and **AP** (sacrifice consistency on partition).

```
CP Systems                         AP Systems
──────────────────────────────     ──────────────────────────────
HBase, Zookeeper                   Cassandra, DynamoDB, CouchDB
MongoDB (strong consistency mode)  Redis (cluster mode)
CockroachDB, Google Spanner        Riak
```

Traditional single-node SQL is **CA** (not distributed, so partition tolerance isn't a concern). Distributed SQL (Spanner, CockroachDB) achieves **CP** with strong consistency across nodes.

---

## Scaling

### How SQL scales

- **Vertical scaling (scale up):** add more CPU, RAM, faster SSDs to one machine. Has a hard ceiling and is expensive.
- **Read replicas:** replicate data to read-only standbys to distribute read load. Writes still go to one primary.
- **Manual sharding:** partition data across multiple nodes by a shard key (e.g., user ID % N). Hard to maintain: cross-shard JOINs break, foreign keys across shards don't exist, resharding is painful.
- **Tools:** Vitess (MySQL sharding proxy), Citus (Postgres sharding extension), PlanetScale

### How NoSQL scales

- **Horizontal scaling (scale out):** add more nodes; data is automatically redistributed.
- Cassandra uses **consistent hashing** — each node owns a token range; data is routed by partition key hash.
- DynamoDB auto-scales partitions transparently.
- MongoDB has native sharding with a configurable shard key.
- Scales both reads and writes (unlike SQL read replicas which only help reads).

```
SQL scaling path
────────────────
Single node → Add RAM/CPU (vertical) → Read replicas → Manual sharding (painful)

NoSQL scaling path
──────────────────
Cluster of N nodes → Add node → Data re-balances automatically
```

---

## Schema and Query Flexibility

### Schema

SQL requires a **schema-first** approach: define tables, columns, types, and constraints before inserting data. Changing a column later requires a migration (ALTER TABLE), which can lock a large table.

NoSQL (document/key-value) offers **schema-on-read**: no upfront schema definition. Documents in the same collection can have different fields. This is powerful for rapidly evolving data models but shifts the enforcement burden to application code — missing fields must be handled in every query and transformation.

Wide-column stores (Cassandra) still require schema design, but the schema must be **designed around access patterns** rather than around the data model. This is the biggest mental shift from relational databases.

### Query Flexibility

SQL's relational model allows **ad-hoc queries** — you can ask questions you didn't anticipate when designing the schema, using arbitrary JOINs, subqueries, window functions, and aggregations.

NoSQL generally requires you to know your access patterns upfront:

- Document store: good queries within a document; poor cross-document JOINs
- Key-Value: only get/put/delete by primary key (no range scans in most implementations)
- Wide-Column (Cassandra): queries must filter by the partition key first; secondary index support is limited
- Graph: excellent for multi-hop traversals; poor for analytical aggregations

**Cassandra query design example:**

If you need "get all messages in a chat room, sorted by time," your schema must have `room_id` as the partition key and `timestamp` as a clustering column. You cannot query by message content or author without a full scan.

---

## When to Use Which

### Use SQL when:

- **Strong relationships** — data has many foreign keys and JOINs are common (orders → line items → products → inventory)
- **ACID compliance is critical** — financial transactions, booking systems, inventory where double-booking is unacceptable
- **Schema is stable** — structure is known upfront and changes slowly
- **Complex ad-hoc queries** — reporting, analytics dashboards, business intelligence
- **Team familiarity** — SQL expertise is widespread; hiring is easier

**Examples:** banking systems, e-commerce order management, payroll, ERP, reservation systems

### Use NoSQL when:

- **Massive scale** — millions of writes per second that a single SQL primary cannot absorb
- **Document-shaped data** — user profiles, product catalogs where each entity has different attributes
- **Rapidly evolving schema** — early-stage product where the data model changes weekly
- **Predefined access patterns** — you know exactly how data will be queried; no ad-hoc queries
- **Low latency with eventual consistency** — social feeds, activity logs where stale reads are acceptable
- **Specific data structures** — graphs, time-series, leaderboards (Redis sorted sets)

**Examples:** social media feeds, IoT sensor streams, real-time analytics, content management, session storage, recommendation engines

---

## The Hybrid Reality

Almost no production system at scale is purely SQL or purely NoSQL. A mature architecture typically layers multiple stores:

```
User-facing Requests
       |
       v
 [API Layer]
  /    |    \
 v     v     v
[Postgres]  [Redis]     [Elasticsearch]
  Core        Cache /     Full-text
  transact.   Sessions    search
  data        Rate-limit
                |
                v
          [Cassandra / DynamoDB]
            Event streams
            Time-series
            High-write logs
```

**Rule of thumb:**
- Start with Postgres (or any SQL DB). It handles most use cases.
- Add Redis when caching or low-latency key lookups become necessary.
- Add a document store or wide-column store when a specific access pattern is genuinely unserviceable by SQL.
- Add a graph DB only when relationship traversals dominate the workload.

---

## Trade-off Summary

```
Dimension          SQL                         NoSQL
───────────────────────────────────────────────────────────────────
Consistency        Strong (ACID)               Eventual (BASE), tunable
Horizontal scale   Hard (manual sharding)      Built-in
Schema             Fixed, schema-first         Flexible, schema-on-read
Query flexibility  High (ad-hoc SQL)           Low (access-pattern-driven)
Joins              Native                      Application-side or none
Write throughput   Limited by single primary   Very high (distributed writes)
Maturity/tooling   Decades of tooling          Varies by DB
Operational cost   Familiar, well-understood   Higher operational complexity
```

---

## Interview Cheat Sheet

**The core question interviewers ask:** "Why did you choose SQL/NoSQL here?" — they want a *justified* choice based on requirements, not familiarity.

**Signal phrases that show depth:**

- "Given that this service needs ACID transactions across user accounts, I'd use Postgres rather than Cassandra."
- "For the event log, which is append-only with high write throughput, Cassandra's wide-column model fits well — I'd partition by user_id and cluster by timestamp."
- "I'd use Redis here for sub-millisecond session lookups; consistency requirements are low because sessions are short-lived."
- "Since this is a social graph with multi-hop traversals, Neo4j or a graph model avoids the N-level JOIN problem in SQL."

**CAP positioning to know cold:**
- Cassandra → AP (tunable with consistency levels: ONE, QUORUM, ALL)
- DynamoDB → AP by default (can enable strong consistency per-read)
- MongoDB → CP (primary always has latest data; secondaries may lag)
- Postgres → CA (single node); CockroachDB → CP (distributed)
- Zookeeper, etcd → CP (used for coordination, not application data)

**Cassandra consistency levels** (worth knowing for senior-level interviews):
- `ONE` — fastest, weakest; reads/writes confirmed by one replica
- `QUORUM` — majority of replicas must confirm; tunable consistency
- `ALL` — strongest; all replicas must confirm; essentially blocking

**Common follow-up questions:**
- "What happens if you need to run a complex report over your NoSQL data?" → Data warehouse / ETL to analytical store (BigQuery, Redshift), or CQRS read models
- "How would you handle a schema change in your document store?" → Application-level migration, dual-read during transition
- "What if your SQL database becomes a bottleneck?" → Read replicas first, then evaluate caching (Redis), then sharding or migration to distributed SQL

**Frequent misconceptions:**
- "NoSQL is faster than SQL" — false in general; it depends on the access pattern and data size. Redis is fast because it's in-memory. Cassandra is fast for specific write patterns, not arbitrary queries.
- "NoSQL doesn't need schema design" — false for wide-column stores; schema design is just as important, only the axis is different.
- "SQL can't scale" — Postgres with Citus, Vitess for MySQL, and distributed SQL like CockroachDB and Spanner scale horizontally.

---

## References

- Martin Fowler, *NoSQL Distilled* (Sadalage & Fowler, 2012)
- Werner Vogels, "Eventually Consistent" — ACM Queue, 2008
- Eric Brewer, CAP Theorem — PODC Keynote, 2000
- [Cassandra Architecture — Apache Docs](https://cassandra.apache.org/doc/latest/cassandra/architecture/)
- [DynamoDB Core Concepts — AWS Docs](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.html)
- [CAP FAQ — Henry Robinson](https://www.the-paper-trail.org/page/cap-faq/)
- [Designing Data-Intensive Applications — Martin Kleppmann, Ch. 2, 5](https://dataintensive.net/)