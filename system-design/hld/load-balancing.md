# Load Balancing — Algorithms, L4 vs L7

## Table of Contents

1. [What It Is](#what-it-is)
2. [Why It Exists](#why-it-exists)
3. [L4 vs L7](#l4-vs-l7)
4. [Algorithms](#algorithms)
5. [Problems It Introduces](#problems-it-introduces)
6. [Interview Cheatsheet](#interview-cheatsheet)
7. [References](#references)

---

## What It Is

A **load balancer** is a component that sits in front of a pool of backend servers and distributes incoming requests across them. Clients talk to the load balancer; the backend servers are invisible to the client.

```
Client
  │
  ▼
┌─────────────┐
│    Load     │
│  Balancer   │
└──┬────┬────┘
   │    │    │
   ▼    ▼    ▼
 [S1] [S2] [S3]
```

---

## Why It Exists

Without a load balancer, a single server is both a **capacity ceiling** and a **single point of failure**.

| Problem | What LB Provides |
|---|---|
| One server can't scale infinitely | Horizontal scaling — add more servers |
| One server crash = full outage | High availability — traffic rerouted around dead servers |
| Some requests are slow, some fast | Even utilisation — prevents hot spots |

---

## L4 vs L7

The numbers refer to layers in the **OSI model**.

### L4 — Transport Layer

Operates on **TCP/UDP** packets. Sees only: source IP, destination IP, source port, destination port. Never reads the payload.

```
Incoming packet:
┌────────────────────────────────┐
│ IP Header │ TCP Header │ ░░░░  │
└────────────────────────────────┘
                            ▲
            payload is opaque to an L4 LB
```

**How it works internally:** The LB maintains a NAT table. When a packet arrives for `203.0.113.1:80`, it rewrites the destination to a chosen backend (e.g. `10.0.0.3:80`) and remembers the mapping so return packets can be rewritten back.

**Characteristics:**
- Very fast — just IP/port rewrite, no payload parsing
- Protocol-agnostic (HTTP, gRPC, raw TCP, databases)
- Cannot route by URL path, headers, or cookies
- Connection-level stickiness: once a TCP connection is assigned to a backend, all packets in that connection stay there

**Examples:** AWS NLB, HAProxy (TCP mode), Linux IPVS

---

### L7 — Application Layer

Operates at the **HTTP/HTTPS layer**. The LB terminates the TCP connection, fully parses the HTTP request, makes a routing decision, and opens a new connection to the backend.

```
Incoming HTTP request (LB reads all of this):
┌─────────────────────────────────────────────┐
│ GET /api/v2/users HTTP/1.1                  │
│ Host: api.example.com                       │
│ Cookie: session=abc123                      │
│ Authorization: Bearer <token>               │
└─────────────────────────────────────────────┘
```

**Characteristics:**
- Routes by URL path, HTTP method, headers, cookies, query params
- Terminates TLS (holds the private key — high-value target)
- Higher overhead than L4 (full HTTP parse per request)
- Can do request rewriting, auth injection, header manipulation
- Cookie-based sticky sessions (more reliable than IP hash)

**Examples:** AWS ALB, Nginx, Envoy, HAProxy (HTTP mode), Traefik

---

### L4 vs L7 Comparison

| Dimension | L4 | L7 |
|---|---|---|
| OSI Layer | Transport (4) | Application (7) |
| Sees | IP + Port only | Full HTTP headers, URL, body |
| Routing basis | IP/Port | URL path, headers, cookies, method |
| Performance | Very fast | Slower (full parsing) |
| SSL termination | No (pass-through) | Yes |
| Sticky sessions | IP hash only | Cookie-based (more reliable) |
| Protocol-aware | No | HTTP, gRPC, WebSocket |
| Use case | Raw TCP, non-HTTP, ultra-low latency | Web APIs, microservices, content routing |

---

## Algorithms

### 1. Round Robin

Cycle through servers in order. Each request goes to the next server.

```
Servers: [S1, S2, S3]

Request 1 → S1
Request 2 → S2
Request 3 → S3
Request 4 → S1   ← wraps around
```

- ✅ Simple, zero state, even distribution for uniform requests
- ❌ Ignores server capacity and actual load; expensive requests can pile up on one server

---

### 2. Weighted Round Robin

Each server has a weight proportional to its capacity. Higher weight = more requests.

```
S1 (weight=3), S2 (weight=1), S3 (weight=2)
Sequence: S1, S1, S1, S2, S3, S3, S1, S1, S1, S2, ... - (basic/naive WRR - chunked (S1,S1,S1...))

In real systems like Nginx - (smooth WRR - Interleaved (more balanced over time))
  It can produce sequences like:
  S1, S3, S1, S2, S1, S3, ...
  S1, S2, S3, S1, S3, S1, S1...
```

- ✅ Handles heterogeneous hardware
- ❌ Weights are static; doesn't react to runtime load

---

### 3. Least Connections

Routes to the server with the fewest **active connections** right now.

```
S1: 5 active connections
S2: 2 active connections  ← new request goes here
S3: 8 active connections
```

- ✅ Adapts to variable request duration; good for long-lived connections (WebSockets, streaming)
- ❌ Doesn't account for server capacity differences

---

### 4. Weighted Least Connections

Combines weight and connection count: `score = active_connections / weight`. Route to lowest score.

- ✅ Handles heterogeneous servers with variable-cost, long-lived connections
- ❌ More complex bookkeeping

---

### 5. IP Hash (Session Affinity)

Hash the client's IP to always route them to the same server:
`server = hash(client_ip) % num_servers`

```
192.168.1.1  →  always  →  S2
10.0.0.5     →  always  →  S1
172.16.0.3   →  always  →  S3
```

- ✅ Stateful apps where session data lives in-process on the server
- ❌ Many clients behind the same NAT → uneven distribution
- ❌ Adding/removing a server remaps most clients (modular hashing problem)

---

### 6. Consistent Hashing

Places servers and keys on a conceptual ring. A request maps to the first server clockwise from its hash position.

```
           0° (top)
             │
  S3 ────────┤──────── S1
             │
           180°
             │
            S2
```

When a server is removed, only that server's keys move to its neighbour. All other assignments are unchanged.

- ✅ Minimises redistribution when topology changes (critical for cache clusters)
- ✅ Used in: Memcached, Redis Cluster, Cassandra, DynamoDB
- ❌ Hot spots without virtual nodes; adds complexity

> **Virtual Nodes:** Each physical server is represented by multiple positions on the ring, improving even distribution.

---

### 7. Least Response Time

Routes to the server with the lowest measured average response latency (often combined with least connections).

- ✅ Adapts to latency differences between backends
- ❌ Requires active monitoring; a momentarily fast-but-overloaded server can attract a flood of requests

---

### 8. Resource-Based (Adaptive)

Backends report health metrics (CPU %, memory %) to the LB via an agent/sidecar. LB routes to the least-loaded server by real metrics.

- ✅ Most accurate picture of server health
- ❌ Adds operational complexity; agent must run alongside every backend

---

### Algorithm Selection Guide

| Scenario | Recommended Algorithm |
|---|---|
| Homogeneous servers, short uniform requests | Round Robin |
| Mixed hardware capacities | Weighted Round Robin |
| Long-lived connections, WebSockets | Least Connections |
| Stateful app, session in memory | IP Hash or Cookie-based sticky (L7) |
| Distributed cache cluster | Consistent Hashing |
| Mixed latency backends | Least Response Time |

---

## Problems It Introduces

### 1. The LB Is Now a SPOF

You added redundancy for your servers but the LB itself is a single point of failure.

**Solution:** Run two LBs in active-passive (or active-active) with a shared Virtual IP (VIP). A heartbeat daemon (VRRP (Virtual Router Redundancy Protocol) / Keepalived) promotes the passive LB if the active one goes down.

```
        VIP: 203.0.113.1
               │
       ┌───────┴──────┐
       ▼              ▼
  LB-Primary    LB-Standby
  (active)      (passive, takes over on failure)
       │              │
       └──────┬───────┘
              │
         [S1] [S2] [S3]
```



### 2. Session Affinity vs Scalability Tension

Sticky sessions undermine even distribution. If S2 gets many long-running sessions, it becomes hot.

**Solution:** Externalise session state (Redis, a database) so all servers are stateless. Now any server can handle any request.

### 3. Health Check Lag

If a server crashes, the LB won't know until the next health check fires and the failure threshold is reached. During this window, requests go to a dead server.

**Mitigate:** Short health check intervals + low failure thresholds, at the cost of more probe traffic.

### 4. TLS Private Key Exposure

An L7 LB that terminates TLS holds your certificate's private key. It becomes a high-value attack target.

**Mitigate:** mTLS between LB and backends; rotate certificates regularly; use HSMs for key storage.

### 5. Consistent Hashing Rebalancing (Thundering Herd)

Even with consistent hashing, adding a new cache node causes a fraction of cache misses to fall through to the DB simultaneously.

**Mitigate:** Gradual rollout, request coalescing, or pre-warming the new node.

---

## Interview Cheatsheet

| Question | Key Point |
|---|---|
| L4 vs L7? | L4 = IP/port only, fast, protocol-agnostic. L7 = full HTTP parse, enables path routing / auth injection |
| LB goes down? | Active-passive pair + VIP. DNS failover. Anycast for global LBs |
| Stateful services? | Sticky sessions → leads to hot servers. Better: externalise state, make servers stateless |
| Round Robin vs Least Connections? | Round Robin = uniform short requests. Least Connections = variable duration or long-lived connections |
| Reverse proxy vs LB? | A LB is a reverse proxy. A reverse proxy isn't always a LB (it may not distribute load) |
| Why consistent hashing in caches? | Minimises cache misses when nodes are added/removed — only 1/n of keys remapped vs full rehash |
| Health checks? | Active (LB probes backend), passive (LB observes error rates on real traffic). Most use both |

---

## References

- [NGINX Load Balancing](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)
- [AWS ALB vs NLB](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html)
- [Consistent Hashing — Wikipedia](https://en.wikipedia.org/wiki/Consistent_hashing)
- [HAProxy Documentation](https://www.haproxy.org/download/2.4/doc/configuration.txt)
- [System Design Primer — Load Balancing](https://github.com/donnemartin/system-design-primer#load-balancer)
- [Envoy Proxy Architecture](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/intro/arch_overview)