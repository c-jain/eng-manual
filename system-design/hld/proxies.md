# Proxies — Forward Proxy, Reverse Proxy, Sidecar Pattern

## Table of Contents

- [What Is a Proxy](#what-is-a-proxy)
- [Forward Proxy](#forward-proxy)
- [Reverse Proxy](#reverse-proxy)
- [Forward vs. Reverse — The Key Distinction](#forward-vs-reverse--the-key-distinction)
- [Sidecar Pattern](#sidecar-pattern)
- [Trade-offs Summary](#trade-offs-summary)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [How to Remember It](#how-to-remember-it)
- [References](#references)

---

## What Is a Proxy

A proxy is an intermediary that sits between two communicating parties and acts on behalf of one of them. The word comes from the legal sense: someone authorised to act for another.

Proxies exist because direct communication is often insufficient:

- You want to hide identity (anonymity, privacy)
- You want to enforce policy (access control, rate limiting)
- You want to improve performance (caching, compression)
- You want to observe traffic (logging, distributed tracing)
- You want to add reliability (retries, circuit breaking, failover)

The critical question: **whose behalf** is the proxy acting on?

---

## Forward Proxy

### What It Is

A forward proxy sits in front of **clients** and acts on their behalf when communicating with external servers. The destination server sees the proxy's IP, not the client's.

```
Client --> Forward Proxy --> Internet --> Server
                   (server sees proxy,
                    not client)
```

### Why It's Called "Forward"

It *forwards* the client's request outward, toward the destination. The directionality is from the client's perspective.

### Why It Exists

- **Anonymity / privacy** — the client's real IP is hidden from the server
- **Content filtering** — corporate or school networks block social media, malware sites, etc.
- **Shared caching** — an ISP-level proxy caches responses for many clients simultaneously
- **Geo-bypass** — the proxy sits in a different region, bypassing geo-restrictions

### Classic Examples

- Squid Proxy (enterprise content filtering)
- VPN-like tools at the network layer
- `HTTP_PROXY` / `HTTPS_PROXY` environment variables — you are configuring a forward proxy for your process

### Problems It Brings

- Single point of failure if not made highly available
- Privacy paradox: you trust the proxy operator rather than the original server
- TLS interception is complex — requires CONNECT tunneling and careful handling of certificate pinning
- Adding a hop increases latency; must be accounted for in SLA budgets

---

## Reverse Proxy

### What It Is

A reverse proxy sits in front of **servers** and acts on their behalf when communicating with clients. The client sees the proxy's address; the backend topology is invisible to it.

```
                              +--> Backend A
Client --> Reverse Proxy  -->-+--> Backend B
                              +--> Backend C
       (client sees proxy,
        not backends)
```

### Why It's Called "Reverse"

It is the *reverse* of a forward proxy. Instead of hiding the client from the server, it hides the server from the client.

### Why It Exists

This is the workhorse of modern infrastructure:

- **Load balancing** — distribute traffic across multiple backend instances
- **TLS termination** — handle HTTPS at the proxy boundary; backends receive plain HTTP internally, simplifying certificate management
- **Caching** — cache backend responses to reduce load and latency (a CDN is a geographically distributed reverse proxy)
- **Compression** — gzip/brotli responses centrally rather than per-service
- **Rate limiting and WAF** — inspect and block malicious traffic before it reaches application code
- **Canary / blue-green deployments** — route a percentage of traffic to a new version without changing client behaviour
- **Health checking** — remove unhealthy backends from the rotation automatically

### Classic Examples

- Nginx, HAProxy, Envoy, Traefik (self-hosted)
- AWS ALB / Cloudflare / Fastly (managed)
- API Gateway (a specialised reverse proxy with auth, throttling, and routing)

### Problems It Brings

- Adds a network hop and latency — usually acceptable but matters at extreme RPS
- The proxy itself becomes a critical component and must be made HA
- Complex configuration (routing rules, TLS certs, retries, timeouts) creates an operational burden
- Can obscure backend errors; debugging requires correlating proxy logs with application logs
- Misconfigured caching can serve stale data silently

---

## Forward vs. Reverse — The Key Distinction

| Dimension | Forward Proxy | Reverse Proxy |
|---|---|---|
| Acts on behalf of | Client | Server |
| Hides | Client from server | Server from client |
| Who configures it | Client (or client's OS) | Server operator |
| Primary use cases | Anonymity, content filtering, caching | Load balancing, TLS termination, WAF |
| Examples | Squid, corporate firewall, VPN | Nginx, AWS ALB, Cloudflare |

**One-line test:**
> If you had to set `HTTP_PROXY=...` yourself to use it, it's a forward proxy.
> If you just hit `api.example.com` and had no idea there were backends behind it, it's a reverse proxy.

---

## Sidecar Pattern

### What It Is

A sidecar is a process (or container) co-located with your application that handles cross-cutting concerns on the application's behalf. The name comes from a motorcycle sidecar — it goes everywhere the motorcycle goes, but it is a separate vehicle.

In Kubernetes: a sidecar is an additional container in the same Pod as your application container, sharing the same network namespace and storage volumes.

### Why It Exists

Every microservice needs the same operational capabilities: mutual TLS, retries, circuit breaking, distributed tracing, metrics collection, log shipping. Without a sidecar:

- Each team re-implements these in their service
- Different languages get different library versions (or none at all)
- Business logic and operational code become entangled
- A bug in the tracing library requires updating every service

The sidecar externalises all of this into a separate process. Your business logic speaks plain HTTP or gRPC; the sidecar handles everything else transparently.

### How It Works

The service mesh control plane (e.g., Istio's Istiod) injects the sidecar container and installs iptables rules that redirect all inbound and outbound traffic through it. The application is unaware.

```
+--------------------------------- Pod ---------------------------------+
|                                                                       |
|  +---------------------+           +-----------------------------+   |
|  |   App Container     |           |   Sidecar (Envoy)           |   |
|  |   (business logic)  |<--------->|   - mTLS                    |   |
|  |   plain HTTP/gRPC   |           |   - retries & timeouts      |   |
|  |                     |           |   - circuit breaking        |   |
|  +---------------------+           |   - distributed tracing     |   |
|                                    |   - metrics / access log    |   |
|                                    +-----------------------------+   |
|                 both communicate over localhost (loopback)           |
+-----------------------------------------------------------------------+
                        |                     |
               inbound traffic          outbound traffic
               intercepted by           intercepted by
               iptables rules           iptables rules
```

### Sidecar as a Local Reverse Proxy

Each sidecar is effectively a **local reverse proxy** for that pod — it proxies outbound calls to other services and routes inbound calls to the app. The collection of all sidecars across a cluster forms the **data plane** of a service mesh. Envoy is the dominant sidecar implementation.

### Problems It Brings

- **Latency** — two extra network hops per inter-service call (outbound sidecar → inbound sidecar). Typically 1–2 ms overhead; significant at high RPS or when service graphs are deep
- **Resource overhead** — every pod gets a sidecar; at 1,000 pods that is 1,000 Envoy processes consuming CPU and memory
- **Operational complexity** — debugging requires correlating application logs with sidecar proxy logs
- **iptables opacity** — transparent traffic interception is powerful but hard to reason about when things break unexpectedly
- **Startup ordering** — the sidecar must be ready before the app container begins accepting traffic; a known pain point in Kubernetes until native sidecar container support (KEP-753) landed in 1.29

### When to Use It

Use it when you have many services across multiple languages and need consistent observability, security (mTLS), and reliability policies cluster-wide.

Avoid it for a monolith or two-service system — the operational overhead is not justified.

---

## Trade-offs Summary

| Proxy Type | Latency Impact | Operational Cost | Primary Benefit |
|---|---|---|---|
| Forward proxy | Low (single hop) | Low–medium | Client anonymity, content policy |
| Reverse proxy | Low (single hop) | Medium | Load balancing, TLS termination, WAF |
| Sidecar | Medium (two extra hops) | High | Uniform cross-cutting concerns across polyglot services |

---

## Interview Cheat Sheet

**Strong signal phrases:**

- "A reverse proxy terminates TLS at the edge so backends don't each need certificate management."
- "The sidecar pattern lets us enforce mTLS between all services without changing application code."
- "A CDN is just a geographically distributed reverse proxy that caches at the edge."
- "Forward proxies hide clients; reverse proxies hide servers."
- "In a service mesh, the sidecar data plane handles retries and circuit breaking; the control plane pushes configuration to it."
- "The main cost of sidecars is the extra latency hop and the resource overhead of running a proxy per pod."

**Red flags to avoid:**

- Confusing forward and reverse proxies — know which party is being hidden
- Saying "Nginx is a forward proxy" without context — it can be either depending on configuration
- Ignoring the operational cost of the sidecar pattern; don't present it as free
- Treating a CDN and a reverse proxy as entirely different things rather than related concepts
- Forgetting that the sidecar intercepts traffic transparently via iptables, not via application configuration

**Common interview scenarios:**

- Design a rate-limited API — use a reverse proxy (API Gateway / Nginx) as the enforcement point
- Design a secure service mesh — use mutual TLS via sidecar proxies (Envoy/Istio)
- "How does Cloudflare work?" — it is a reverse proxy + CDN at the network edge
- "How do you add distributed tracing without touching every service?" — sidecar injection

---

## How to Remember It

**Forward** = hides the **F**ront (client-side). Think **F**irewall, **F**an-out to the internet.

**Reverse** = hides the **R**ear (server-side). **R**outes inbound traffic.

**Sidecar** = your app's personal assistant co-located in the same pod. It handles all the boring cross-cutting work (mTLS, retries, tracing) so your app can focus purely on business logic. Like a motorcycle sidecar: always along for the ride, always separate.

---

## References

- [NGINX Reverse Proxy Guide](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
- [Envoy Proxy Documentation](https://www.envoyproxy.io/docs/envoy/latest/)
- [Istio Architecture — Data Plane and Control Plane](https://istio.io/latest/docs/ops/deployment/architecture/)
- [Kubernetes Native Sidecar Containers (KEP-753)](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
- [The Sidecar Pattern — Microsoft Azure Architecture Patterns](https://learn.microsoft.com/en-us/azure/architecture/patterns/sidecar)