# NoSQL Databases — Key-Value, Document, Wide-Column, Graph

```
Status:       🌳 Evergreen
Created:      2026-06-10
Last Updated: 2026-06-10
```

> Companion to `hld/sql-vs-nosql.md` which covers when to choose NoSQL over SQL.
> This file digs into what each NoSQL family actually is, how it works internally,
> and how to reason about trade-offs in an interview.


## Table of Contents

- [What Is NoSQL](#what-is-nosql)
- [The Four Families at a Glance](#the-four-families-at-a-glance)
- [Key-Value Stores](#key-value-stores)
- [Document Stores](#document-stores)
- [Wide-Column Stores](#wide-column-stores)
- [Graph Databases](#graph-databases)
- [Choosing the Right Type](#choosing-the-right-type)
- [Comparison Table](#comparison-table)
- [How to Remember This](#how-to-remember-this)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)


## What Is NoSQL

"NoSQL" was coined in 2009 as a Twitter hashtag for a meetup on non-relational databases — originally meaning "No SQL," later rebranded to **"Not Only SQL"** once the community realised these databases complement rather than replace relational systems.

The driving force was scale. Around 2005–2010, Google, Amazon, and Facebook faced data volumes and write rates that traditional RDBMS couldn't serve cost-effectively:

- Horizontal sharding in Postgres is manual and painful
- Schema migrations on billion-row tables take hours with table locks
- `JOIN` operations don't distribute cleanly across machines
- Not all data fits naturally into rows and tables — events, graphs, and documents are awkward

The result: a wave of purpose-built databases, each optimised for a specific **data shape** and **access pattern**. The four major families are not interchangeable — picking the wrong one is a system design failure.


## The Four Families at a Glance

```
Data Shape                      Family              Example Systems
──────────────────────────────  ──────────────────  ──────────────────────────────
key → opaque value              Key-Value           Redis, DynamoDB, Memcached
self-contained nested object    Document            MongoDB, Firestore, CouchDB
entity + time-ordered events    Wide-Column         Cassandra, HBase, Bigtable
nodes + relationship edges      Graph               Neo4j, Amazon Neptune
```


## Key-Value Stores

### What It Is

The simplest possible database: a distributed hash map. Every value is stored and retrieved by a single opaque key. The database has no knowledge of what's inside the value — it treats it as a blob.

### Data Model

```
+------------------+-----------------------------+
|       Key        |           Value             |
+------------------+-----------------------------+
| "session:u42"    | { binary blob / string }    |
| "rate:1.2.3.4"   | "47"                        |
| "cart:user_9"    | { serialised JSON bytes }   |
| "lock:order_71"  | "worker-3"                  |
+------------------+-----------------------------+
         O(1) lookup by key only
```

### Internals — Redis

Redis is the canonical reference for this family.

- Entirely **in-memory**; commands processed single-threaded (I/O multithreaded since Redis 6)
- Different value types use different internal data structures:
  - Strings → SDS (Simple Dynamic String)
  - Sorted sets → skip list + hash table
  - Small hashes/sets → zip list (memory-compact encoding)
- **Persistence options** (two can run together):
  - **RDB** — point-in-time snapshots written to disk on a schedule
  - **AOF** — every write command appended to a log file; replayed on restart
- **Replication** — primary/replica streaming replication
- **Redis Cluster** — consistent hashing across 16,384 virtual hash slots; each node owns a range

### Internals — DynamoDB

- Stores data on SSDs partitioned by a hash of the partition key
- Supports two key schemas: partition key only, or partition key + sort key (acts as a key-value and a simple document store)
- **Provisioned vs on-demand** capacity modes
- Eventually consistent reads by default; strongly consistent reads available at higher cost

### Strengths

- Sub-millisecond reads and writes at any scale
- Trivially sharded — no schema to co-ordinate
- Perfect for high-frequency, simple value types

### Weaknesses and Problems It Brings

- **No query by value** — you cannot find all users whose session expires before midnight; you can only look up by exact key
- The database is blind to the value — no secondary indexes on value fields
- No relationships or referential integrity
- Cache invalidation and TTL management are entirely the application's responsibility
- Hot keys are a real problem: a single viral user-id can overwhelm one shard

### Best Use Cases

Session storage, caching layers, rate limiting counters, distributed locks, leaderboards (Redis sorted sets), feature flags, real-time pub/sub


## Document Stores

### What It Is

Stores self-contained, semi-structured records called **documents** — typically JSON or BSON. A document is the unit of storage, analogous to a row in SQL but with a flexible, nested schema. Documents are grouped into **collections** (not tables).

**Why "document"?** Each record is a standalone artefact that contains all its own data — like a paper document rather than a row that references other rows via foreign keys.

### Data Model

```
Collection: users
+------------------------------------------------------+
| { "_id": "u1",                                       |
|   "name": "Alice",                                   |
|   "address": { "city": "NYC", "zip": "10001" },      |
|   "tags": ["admin", "beta-user"],                    |
|   "created_at": ISODate("2024-01-15")                |
| }                                                    |
+------------------------------------------------------+
| { "_id": "u2",                                       |
|   "name": "Bob",                                     |
|   "age": 29              <- different fields are OK  |
| }                                                    |
+------------------------------------------------------+

Secondary indexes allow querying on any field:
  db.users.find({ "address.city": "NYC" })
  db.users.find({ "tags": "admin" })
```

### Internals — MongoDB

- Documents stored in **BSON** (Binary JSON) — richer type system than plain JSON (ObjectId, Date, Decimal128)
- **WiredTiger** storage engine: B-tree indexed files for both data and indexes
- **Replica sets** for high availability: one primary, multiple secondaries, automatic failover via Raft-like election
- **Sharding** via a `mongos` query router: range-based or hash-based on a designated shard key
  - The shard key is chosen at collection creation time and is immutable — choose badly and you cannot change it without a full migration
- **Aggregation pipeline** for GROUP BY, filtering, reshaping, and joining across collections (`$lookup`)

### Strengths

- Schema flexibility — add fields freely without ALTER TABLE migrations
- Natural fit for object graphs (the document maps directly to a struct or class)
- Embedded sub-documents avoid many joins at read time
- Rich query capability — not limited to key lookup

### Weaknesses and Problems It Brings

- **No referential integrity** — orphaned references accumulate silently
- **Data duplication** from denormalisation — updating a user's name can mean touching thousands of documents
- Multi-document transactions only from MongoDB 4.0; they carry significant write amplification and should be used sparingly
- Cross-collection joins (`$lookup`) are expensive; the document model nudges you to denormalise or accept application-level joins
- A poorly chosen shard key creates **hot partitions** — one shard handles all writes while others sit idle

### Best Use Cases

Content management systems, user profiles and preferences, product catalogs, real-time applications, any domain where the entity is self-contained and schema evolves frequently


## Wide-Column Stores

### What It Is

A database where each row has a **partition key** that routes it to a shard, and within a partition the columns are sorted by a **clustering key** into an ordered sequence. Different partitions can have different column sets; a single partition can hold thousands of columns — hence the name "wide."

> **Critical distinction**: wide-column ≠ columnar/column-oriented. Wide-column stores (Cassandra, HBase) are optimised for row-level writes and point lookups. Columnar stores (Redshift, BigQuery, Parquet) store data column-by-column for analytical full-table scans. Conflating these two is a red flag in interviews.

### Data Model — Cassandra

```
Keyspace: analytics          (like a schema/database)
  └── Table: user_events

Partition Key:  user_id      (determines which node owns this data)
Clustering Key: event_ts     (sort order within the partition)

user_id  | event_ts            | action  | metadata
---------+---------------------+---------+-----------------
user_1   | 2024-01-01 09:00    | login   | {ip: "1.2.3.4"}
user_1   | 2024-01-01 10:15    | click   | {btn: "A"}
user_1   | 2024-01-02 08:30    | logout  | {}
---------+---------------------+---------+-----------------
user_2   | 2024-01-01 11:00    | signup  | {src: "email"}

All user_1 rows live on the same node, sorted by event_ts.
Query: "all events for user_1 in January" is a single-shard sequential scan.
Query: "all users who logged in today" requires a full cluster scan — bad.
```

The table is designed around a query. You do not design around entities and then query freely — you design the schema to serve a specific query pattern.

### Internals — Cassandra Write Path (LSM Tree)

Cassandra uses a **Log-Structured Merge Tree** (LSM tree) which makes writes extremely fast at the cost of more expensive reads.

```
Write Request
     |
     v
CommitLog  (sequential append to disk — survives crash, never read by clients)
     |
     v
MemTable   (sorted in-memory structure — accumulates recent writes)
     |
     | flush when MemTable reaches threshold
     v
SSTable    (immutable, sorted file on disk — written once, never modified)
     |
     | background compaction merges multiple SSTables
     v
Fewer, larger SSTables  (reclaims tombstone space, improves read speed)
```

**Why writes are fast:** writing to CommitLog is a sequential disk append + in-memory insert — no random I/O.

**Why reads are more expensive:** a read must check the MemTable and potentially multiple SSTables, then merge the results. Bloom filters are used to skip SSTables that definitely do not contain the key.

**Deletes are tombstones:** a delete writes a tombstone marker, not an actual removal. The row appears deleted to readers but the space is only reclaimed at compaction time. Accumulating tombstones without timely compaction is a real operational footgun.

### Replication and Consistency

- Cassandra uses a **leaderless** peer-to-peer ring with consistent hashing and virtual nodes (vnodes)
- Each row is replicated to N nodes (replication factor)
- Tunable consistency per query: `ONE`, `QUORUM`, `ALL`
- `W + R > N` gives strong consistency; `W=QUORUM, R=QUORUM` is the common production setting
- **Hinted handoff**: if a replica is temporarily down, another node stores the write and forwards it when the replica recovers

### Strengths

- Extremely write-optimised — sequential CommitLog append + in-memory MemTable
- Scales horizontally with near-linear write throughput (leaderless ring)
- No single point of failure
- Clustering key gives free time-range queries within a partition

### Weaknesses and Problems It Brings

- Reads are more expensive than writes — must merge MemTable + multiple SSTables
- **Query-driven design is mandatory** — you must know your queries at schema design time; ad-hoc queries across partitions require full cluster scans
- No joins, no foreign keys, no cross-partition secondary indexes (global secondary indexes exist but are expensive)
- Compaction causes periodic I/O spikes that must be scheduled carefully in production
- Tombstone accumulation from frequent deletes can seriously degrade read performance

### Best Use Cases

Time-series data (IoT sensor streams, metrics, telemetry), activity and audit logs, write-heavy analytics, user event history, anything with a natural partition key and a time-ordered access pattern


## Graph Databases

### What It Is

Stores data as **nodes** (entities) and **edges** (named, directed relationships between entities). Both nodes and edges can carry properties. Querying means traversing the graph — following edges from node to node.

**Why "graph"?** Based on the mathematical definition: a graph G = (V, E) where V is the set of vertices (nodes) and E is the set of edges connecting them.

### Data Model

```
Scenario A: Social graph

  (Alice)--[:FOLLOWS]-->(Bob)
  (Alice)--[:FOLLOWS]-->(Carol)
  (Bob)--[:FOLLOWS]-->(Carol)

  Cypher: MATCH (a:User {name:"Alice"})-[:FOLLOWS*2]->(rec)
          RETURN rec.name   -- friends-of-friends

Scenario B: Recommendation

  (Alice)--[:PURCHASED]-->(ProductX)
  (Bob)--[:PURCHASED]-->(ProductX)
  (Bob)--[:PURCHASED]-->(ProductY)

  Inference: Alice might like ProductY (shared co-purchase path)
```

### Internals — Neo4j

- **Index-free adjacency**: every node record stores a direct pointer to its first relationship record. Traversing to a neighbour is O(1) — you follow a pointer, regardless of total graph size. Compare to a relational self-join where each hop is O(log N) in an index.
- Storage layout: fixed-size node records, fixed-size relationship records (each stores pointers to start node, end node, previous/next relationship in chain), property store (key-value linked list per node/edge)
- Full **ACID transactions** — Neo4j is strongly consistent within a single instance
- **Cypher** query language (`MATCH`, `WHERE`, `RETURN`) — pattern-matching against the graph structure
- Replication via a Raft-based cluster; the primary node handles all writes

### Strengths

- Relationship-heavy traversals that require many self-joins in SQL are natural and fast
- Traversal depth does not degrade performance (index-free adjacency keeps each hop O(1))
- Flexible schema — add node labels and edge types without migrations
- Cypher is expressive and readable for graph queries

### Weaknesses and Problems It Brings

- **Horizontal scaling is hard** — graph partitioning is NP-hard in general; edges that cross partition boundaries require network hops, which destroys the O(1) adjacency advantage
- Most graph databases are not suitable for full-graph analytical scans (e.g., "PageRank over 10 billion nodes")
- Niche query languages (Cypher, Gremlin) have a smaller ecosystem and hiring pool
- Write performance is generally lower than Cassandra or Redis because writes must maintain the bidirectional pointer structure
- Not a good fit for data that is mostly tabular with a few relationships

### Best Use Cases

Social networks (friend graphs, follower graphs), recommendation engines, fraud detection (follow money flows across accounts), knowledge graphs, network topology, access control and permission systems


## Choosing the Right Type

```
Primary question: what shape is your data, and what is your access pattern?

Single value lookup by ID, ephemeral or simple types
    => Key-Value  (Redis, DynamoDB)

Self-contained nested object, queried by multiple fields, schema evolves
    => Document  (MongoDB, Firestore)

Write-heavy stream of events or measurements, queried by entity + time range
    => Wide-Column  (Cassandra, Bigtable)

Rich relationships, traversal queries, "friends of friends", fraud paths
    => Graph  (Neo4j, Neptune)
```

Secondary questions that sharpen the choice:

- **Consistency requirement?** Graph (Neo4j) and relational are ACID-native. Cassandra and DynamoDB default to eventual consistency with tunable options.
- **Write volume?** Cassandra's LSM tree wins. Redis wins if data fits in memory.
- **Horizontal scale?** Cassandra and Redis Cluster scale best. Graph databases scale worst.
- **Query flexibility?** Document stores are most flexible at query time. Wide-column stores are the least flexible — queries must be anticipated at design time.


## Comparison Table

| Dimension           | Key-Value         | Document          | Wide-Column           | Graph               |
|---------------------|-------------------|-------------------|-----------------------|---------------------|
| Data unit           | key + blob        | JSON/BSON doc     | partition + columns   | nodes + edges       |
| Schema              | none              | flexible          | semi-rigid            | flexible            |
| Primary access      | key lookup        | field queries     | partition + range     | graph traversal     |
| Write pattern       | any               | any               | write-optimised       | write-pessimistic   |
| Horizontal scale    | excellent         | good              | excellent             | limited             |
| Consistency         | tunable           | tunable           | tunable               | ACID (Neo4j)        |
| Joins               | none              | none (app-level)  | none                  | traversal = join    |
| Common pitfall      | hot keys          | bad shard key     | tombstone accumulation| partition boundary  |
| Canonical system    | Redis             | MongoDB           | Cassandra             | Neo4j               |


## How to Remember This

**Mnemonic — "KDWG: Kids Don't Watch Graph"**

| Letter | Type | Mental Image |
|--------|------|--------------|
| K | Key-Value | A locker — insert your key, get the contents, no questions asked |
| D | Document | A filing cabinet — each folder is self-contained with its own fields |
| W | Wide-Column | A spreadsheet turned sideways — each row can be very wide |
| G | Graph | A web — nodes are dots, edges are threads connecting them |

**For Cassandra internals:** think POST OFFICE
- **P**artition key (address → which post office branch)
- **C**lustering key (sorted order within the branch)
- **L**SM tree (CommitLog → MemTable → SSTable → Compaction)

**Wide-column ≠ columnar** — say it once per interview if the topic comes up, it immediately signals depth.


## Interview Cheat Sheet

### Signal Phrases

- "The right NoSQL type depends on access pattern first, data shape second — not the other way around."
- "Wide-column stores are query-driven — you model the schema around the query, not the entity."
- "Cassandra's LSM tree makes writes O(1) sequential appends, but reads require merging MemTable and SSTables — it's a write-optimised trade-off."
- "Graph databases use index-free adjacency so each hop is O(1), whereas a relational self-join is O(log N) per level of depth."
- "Wide-column and columnar are different things — Cassandra is not an analytical database."

### Red Flags to Avoid

- "NoSQL is better/faster than SQL" — wrong framing; it is use-case dependent
- "Document stores support JOINs" — they do not natively; `$lookup` in MongoDB is expensive and not a true join
- "Cassandra rows are like SQL rows" — they are sorted maps within a partition, not flat rows
- "Graph databases scale horizontally as well as Cassandra" — they do not; graph partitioning is fundamentally hard
- Treating a tombstone as a delete — it is a marker; the space is only reclaimed at compaction

### Common Interview Probes

- "How would you model a Twitter-style activity feed for 100M users?"
  - Wide-column (Cassandra): partition = user_id, clustering = tweet_timestamp
- "Why would you choose Cassandra over DynamoDB for time-series IoT data?"
  - Cassandra: open-source, tunable consistency, better clustering key semantics; DynamoDB: fully managed, serverless scaling, simpler ops
- "What happens when a Cassandra MemTable fills up?"
  - It is flushed to a new immutable SSTable on disk; the CommitLog segment is eventually recycled
- "How does Neo4j traverse a graph faster than a relational database doing self-joins?"
  - Index-free adjacency — direct pointer to neighbour vs. index lookup per join level
- "A MongoDB collection has 500M documents. Queries are slow. What do you look at?"
  - Missing secondary index, bad shard key causing hot partition, large documents causing full collection scans, aggregation without `$match` before `$group`


## References

- Alex Xu, *System Design Interview Vol. 1* — Chapter on storage systems
- Martin Kleppmann, *Designing Data-Intensive Applications* — Chapters 2 (data models) and 3 (storage engines, LSM trees vs B-trees)
- Apache Cassandra documentation — [cassandra.apache.org](https://cassandra.apache.org/doc/latest/)
- MongoDB documentation — [mongodb.com/docs](https://www.mongodb.com/docs/)
- Neo4j documentation — [neo4j.com/docs](https://neo4j.com/docs/)
- Gaurav Sen — "What is a Wide-Column Database?" (YouTube)
- Concept && Coding — NoSQL database deep dives (YouTube)