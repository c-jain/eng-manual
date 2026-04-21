# CAP Theorem & PACELC Theorem

## Table of Contents

- [Why This Matters in Interviews](#why-this-matters-in-interviews)
- [CAP Theorem](#cap-theorem)
  - [The Three Properties](#the-three-properties)
  - [The Real Trade-off](#the-real-trade-off)
  - [System Classification](#system-classification)
  - [Real-World Examples](#real-world-examples)
  - [Common Misconceptions](#common-misconceptions)
- [PACELC Theorem](#pacelc-theorem)
  - [Why PACELC Was Needed](#why-pacelc-was-needed)
  - [The Four Dimensions](#the-four-dimensions)
  - [System Classification Under PACELC](#system-classification-under-pacelc)
- [CAP vs PACELC: Putting It Together](#cap-vs-pacelc-putting-it-together)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

---

## Why This Matters in Interviews

Every distributed system design question implicitly asks you to make a consistency vs. availability trade-off. CAP and PACELC give you the vocabulary and framework to make that trade-off explicit and defend it. When you say "I'll use Cassandra here," the interviewer expects you to know what consistency guarantees you're giving up and why that's acceptable for the use case.

---

## CAP Theorem

**Coined by:** Eric Brewer, 2000 (keynote). Formally proven by Gilbert and Lynch, 2002.

**What it says:** A distributed system can guarantee at most two of the following three properties simultaneously — Consistency, Availability, and Partition Tolerance.

**Why it's called CAP:** It's an acronym of the three properties — **C**onsistency, **A**vailability, **P**artition Tolerance.

**Why it exists:** As systems grew beyond a single machine, engineers needed a principled way to reason about the fundamental limits of distributed data systems. CAP made the trade-off explicit and formal.

### The Three Properties

**Consistency (C)**

Every read returns the most recent write, or an error. All nodes in the cluster reflect the same data at the same point in time. Think of it as a system-wide single source of truth — you'll never observe stale data.

- Not to be confused with the "C" in ACID (which is about invariants/integrity, not about distributed agreement).
- Also called "linearizability" or "strong consistency" in formal literature.

```
  Scenario 1 — replication completed before the read:
 
    Write x=5 ──▶ Node A ──(replicates)──▶ Node B (x=5)
    Read x    ──▶ Node B ──▶ Returns 5   ✓  (consistent)
 
  Scenario 2 — read arrives before replication completes:
 
    Write x=5 ──▶ Node A       Node B (still x=3, not yet synced)
    Read x    ──▶ Node B ──▶ Returns 3   ✗  (stale — inconsistent)
```

**Availability (A)**

Every request to a non-failing node receives a response (not necessarily the most recent write). The system is always operational; it never returns an error, even if it means serving potentially stale data.

- "Available" here means "no error, no timeout" — not "fast."
- A system returning a cached or outdated value is available.

**Partition Tolerance (P)**

The system continues to function even when the network drops or delays messages between nodes (a "network partition"). Nodes on different sides of the partition keep processing requests.

```
   Node A  ─── X ──────── Node B
                (partition)

   Both sides keep processing — that's partition tolerance.
```

### The Real Trade-off

Here's the key insight that most people miss:

**In any real distributed system, you cannot sacrifice P.** Network partitions are not hypothetical — hardware failures, routing issues, and data centre splits happen. A system that fails entirely when a partition occurs is not a useful distributed system.

So the real choice is:

```
  During a partition, you must choose:

  ┌────────────────────────────────────────────────────────────────┐
  │  CP: Sacrifice Availability                                    │
  │      Return an error or block until partition heals.           │
  │      Data is always consistent, but some reads/writes fail.    │
  │                                                                │
  │  AP: Sacrifice Consistency                                     │
  │      Return a response (possibly stale). Never error.          │
  │      Data may diverge across nodes; reconcile later.           │
  └────────────────────────────────────────────────────────────────┘
```

CA (no partition tolerance) is only achievable on a single node — not a distributed system. When people say "CA database" they mean a single-node RDBMS (Postgres, MySQL) where partitions simply don't occur.

### System Classification


```
  Each tier lives on the EDGE between two properties it satisfies.
  The vertex it doesn't touch is the property it sacrifices.
 
                    Consistency
                         *
                        / \
                       /   \
                      /     \
                    CA       CP
                    /         \
                   /           \
                  * ─────────── *
          Availability    Partition Tolerance
                      AP (bottom edge)
 
  CA — satisfies Consistency + Availability. No partition tolerance.
       Only possible on a single node (no network to partition).
 
  CP — satisfies Consistency + Partition Tolerance. Sacrifices Availability.
       Blocks or errors during a partition rather than serve stale data.
 
  AP — satisfies Availability + Partition Tolerance. Sacrifices Consistency.
       Always responds, but may return stale data during a partition.
```

| Tier | Meaning | Behaviour During Partition |
|------|---------|---------------------------|
| CP   | Consistent + Partition Tolerant | Returns error, blocks, or refuses write rather than serve stale data |
| AP   | Available + Partition Tolerant  | Returns potentially stale data, never errors |
| CA   | Consistent + Available (no partition) | Single-node only; partition causes full failure |

### Real-World Examples

**CP Systems — Prefer Correctness Over Uptime**

- **ZooKeeper / etcd:** Leader-based consensus (Zab / Raft). During partition, followers that can't reach the leader reject reads/writes. Used for distributed coordination, leader election — correctness is non-negotiable.
- **HBase:** Uses HDFS and ZooKeeper. Region servers that lose ZooKeeper contact stop serving. Data integrity is prioritised.
- **MongoDB (default write concern `majority`):** Will refuse writes if a majority quorum can't be reached.

**AP Systems — Prefer Uptime Over Correctness**

- **Cassandra:** Uses tunable consistency, but defaults to eventual consistency. During partition, all nodes keep accepting writes. Conflicts resolved via last-write-wins or vector clocks.
- **DynamoDB:** Highly available by design. Eventual consistency by default; strong consistency is opt-in per read but has higher latency.
- **CouchDB:** AP with MVCC and eventual sync. Designed for offline-first scenarios.
- **DNS:** Caches stale records across resolvers. Availability is everything.

**CA (Single-Node)**

- **PostgreSQL, MySQL (single instance):** No partitions possible. Both consistent and available — until you add replication, at which point you're back to CP/AP territory.

### Common Misconceptions

**"CAP means you choose two of three always."**
Wrong. P is always required in a distributed system. You only choose between C and A *when a partition occurs*.

**"Consistency in CAP = Consistency in ACID."**
Different concepts. CAP-C is about all nodes agreeing on the same value (linearizability). ACID-C is about constraints/invariants staying valid after a transaction.

**"AP systems have no consistency."**
Wrong. AP systems often offer *eventual consistency* — nodes converge to the same value once the partition heals. It's not zero consistency; it's weaker consistency.

---

## PACELC Theorem

**Proposed by:** Daniel Abadi, 2012.

**Why it was needed:** CAP only talks about behaviour during partitions. But partitions are rare. What about normal operation? There, every distributed system faces another trade-off: **latency vs. consistency**. Replicating a write synchronously to all nodes (consistency) takes time; returning immediately and replicating asynchronously (low latency) risks stale reads. CAP says nothing about this. PACELC fills the gap.

**Why it's called PACELC:**
- **PAC** = Partition → choose between Availability and Consistency (same as CAP)
- **ELC** = Else (no partition) → choose between Latency and Consistency

Read it as: "If there's a **P**artition, choose **A** or **C**; **E**lse, choose **L** or **C**."

### The Four Dimensions

```
  Is there a network partition?
         │
    YES ─┤                   NO ─┐
         │                       │
  Choose: A or C          Choose: L or C
  ┌──────────────┐         ┌──────────────────────┐
  │ A: Keep      │         │ L: Respond fast,     │
  │    responding│         │    replicate async   │
  │    (may be   │         │    (stale reads OK)  │
  │    stale)    │         │                      │
  │ C: Block or  │         │ C: Wait for all      │
  │    error     │         │    replicas to ack   │
  └──────────────┘         └──────────────────────┘
```

**Latency vs. Consistency in Normal Operation**

Even with no partition, replication takes time. If you synchronously wait for all replicas to confirm a write before returning to the client, you get strong consistency but higher latency. If you return immediately and replicate in the background, you get low latency but risk a node serving a slightly old value before it receives the update.

```
  Client ──▶ Leader ──▶ Replica 1 ──▶ Ack
                    ──▶ Replica 2 ──▶ Ack
                    ──▶ Replica 3 ──▶ Ack
                    ──▶ Client: Done  (high latency, strong consistency)

  Client ──▶ Leader ──▶ Client: Done  (low latency)
                    ──▶ Replica 1 (async)
                    ──▶ Replica 2 (async)   ← stale window
```

### System Classification Under PACELC

PACELC notation: **`PA/EL`** = Prioritise Availability during partitions, Latency during normal ops.

| System | PACELC | Reasoning |
|--------|--------|-----------|
| DynamoDB (default) | PA/EL | Stays up during partition; async replication for speed |
| Cassandra (default) | PA/EL | Tunable but defaults to AP + low latency |
| Cassandra (quorum reads/writes) | PA/EC | Still AP on partition, but can be configured for consistency at cost of latency |
| HBase | PC/EC | Blocks on partition; synchronous replication for consistency |
| ZooKeeper / etcd | PC/EC | Leader consensus; no speed shortcuts |
| MongoDB (`majority` write concern) | PC/EC | Won't ack write until majority confirms |
| CRDT-based systems (Riak) | PA/EL | Conflict-free merge; always available, always fast |
| Spanner (Google) | PC/EC | TrueTime for global consistency; pays latency for it |
| MySQL (async replication) | PA/EL | Primary returns fast; replicas lag |
| MySQL (semi-sync replication) | PC/EL | At least one replica acks before primary responds — middle ground |

---

## CAP vs PACELC: Putting It Together

```
  CAP:      Normal op ──────────────── Partition occurs
                  │                          │
                  ?  (CAP says nothing)    CP or AP?

  PACELC:   Normal op ──────────────── Partition occurs
                  │                          │
             LC or LL?                   AC or PC?
          (Latency/Consistency)       (Availability/Consistency)
```

**When to use CAP in an interview:** When talking about what happens under failure — does the system block or serve stale data?

**When to use PACELC in an interview:** When talking about replication strategy, read/write latency, and consistency levels in normal (non-failure) operation. This is where most real product decisions live.

**The combined mental model:**

```
  Design question: "Should I use Cassandra or ZooKeeper for X?"

  Step 1 — Partition behaviour (CAP):
    Does X require every read to be current? (CP → ZooKeeper)
    Or is it OK to serve slightly old data rather than fail? (AP → Cassandra)

  Step 2 — Normal operation (PACELC):
    Does X need the lowest possible write latency? (EL → async replication)
    Or does X need every read to see the most recent write? (EC → synchronous quorum)
```

---

## Interview Cheat Sheet

**"Which would you choose for a social media timeline?"**
AP/EL — It's acceptable to see a post a second late. Availability and speed matter far more than strict ordering.

**"Which for a bank ledger or financial transaction log?"**
CP/EC — Stale balance reads are unacceptable. You trade availability and latency for correctness.

**"Which for a distributed lock or leader election?"**
CP/EC — etcd or ZooKeeper. Correctness is the entire point. A split-brain (two leaders thinking they both hold the lock) is catastrophic.

**"Which for a shopping cart?"**
AP/EL — Amazon's Dynamo paper (2007) is the canonical reference here. Items lost from a cart are worse for UX than seeing a slightly stale cart, so they chose AP. Conflicts resolved at checkout.

**"How does Cassandra achieve tunable consistency?"**
Cassandra lets you set `QUORUM`, `ONE`, `ALL` per read/write. At `QUORUM` (majority of replicas must respond), you get strong consistency at the cost of latency. At `ONE`, you get lowest latency but may read stale. This is PACELC in action — you're trading L for C on the ELC side.

**"What is eventual consistency?"**
All replicas will converge to the same value, given no new writes and enough time for gossip/propagation. It's the consistency model of AP/EL systems. Not "no consistency" — just weaker than linearizability.

**"What problems does AP bring?"**
- Stale reads — clients may see old data.
- Conflicts — two nodes accept concurrent conflicting writes during partition; reconciliation needed (LWW, vector clocks, CRDTs).
- No global ordering — events may arrive at different nodes in different orders.

**"What problems does CP bring?"**
- Reduced availability — during a partition, the system may refuse requests.
- Latency — synchronous replication adds round-trip time.
- Complexity — consensus protocols (Raft, Paxos) are hard to implement correctly.

---

## References

- [Brewer's CAP Theorem (Original Keynote, 2000)](https://people.eecs.berkeley.edu/~brewer/cs262b-2004/PODC-keynote.pdf)
- [Gilbert & Lynch — Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services (2002)](https://dl.acm.org/doi/10.1145/564585.564601)
- [Daniel Abadi — Consistency Tradeoffs in Modern Distributed Database System Design: CAP is Only Part of the Story (2012)](https://www.cs.umd.edu/~abadi/papers/abadi-pacelc.pdf)
- [Amazon Dynamo Paper (2007)](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- [Martin Kleppmann — Designing Data-Intensive Applications, Chapter 9](https://dataintensive.net/)
- [AWS DynamoDB Consistency Models](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html)
- [Cassandra Tunable Consistency](https://docs.datastax.com/en/cassandra-oss/3.0/cassandra/dml/dmlConfigConsistency.html)