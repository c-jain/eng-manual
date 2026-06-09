---
Status: 🌳 Evergreen
Created: 2026-06-09
Last Updated: 2026-06-09
---

# Relational Databases — Indexes, Transactions, ACID

## Table of Contents

1. [Why This Matters](#why-this-matters)
2. [Indexes](#indexes)
   - [Why Indexes Exist](#why-indexes-exist)
   - [B-Tree Internals](#b-tree-internals)
   - [Hash Index](#hash-index)
   - [Index Variants](#index-variants)
   - [Index Trade-offs](#index-trade-offs)
3. [Transactions](#transactions)
4. [ACID](#acid)
   - [Atomicity](#atomicity)
   - [Consistency](#consistency)
   - [Isolation](#isolation)
   - [Durability](#durability)
5. [Isolation Levels and Concurrency Anomalies](#isolation-levels-and-concurrency-anomalies)
   - [The Five Anomalies](#the-five-anomalies)
   - [Isolation Level Comparison](#isolation-level-comparison)
   - [Implementation Approaches](#implementation-approaches)
6. [ACID vs BASE](#acid-vs-base)
7. [Interview Cheat Sheet](#interview-cheat-sheet)
8. [Mnemonics](#mnemonics)
9. [References](#references)

---

## Why This Matters

Relational databases underpin most production systems. In system design interviews, knowing *why* a database behaves a certain way under concurrent load or partial failure separates surface-level answers from architectural ones. This file covers indexes (how reads become fast), transactions (how multi-step writes stay safe), and ACID (the four guarantees that make it all reliable).

---

## Indexes

### Why Indexes Exist

A table is stored as a **heap file** — an unordered collection of disk pages, each holding multiple rows. Without an index, every query performs a **full table scan**: every page is read sequentially until matching rows are found. That is O(N) disk reads, catastrophic at scale.

An index is a **separate data structure** that stores a copy of one or more column values in a form that enables fast lookup, with pointers back to the heap rows. The name is literal — like the index at the back of a book that maps a term to a page number, a database index maps a column value to a row location.

**Problems indexes introduce:**
- Every `INSERT`, `UPDATE`, `DELETE` must update every index on the table — write amplification
- Too many indexes on a write-heavy table can be worse than no indexes
- Indexes consume disk space and buffer pool memory
- The query planner can choose the wrong index, or ignore a useful one
- Index bloat accumulates on high-churn tables (B-Tree pages with deleted entries are not reused immediately)

---

### B-Tree Internals

The **B-Tree** (Balanced Tree) is the default index structure in PostgreSQL, MySQL, SQLite, and most relational databases. Every `CREATE INDEX` produces a B-Tree unless you specify otherwise.

**Why not a Binary Search Tree?**
A BST node holds one key and two child pointers, wasting nearly an entire disk page (8 KB) per read. A B-Tree node is sized to fill one disk page, packing hundreds of keys and child pointers. This gives a branching factor of 100–500, keeping the tree extremely shallow. A B-Tree over 100 million rows is typically 3–4 levels deep — 3–4 disk reads to reach any row.

```
B-Tree Index on users.age

              [ 50 ]                       <- Root node
             /      \
         [20]        [80]                 <- Internal nodes
        /    \       /    \
    [8,18]  [35] [65]  [82,91]           <- Leaf nodes

  Each leaf entry holds: (key value, pointer → heap row)
  Leaf nodes are doubly linked left ↔ right for efficient range scans.
```

> The "B" in B-Tree is historically unclear. The inventor Rudolf Bayer never clarified. Common attributions: **B**alanced, **B**ayer, **B**oeing (where it was developed). Balanced is the most useful mental model.

**Supported operations:**

| Operation | Complexity | Notes |
|-----------|------------|-------|
| Point lookup | O(log N) | Root-to-leaf traversal |
| Range scan | O(log N + K) | K = number of matching rows |
| Insert | O(log N) | May trigger node splits up the tree |
| Delete | O(log N) | May trigger rebalancing |

**B-Tree supports:** `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `ORDER BY`, `LIKE 'prefix%'`

**B-Tree does NOT help:** `LIKE '%suffix'`, `LIKE '%middle%'` — leading wildcards discard the sorted key order entirely.

---

### Hash Index

A hash index applies a hash function to the column value and maps it to a bucket containing row pointers.

- **Lookup:** O(1) — compute hash, find bucket, follow pointer
- **Only for equality:** `WHERE id = 123`
- **Cannot do:** range queries, ordering, prefix matching
- PostgreSQL: `CREATE INDEX ... USING HASH`
- MySQL InnoDB: builds **adaptive hash indexes** in memory automatically, layered atop the B-Tree

> **Rule of thumb:** B-Tree does **Both** (equality and range). Hash is **Half** (equality only). Default to B-Tree.

---

### Index Variants

**Composite Index**

An index on multiple columns: `CREATE INDEX ON orders (user_id, created_at)`.

The **left-prefix rule** governs usability. This index accelerates queries filtering on `user_id` alone, or `user_id + created_at` together. It cannot accelerate a query filtering on `created_at` alone — without constraining `user_id`, the entire index must be scanned.

> **Phone book model:** Sorted by last name then first name. You can jump to "Smith" (last name alone) or "Smith, John" (both). You cannot jump to all "Johns" without reading every page.

Column ordering guidelines:
1. Equality-filter columns first
2. Range-filter columns last
3. Among equality columns, put higher-selectivity columns first

**Covering Index**

An index that contains all columns a query needs. The database answers the query entirely from the index, never fetching the heap row. PostgreSQL calls this an **Index Only Scan**.

```sql
-- Query:
SELECT name FROM users WHERE email = 'a@b.com';

-- Covering index:
CREATE INDEX ON users (email, name);
-- email lookup + name retrieval happen entirely within the index — no heap access.
```

Covering indexes are among the highest-leverage read performance optimisations.

**Partial Index**

An index on a subset of rows satisfying a `WHERE` condition.

```sql
CREATE INDEX ON orders (user_id) WHERE status = 'pending';
```

Much smaller and faster to maintain. Only helps queries that also filter on that condition.

**Clustered vs Non-Clustered**

| | Clustered | Non-Clustered (Secondary) |
|-|-----------|--------------------------|
| Data layout | Heap rows ARE the leaf nodes — table stored in key order | Separate structure, points to heap |
| Count per table | One only (you can only physically sort data one way) | Many |
| Lookup path | Single traversal | Index → primary key → clustered index → row (in InnoDB: two traversals) |
| PostgreSQL | `CLUSTER` command re-orders heap once; not maintained | All indexes |
| MySQL InnoDB | Primary key always | All non-primary indexes |

> In InnoDB, secondary index leaf nodes store the **primary key value**, not a direct heap pointer. A secondary index lookup requires: secondary B-Tree → primary key value → primary B-Tree → row. Two B-Tree traversals.

---

### Index Trade-offs

| Index | Best For | Avoid When |
|-------|----------|------------|
| B-Tree | Range + equality, ORDER BY, general use | — (nearly always the right default) |
| Hash | Exact equality only, very high cardinality | Any range or sort needed |
| Composite | Multi-column WHERE, covering queries | Column order is wrong; left-prefix not satisfied |
| Partial | Sparse condition (status='active') | Condition not always in query |
| Clustered | Primary key range scans, locality | Table has no natural clustering key |

**When to NOT add an index:**
- Low-cardinality columns (boolean, status with 3 values) — the planner may prefer a full scan
- Very small tables — full scan is faster than B-Tree traversal overhead
- Write-heavy tables already burdened with many indexes
- Columns only used in leading-wildcard `LIKE` patterns

---

## Transactions

A **transaction** is a unit of work treated as a single, indivisible operation. It either fully commits (all changes become visible) or fully rolls back (no changes take effect).

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1; -- debit
  UPDATE accounts SET balance = balance + 100 WHERE id = 2; -- credit
COMMIT;
```

Without a transaction, a crash between the two `UPDATE` statements leaves money subtracted from account 1 but never added to account 2. The transaction guarantees both happen or neither does.

> **The name:** From financial banking — a discrete, complete exchange. Databases borrowed it to mean any discrete, all-or-nothing unit of work.

---

## ACID

ACID is the set of four properties that define what a reliable database transaction means.

### Atomicity

**"All or nothing."** Every operation in a transaction either executes or is undone. There is no observable partial state.

**Implementation — Undo Logs:**
Before modifying data, the database writes the *old value* to an undo log. If the transaction rolls back or the system crashes mid-transaction, the undo log is replayed to restore original values.

```
Undo Log — Crash Recovery

  BEGIN
  UPDATE accounts SET balance=450 WHERE id=1
    Undo log entry: "accounts id=1, old=500"   <- written before the change

  -- crash here --

  Recovery:
    Read undo log → SET balance=500 WHERE id=1  <- original value restored
```

### Consistency

**"Constraints are always preserved."** Every transaction moves the database from one valid state to another. Valid means all defined schema constraints — foreign keys, `CHECK` constraints, `NOT NULL`, uniqueness — hold both before and after.

The database enforces its schema constraints. Application-level business invariants (e.g., "total inventory cannot be negative") are the application's responsibility.

> **Critical distinction:** In the **CAP theorem**, "C" means **linearizability** (every read reflects the most recent write). In **ACID**, "C" means **constraint preservation** (schema and application invariants hold). These are entirely different. Conflating them is a common interview error.

### Isolation

**"Concurrent transactions do not interfere."** The result of running N transactions concurrently is identical to running them serially, one at a time. This is the most nuanced ACID property — see [Isolation Levels and Concurrency Anomalies](#isolation-levels-and-concurrency-anomalies).

### Durability

**"Committed data survives crashes."** Once `COMMIT` returns to the client, the data is permanently on disk. A power cut immediately after cannot undo it.

**Implementation — Write-Ahead Log (WAL):**

The rule: *write the log before writing the data page*. The WAL is a sequential, append-only file on disk. Changes are written and `fsync`'d to the WAL before the commit is acknowledged. Data pages are flushed to disk lazily in the background (checkpointing).

```
WAL Commit Path

  1. BEGIN TXN 42
  2. Write to WAL: "TXN 42: SET accounts.balance=450 WHERE id=1"
  3. fsync WAL to disk              <- durability is guaranteed from here
  4. Modify data page in buffer pool (memory)
  5. COMMIT
  6. Write "TXN 42: COMMITTED" to WAL, fsync

  Crash before step 3: WAL incomplete → recovery ignores TXN 42 (no visible change)
  Crash after step 3:  WAL replayed on recovery → committed data restored
```

**Why WAL is fast:** The WAL is sequential writes, which are orders of magnitude faster than random writes to scattered data pages. Data pages are flushed lazily; the WAL is the authoritative record for crash recovery.

> **fsync matters.** Disabling `fsync` lets the OS buffer WAL writes in memory. A kernel panic or power loss before the OS flushes loses committed data. This caused real data-loss incidents in production PostgreSQL deployments.

---

## Isolation Levels and Concurrency Anomalies

### The Five Anomalies

**Dirty Read**
Txn A reads data written by Txn B, which has not yet committed. If B rolls back, A read data that never officially existed.

**Non-Repeatable Read**
Txn A reads row X. Txn B updates and commits row X. Txn A reads row X again within the same transaction and gets a different value. "Repeatable" means: the same query within one transaction always returns the same result.

**Phantom Read**
Txn A runs `SELECT * FROM orders WHERE amount > 100`. Txn B inserts a new row with `amount=200` and commits. Txn A reruns the same range query — a new "phantom" row appears. It's a row that didn't exist at the start of A's transaction.

**Lost Update**
Txn A and Txn B both read `counter=10`. Both compute `10+1=11`. Both write 11. One write overwrites the other silently. The counter should be 12.

**Write Skew** (most subtle — the reason Serializable exists)
Txn A and Txn B both read a shared condition. Each makes a valid independent decision and writes to *different* rows. The combination of writes violates an invariant that neither transaction individually violated.

Classic example: two doctors on-call. Both read "2 doctors on-call." Both decide they can go off-call (invariant: ≥1 must remain). Both write "off-call" to their own row. Result: 0 doctors on-call — not caught by Repeatable Read, since neither transaction re-read the other's row.

---

### Isolation Level Comparison

| Isolation Level | Dirty Read | Lost Update | Non-Repeatable Read | Phantom Read | Write Skew |
|-----------------|-----------|------------|---------------------|--------------|------------|
| Read Uncommitted | possible | possible | possible | possible | possible |
| Read Committed | prevented | possible | possible | possible | possible |
| Repeatable Read | prevented | prevented (2PL) / possible (some MVCC) † | prevented | possible | possible |
| Snapshot Isolation ‡ | prevented | prevented | prevented | prevented | possible |
| Serializable | prevented | prevented | prevented | prevented | prevented |

† **Lost update at Repeatable Read is implementation-dependent.** Under 2PL, every read row is held under a shared lock until the transaction ends. When both transactions attempt to upgrade to an exclusive lock for writing, a deadlock is detected and one is aborted — so lost updates cannot occur silently. Under some MVCC implementations that only snapshot or lock the rows actually read (without applying first-committer-wins to write conflicts), two concurrent read-modify-write cycles on the same row can both succeed, losing one update. This is the gap that Snapshot Isolation closes.

‡ **Snapshot Isolation is not a standard SQL level** but sits between Repeatable Read and Serializable in strength. The entire database is snapshotted at transaction start, and write conflicts use first-committer-wins: if two transactions both modify the same row, the second to commit is aborted. PostgreSQL's `REPEATABLE READ` and MySQL InnoDB's `REPEATABLE READ` both implement Snapshot Isolation — stronger than what the SQL standard requires of that level (which only mandates preventing dirty reads and non-repeatable reads). The one anomaly SI does not prevent is write skew, which is why Serializable still exists.

> **PostgreSQL default:** Read Committed
> **MySQL InnoDB default:** Repeatable Read (implemented as Snapshot Isolation)

The higher the isolation level, the stronger the guarantee — and the more lock contention (or version overhead with MVCC) is incurred.

---

### Implementation Approaches

**Lock-Based — Two-Phase Locking (2PL)**

- **Shared lock (S):** Acquired for reads. Multiple transactions hold S locks concurrently.
- **Exclusive lock (X):** Acquired for writes. Blocks both S and X from other transactions.
- **Growing phase:** Transaction only acquires locks.
- **Shrinking phase:** Transaction only releases locks. Once it releases one lock, it may acquire no more.
- Deadlocks are possible. The DB detects cycles in the waits-for graph and aborts one transaction (the "victim").
- Readers block writers; writers block readers.

**MVCC — Multi-Version Concurrency Control (PostgreSQL, MySQL InnoDB)**

Core insight: **writers create new versions; readers read old versions from their snapshot.** Reads and writes do not block each other.

```
MVCC — Row Versions

  Row: accounts WHERE id=1

  Version A: { xmin=50,  xmax=100, balance=500 }  <- deleted by txn 100
  Version B: { xmin=100, xmax=NULL, balance=450 }  <- current

  Txn 80  (snapshot before txn 100 committed): sees Version A — balance=500
  Txn 150 (snapshot after  txn 100 committed): sees Version B — balance=450

  Readers and writers do not block each other.
```

- `xmin`: Transaction ID that created this version.
- `xmax`: Transaction ID that deleted/replaced it (`NULL` = version is still live).
- A transaction's snapshot sees a row version if `xmin` committed before the snapshot timestamp AND `xmax` is `NULL` or committed after.

**Dead tuple cleanup:** Superseded row versions (dead tuples) accumulate. PostgreSQL's `VACUUM` process reclaims their space. Neglecting `VACUUM` causes table bloat and, on very long-running write workloads, transaction ID wraparound — a critical failure mode.

**Serializable Snapshot Isolation (SSI)** — PostgreSQL's `SERIALIZABLE` level uses SSI: transactions run using MVCC snapshots but track read/write dependencies. At commit time, if a cycle is detected in the dependency graph (indicating a serialisation anomaly), one transaction is aborted. This catches write skew without requiring lock escalation for most workloads.

---

## ACID vs BASE

When horizontal scale and partition tolerance outweigh strict consistency (e.g., distributed NoSQL), databases relax ACID in favour of **BASE**:

| | ACID | BASE |
|-|------|------|
| Stands for | Atomicity, Consistency, Isolation, Durability | Basically Available, Soft state, Eventually consistent |
| Consistency | Strong, synchronous | Eventual |
| Availability | May be reduced (blocked transactions) | Maximised |
| Typical systems | PostgreSQL, MySQL, Oracle | Cassandra, DynamoDB, Riak |
| CAP alignment | CP | AP |

ACID is not strictly better than BASE — they serve different requirements. High-throughput, globally distributed systems often must choose BASE to remain available across network partitions.

---

## Interview Cheat Sheet

**Signal Phrases (Demonstrate Depth)**
- "PostgreSQL defaults to Read Committed, so non-repeatable reads are possible within a transaction — is that acceptable here, or do we need Repeatable Read?"
- "I'd use a composite index on `(user_id, created_at)` — the left-prefix rule means queries filtering on `user_id` alone still use it."
- "A covering index on `(email, name)` avoids the heap fetch entirely — PostgreSQL calls it an Index Only Scan."
- "MVCC means readers don't block writers in PostgreSQL, which is why read-heavy workloads scale well without lock contention."
- "WAL writes are sequential, which is why durability doesn't kill write throughput — data pages are flushed lazily."
- "Write skew is the gap between Snapshot Isolation and Serializable — two individually valid transactions whose combined writes violate an invariant that neither violated alone."
- "PostgreSQL's Repeatable Read is actually Snapshot Isolation — it prevents phantom reads even though the SQL standard doesn't require it at that level."
- "Lost update under MVCC is prevented by first-committer-wins: the second transaction to commit an update to the same row gets aborted."
- "In InnoDB, a secondary index lookup requires two B-Tree traversals — secondary index finds the primary key, then the primary key finds the row."

**Red Flags to Avoid**
- Indexing every column "to speed up queries" — this kills write performance
- Confusing CAP's "C" (linearizability) with ACID's "C" (constraint preservation)
- Saying "transactions are slow" without understanding why (fsync, lock contention, not transactions themselves)
- Not knowing your database's default isolation level
- Assuming MVCC means "no conflicts ever" — write-write conflicts still occur and are detected at commit time under SSI

**Common Interview Probes**
- "Walk me through what happens when a bank transfer crashes halfway."
- "Your query is slow on a 10M-row table. Where do you start?"
- "What is the difference between a clustered and non-clustered index?"
- "Explain a phantom read and how to prevent it."
- "Why might you choose Read Committed over Serializable?"
- "What is write skew? Give an example."

---

## Mnemonics

**ACID Properties**
- **A**tomic → Indivisible, like an atom — all or nothing
- **C**onsistent → Constraints preserved — DB never violates its own rules
- **I**solated → Invisible to others until committed
- **D**urable → Disk-persisted after commit (WAL + fsync)

> "A transaction's **A**ctions **C**annot **I**ntrude on others, **D**isk-saved forever."

**ACID's C vs CAP's C**
- ACID-C: **C**onstraints (foreign keys, CHECK, NOT NULL hold)
- CAP-C: **C**urrent (all reads see the latest write — linearizability)

**B-Tree vs Hash**
- B-Tree does **Both** (equality and range)
- Hash is **Half** (equality only, no range)

**Isolation Anomalies — Severity Ladder**
Dirty → Lost Update → Non-Repeatable → Phantom → Write Skew (each requires a stronger isolation guarantee to prevent)

**Composite Index — The Phone Book**
Sorted by last name, then first name. You can find "Smith" (A alone) or "Smith, John" (A+B). You cannot jump to all "Johns" without scanning the whole book.

---

## References

- *Designing Data-Intensive Applications* — Martin Kleppmann, Ch. 3 (Storage & Retrieval), Ch. 7 (Transactions)
- PostgreSQL Documentation — [Indexes](https://www.postgresql.org/docs/current/indexes.html)
- PostgreSQL Documentation — [MVCC](https://www.postgresql.org/docs/current/mvcc.html)
- PostgreSQL Documentation — [WAL](https://www.postgresql.org/docs/current/wal-intro.html)
- *Use The Index, Luke* — https://use-the-index-luke.com (free; excellent B-Tree internals)
- *High Performance MySQL* — Schwartz et al., Ch. 5 (Indexing for High Performance)