# Consistent Hashing

## Table of Contents

- [What It Is and Why It Exists](#what-it-is-and-why-it-exists)
- [The Problem with Naive Hashing](#the-problem-with-naive-hashing)
- [The Hash Ring](#the-hash-ring)
- [Node Addition and Removal](#node-addition-and-removal)
- [Uneven Distribution and Virtual Nodes](#uneven-distribution-and-virtual-nodes)
- [Implementation: O(log N) Lookup](#implementation-olog-n-lookup)
- [Real-World Usage](#real-world-usage)
- [Trade-offs](#trade-offs)
- [How to Remember It](#how-to-remember-it)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

## What It Is and Why It Exists

Consistent hashing is a distributed hashing technique that minimises the number of keys remapped when the set of nodes changes. It solves a fundamental problem in distributed systems: how do you assign keys to nodes in a way that remains stable when the cluster grows or shrinks?

The name comes from its core guarantee — the mapping of keys to nodes is *consistent* through topology changes. Adding or removing a single node disturbs only ~1/N of keys, not the entire keyspace.

## The Problem with Naive Hashing

In a distributed cache or data store, a natural starting point is modulo hashing:

```
node = hash(key) % N     # N = number of nodes
```

This works correctly at a fixed size. The problem appears the moment N changes.

Add one node: N becomes N+1. The formula now produces almost entirely different results for every key. Approximately `(N-1)/N` of all keys remap to a new node — for N=4, that is ~80% of the entire keyspace. For a caching layer, this triggers a near-total miss rate spike on the next request wave. For a sharded database, it means a mass data migration.

The root cause: **modulo hashing couples every key's assignment to the total node count**. It is not resize-stable.

## The Hash Ring

Consistent hashing decouples key assignment from node count by introducing a **hash ring** (also called a token ring or continuum).

Take all possible hash output values — say 0 to 2³²−1 — and arrange them in a circle. Both nodes and keys are mapped onto this same space:

- Node placement: `position(node) = hash(node_name)`
- Key placement: `position(key) = hash(key)`

To find which node owns a given key: start at the key's position on the ring and walk **clockwise** until you reach a node. That node is the key's owner.

```
             0 / 2^32
                |
      [A] ------+------ [B]
     /                       \
   [D]                       [C]
     \                       /
      +---------------------+

hash(key1) lands between A and B  →  key1 owned by B
hash(key2) lands between C and D  →  key2 owned by D
```

## Node Addition and Removal

**Adding a node E between A and B:**

Only the keys in the A→B arc that now fall in the A→E sub-arc need to move to E. All other keys are unaffected. On average, ~1/N of keys move.

```
Scenario 1: Before adding E
  A→B arc owns: k1, k2, k3, k4

  [A] ----k1--k2--k3--k4---- [B]

Scenario 2: After adding E between A and B
  A→E arc owns: k1, k2  (remapped to E)
  E→B arc owns: k3, k4  (remain on B)

  [A] ----k1--k2---- [E] ----k3--k4---- [B]
```

**Removing a node B:**

Keys owned by B are those in the A→B arc — they walk clockwise and hit B first. When B is removed, those keys continue clockwise and land on C. C's responsibility expands to cover the full A→C arc.

```
Scenario 1: B exists
  A→B arc: k1, k2 owned by B
  B→C arc: k3, k4 owned by C

  [A] ---k1---k2--- [B] ---k3---k4--- [C]

Scenario 2: B removed
  k1, k2 walk clockwise past where B was → now owned by C
  k3, k4 were already owned by C → unchanged

  [A] ---k1---k2---k3---k4--- [C]
```

In both cases, every other key on the ring is completely undisturbed.

## Uneven Distribution and Virtual Nodes

With few physical nodes placed by hashing their names, random positioning can leave one node covering 50% of the ring while another covers 5%. That large-arc node becomes a hotspot — it receives disproportionately more traffic and stores more data.

**Virtual nodes (vnodes)** solve this. Each physical node is assigned multiple positions on the ring — typically 100–200 — by hashing variants of its identifier:

```
hash("server-A-1"), hash("server-A-2"), ..., hash("server-A-150")
```

All 150 positions belong to the same physical node A, but they spread A's coverage evenly across the entire ring.

```
Physical nodes: A, B, C — each with 3 vnodes (for illustration)

Ring: A1 → C2 → B1 → A2 → C1 → B2 → A3 → C3 → B3 → (wraps to A1)
```

Benefits:
- Distribution becomes statistically uniform as vnode count increases.
- When a node is removed, its load redistributes across *many* other nodes (one vnode at a time), not just a single successor. No one node absorbs all the rebalancing work.
- Heterogeneous capacity: assign more vnodes to nodes with more resources — they receive proportionally more keys.

## Implementation: O(log N) Lookup

All vnode positions are stored in a sorted array. To find the owner of a key:

1. Compute `pos = hash(key)`.
2. Binary search the sorted position array for the first entry ≥ `pos`.
3. If none found (pos > all entries), wrap around to index 0.
4. Return the physical node that vnode position maps to.

This gives O(log N) lookup time, where N is the total number of vnode positions (e.g., 150 vnodes × 100 nodes = 15,000 entries).

```go
import "sort"

type Ring struct {
    positions []int            // sorted list of all vnode hash positions
    nodeOf    map[int]string   // vnode position → physical node name
}

func (r *Ring) Get(key string) string {
    pos := hash(key)
    i := sort.SearchInts(r.positions, pos)
    if i == len(r.positions) {
        i = 0 // wrap around
    }
    return r.nodeOf[r.positions[i]]
}
```

Adding a node means inserting its vnode positions into `r.positions` (maintaining sort order) and populating `nodeOf`. Removing a node means deleting its entries from both.

## Real-World Usage

- **Apache Cassandra**: data is distributed across nodes using token ranges — consistent hashing positions. Vnodes are enabled by default and are the recommended configuration.
- **Amazon DynamoDB**: the internal partitioning layer uses consistent hashing to assign key ranges to storage nodes.
- **Memcached**: the `libketama` client library was the first widely-adopted consistent hashing implementation for cache clusters. Most Memcached clients today use it.
- **Redis Cluster**: uses 16,384 fixed hash slots — a discrete, pre-partitioned variant of the same idea. Each node owns a range of slots.
- **CDNs**: edge node selection for a given URL uses consistent hashing so the same URL consistently lands on the same edge node, maximising cache hit rate without explicit coordination.

## Trade-offs

Advantages:
- Only ~1/N keys remapped on topology changes — no mass migration or cache invalidation storm.
- Vnodes give near-uniform distribution without manual placement tuning.
- Node heterogeneity is handled gracefully via vnode count weighting.
- Scales horizontally without changing the routing formula.

Disadvantages:
- Without vnodes, distribution is uneven and hotspots are near-certain.
- With vnodes, the position index grows large and must be managed (e.g., 15,000 entries for 100 nodes × 150 vnodes).
- Consistent hashing distributes *keys* evenly, not *access frequency*. A hot key receiving millions of requests per second still lands on a single node. Hot-key problems require different solutions: key sharding with a suffix, local in-process caching, or read replicas.
- Ring membership state must be consistent across all nodes. A coordination mechanism — ZooKeeper, etcd, or a gossip protocol — is required to propagate topology changes.
- Implementation and operational complexity is higher than simple modulo hashing.

## How to Remember It

Think of a **clock face**. Servers are placed at various hour and minute markers. A task (key) lands somewhere on the clock face and is handled by the next marker clockwise. Add a new server marker and only the tasks between the previous marker and the new one migrate — everything else stays.

Mnemonics:
- **Consistent** → the *majority* of keys consistently stay on the same node through cluster changes.
- **Virtual nodes** → multiple clock positions per server, so load spreads evenly.
- **Clockwise** → always walk forward to find your node.

## Interview Cheat Sheet

**"Why not use modulo hashing?"**
> "Modulo hashing is not resize-stable. Changing N by 1 remaps ~(N-1)/N of all keys. For a cache, that means a near-total miss rate spike. Consistent hashing limits remapping to ~1/N of keys by assigning ownership based on position on a hash ring rather than a count-dependent formula."

**"Walk me through how the hash ring works."**
> "Hash values form a circle. Nodes are placed at hash(node_id) positions. A key maps to the first node clockwise from hash(key). Adding a node only displaces keys in the arc between that node and its predecessor — ~1/N keys on average."

**"What is the problem with a plain ring and how do you fix it?"**
> "Random placement of a few nodes leads to uneven arc sizes and hotspots. Virtual nodes fix this — each physical node gets 100–200 positions by hashing variants of its name. Distribution becomes statistically uniform and rebalancing on node change scatters across many nodes rather than dumping load on one successor."

**"What is the time complexity of a lookup?"**
> "O(log N) using binary search on a sorted array of vnode positions, where N is the total number of vnode entries."

**"Where does consistent hashing not help?"**
> "It distributes keys evenly, not access frequency. A key that receives millions of requests per second still hits a single node. Hot-key problems need different solutions — suffix-based sharding, local caching, or read replicas."

**"Name real systems that use it."**
> "Cassandra uses token ranges with vnodes. DynamoDB uses it for partitioning. libketama brought it to Memcached. CDNs use it for edge node selection."

## References

- [Consistent Hashing — Wikipedia](https://en.wikipedia.org/wiki/Consistent_hashing)
- [Amazon Dynamo Paper — Section 4.2, Partitioning](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- [Cassandra — Token Ring and Vnodes](https://cassandra.apache.org/doc/stable/cassandra/architecture/dynamo.html)
- [libketama — Original Consistent Hashing for Memcached](https://github.com/RJ/ketama)
- [Redis Cluster Specification — Hash Slots](https://redis.io/docs/reference/cluster-spec/)