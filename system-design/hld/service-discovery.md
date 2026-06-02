# Service Discovery & Coordination — Zookeeper, etcd

## Table of Contents

- [Overview](#overview)
- [The Core Problem: Distributed Consensus](#the-core-problem-distributed-consensus)
- [Zookeeper](#zookeeper)
  - [Data Model — Znodes](#data-model--znodes)
  - [Znode Types](#znode-types)
  - [Watches](#watches)
  - [Session Semantics and Health Detection](#session-semantics-and-health-detection)
  - [Leader Election](#leader-election)
  - [Distributed Lock](#distributed-lock)
- [etcd](#etcd)
  - [Data Model — Flat Key-Value](#data-model--flat-key-value)
  - [Raft Consensus](#raft-consensus)
  - [Leases](#leases)
  - [Watch — Streaming and Prefix-Based](#watch--streaming-and-prefix-based)
  - [Go Code Example](#go-code-example)
- [Service Discovery Patterns](#service-discovery-patterns)
  - [Self-Registration](#self-registration)
  - [Third-Party Registration](#third-party-registration)
  - [Client-Side vs Server-Side Load Balancing](#client-side-vs-server-side-load-balancing)
- [Zookeeper vs etcd](#zookeeper-vs-etcd)
- [Consul — A Practical Alternative](#consul--a-practical-alternative)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

---

## Overview

**What it is:** Service discovery is the mechanism by which services in a distributed system locate live, healthy instances of one another. Coordination is the broader class of problems — leader election, distributed locking, configuration management, membership tracking — that require multiple distributed nodes to agree on shared state.

**Why it exists:** In a monolith, components call each other in-process; finding them is trivial. In a distributed system, service instances start, crash, restart, scale, and migrate across machines continuously. Hardcoding IP addresses is brittle. DNS TTLs are too slow. You need a dedicated system that tracks live instances in near-real-time and notifies dependents when things change.

**Why it is called what it is:** Zookeeper was built at Yahoo to manage the "zoo" of distributed systems they operated — HBase, Kafka, Hadoop, and others. The name reflects the idea of coordinating many animals. etcd is named for a directory (`/etc` in Unix) that stores configuration, distributed — hence `etcd`.

**The problem it brings:** Both systems are additional infrastructure to deploy, monitor, and operate. They are CP systems (see CAP theorem), meaning they sacrifice availability during network partitions to guarantee consistency. An operator must understand quorum, leader election, and data retention to run them safely.

---

## The Core Problem: Distributed Consensus

Consider five nodes that must agree on who the current leader is. If two nodes simultaneously believe they are the leader, you have a split-brain — both accept writes, data diverges, and recovery is painful.

Solving this requires a **consensus algorithm**: a protocol that guarantees only one value wins even when nodes can fail or messages can be delayed.

Both Zookeeper and etcd are built on consensus algorithms:

- Zookeeper uses **ZAB** (Zookeeper Atomic Broadcast), a Paxos variant developed at Yahoo
- etcd uses **Raft**, designed at Stanford as a deliberately more understandable alternative to Paxos

Raft decomposes the problem into three distinct sub-problems: leader election, log replication, and safety. This separation makes it easier to reason about, implement correctly, and debug — which is why etcd tends to have fewer operational surprises than Zookeeper.

Both systems require a quorum (strict majority) to make progress. Quorum = ⌊n/2⌋ + 1. With three nodes, quorum is two — the cluster tolerates one failure. Always deploy an odd number of nodes; four nodes gives the same fault tolerance as three (quorum = 3) at higher cost.

**Memory aid:** Think of Raft as "Paxos but explainable to a colleague in 20 minutes." ZAB came first; Raft came later, explicitly to solve Paxos's understandability problem.

---

## Zookeeper

### Data Model — Znodes

Zookeeper models shared state as a **hierarchical namespace**, structurally identical to a Unix filesystem. Nodes in this tree are called **znodes**. Each znode can store a small payload — up to 1MB, but in practice only a few kilobytes of metadata.

```
/
├── /services
│   ├── /services/auth
│   │   ├── /services/auth/instance-0001   →  "192.168.1.10:8080"
│   │   └── /services/auth/instance-0002   →  "192.168.1.11:8080"
│   └── /services/payment
│       └── /services/payment/instance-0001
└── /config
    └── /config/feature-flags              →  "{\"dark_mode\": true}"
```

Znodes are not designed to store application data — they store coordination metadata: addresses, versions, flags, and sequence numbers.

### Znode Types

Zookeeper has four types of znodes, and the combination of types is what makes most coordination patterns possible:

- **Persistent** — survives client disconnection; must be explicitly deleted
- **Ephemeral** — automatically deleted when the creating client's session ends (crash or timeout)
- **Sequential** — Zookeeper appends a monotonically increasing counter suffix to the node name on creation
- **Ephemeral Sequential** — both properties combined; this combination is the foundation of leader election and distributed locks

The key insight: **ephemeral znodes give you automatic failure detection for free**. When a service dies, its session expires, its ephemerals disappear, and watchers are notified — with no manual cleanup required.

### Watches

A client can set a **watch** on any znode. When that znode changes — created, deleted, data modified, or children changed — Zookeeper sends a one-time notification to the watching client.

The watch is intentionally one-shot. The client must re-register after each notification. This prevents thundering-herd scenarios where a single event floods every watcher. In practice, clients immediately re-register the watch inside their notification callback.

### Session Semantics and Health Detection

When a client connects, Zookeeper creates a **session** with a configurable timeout. The client sends periodic heartbeats to keep the session alive. If the client crashes or loses network connectivity, heartbeats stop, the session times out, and Zookeeper deletes all ephemeral znodes created in that session.

This is the fundamental health detection mechanism:
- Service starts → creates ephemeral znode → other services watch it → service dies → znode disappears → watchers notified

No health check polling. No TTL renewal. The session timeout acts as an implicit TTL.

### Leader Election

```
Step 1: all candidates create an ephemeral sequential znode under /election

  /election/
    candidate-0001   ← node A (created first)
    candidate-0002   ← node B
    candidate-0003   ← node C

Step 2: each node lists children and finds the lowest sequence number

  Node A holds the lowest → A is leader

Step 3: each non-leader watches only the znode IMMEDIATELY before it
  (not the current leader)

  C watches candidate-0002 (B)
  B watches candidate-0001 (A)

Step 4: if A dies, candidate-0001 is deleted

  B is notified (it was watching 0001)
  B takes leadership, C is implicitly notified next round
```

**Why watch the predecessor and not the leader directly?**
If every node watches the leader, when the leader dies, all N-1 nodes are simultaneously notified and simultaneously try to become leader — this is the **herd effect**. By watching only the predecessor, exactly one node is notified per failure. Leadership transfer is O(1) notifications instead of O(N).

### Distributed Lock

The distributed lock is structurally identical to leader election. A lock has one holder at a time: whichever node holds the lowest sequential znode under the lock path owns the lock. When it releases (or crashes), the next node in sequence is notified and acquires the lock.

---

## etcd

etcd is a distributed key-value store purpose-built for storing Kubernetes cluster state, but general-purpose by design. It is the backing store for all objects in a Kubernetes cluster — Pods, Services, ConfigMaps, Secrets, etc.

### Data Model — Flat Key-Value

Unlike Zookeeper's filesystem tree, etcd exposes a **flat key-value API**. Keys are arbitrary strings, typically structured with `/` separators to simulate hierarchy, and an entire prefix can be watched or listed.

```
/registry/services/auth/instance-1  →  "192.168.1.10:8080"
/registry/services/auth/instance-2  →  "192.168.1.11:8080"
/registry/config/feature-flags      →  "{\"dark_mode\": true}"
```

Every write in etcd receives a monotonically increasing **revision number** (a global logical clock). This revision can be used to replay history — you can ask for all events since revision 42 — which enables reliable catch-up after reconnection.

### Raft Consensus

```
etcd cluster — always odd number of nodes (3 or 5)

        [ Leader ]
            |
      ┌─────┴─────┐
  [ Follower 1 ]  [ Follower 2 ]

Write flow:
  1. Client sends write to any node
  2. Non-leader nodes forward to leader
  3. Leader appends to its log, sends AppendEntries RPC to followers
  4. Once quorum (n/2 + 1) acknowledge, write is committed
  5. Leader responds to client

Read options:
  - Linearised read  → goes through leader; always consistent
  - Serialisable read → served locally; may be stale
```

Quorum examples:
- 3 nodes → quorum = 2 → tolerates 1 failure
- 5 nodes → quorum = 3 → tolerates 2 failures
- 4 nodes → quorum = 3 → tolerates 1 failure (same as 3, but costs more)

Always use odd counts. Even counts do not improve fault tolerance.

### Leases

etcd's equivalent of Zookeeper's ephemeral znodes is **leases** — a TTL object that can be attached to one or more keys. When a client stops renewing its lease (because it crashed), the lease expires and all associated keys are automatically deleted.

Leases are explicit: you create one, attach keys to it, and renew it in a background goroutine. This is more transparent than Zookeeper's implicit session-based ephemerals, but the effect is the same.

### Watch — Streaming and Prefix-Based

etcd's watch API is more powerful than Zookeeper's:

- **Streaming:** watches are persistent gRPC streams — no re-registration required after each event
- **Prefix-based:** you can watch all keys under `/services/auth/` in a single call
- **Revision-based:** you can start a watch from a specific revision, enabling reliable catch-up after reconnection — no events are missed

### Go Code Example

```go
package main

import (
    "context"
    "fmt"
    "log"

    clientv3 "go.etcd.io/etcd/client/v3"
)

func registerService(ctx context.Context, cli *clientv3.Client) {
    // 1. Create a lease with a 30-second TTL
    lease, err := cli.Grant(ctx, 30)
    if err != nil {
        log.Fatal(err)
    }

    // 2. Register this service instance under its key, tied to the lease.
    //    When the lease expires (process dies), the key is automatically deleted.
    _, err = cli.Put(ctx, "/services/auth/instance-1", "192.168.1.10:8080",
        clientv3.WithLease(lease.ID))
    if err != nil {
        log.Fatal(err)
    }

    // 3. Keep the lease alive by sending renewals in the background.
    //    If this goroutine stops (process crash), lease expiry cascades to key deletion.
    keepAliveCh, err := cli.KeepAlive(ctx, lease.ID)
    if err != nil {
        log.Fatal(err)
    }

    go func() {
        for range keepAliveCh {
            // drain keepalive responses; non-nil response = still alive
        }
        // channel closed → lease expired
    }()
}

func watchServices(ctx context.Context, cli *clientv3.Client) {
    // Watch all keys under the /services/auth/ prefix — streaming, persistent
    watchCh := cli.Watch(ctx, "/services/auth/", clientv3.WithPrefix())

    for resp := range watchCh {
        for _, ev := range resp.Events {
            switch ev.Type {
            case clientv3.EventTypePut:
                fmt.Printf("instance registered: %s → %s\n", ev.Kv.Key, ev.Kv.Value)
            case clientv3.EventTypeDelete:
                fmt.Printf("instance deregistered: %s\n", ev.Kv.Key)
            }
        }
    }
}

func main() {
    cli, err := clientv3.New(clientv3.Config{
        Endpoints: []string{"localhost:2379", "localhost:2380", "localhost:2381"},
    })
    if err != nil {
        log.Fatal(err)
    }
    defer cli.Close()

    ctx := context.Background()
    go registerService(ctx, cli)
    watchServices(ctx, cli)
}
```

---

## Service Discovery Patterns

### Self-Registration

The service instance itself registers with the registry on startup and deregisters on shutdown. Crash deregistration is handled by lease/session expiry — no explicit deregister call is needed.

```
Service Instance                      Registry (etcd / ZK)
      │                                       │
      │── on startup: PUT key + lease ───────→│
      │── background: keepalive renewals ─────→│
      │                                       │
      │── (crash: renewals stop) ─────────────│ key deleted automatically
      │                                       │
                                              │
Client
      │── Watch /services/auth/* ────────────→│
      │←── streaming notifications ───────────│
      │── maintains local copy of instances   │
```

**Trade-off:** Simple to implement. Ties service code to the registry's client library. Works poorly if the service crashes before deregistering and the lease TTL is long — there's a window during which dead instances are still in the registry.

### Third-Party Registration

A sidecar process (or external agent) handles registration on the service's behalf. The service itself is registry-unaware.

```
  ┌──────────────────────────┐
  │  Service Pod             │
  │                          │
  │  ┌───────────┐           │
  │  │  App      │           │       ┌──────────────┐
  │  └───────────┘           │       │   Registry   │
  │                          │       │  (etcd / ZK) │
  │  ┌───────────┐           │       │              │
  │  │  Sidecar  │───────────┼──────→│              │
  │  └───────────┘           │       └──────────────┘
  └──────────────────────────┘
    registers on behalf of app
```

Kubernetes uses this pattern: kubelet registers pods; pods do not register themselves. Consul uses its agent per host for the same purpose.

**Trade-off:** Application code is decoupled from registry logic. Adds operational complexity (sidecar lifecycle, health checking). The sidecar itself can be a failure point.

### Client-Side vs Server-Side Load Balancing

Once a client has discovered a list of healthy instances, it must choose one to call.

**Client-side:** The client queries the registry directly, maintains a local list of instances, and performs load balancing (round-robin, least-connections) itself. Netflix's Ribbon and Eureka use this model.

- Pros: no central routing bottleneck; the client has full visibility into instance health
- Cons: every client needs registry-awareness; client logic is more complex

**Server-side:** The client calls a fixed endpoint (a load balancer). The load balancer queries the registry and routes.

- Pros: clients are simple; routing logic is centralised
- Cons: the load balancer is a potential bottleneck and single point of failure

Modern service meshes (Istio, Linkerd) blur this boundary: a sidecar proxy on each pod handles both discovery and client-side load balancing transparently.

---

## Zookeeper vs etcd

| Dimension | Zookeeper | etcd |
|---|---|---|
| Data model | Hierarchical znodes (filesystem-like) | Flat key-value with prefix simulation |
| Consensus algorithm | ZAB (Paxos variant, Yahoo) | Raft (Stanford, simpler to reason about) |
| Watch semantics | One-shot, node-level | Streaming, prefix-level, revision-aware |
| Ephemeral state mechanism | Ephemeral znodes tied to session | Leases with explicit TTL and renewal |
| Client API complexity | High — ZK client is complex, Java-first | Low — simple gRPC / HTTP API, Go-first |
| Kubernetes native | No | Yes — etcd is Kubernetes' backing store |
| Operational overhead | High | Medium |
| Typical write throughput | ~10k ops/sec | ~10k ops/sec |
| When you see it today | Kafka (legacy), HBase, older infra | Kubernetes, modern distributed systems |

**The practical answer for new systems:** Prefer etcd. Simpler client API, better watch semantics, Raft is more understandable than ZAB, and it integrates naturally with any Kubernetes-native infrastructure. Zookeeper still appears in Kafka clusters, though Kafka is migrating to its own built-in consensus (KRaft) and will eventually drop the Zookeeper dependency.

---

## Consul — A Practical Alternative

Consul is worth knowing as a third option. It bundles:

- Key-value store (like etcd)
- Service registry with health checking built in (HTTP, TCP, or script checks)
- DNS interface (services discoverable as `service-name.service.consul`)
- Service mesh capabilities (Consul Connect)

The advantage over etcd for non-Kubernetes environments: health checks are first-class citizens, not bolted on. Services are deregistered when health checks fail, not just when leases expire. The trade-off is more moving parts and a more opinionated architecture.

---

## Interview Cheat Sheet

**"How do services find each other in your design?"**

Name the registry explicitly: etcd, Consul, or a managed equivalent (AWS Cloud Map, GCP Service Directory). Distinguish discovery (finding instances) from routing (choosing among them). A load balancer alone is not discovery.

**"What happens when a service instance crashes?"**

Leases (etcd) or ephemeral znodes (Zookeeper) — automatic deregistration. No manual cleanup. The TTL is the only latency window. Watchers are notified and update their local view.

**"Why not just use a database as a service registry?"**

Databases lack push-based notification semantics. You would be forced to poll, which introduces propagation delay and unnecessary load. ZK/etcd push change events to watchers in near-real-time with minimal overhead.

**"Why deploy an odd number of nodes?"**

Quorum requires a strict majority. With four nodes, quorum is still three — same fault tolerance as three nodes, higher cost. Odd numbers maximise fault tolerance per node. Always use 3 or 5 in production.

**"Zookeeper vs etcd — which would you choose?"**

etcd for new systems. Simpler API, streaming watches, Raft is more understandable than ZAB, Kubernetes-native. Zookeeper if integrating with existing Kafka or HBase infrastructure where changing the coordination layer is impractical.

**"What consistency model does etcd provide?"**

Linearisability — every read reflects the most recent completed write, as if operations happened on a single node. This is the strongest consistency guarantee and is why etcd is suitable as a source of truth for cluster state.

**"What is the CAP trade-off here?"**

Both ZK and etcd are CP systems. They choose consistency over availability during partitions. This is the right choice for a service registry: a stale registry that routes traffic to dead instances (availability over consistency) causes worse failures than a registry that briefly refuses requests during a partition.

**Signal phrases to use:** "ephemeral nodes for automatic deregistration," "watch semantics for push-based notification," "quorum-based writes," "linearisable reads," "lease renewal as a heartbeat."

**Red flags to avoid:**
- Saying services find each other "by hostname" without explaining DNS-based or registry-based discovery
- Conflating service discovery with load balancing
- Not knowing why CP is the right choice for a registry
- Proposing polling a database instead of a watch-capable registry

---

## References

- [ZooKeeper: Distributed Process Coordination (O'Reilly book)](https://www.oreilly.com/library/view/zookeeper/9781449361297/)
- [etcd documentation — learner, raft, and data model](https://etcd.io/docs/)
- [Raft paper — In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf)
- [ZAB — A Simple Totally Ordered Broadcast Protocol (Yahoo Research)](https://distrinet.cs.kuleuven.be/events/ew/ew2007/programme/papers/s5p1.pdf)
- [Consul service discovery documentation](https://developer.hashicorp.com/consul/docs/concepts/service-discovery)
- [Kubernetes etcd usage — concepts/storage](https://kubernetes.io/docs/concepts/overview/components/#etcd)
- [etcd Go client v3 (official)](https://pkg.go.dev/go.etcd.io/etcd/client/v3)