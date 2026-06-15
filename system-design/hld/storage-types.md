---
Status: 🌳 Evergreen
Created: 2026-06-15
Last Updated: 2026-06-15
---

# Blob Storage vs Object Storage vs Block Storage

## Table of Contents

- [The Core Confusion](#the-core-confusion)
- [Block Storage](#block-storage)
- [Object Storage](#object-storage)
- [Blob Storage](#blob-storage)
- [Bonus: Where Does File Storage Fit?](#bonus-where-does-file-storage-fit)
- [Side-by-Side Comparison](#side-by-side-comparison)
- [How to Remember This](#how-to-remember-this)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

## The Core Confusion

Three terms get thrown around loosely — and often interchangeably — in interviews and job descriptions:

- Block Storage
- Object Storage
- Blob Storage

The headline fact that resolves most of the confusion: **Object Storage and Blob Storage are (almost) the same thing.** "Object storage" is the generic industry term (AWS S3, Google Cloud Storage). "Blob Storage" is Microsoft Azure's product name for its equivalent service. Block Storage is the genuinely different one — a completely different abstraction, sitting much closer to the hardware.

The trap: "Block" and "Blob" sound nearly identical but sit at opposite ends of the storage spectrum. Mixing them up in an interview is a real signal of shallow understanding.

## Block Storage

### What It Is and Why It Exists

Block storage exposes storage as a set of fixed-size, independently addressable chunks called blocks — historically 512 bytes, commonly 4KB today. It's presented to an operating system or hypervisor as a raw device: no files, no folders, just a flat sequence of addressable blocks. The OS layers a filesystem (ext4, xfs, NTFS) on top, and the filesystem is what actually understands "files" and "directories."

This is literally how physical disks have always worked — platters and SSD cells are organised into sectors/blocks at the hardware level. Cloud providers built network block storage (AWS EBS, GCP Persistent Disk, Azure Managed Disks) to take that same raw-disk abstraction and detach it from any single physical machine. The "disk" lives on the network and can be attached to, detached from, and reattached to different VMs, persisting independently of any one VM's lifecycle.

### Why It's Called "Block"

The fundamental unit of storage and addressing is the block — a fixed-size chunk of bytes identified by a Logical Block Address (LBA). The storage layer has zero understanding of what's inside a block. It doesn't know about files, filenames, or structure — that's entirely the filesystem's job, layered on top. "Block" names exactly what the storage layer operates on, and nothing more.

### Internals

```
+----------------------------------------+
|               Application                 |
+----------------------------------------+
                    |
                    v
+----------------------------------------+
|     Filesystem (ext4 / xfs / NTFS)        |
|   maps file paths -> block addresses      |
+----------------------------------------+
                    |
                    v
+----------------------------------------+
|          Block Device / Volume            |
|   [Blk 0][Blk 1][Blk 2][Blk 3] ...         |
+----------------------------------------+
```

For network-attached block storage (EBS-style), the volume sits across the network rather than on local disk:

```
+--------------------------------+
|        Compute Instance           |
|  (App + Filesystem layers above)  |
+--------------------------------+
                |
                | block-level protocol
                | (iSCSI / NVMe over Fabrics)
                v
+--------------------------------+
|       Remote Block Volume          |
|   [Blk 0][Blk 1][Blk 2] ...         |
+--------------------------------+
```

Random reads and writes happen directly at the block level — the filesystem can update a single block in place. This is what gives block storage its low, predictable latency: it's the closest abstraction to "talking to a disk."

### Problems It Brings

- Needs a filesystem layered on top — raw blocks alone aren't directly useful to applications.
- Mostly single-attach — a volume is typically attached to one compute instance at a time. Multi-attach variants exist but need a cluster-aware filesystem to avoid corruption from concurrent writers.
- Doesn't scale horizontally on its own — a volume has a maximum size, so scaling beyond it means the application (e.g., a sharded database) has to manage multiple volumes itself.
- Durability and replication are the operator's responsibility — snapshots, cross-AZ replication, and backups must be explicitly configured; they aren't automatic the way they are with object storage.

## Object Storage

### What It Is and Why It Exists

Object storage manages data as objects, where each object is a bundle of three things: the raw data/payload, metadata (key-value attributes — both system-managed, like content type, and user-defined), and a unique key/identifier. Objects live in a flat namespace (a "bucket") and are accessed entirely through an HTTP(S) REST API — `GET`, `PUT`, `DELETE`, `HEAD`, `LIST`. There's no mounting, no block-level access, no filesystem semantics.

As unstructured data — images, video, logs, backups, ML datasets — exploded in scale, traditional filesystems and block devices hit real limits: directory-tree bottlenecks at huge file counts, single-machine capacity ceilings, and the operational burden of building your own replication and durability. Object storage was designed cloud-native from the start: virtually unlimited capacity, built-in multi-AZ replication or erasure coding for durability, and access from anywhere over plain HTTP — fully decoupling storage from any single compute instance.

### Why It's Called "Object"

Each stored item is managed as a self-contained object — not just bytes (like a block), and not just bytes-with-a-path (like a file in a filesystem), but data + metadata + identity, bundled together and managed as one indivisible unit by the storage system. The name draws a deliberate contrast with both "block" (no identity, no metadata) and "file" (identity via a path, but still owned by a filesystem that lets you open, seek, and append within it).

### Internals

```
                  Client
                    |
                    | HTTP: GET / PUT / DELETE / HEAD
                    v
+------------------------------------------------+
|              Object Storage Service               |
|                                                    |
|  Bucket: "user-uploads"                            |
|    Key "images/avatar-123.png"                     |
|      -> Object [ data | metadata | ETag/version ]  |
+------------------------------------------------+
                    |
                    v
   Object is split, replicated, and/or erasure-coded
   across many disks, racks, and availability zones
```

Internally, the bucket/key pair is hashed or partitioned to locate which storage nodes hold (or should hold) the object's data and replicas — the same kind of partitioning scheme that shows up in distributed databases and caches (see `consistent-hashing.md`). Durability comes from spreading copies, or erasure-coded fragments, across failure domains, so the loss of a disk, rack, or even an AZ doesn't lose the object.

### Problems It Brings

- Effective immutability — there's no in-place partial write. "Updating" an object means uploading a new version (or a whole new object). Multipart upload helps assemble large objects, but doesn't give random in-place edits.
- Higher, more variable latency — every operation is a network call through an HTTP API, with overhead well above a local block device. This makes object storage unsuitable as a database's primary storage engine.
- No POSIX semantics — no file locking, no random-access editing, no atomic directory rename. "Renaming a folder" means copying and deleting every object under that prefix.
- Consistency model matters — historically some object stores offered only eventual consistency for overwrites and deletes. Whatever guarantees the provider currently offers directly affect correctness for read-after-write patterns, so it's worth confirming rather than assuming.
- Cost has hidden axes — storage class/tier, per-request pricing (GETs and PUTs cost money), and egress bandwidth all factor in. A "store once, read constantly" pattern without a cache or CDN in front can get surprisingly expensive.

## Blob Storage

### What It Is and Why It's Called That

"Blob" stands for **B**inary **L**arge **OB**ject. In modern cloud vocabulary, Blob Storage is essentially a synonym for object storage — it's Microsoft Azure's product name for its object-storage service (Azure Blob Storage), conceptually equivalent to S3 or Google Cloud Storage.

The term predates the cloud. "BLOB" originated as a relational database column type for storing large binary data (images, documents, audio) directly in a row, for data that didn't fit naturally into typed columns like `INT` or `VARCHAR`. The name signalled that the database treats this data as opaque — it doesn't interpret or index the bytes, it just stores and returns them as a "blob." When Azure built its object-storage service, it reused this familiar database term as the product name, rather than adopting the generic "object storage" label AWS and GCP use.

### Azure's Three Blob Types

This is where Azure Blob Storage is actually broader than "just object storage," and it's a detail that shows depth in an interview:

- **Block Blobs** — composed of blocks uploaded independently and then committed as one object. This is the "object storage" use case: large files like images, videos, and backups. Roughly equivalent to an S3 object.
- **Append Blobs** — optimised for append-only writes (e.g., log files). New data can be appended without rewriting the whole blob, something Block Blobs and plain object storage generally can't do efficiently.
- **Page Blobs** — optimised for random read/write access in fixed-size, page-aligned chunks, up to large total sizes. These back Azure Managed Disks — i.e., Azure's *block storage* product is implemented on top of a special Page Blob inside the Blob Storage service.

The takeaway: "Blob Storage" as Azure uses the term is a broader umbrella than "object storage" — it includes a block-storage-like mode (Page Blobs) under the hood. For interview purposes, **Block Blobs ≈ object storage** is the mapping that matters most of the time.

## Bonus: Where Does File Storage Fit?

Cloud providers usually present three storage "pillars" — block, file, and object (AWS EFS, Azure Files, and GCP Filestore are the "file" leg).

File storage is network-attached storage exposed via standard file-sharing protocols (NFS/SMB), presenting a hierarchical directory tree that multiple clients can mount and access concurrently. It sits between the other two: richer semantics than object storage (a real hierarchy, POSIX file operations, multiple simultaneous client access) but not as low-latency or "raw" as block storage, and not as elastically scalable as object storage.

It exists because many applications and shared workloads — content management systems, shared home directories, build-artifact caches — expect a real shared filesystem with paths and POSIX semantics, something neither block storage (mostly single-attach) nor object storage (no real hierarchy, no POSIX) provides.

## Side-by-Side Comparison

| Aspect | Block Storage | Object Storage | Blob Storage (Azure) |
|---|---|---|---|
| Unit | Fixed-size block (LBA-addressed) | Object: data + metadata + key | Same as object storage; Page Blobs add block-like semantics |
| Access | Raw device, mounted via filesystem | HTTP REST API (GET/PUT/DELETE/HEAD) | Azure Blob REST API |
| Namespace | None — raw address space | Flat: bucket / key | Flat: container / blob name |
| Mutability | In-place random read/write | Effectively immutable — replace the object | Block/Append Blobs: append or replace; Page Blobs: random read/write |
| Latency | Very low, sub-millisecond | Higher — network + API overhead | Same profile as object storage |
| Attach model | Usually single VM; multi-attach needs a cluster filesystem | No "attach" — globally accessible over HTTP | Globally accessible over HTTP |
| Scaling | Per-volume size limit; app must shard beyond it | Virtually unlimited | Virtually unlimited |
| Typical use | Databases, VM boot disks, low-latency workloads | Static assets, backups, data lakes, media, logs | Same as object storage; Page Blobs back Azure VM disks |
| Rough equivalents | AWS EBS, GCP Persistent Disk | AWS S3, GCP Cloud Storage | Azure Blob Storage |

## How to Remember This

- **Block = LEGO bricks.** Tiny, uniform, identical, unlabelled pieces. Something else — the filesystem — has to assemble them into anything meaningful. Low-level, fast, dumb.
- **Object = a shipping package.** Contents (data) plus a shipping label with a tracking number and details (metadata + key) — self-describing, sent over a network (HTTP), and handled as one indivisible unit.
- **Blob = "Binary Large OBject," spelled out.** It's Azure's name for the same "package" idea, inherited from the old database column type for "big binary stuff we don't look inside."
- **The trap to remember:** "Block" and "Blob" sound similar but are nearly opposites. Block = raw, low-level, low-latency, usually single-attach. Blob/Object = high-level, API-driven, globally accessible, effectively immutable.

## Interview Cheat Sheet

### Signal Phrases

- "For a database or any low-latency random-access workload, I'd attach block storage — e.g., EBS — directly to the instance."
- "For user-uploaded media, static assets, or backups at scale, I'd use object storage — durable, virtually unlimited, and served through a CDN."
- "Object storage objects are effectively immutable — any 'edit' is really a new version or a full re-upload."

### Red Flags to Avoid

| Mistake | Correct Framing |
|---|---|
| Treating "Blob Storage" and "Block Storage" as the same thing | Blob ≈ Object (Azure's naming); Block is a different, lower-level tier |
| Proposing object storage as a database's primary storage engine | Object storage is too high-latency and effectively immutable for OLTP |
| Proposing block storage for "millions of profile photos accessible globally" | Block storage is tied to a single instance, not a globally accessible store |
| Ignoring request/egress costs for object storage | A hot, frequently-read object without a cache/CDN in front gets expensive |

### Common Interviewer Probes

| Probe | What They're Looking For |
|---|---|
| "How would you handle a partial update to a large object in S3?" | Recognise there's no true in-place write — re-upload, restructure into smaller objects, or move the mutable part to a DB/block-backed store |
| "What gives S3-style storage its durability guarantees?" | Replication and/or erasure coding of object data across multiple devices, racks, and availability zones |
| "Why can't you just mount object storage like a disk?" | No POSIX semantics, no random in-place writes, API-based access model rather than a block-device interface |
| "When would you pick block over object storage, and vice versa?" | Map access pattern (random low-latency vs. bulk/sequential, single-instance vs. globally accessible) to the right tier |

## References

- AWS — Amazon S3 and Amazon EBS documentation
- Microsoft Azure — Blob Storage documentation (Block, Append, and Page blob types; Managed Disks)
- Alex Xu — *System Design Interview*, Vol. 1 & 2 (storage chapters)
- Kleppmann — *Designing Data-Intensive Applications* (storage and distributed systems chapters)