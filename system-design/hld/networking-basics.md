# Client-Server model, IP, DNS, HTTP/HTTPS

## Table of Contents

- [Client-Server model](#Client-Server-model)
- [IP Addressing](#IP-Addressing)
- [DNS](#dns)
- [HTTP and HTTPS](#http-and-https)
- [Interview cheat sheet](#Interview-cheat-sheet)
- [References](#References)

---

## Client-Server model

The client-server model is an architectural pattern where two parties communicate over a network: one **requests** resources or services (the client), and the other **provides** them (the server).

- **client** — initiates the communication. It sends a request and waits for a response. Examples: a browser, a mobile app, a CLI tool like `curl`.
- **server** — listens passively for incoming requests on a known address and port. It processes requests and sends back responses. Examples: a web server (nginx), an API server, a database.

The model is asymmetric: clients start conversations; servers respond to them. A server can handle many clients simultaneously, but a single client typically talks to one server per connection at a time.

```
  CLIENT                          SERVER
  ┌──────────┐                   ┌──────────────┐
  │  Browser │ ── request ────►  │  Web Server  │
  │          │ ◄── response ──   │  (port 443)  │
  └──────────┘                   └──────────────┘
```

**Variants worth knowing:**

- **peer-to-peer (P2P)** — every node acts as both client and server (BitTorrent, WebRTC). There is no central authority.
- **three-tier architecture** — client talks to an application server, which in turn talks to a database server. Responsibilities are cleanly separated across layers.
- **microservices** — multiple backend services communicate with each other as both clients and servers. A single user request may chain through many services.

### Stateless vs Stateful servers

HTTP is stateless by design: the server holds no memory of previous requests. This is why session state must live somewhere explicit:

- **Cookie** — stored client-side, sent on every request in the `Cookie` header.
- **Session store** — a server-side key-value store (Redis, Memcached) keyed by a session ID in a cookie.
- **JWT** — a self-describing signed token. Stateless on the server side; the client carries all state. The server only needs the signing key to verify.

Stateless servers are horizontally scalable with no coordination. Stateful servers require sticky sessions or shared state — a design constraint that comes up constantly in scaling interviews.

### request-response cycle

```
Client                          Server
  │                               │
  │── TCP connect ───────────────>│
  │<─ TCP accept ─────────────────│
  │                               │
  │── HTTP request ──────────────>│  (GET /api/users HTTP/1.1)
  │<─ HTTP response ──────────────│  (200 OK + body)
  │                               │
  │── keep-alive: next request ──>│
```

In HTTP/1.1 connections persist (`keep-alive`) by default. Each request still waits for the previous response (head-of-line blocking at the application layer). HTTP/2 fixes this with multiplexing.

**Why this abstraction matters for system design:** when you design a system, the client-server boundary is where you define your API contract, decide on protocols, and think about latency, reliability, and security. Every architectural decision downstream — load balancing, caching, authentication — exists to make this boundary more reliable or performant.

---

## IP Addressing

IP is the network layer protocol. Every device on a network is assigned an **IP address** — a numeric label that uniquely identifies it within a network. IP addresses are how the internet routes packets from one machine to another.

**IPv4**

- 32-bit addresses written as four octets: `192.168.1.10`
- Supports ~4.3 billion unique addresses — a space that has been exhausted in practice
- Private ranges (RFC 1918) are not routable on the public internet:
  - `10.0.0.0/8`
  - `172.16.0.0/12`
  - `192.168.0.0/16`
- `127.0.0.1` is the loopback address — a machine sending traffic to itself
- Exhausted in the early 2010s; NAT buys time

**IPv6**

- 128-bit addresses written in hex: `2001:0db8:85a3::8a2e:0370:7334`
- Supports ~3.4 × 10³⁸ unique addresses
- `::1` is the IPv6 loopback
- Adoption is accelerating but coexistence with IPv4 (dual-stack) is common

**Ports**

An IP address identifies a machine. A **port** identifies a specific process or service running on that machine. Together they form a **socket address**: `192.168.1.10:8080`.

- Ports range from 0 to 65535
- **Well-known ports (0–1023)** are reserved by convention:
  - `80` → HTTP
  - `443` → HTTPS
  - `22` → SSH
  - `53` → DNS
- **Ephemeral ports (49152–65535)** are assigned temporarily by the OS to client-side connections

### CIDR notation (Classless Inter-Domain Routing)

`10.0.0.0/24` — the `/24` means the first 24 bits are the **network prefix**. The remaining 8 bits identify hosts: `2^8 = 256` addresses (`.0` to `.255`; `.0` is the network address, `.255` is broadcast — 254 usable).

```
10.0.0.0/24  → hosts: 10.0.0.1 – 10.0.0.254
10.0.0.0/16  → hosts: 10.0.0.1 – 10.0.255.254   (65,534 usable)
10.0.0.0/8   → hosts: 10.0.0.1 – 10.255.255.254
```

CIDR notation appears everywhere in VPC subnet design. A `/16` VPC broken into `/24` subnets gives you 256 subnets of 254 hosts each.

### Public vs Private addresses (RFC 1918)

Private ranges (not routable on the public internet):

```
10.0.0.0/8          — Class A private
172.16.0.0/12       — Class B private
192.168.0.0/16      — Class C private (home networks)
127.0.0.0/8         — loopback (localhost)
169.254.0.0/16      — link-local (APIPA, no DHCP found)
```

### NAT (Network Address Translation)

Your home router has one public IP. All devices on your LAN have private IPs. NAT translates outbound packets (source private IP → public IP + unique port) and tracks the mapping to reverse-translate inbound responses.

In AWS VPC design:
- **Public subnet** — instances have a public IP and an internet gateway route.
- **Private subnet** — instances have only private IPs. A **NAT gateway** in the public subnet allows outbound traffic from private instances while keeping them unreachable from the internet.
- Databases should always live in private subnets.

---

## DNS

DNS (Domain Name System) is the internet's phone book. It translates human-readable domain names (`github.com`) into IP addresses (`140.82.114.4`) that machines can route to. It is the distributed, hierarchical system.  It is itself a distributed database — no single server holds all records.

Without DNS, you would have to remember IP addresses for every service you use. DNS lets the IP behind a name change without users noticing.

**Resolution process**

```
Browser / OS cache
    │ miss
    ▼
Recursive resolver (ISP or 8.8.8.8 / 1.1.1.1)
    │ cache miss
    ├── asks Root nameserver → "for .com, ask Verisign TLD servers"
    ├── asks TLD nameserver (.com) → "for github.com, ask NS1/NS2.p16.dynect.net"
    └── asks Authoritative nameserver → "github.com → 140.82.121.4"
    │
    ▼
Answer returned, cached at each layer with TTL
```

Steps in full:

1. Browser checks its own DNS cache.
2. OS resolver checks `/etc/hosts`, then its own cache.
3. Query sent to configured **recursive resolver** (from DHCP or manual config).
4. Recursive resolver checks its cache. On miss, it walks the hierarchy:
    - Asks a **root nameserver** (13 root server clusters, anycast-distributed worldwide).
    - Root returns the address of the relevant **TLD nameserver** (`.com`, `.io`, `.org`).
    - Recursive resolver asks the TLD nameserver.
    - TLD returns the **authoritative nameserver** for the domain.
    - Recursive resolver asks the authoritative nameserver.
    - Authoritative nameserver returns the actual record.
5. Answer travels back to the browser. Each intermediary caches the response for its TTL.

When you type `github.com` in your browser:

```
Browser                Recursive Resolver         Root NS         TLD NS (.com)       Authoritative NS
  │                          │                       │                  │                     │
  │── "github.com?" ────────►│                       │                  │                     │
  │                          │── "github.com?" ─────►│                  │                     │
  │                          │◄─ ".com NS address" ──│                  │                     │
  │                          │── "github.com?" ─────────────────────────►│                   │
  │                          │◄─ "github.com NS addr"──────────────────  │                   │
  │                          │── "github.com?" ───────────────────────────────────────────── ►│
  │                          │◄─ "140.82.114.4" ──────────────────────────────────────────── │
  │◄─ "140.82.114.4" ────────│
```

1. **browser cache** — checked first. OS cache checked next.
2. **recursive resolver** — usually your ISP or a public resolver (Cloudflare `1.1.1.1`, Google `8.8.8.8`). Does the heavy lifting on your behalf.
3. **root nameservers** — 13 logical root servers that know the addresses of TLD nameservers.
4. **TLD nameservers** — responsible for `.com`, `.org`, etc. Know who is authoritative for `github.com`.
5. **authoritative nameserver** — the final authority. Returns the actual IP.

### DNS record types

| Record | Maps | Notes |
|--------|------|-------|
| `A` | hostname → IPv4 | Most common |
| `AAAA` | hostname → IPv6 | |
| `CNAME` | hostname → hostname (alias) | Cannot be set at zone apex (`@`). Adds a resolution hop. |
| `MX` | domain → mail server hostname | Priority value for fallback |
| `TXT` | domain → arbitrary text | SPF, DKIM, domain verification |
| `NS` | domain → nameserver | Delegates authority for the zone |
| `SOA` | — | Zone metadata: primary NS, contact email, serial, TTL defaults |
| `PTR` | IP → hostname | Reverse DNS |
| `SRV` | service → host + port | Used by service discovery (Kubernetes, etcd) |

### TTL trade-offs

- **Low TTL (30–300s):** Changes propagate fast. More DNS queries = more load on resolvers and latency per lookup. Use when you're about to do a migration or failover.
- **High TTL (3600–86400s):** Fewer queries, less latency. Stale on IP changes. Lower TTL well in advance (24–48h) before any planned IP change.

### CNAME vs A at the apex

You cannot set a `CNAME` for `example.com` (the zone apex) because an apex must also have an `SOA` and `NS` record — and a `CNAME` would conflict (RFC 1034). Subdomains (`api.example.com`) can be `CNAME`s.

Workarounds:
- **Flatten to A** — the DNS provider (Cloudflare, Route 53) resolves the CNAME target at query time and returns the A record. Called `ALIAS` or `ANAME`.
- **Use a redirect** — redirect `example.com` → `www.example.com`.

### DNS-based load balancing

DNS can return multiple A records (round-robin). Clients pick one, usually the first. This is coarse-grained — DNS has no knowledge of server health or load. Real load balancing requires a load balancer, not DNS alone.

Route 53 weighted routing, latency-based routing, and health-checked failover are more sophisticated versions of DNS-level routing.

**Why DNS matters for system design:**

- **latency** — uncached DNS lookups add round-trip time before your first byte is sent. CDNs and browser pre-fetching (`dns-prefetch`) mitigate this.
- **DNS-based load balancing** — returning multiple A records for a domain distributes traffic across servers.
- **failover** — changing a DNS record to point to a backup server is a coarse but effective failover mechanism.
- **split-horizon DNS** — internal and external queries for the same name resolve to different IPs (useful for private services).

---

## HTTP and HTTPS

### HTTP versions

**HTTP (HyperText Transfer Protocol)** is the application-layer protocol that governs how clients and servers exchange data on the web. It is stateless — each request is independent; the server holds no memory of previous requests by default.

**Request structure**

```
GET /api/users/42 HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer eyJhbGci...

[optional body]
```

**Response structure**

```
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=3600

{"id": 42, "name": "Chakshu"}
```

```
HTTP/1.0  — one TCP connection per request. Teardown after response.
HTTP/1.1  — persistent connections (keep-alive), but requests on a single connection are sequential. Browsers open multiple parallel connections to work around this (head-of-line blocking at app layer).
HTTP/2    — multiplexing: many requests over one TCP connection concurrently. Binary framing instead of plaintext. Header compression (HPACK). Server push.
           TCP head-of-line blocking still exists (one lost packet stalls all streams).
HTTP/3    — runs over QUIC (UDP). Eliminates TCP HoL blocking. Faster connection setup. Better for lossy networks.
           Connection migration: switch from Wi-Fi to cellular without reconnecting.
```

### Methods and Semantics

| Method | Idempotent | Safe | Typical use |
|--------|-----------|------|-------------|
| GET | yes | yes | Fetch resource |
| HEAD | yes | yes | Fetch headers only (for caching checks) |
| POST | no | no | Create resource, trigger action |
| PUT | yes | no | Replace entire resource |
| PATCH | no | no | Partial update |
| DELETE | yes | no | Delete resource |
| OPTIONS | yes | yes | CORS preflight, capabilities |

**Idempotent** — repeating the same request produces the same server state. Retrying a safe DELETE is fine; retrying a payment POST is dangerous.

### Status Codes

```
1xx Informational
    101 Switching Protocols
2xx Success
    200 OK              — standard success
    201 Created         — resource created, Location header points to it
    202 Accepted        — async: request accepted, not yet processed
    204 No Content      — success with no body (common for DELETE, PATCH)

3xx Redirection
    301 Moved Permanently — clients and crawlers cache this permanently
    302 Found             — temporary redirect, not cached
    304 Not Modified      — conditional GET: cached version still valid
    307 Temporary Redirect — same as 302 but preserves method (no GET downgrade)
    308 Permanent Redirect — same as 301 but preserves method

4xx Client Error
    400 Bad Request       — malformed request syntax
    401 Unauthorized      — not authenticated (no or invalid credentials)
    403 Forbidden         — authenticated but not authorised
    404 Not Found         — resource does not exist
    405 Method Not Allowed
    409 Conflict          — state conflict (e.g. duplicate create)
    422 Unprocessable     — syntactically valid but semantically wrong
    429 Too Many Requests — rate limited. Include Retry-After header.

5xx Server Error
    500 Internal Server Error — unhandled exception
    502 Bad Gateway           — upstream returned invalid response
    503 Service Unavailable   — server down or overloaded. Include Retry-After.
    504 Gateway Timeout       — upstream timed out
```

**Interview trap:** 401 vs 403. `401` means the client is not authenticated — send credentials or log in. `403` means the client is authenticated but lacks permission.

### key headers

```
Request headers:
  Authorization: Bearer <token>
  Content-Type: application/json
  Accept: application/json
  If-None-Match: "abc123"        (conditional GET — 304 if ETag matches)
  If-Modified-Since: <date>
  X-Request-ID: <uuid>           (distributed tracing)

Response headers:
  Content-Type: application/json; charset=utf-8
  Cache-Control: max-age=3600, must-revalidate
  ETag: "abc123"
  Location: /api/users/42        (for 201 Created)
  Retry-After: 60                (for 429, 503)
  Strict-Transport-Security: max-age=31536000; includeSubDomains
  X-Content-Type-Options: nosniff
  X-Frame-Options: DENY
```

### Caching

```
Cache-Control: no-store            — never cache, never serve from cache
Cache-Control: no-cache            — cache, but always revalidate before serving
Cache-Control: max-age=3600        — cache for 1 hour
Cache-Control: s-maxage=600        — CDN-specific TTL (overrides max-age for shared caches)
Cache-Control: private             — browser can cache, CDN must not
Cache-Control: public              — CDN may cache

ETag / If-None-Match:              — conditional request cycle
  Server sends:    ETag: "v42"
  Client sends:    If-None-Match: "v42"
  Server responds: 304 Not Modified (if unchanged) or 200 + new ETag
```

### HTTPS and TLS

HTTPS = HTTP over TLS. TLS handles authentication (via certificates), encryption (symmetric after handshake), and integrity (MAC on every record).

**TLS 1.3 handshake (1-RTT):**

```
Client                              Server
  │                                   │
  │─── TCP SYN ──────────────────────>│
  │<── TCP SYN-ACK ───────────────────│
  │─── TCP ACK ──────────────────────>│  (TCP done: ~1 RTT)
  │                                   │
  │─── ClientHello ──────────────────>│  (cipher suites, key_share: ECDHE pubkey)
  │<── ServerHello ───────────────────│  (chosen cipher, server ECDHE pubkey)
  │<── Certificate ───────────────────│  (server cert chain)
  │<── CertificateVerify ─────────────│  (proof server holds private key)
  │<── Finished (encrypted) ──────────│
  │                                   │
  │    Both derive session key from ECDHE. Private key never transmitted.
  │                                   │
  │─── Finished (encrypted) ─────────>│  (TLS done: total ~2 RTT from TCP start)
  │                                   │
  │─── GET /api/data (encrypted) ────>│
  │<── 200 OK (encrypted) ────────────│
```

TLS 1.2 required 2 RTTs for the handshake; TLS 1.3 reduced to 1 RTT by sending the key share in the first message. TLS 1.3 also added **0-RTT resumption** for reconnecting clients (session ticket), though 0-RTT has replay attack implications.

**ECDHE (Elliptic Curve Diffie-Hellman Ephemeral)** — each session uses a fresh key pair. Even if the server's long-term private key is later compromised, past sessions cannot be decrypted. This property is called **forward secrecy**.

**Certificate chain:**
```
Root CA (in your OS/browser trust store)
  └── Intermediate CA (cross-signed by Root)
       └── Server cert (issued for github.com, signed by Intermediate)
```

The client validates the signature chain from the server cert up to a trusted root. If any cert is expired, self-signed without a trusted root, or issued for the wrong hostname — the browser rejects it.

**HSTS (HTTP Strict Transport Security):**
```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```
Tells the browser to always use HTTPS for this domain, even if the user types `http://`. `preload` allows inclusion in browsers' built-in HSTS list — effective before the first connection.

**SNI (Server Name Indication)**

A TLS extension that lets a server host multiple certificates on the same IP, since the hostname is sent unencrypted during the handshake.

### Cookies and security headers

```go
// Setting a secure cookie in Go
http.SetCookie(w, &http.Cookie{
    Name:     "session_id",
    Value:    sessionToken,
    HttpOnly: true,    // not accessible from JS — XSS mitigation
    Secure:   true,    // HTTPS only
    SameSite: http.SameSiteLaxMode,  // CSRF mitigation
    MaxAge:   86400,
    Path:     "/",
})
```

`SameSite` modes:
- `Strict` — cookie never sent on cross-site requests at all (breaks OAuth flows).
- `Lax` (default in modern browsers) — sent on top-level navigations (GET), not on sub-resource requests from other sites.
- `None` — always sent, but requires `Secure: true`. Used for legitimate third-party embeds.

---

## Interview cheat sheet

**"Design a URL shortener"** — DNS, HTTP redirects (301 vs 302 — 301 is cached by browsers, 302 lets you track clicks), distributed ID generation.

**"How would you reduce API latency globally?"** — CDN (edge caching), HTTP/2 multiplexing, Keep-Alive connection pools, gzip/br compression, DNS latency-based routing, regional deployments.

**"Walk me through what happens when you type a URL in a browser."**
```
1. Browser checks DNS cache → OS resolver → recursive resolver → full DNS chain
2. TCP connection to resolved IP (3-way handshake)
3. TLS handshake (1 RTT for TLS 1.3)
4. HTTP GET request
5. Server processes and responds
6. Browser parses HTML, fires sub-requests (CSS, JS, images)
7. Subsequent requests reuse the TCP/TLS connection (keep-alive / HTTP/2 streams)
```

Every layer — DNS, IP routing, TCP, TLS, HTTP — is a separate abstraction with its own failure modes. When debugging latency or failures in production, knowing which layer the problem lives in is the first step.

**"Difference between 401 and 403"** — 401: not authenticated. 403: authenticated, not authorised.

**"How does HTTPS prevent MITM?"** — Server cert is signed by a CA the client trusts. ECDHE gives forward secrecy. The private key is never transmitted.

**"What happens if DNS TTL is too high during a migration?"** — Clients keep connecting to the old IP for up to TTL seconds. Mitigation: lower TTL 24–48h before the migration, migrate, raise TTL again after confirming stability.

---

## References

- [MDN HTTP docs](https://developer.mozilla.org/en-US/docs/Web/HTTP)
- [RFC 7230 — HTTP/1.1 Message Syntax](https://datatracker.ietf.org/doc/html/rfc7230)
- [RFC 8446 — TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [RFC 1034 — DNS Concepts](https://datatracker.ietf.org/doc/html/rfc1034)
- [High Performance Browser Networking — Ilya Grigorik](https://hpbn.co) (chapters on HTTP/2, TLS, QUIC)
- [Cloudflare Learning Center — DNS](https://www.cloudflare.com/learning/dns/what-is-dns/)
- [IANA — HTTP Status Code Registry](https://www.iana.org/assignments/http-status-codes/http-status-codes.xhtml)