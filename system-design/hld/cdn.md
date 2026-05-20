# Content Delivery Networks (CDN)

## Table of Contents

- [What Is a CDN](#what-is-a-cdn)
- [Why CDNs Exist](#why-cdns-exist)
- [Core Terminology](#core-terminology)
- [How a CDN Request Works](#how-a-cdn-request-works)
- [Traffic Routing Mechanisms](#traffic-routing-mechanisms)
- [Push vs. Pull CDN](#push-vs-pull-cdn)
- [Cache Control](#cache-control)
- [Cache Invalidation Strategies](#cache-invalidation-strategies)
- [CDN for Dynamic Content](#cdn-for-dynamic-content)
- [Security at the CDN Layer](#security-at-the-cdn-layer)
- [Trade-offs](#trade-offs)
- [When Not to Use a CDN](#when-not-to-use-a-cdn)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)


## What Is a CDN

A CDN is a geographically distributed network of servers — called **edge servers** or **Points of Presence (PoPs)** — that cache and serve content to users from a location physically close to them, rather than always routing requests to a central origin server.

The name is descriptive: **Content** (what is being moved) + **Delivery** (the act of serving it) + **Network** (the infrastructure coordinating it).


## Why CDNs Exist

Latency is fundamentally bounded by physics. Light through fiber travels at roughly 2/3 the speed of light in a vacuum. A round trip from Tokyo to a Virginia-based origin server adds ~150–200 ms before any application processing begins. No amount of software optimization eliminates that distance tax.

CDNs solve this by placing copies of content at dozens or hundreds of globally distributed edge locations, so that most users receive responses from a server that is tens of milliseconds away rather than hundreds.

Secondary motivations:

- **Origin offload** — a cache hit at the edge never touches the origin, dramatically reducing origin server load and bandwidth costs
- **Availability** — if the origin goes down, cached content at the edge can still be served (for the TTL duration)
- **DDoS resilience** — the attack surface is distributed; volumetric attacks are absorbed across PoPs instead of overwhelming a single origin


## Core Terminology

- **Origin server** — the authoritative source of truth for content; your application server or object storage bucket
- **Edge server** — a CDN server at a PoP that caches and serves content to nearby users
- **PoP (Point of Presence)** — a CDN data centre location; a cluster of edge servers in a city or region
- **Cache hit** — the requested content is found in the edge cache; served without contacting the origin
- **Cache miss** — content not in edge cache; edge fetches from origin, caches it, then serves
- **TTL (Time to Live)** — how long a cached object is considered fresh before revalidation is needed


## How a CDN Request Works

### Scenario A: Cache Hit

```
User (Tokyo)
    |
    | 1. DNS query: "who is cdn.example.com?"
    v
GeoDNS / Anycast
    |
    | 2. returns IP of nearest Tokyo PoP
    v
Tokyo Edge Server
    |
    | 3. cache HIT — content found, TTL still valid
    v
User (Tokyo)   <-- response served at low latency; origin never contacted
```

### Scenario B: Cache Miss (Origin Fetch)

```
User (Tokyo)
    |
    | 1. DNS query → Tokyo Edge Server IP
    v
Tokyo Edge Server
    |
    | 2. cache MISS — content absent or TTL expired
    v
Origin Server (Virginia)
    |
    | 3. returns content + cache-control headers
    v
Tokyo Edge Server
    | 4. stores content in local cache per TTL
    v
User (Tokyo)   <-- response served (higher latency this one time)
                   [subsequent requests: cache hit]
```


## Traffic Routing Mechanisms

### Anycast

Multiple edge servers share the **same IP address**. The BGP (Border Gateway Protocol) routing protocol naturally forwards packets to the topologically nearest server. No DNS lookup overhead per request once the IP is resolved.

- Used heavily by Cloudflare
- Failover is automatic — if one PoP is unreachable, BGP re-routes to the next nearest

### GeoDNS

DNS returns **different IP addresses** based on the requester's geographic location (determined from their resolver's IP). Each region gets a response pointing to its nearest PoP.

- Simpler to reason about operationally
- Subject to DNS propagation delays and resolver locality issues (a user in Tokyo whose resolver is in Singapore gets routed to Singapore PoP)
- Most CDNs use a combination of GeoDNS for initial routing and Anycast within a region


## Push vs. Pull CDN

### Pull CDN (the default mental model)

The CDN is initially empty. On a cache miss, the edge server fetches content from the origin, caches it, and serves it. All subsequent requests hit the cache until TTL expires.

```
First Request (cache miss):
User --> Edge --> Origin --> Edge (caches) --> User

Subsequent Requests (cache hit):
User --> Edge (serves from cache) --> User
```

- Good for: web assets, images, API responses, any content with uncertain access patterns
- Downside: first user after a cold start (or post-TTL-expiry) bears origin latency
- Examples: CloudFront (pull mode), Cloudflare

### Push CDN

You proactively upload content to the CDN. The CDN never fetches from origin; it serves only what you have explicitly pushed.

```
Publisher --> CDN Storage (all PoPs pre-loaded) --> Users
```

- Good for: large files with known access patterns (software releases, video archives)
- Downside: you are responsible for keeping CDN content in sync; stale content stays until you remove it
- Examples: Akamai NetStorage, some object-storage CDN setups


## Cache Control

CDNs respect standard HTTP caching headers. Setting these correctly on origin responses determines CDN caching behaviour.

### Key Headers

- **`Cache-Control: max-age=3600`** — cache for 3600 seconds (1 hour); applies to both browser and CDN
- **`Cache-Control: s-maxage=3600`** — CDN-specific TTL; overrides `max-age` for shared caches (CDN), while browser uses `max-age`
- **`Cache-Control: no-store`** — do not cache at all; used for sensitive/private data
- **`Cache-Control: no-cache`** — store in cache but always revalidate with origin before serving (uses ETag or Last-Modified)
- **`Cache-Control: stale-while-revalidate=60`** — serve stale content immediately; trigger background refresh if content is up to 60 s past expiry
- **`ETag`** — a fingerprint (hash) of the content; CDN sends it back to origin via `If-None-Match` to check if content has changed; origin replies `304 Not Modified` if unchanged (saves bandwidth)
- **`Vary: Accept-Encoding`** — instructs CDN to store separate cache entries per encoding variant (gzip vs. brotli); important when serving compressed assets

### Common Mistake

Setting `max-age` too high (e.g., one week) on assets without versioned URLs. When you deploy a new version, users and CDN edges continue serving the old file until TTL expires.

**Fix:** use cache busting — versioned or hashed filenames (`app.a3f9b2.js`) so new deployments are a cache miss for a new URL, while the old URL expires naturally.


## Cache Invalidation Strategies

Cache invalidation is famously one of the two hard problems in computer science. Three main approaches:

### 1. TTL Expiry (Passive Invalidation)

Set a TTL on cached objects. After expiry, the next request triggers a fresh origin fetch. Simple and zero-operational-overhead, but coarse — content remains stale for up to TTL duration after a change.

Best for: content that can tolerate some staleness (blog posts, product descriptions).

### 2. Versioned URLs / Cache Busting (Proactive Prevention)

Embed a version or content hash in the URL: `styles.v4.css` or `styles.a3f9b2.css`. Each deploy produces a new URL; the old URL's cached version is irrelevant because no one requests it anymore. The new URL starts as a cold cache miss.

```
Deploy v1: /static/bundle.abc123.js  (TTL: 1 year, safe)
Deploy v2: /static/bundle.def456.js  (new URL, new cache entry)
           /static/bundle.abc123.js  (still in CDN cache, but no longer linked)
```

Best for: static assets (JS, CSS, images). This is the recommended approach for immutable assets.

### 3. Purge API (Active Invalidation)

Most CDN providers expose an API to immediately invalidate specific URLs or URL patterns across all PoPs. Useful after content updates that need to propagate quickly (news articles, price changes).

Downsides: propagation across all PoPs can take seconds to minutes; some providers charge per purge; purging patterns (`/products/*`) can be slow.


## CDN for Dynamic Content

This is where interviews get more nuanced. CDNs are not just for static files.

### Short-TTL Caching of API Responses

For content that is "fresh enough" rather than strictly real-time — product listings, sports scores, weather data — caching API responses at the edge with a short TTL (5–60 s) dramatically reduces origin load with acceptable staleness.

### stale-while-revalidate

Serve the cached (potentially slightly stale) response immediately; simultaneously trigger an async revalidation request to origin. The user gets low latency; origin gets updated cache in the background. Good user experience trade-off for mildly time-sensitive data.

```
Request arrives, TTL expired by < revalidate window:
  Edge --> User  (serves stale content immediately)
  Edge --> Origin (async, background refresh)
  [next request gets fresh content]
```

### Edge Functions / Edge Computing

Run code at the PoP itself. No round trip to origin for logic that does not require full application state.

Use cases at the edge:
- Authentication token validation (verify JWT at edge, reject bad requests before origin)
- A/B testing (route 10% of traffic to `/v2` at edge, no origin change required)
- HTTP header manipulation (add security headers, CORS headers)
- Geolocation-based redirects
- Request coalescing (collapse N simultaneous cache-miss requests into one origin fetch)

Examples: Cloudflare Workers, AWS Lambda@Edge, Vercel Edge Functions.


## Security at the CDN Layer

- **DDoS absorption** — volumetric attacks (floods of traffic) are distributed across PoPs rather than concentrated on the origin; CDN providers have significant capacity headroom
- **WAF (Web Application Firewall)** — run firewall rules at the edge to block SQL injection, XSS, bad bots before they reach origin (Cloudflare, Akamai both offer this)
- **TLS termination** — the CDN handles TLS handshakes with end users; the CDN-to-origin connection uses a separate (often internal or private) TLS session; reduces TLS overhead at origin
- **Rate limiting at edge** — limit requests per IP/endpoint without touching application servers
- **Bot detection** — CDN-level CAPTCHA challenges, fingerprinting, behavioural analysis

Important: CDN security is defence-in-depth, not a replacement for application-level authentication and authorisation.


## Trade-offs

| Concern | CDN Helps | CDN Does Not Help (or Hurts) |
|---|---|---|
| Latency | Static assets, cacheable responses | Real-time, per-user personalised data |
| Origin load | Cache hits eliminate origin calls | Cold starts still hit origin |
| Bandwidth cost | Reduces origin egress | CDN egress has its own (often lower) cost |
| Availability | Cached content survives short origin outages | Not a substitute for proper origin HA |
| Security | DDoS, WAF, bot filtering | Not a substitute for app-layer auth |
| Staleness | Versioned URLs eliminate stale-content risk | TTL tuning is genuinely difficult |
| Observability | Some CDNs expose detailed analytics | Adds a hop to debug; is it CDN or origin? |


## When Not to Use a CDN

- **Highly dynamic, personalised content** — user dashboards, banking pages, shopping carts; the cache-key would need to include user identity, making it effectively uncacheable
- **Strictly real-time data** — live stock prices, live sensor feeds where even a 1 s stale response is wrong
- **Low-traffic or geographically concentrated apps** — if all users are in one region and close to the origin, CDN adds cost and a layer of complexity with minimal gain
- **Regulatory / data residency constraints** — some jurisdictions prohibit content transiting foreign CDN PoPs; verify compliance before adopting a global CDN


## Interview Cheat Sheet

### Likely Questions and What Interviewers Are Probing

**"How does a CDN work?"**
Walk through the DNS resolution (GeoDNS / Anycast), cache hit vs. cache miss flow, TTL, and origin fetch. Show you understand the full request lifecycle, not just "it caches stuff near users."

**"How do you invalidate CDN cache after a deployment?"**
Mention all three strategies: TTL expiry (passive, acceptable for some content), versioned URLs / cache busting (best practice for static assets, zero invalidation needed), and purge API (immediate but operationally heavier). Explain why versioned URLs are preferred for immutable assets.

**"Can you cache dynamic content on a CDN?"**
Yes — short TTL, stale-while-revalidate, and edge functions. Show you know CDNs are not limited to static files.

**"What's the difference between push and pull CDN?"**
Pull: CDN fetches on miss; good for unpredictable access patterns. Push: you load content proactively; good for known large files. Most web apps use pull.

**"What are the downsides of a CDN?"**
Stale content risk, cache invalidation complexity, vendor lock-in, debugging complexity (an extra hop), cost at scale, not suitable for personalised/real-time data.

**"How does CDN help with DDoS?"**
Distributed surface area; attack traffic is absorbed across global PoPs rather than concentrated on one origin. CDN providers have far more aggregate capacity than a single origin deployment.

### Signal Phrases That Show Depth

- "I'd use `s-maxage` for the CDN TTL and `max-age` for browser caching to control them independently."
- "For static assets I'd use content-hashed filenames with a 1-year TTL — no invalidation needed."
- "stale-while-revalidate gives us the latency of a cache hit with eventual consistency to origin."
- "Edge functions let us run auth or routing logic at the PoP with sub-millisecond overhead."
- "We'd need to set `Vary: Accept-Encoding` correctly or we'd serve gzipped content to clients that don't support it."

### Common Misconceptions to Avoid

- "CDN caches everything" — it caches what you tell it to via headers; it respects `no-store` and `private`
- "CDN makes my app always available" — CDN availability requires the cache to be warm; cache misses during origin downtime will fail
- "A CDN is only for images and video" — API responses, HTML, even personalised content (with care) can be edge-cached


## References

- [MDN: HTTP Caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
- [Cloudflare Learning Center: What is a CDN?](https://www.cloudflare.com/learning/cdn/what-is-a-cdn/)
- [AWS CloudFront Developer Guide](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html)
- [Web.dev: stale-while-revalidate](https://web.dev/articles/stale-while-revalidate)
- [Cloudflare Workers Documentation](https://developers.cloudflare.com/workers/)
- System Design Interview – An Insider's Guide, Alex Xu (Chapter on CDN)