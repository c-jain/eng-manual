---
Status: 🌳 Evergreen
Created: 2026-06-11
Last Updated: 2026-06-11
---

# Distributed File Storage — S3, HDFS, GFS

## Table of Contents

- [Why This Exists](#why-this-exists)
- [The Three Systems at a Glance](#the-three-systems-at-a-glance)
- [GFS — The Blueprint](#gfs--the-blueprint)
- [HDFS — The Open-Source Clone](#hdfs--the-open-source-clone)
- [S3 — Object Storage, Not a File System](#s3--object-storage-not-a-file-system)
- [Core Concepts Across All Three Systems](#core-concepts-across-all-three-systems)
- [HDFS vs S3 — When to Use Which](#hdfs-vs-s3--when-to-use-which)
- [How to Remember This](#how-to-remember-this)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

---

## Why This Exists

Before 2003, storing a 1 TB file reliably meant buying expensive RAID arrays. Google needed
to process petabytes of web-crawl data on commodity hardware that fails regularly. The answer:
spread data across many cheap machines, replicate each chunk, and let software handle failures.
That 2003 GFS paper changed how everyone thinks about storage.

```
Timeline

  2003 ── GFS paper published (Google)      ← the blueprint
  2006 ── HDFS released (Apache / Yahoo)    ← open-source GFS clone, Hadoop ecosystem
  2006 ── Amazon S3 launches                ← object store, HTTP API, fully managed
  2010 ── HDFS NameNode HA added            ← ZooKeeper-based standby NameNode
  2013 ── HDFS Federation                   ← multiple NameNodes, shared DataNodes
  2020 ── S3 strong consistency             ← S3 drops eventual-consistency quirks
```

---

## The Three Systems at a Glance

| System | Type | Key Innovation |
|--------|------|----------------|
| GFS (2003) | Distributed file system | Single master, 64 MB chunks, relaxed consistency |
| HDFS (2006) | Distributed file system | Open-source GFS; powers Hadoop/Spark ecosystem |
| S3 (2006) | Object store | Flat key-value, HTTP API, immutable objects, managed |

The first two share nearly the same architecture. S3 breaks from file-system semantics entirely.

---

## GFS — The Blueprint

### Architecture

```
GFS / HDFS Architecture

                  ┌─────────────────────┐
                  │  Master / NameNode  │   stores: namespace tree,
                  │  (single process)   │   chunk → chunkserver map,
                  └──────────┬──────────┘   access control metadata
                             │
               ┌─────────────┼─────────────┐
               │             │             │
       ┌───────┴──────┐ ┌────┴──────┐ ┌────┴──────────┐
       │ ChunkServer  │ │ChunkServer│ │  ChunkServer  │
       │   Rack A     │ │  Rack B   │ │    Rack C     │
       ├──────────────┤ ├───────────┤ ├───────────────┤
       │  chunk B1r1  │ │ chunk B1r2│ │   chunk B1r3  │
       │  chunk B2r1  │ │ chunk B2r2│ │   chunk B2r3  │
       └──────────────┘ └───────────┘ └───────────────┘

  Client contacts Master for metadata ONLY.
  Client reads/writes directly to ChunkServers.
  Master is never in the data path.
```

### Key Design Decisions

**64 MB chunk size**

Before GFS, filesystems used 4–8 KB blocks. GFS uses 64 MB because Google files were enormous
(web-crawl data), sequential reads dominated, and large chunks meant far fewer metadata entries —
the master's entire namespace fits in RAM.

Downside: small files become hot spots. A 1 KB config file stored in a 64 MB chunk on one server
gets hammered if many clients request it.

**Single master**

Radical simplification. One process owns all namespace state, so no distributed consensus is
needed for metadata operations. The master never proxies data — it only answers "which servers
hold chunk X?". Its state fits in memory (string maps + 64-byte chunk handles). An operation log
and periodic fsimage checkpoints make it recoverable.

**Relaxed consistency**

Concurrent appends to the same file can produce undefined regions (bytes may be duplicated or
interleaved). Applications using GFS (like MapReduce) were designed to tolerate duplicates.
This removed an entire class of distributed coordination problems at the cost of unusual semantics.

---

## HDFS — The Open-Source Clone

HDFS is GFS ported to Java, released as part of Apache Hadoop. The mental model is identical;
the terminology differs.

| GFS Term | HDFS Term | Default Size / Factor |
|----------|-----------|-----------------------|
| Master | NameNode | — |
| ChunkServer | DataNode | — |
| Chunk | Block | 128 MB (was 64 MB) |
| Replication factor | Replication factor | 3 |
| Shadow master | Secondary NameNode* | — |

> **Important:** The Secondary NameNode is **not** a hot standby. It periodically merges the
> edit log with the fsimage checkpoint to prevent unbounded log growth. For true HA, you need a
> Standby NameNode + Quorum Journal Manager (ZooKeeper-based shared edit log).

### HDFS Write Path

```
Phase 1 — Metadata

  Client ──── (1) create file, request block ─────► NameNode
  Client ◄─── (2) pipeline: [DN-1, DN-2, DN-3] ──── NameNode

Phase 2 — Pipeline Replication

  Client ──► DN-1 ──► DN-2 ──► DN-3    (data streams left to right)
  Client ◄── DN-1 ◄── DN-2 ◄── DN-3    (ACKs flow right to left)

Phase 3 — Commit

  Client ──── (3) block written ───────────────────► NameNode
              NameNode updates block → DataNode map
```

The pipeline model means the client only speaks to DN-1. DN-1 forwards to DN-2 simultaneously
while receiving from the client. This bounds write latency — you do not wait for all replicas
to confirm before writing the next packet.

### HDFS Read Path

```
  Client ──── (1) open file, get block locations ──► NameNode
  Client ◄─── (2) sorted DataNode list per block ─── NameNode

  Client ──── (3) read block from nearest DN ──────► DN-1
  Client ◄─── (4) data stream ─────────────────────── DN-1

  NameNode is NOT in the data path.
  Client picks the topologically nearest DataNode (same rack preferred).
```

### Rack-Aware Replication

```
3-Replica Placement Strategy

  Replica 1 → same rack as client          (fast first write)
  Replica 2 → different rack               (rack-level fault tolerance)
  Replica 3 → different node, same rack as replica 2  (cheap third copy)

  Rack A               Rack B
  ┌──────────┐         ┌──────────────────────┐
  │  DN-1    │         │  DN-2  │    DN-3     │
  │  (r1)    │         │  (r2)  │    (r3)     │
  └──────────┘         └──────────────────────┘

  If Rack B loses power entirely, Replica 1 on Rack A survives.
```

### Fault Detection and Recovery

- DataNodes send heartbeats to the NameNode every ~3 seconds
- If the NameNode receives no heartbeat for ~10 minutes, it marks the DataNode dead
- NameNode identifies all blocks now below replication factor and schedules re-replication
- Each block carries a checksum; corrupt replicas are discarded and re-replicated automatically

### HDFS Federation

Single NameNode limits namespace scalability — each file/block consumes ~150 bytes of NameNode
heap. At 1 billion files, that is 150 GB of heap.

HDFS Federation introduces multiple independent NameNodes, each managing its own namespace
volume. DataNodes serve all NameNodes via separate block pools. This horizontally scales the
metadata tier while leaving the data tier unchanged.

---

## S3 — Object Storage, Not a File System

### The Conceptual Shift

GFS and HDFS give file system semantics: directories, hierarchical paths, append operations,
atomic renames. S3 is an **object store**: a flat key-value system where a key like
`photos/2024/img.jpg` is just a string, not a path. There are no real directories, no atomic
renames, no append.

Eliminating file-system semantics is what makes S3 infinitely horizontally scalable — there is
no global namespace tree to coordinate.

### S3 Request Flow

```
Object Write (PUT /bucket/key)

  Client ──── HTTPS PUT ────────────────────────► S3 API Endpoint
                              S3 API ─── route ─► Metadata Service
                                       (maps bucket+key → storage location)
                              S3 API ─── store ─► Storage Layer
                                       (replicated across ≥ 3 AZs)
  Client ◄─── 200 OK + ETag ───────────────────── S3 API Endpoint

Object Read (GET /bucket/key)

  Client ──── HTTPS GET ────────────────────────► S3 API Endpoint
                              S3 API ── lookup ─► Metadata Service
                              S3 API ── fetch ──► Storage Layer
  Client ◄─── 200 OK + body ───────────────────── S3 API Endpoint
```

### S3 Internals (Inferred — Amazon Has Not Published)

From re:Invent talks and engineering analysis:

- Objects are **immutable** — a PUT replaces the whole object; partial byte overwrites are
  impossible. (Contrast: HDFS supports append.)
- Internally, objects are split into pieces across storage nodes, likely via consistent hashing.
- A separate **metadata service** maps bucket+key to physical storage location, decoupled from
  the data service.
- **Multipart upload**: for objects >5 GB (required) or >100 MB (recommended), upload parts
  independently and in parallel (minimum 5 MB each), then call `CompleteMultipartUpload` to
  assemble. Enables resumable uploads and parallel throughput.
- **ETags**: MD5 of the object (or hash-of-part-ETags for multipart). Use for integrity checks
  and conditional requests (`If-None-Match`).
- Since **December 2020**: strong read-after-write consistency for all S3 operations. Before
  this, a PUT followed immediately by a GET could return a stale 404.

### S3 Storage Classes

```
Access Frequency → Cost (high frequency at top, low at bottom)

  Standard            ─── frequent access, millisecond retrieval
  Standard-IA         ─── infrequent, same retrieval speed, lower storage cost
  One Zone-IA         ─── infrequent, single AZ (cheaper, less durable)
  Glacier Instant     ─── archive, occasional access, millisecond retrieval
  Glacier Flexible    ─── archive, minutes-to-hours retrieval
  Deep Archive        ─── long-term cold storage, 12-hour retrieval
```

S3 Intelligent-Tiering automatically moves objects between tiers based on access patterns —
useful when access frequency is unpredictable.

---

## Core Concepts Across All Three Systems

### Chunking — Why Files Are Split

- **Parallelism**: different clients or processes read different chunks simultaneously
- **Fault isolation**: one DataNode failure affects only the chunks it holds, not the whole file
- **Load distribution**: chunks placed across nodes spread I/O load naturally
- **Metadata tractability**: the master stores one entry per chunk, keeping its namespace small

### Replication vs Erasure Coding

```
Replication (factor 3) — 1 GB file stored as 3 GB

  Block ──────► DN-1   (replica 1)
  Block ──────► DN-2   (replica 2)
  Block ──────► DN-3   (replica 3)

  Overhead: 3×. Fast reads (any replica), simple recovery.


Erasure Coding (Reed-Solomon 6+3) — 1 GB file stored as 1.5 GB

  Data shards:   [D1][D2][D3][D4][D5][D6]
  Parity shards: [P1][P2][P3]   (calculated from data shards)

  Any 6 of 9 shards reconstruct the file.
  Overhead: 1.5×. More compute to encode/decode; slower degraded reads.
```

HDFS added erasure coding in Hadoop 3.0 — recommended for cold/archival data where storage
efficiency matters more than read speed. S3 uses erasure coding internally (inferred).

### The Metadata Bottleneck and Solutions

```
NameNode Scaling Solutions (in order of complexity)

  Vertical scaling    → give NameNode more RAM              (short-term fix)
  HA / Standby NN     → ZooKeeper failover, shared journal  (availability fix)
  HDFS Federation     → multiple NameNodes, namespace shards (scalability fix)
  Object store (S3)   → no single metadata server; sharded  (architecture fix)
```

### Consistency Models Compared

| System | Consistency | Notes |
|--------|-------------|-------|
| GFS | Relaxed (defined / undefined regions) | Concurrent appends may duplicate records |
| HDFS | Strong per-file (WORM model) | Write-once; no concurrent writers to a file |
| S3 pre-2020 | Eventual consistency | Stale reads possible after PUT / DELETE |
| S3 post-2020 | Strong read-after-write | All operations immediately consistent |

---

## HDFS vs S3 — When to Use Which

| Factor | HDFS | S3 |
|--------|------|----|
| Access pattern | Sequential, large reads | Random access via HTTP |
| Latency | Low (co-located compute) | Higher (network round-trip to AWS) |
| Consistency | Strong per-file | Strong (post-2020) |
| Ops burden | You manage the cluster | Fully managed |
| Cost model | Fixed (hardware you own) | Pay per GB + requests |
| Append support | Yes | No (immutable objects) |
| File system semantics | Yes | No (flat key-value) |
| Typical use | Hadoop/Spark on-prem, legacy | Cloud-native data lake, backups, serving |

**The industry shift since ~2015**: HDFS was the default big-data storage layer. As S3 gained
strong consistency and Spark/Databricks/Snowflake natively target S3, most new workloads use
S3 even for large-scale batch jobs. HDFS is increasingly on-prem legacy.

---

## How to Remember This

**SRC — Split, Replicate, Coordinate**

```
  Split      → chunk files so data is parallel and manageable
  Replicate  → 3 copies so one node failure does not lose data
  Coordinate → master / NameNode / metadata service knows where everything lives
```

**Personality mnemonics for the three systems**

- **GFS** = the *blueprint* — single master, relaxed consistency, built for MapReduce at Google
- **HDFS** = GFS's *open-source Java twin* — same architecture, stricter consistency, powers Hadoop
- **S3** = the *managed rebel* — no file system, HTTP PUT/GET, immutable objects, AWS handles everything

**HDFS write path: Ask → Pipeline → Commit**

Ask the NameNode for a DataNode pipeline, stream data through the pipeline, commit completion
back to the NameNode.

**Secondary NameNode trap**: "secondary" sounds like "backup" — it is not. It is a checkpoint
helper. The standby NameNode (in HA mode) is the actual backup.

---

## Interview Cheat Sheet

**Signal Phrases**

- "The NameNode is purely in the control plane — clients go directly to DataNodes for data
  transfer. The NameNode is never in the data path."
- "Pipeline replication means the client writes to DN-1 and DN-1 forwards to DN-2 simultaneously.
  This bounds write latency — you don't wait for all replicas before the next packet."
- "S3 objects are immutable. You cannot append or partially update. For Spark this means writing
  complete output files — and since S3 has no atomic rename, you use staging prefixes or
  S3-compatible atomic commit protocols."
- "HDFS Federation shards the namespace across multiple independent NameNodes, solving the
  NameNode heap ceiling."
- "S3 gained strong read-after-write consistency in December 2020. Before that you had to
  design around eventual consistency."

**Red Flags**

- Calling the Secondary NameNode a standby — it is a checkpoint process, not HA
- Not distinguishing object storage semantics from file system semantics
- Saying S3 is eventually consistent without noting the 2020 change
- Treating erasure coding as strictly better — it trades compute and degraded-read latency for
  storage efficiency; replication still wins for hot data

**Common Interview Probes**

- *"Design Google Drive / Dropbox"* → chunking strategy, multipart upload, metadata DB for
  file→chunk mapping, deduplication, conflict resolution
- *"Design YouTube storage"* → S3/blob store for video files, CDN for edge delivery,
  transcoding pipeline
- *"How does HDFS handle a DataNode failure?"* → heartbeat timeout → under-replication
  detected → re-replication scheduled to maintain factor
- *"Why doesn't HDFS scale to billions of small files?"* → 150 bytes of NameNode heap per
  file/block; 1 billion files ≈ 150 GB heap
- *"Design a distributed object store from scratch"* → consistent hashing for shard
  assignment, separate metadata and data services, checksums for integrity, replication
  factor, compaction/GC for deleted objects

---

## References

- [The Google File System (2003 paper)](https://research.google/pubs/pub51/)
- [HDFS Architecture Guide](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html)
- [Amazon S3 Strong Consistency (AWS Blog, 2020)](https://aws.amazon.com/blogs/aws/amazon-s3-update-strong-read-after-write-consistency/)
- [HDFS Erasure Coding](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HDFSErasureCoding.html)
- Designing Data-Intensive Applications — Martin Kleppmann, Ch. 3 (Storage Engines)
- System Design Interview Vol. 2 — Alex Xu, Ch. 8 (Distributed File Store)