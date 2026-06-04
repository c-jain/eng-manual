# Rate Limiting

## Table of Contents

- [What Is Rate Limiting](#what-is-rate-limiting)
- [Why It Exists](#why-it-exists)
- [Token Bucket](#token-bucket)
- [Leaky Bucket](#leaky-bucket)
- [Sliding Window](#sliding-window)
- [Algorithm Comparison](#algorithm-comparison)
- [Distributed Rate Limiting](#distributed-rate-limiting)
- [Where It Lives in a System](#where-it-lives-in-a-system)
- [Headers and Client UX](#headers-and-client-ux)
- [Trade-offs](#trade-offs)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [How to Remember It](#how-to-remember-it)
- [References](#references)

---

## What Is Rate Limiting

Rate limiting is a control mechanism that restricts how many requests a client can make to a system within a given time window. Requests beyond the limit are either rejected (HTTP 429) or queued.

**Why it is called that:** "Rate" = requests per unit time. "Limiting" = enforcing a ceiling on that rate. The terminology comes from traffic engineering and telephony, where it was used to police bandwidth usage.

---

## Why It Exists

Without rate limiting, a single misbehaving or malicious client can exhaust server resources, starve other clients, and trigger cascading failures across the system.

Three concrete problems it solves:

- **Abuse and security** — brute-force login attempts, credential stuffing, scraping, DDoS amplification
- **Resource fairness** — prevents one tenant from consuming disproportionate shared capacity
- **Cost control** — third-party services (LLM APIs, SMS gateways, payment processors) charge per call; runaway clients create runaway bills

**Problems rate limiting itself brings:**

- Adds latency on the hot path (especially with a remote store like Redis)
- Requires distributed coordination in multi-node deployments — a non-trivial engineering problem
- Wrong limits harm legitimate users; too loose limits fail to protect

---

## Token Bucket

### What It Is

A bucket holds a fixed maximum number of tokens. Each request consumes one token. Tokens refill at a fixed rate up to the bucket capacity. If there are no tokens, the request is rejected.

**Why it is called that:** Direct physical analogy — a bucket of tokens from telephony/networking, codified in RFC 2697 and RFC 2698.

### Internals

```
state per client: { tokens float64, last_refill time.Time }

on request arrival:
  elapsed   = now - last_refill
  tokens    = min(capacity, tokens + elapsed * refill_rate)
  last_refill = now

  if tokens >= 1:
    tokens -= 1
    allow
  else:
    reject (429)
```

The token count is a float so partial tokens accumulate correctly between refills.

### Diagram

```
Refill (+rate/sec) --> [ T T T T T T T T   ] (max capacity)
                                |
                          request arrives
                                |
                     tokens >= 1? --YES--> allow, tokens--
                                 --NO --> reject (429)
```

### Key Property

Allows **bursting** — if a client has been idle, tokens accumulate and it can fire a burst of requests instantly up to `capacity`. After the burst, it is throttled to the refill rate.

This matches real user behaviour: users are bursty, not uniformly spaced.

### Problems It Brings

- State per client — memory scales with distinct client count
- Distributed deployments need shared state (e.g., Redis) — coordination adds latency
- Clock skew across nodes can cause subtle inaccuracies in token accumulation

---

## Leaky Bucket

### What It Is

Requests pour into a queue (the bucket). A background process drains the queue at a fixed rate — one request per fixed interval. If the queue is full, incoming requests overflow and are rejected.

**Why it is called that:** Water pours in irregularly but drips out at a constant rate — a real leaky bucket.

### Internals

```
state per client: { queue []Request, last_drain time.Time }

on request arrival:
  if len(queue) < queue_capacity:
    enqueue(request)
  else:
    reject (429)

background (every drain_interval):
  if len(queue) > 0:
    process(dequeue())
```

### Diagram

```
Requests (bursty) --> [ req req req req req ] (queue, capacity N)
                                  |
                        drain: 1 req / interval
                                  |
                       Downstream (perfectly smooth output)
```

### Key Property

Output is **perfectly smooth** — downstream always receives requests at exactly the drain rate, regardless of how bursty the input is. No bursting allowed.

### Problems It Brings

- Introduces queuing latency — legitimate requests wait even when the downstream has capacity
- Queue capacity is a hard design choice: too small → too many false rejects; too large → too much latency
- Not well-suited for user-facing APIs where clients expect low, predictable latency

---

## Sliding Window

Fixed window is the naive baseline; sliding window exists to fix its boundary problem.

### Fixed Window (Baseline — and Its Flaw)

Divide time into fixed buckets (e.g., 1-second slots). Allow up to N requests per bucket. Simple O(1) state.

**The boundary problem:**

```
Window:  |------- 0s to 1s -------|------- 1s to 2s -------|
                   N allowed                N allowed

Client sends N requests at t=0.9s  (within first window)
Client sends N requests at t=1.1s  (within second window)

Both windows see N requests → both pass.
But 2N requests arrived in a 200ms span — a real violation.
```

### Sliding Window Log

Store a timestamped log per client. On each request, drop entries older than the window, count remaining, compare to limit.

```
window = 1s, limit = 5
log = [t1, t2, t3, t4]

on request at t_now:
  drop all t where t < t_now - window
  if len(log) < limit:
    log.append(t_now)
    allow
  else:
    reject
```

Accurate but memory grows proportionally to request rate — impractical at high throughput.

### Sliding Window Counter (The Practical Hybrid)

Approximate a true sliding window using two fixed-window counters (current and previous) with a weighted blend. O(1) memory.

```
estimated_count = prev_count * (1 - elapsed_in_current_window / window_size)
                + current_count
```

**Worked example:**

```
window = 1s, limit = 10
prev window: 8 requests
current window is 70% elapsed, has 3 requests so far

estimated = 8 * (1 - 0.7) + 3
          = 8 * 0.3 + 3
          = 2.4 + 3
          = 5.4  → below 10 → allow
```

**Diagram:**

```
Sliding window at t = 1.7s  (window = 1s)

      |<-------------- 1s window ------------->|
      0.7s         1.0s                       1.7s
        |            |                           |
 [-- prev window --][ ----- current window ----- ]
  weight: 30% (0.3)      weight: 100% (1.0)
```

**Problems it brings:**

- Log variant: memory proportional to request rate — does not scale
- Counter variant: approximate — can allow slightly over the limit at exact window edges (bounded error, typically < a few percent)
- All variants need shared state in distributed deployments

---

## Algorithm Comparison

| Property | Token Bucket | Leaky Bucket | Fixed Window | Sliding Window Log | Sliding Window Counter |
|---|---|---|---|---|---|
| Allows bursting | Yes | No | Yes (boundary exploit) | Yes | Yes (approximate) |
| Output smoothness | Variable | Perfectly smooth | Variable | Variable | Variable |
| Memory per client | O(1) | O(queue size) | O(1) | O(requests in window) | O(1) |
| Accuracy | Exact | Exact | Poor at boundaries | Exact | Approximate |
| Best for | User-facing APIs | Downstream protection | Simple internal tools | Low-traffic exact limiting | High-traffic production APIs |

---

## Distributed Rate Limiting

Single-server rate limiting is straightforward: state lives in memory. Distributed systems require coordination.

### Option 1: Centralised Store (Redis + Lua)

All nodes check and update a shared Redis counter. Use a Lua script to make the read-increment-write atomic (Redis executes Lua scripts as a single command).

```lua
-- Lua script (runs atomically in Redis)
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local count = redis.call("INCR", key)
if count == 1 then
  redis.call("EXPIRE", key, window)
end
if count > limit then
  return 0  -- reject
end
return 1    -- allow
```

This is the most common production approach. Redis latency is sub-millisecond and it becomes the single source of truth.

**Problems:** Redis is now a critical dependency — if it goes down, the rate limiter fails. Design a fallback policy: fail open (allow all) or fail closed (reject all).

### Option 2: Local Approximate Limiting

Each node enforces `limit / N` (N = node count). No coordination. Fast. Breaks down if traffic is not evenly distributed across nodes.

### Option 3: Sticky Routing

Route each client consistently to one node (via consistent hashing on client ID). Local rate limiting then works correctly. Does not survive node failure gracefully.

### Redis Sorted Set (Sliding Window Log in Redis)

```
ZREMRANGEBYSCORE client_key 0 (now - window_ms)
ZADD client_key now now
ZCARD client_key  → compare to limit
EXPIRE client_key window_seconds
```

Accurate but memory-heavy at high request rates.

---

## Where It Lives in a System

Rate limiting sits at the API Gateway or a dedicated middleware layer — upstream of the service. The service never wastes resources processing a request that will be rejected.

```
Client
  |
  v
[API Gateway / Reverse Proxy]
  |
  +--> Rate Limiter checks Redis
  |         |
  |    allow/reject
  |
  v (if allowed)
[Upstream Service]
```

Common placements:
- **API Gateway** (Kong, AWS API Gateway, Nginx) — most common
- **Service mesh sidecar** (Envoy) — for east-west (service-to-service) traffic
- **Application middleware** — for fine-grained per-endpoint limits

---

## Headers and Client UX

A well-behaved rate limiter communicates its state to clients:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1717459200     (Unix timestamp when window resets)
Retry-After: 30                   (seconds to wait, on 429)
```

Clients that respect `Retry-After` implement exponential backoff with jitter and avoid thundering-herd retries.

---

## Trade-offs

- **Token bucket vs. leaky bucket:** Token bucket is better for user-facing APIs (bursting is natural). Leaky bucket is better for protecting a downstream system that can only handle a fixed throughput.
- **Simplicity vs. accuracy:** Fixed window is simplest but has boundary exploits. Sliding window counter is accurate enough and still O(1). Sliding window log is exact but memory-heavy.
- **Centralised vs. distributed state:** Redis is accurate but is a dependency and adds latency. Local limiting is fast but inaccurate under uneven traffic.
- **Fail open vs. fail closed:** If the rate limiter itself fails, fail open (allow traffic through) to avoid being a single point of failure for the entire system; fail closed only when the rate limiter is protecting against security threats.

---

## Interview Cheat Sheet

**Strong signal phrases:**

- "I'd use token bucket for user-facing APIs — it allows natural bursting while enforcing a sustained rate ceiling"
- "For smooth downstream protection I'd use leaky bucket — the output rate is perfectly controlled"
- "Fixed window has a 2x boundary exploit; sliding window counter fixes that with just two counters and a weighted blend"
- "In a distributed system I'd use Redis with a Lua script to make the check-and-decrement atomic — no race condition between read and write"
- "I'd expose X-RateLimit-Remaining and Retry-After headers so clients can back off gracefully"
- "If Redis goes down I'd fail open — a brief window of unthrottled traffic is better than taking down the entire API"

**Red flags:**

- Proposing fixed window without acknowledging the boundary exploit
- Saying "just use Redis" without mentioning the atomic Lua script or what data structure to use
- Not mentioning distributed coordination at all
- Confusing token bucket (bursting allowed) with leaky bucket (output smoothed)
- Forgetting that the rate limiter should sit upstream of the service, not inside it

---

## How to Remember It

**Token Bucket = Tokens in a Jar**
You earn tokens over time. You spend tokens to act. Burst when you have savings.

**Leaky Bucket = Garden Hose Nozzle**
Water (requests) pours in fast, but the nozzle (drain) controls the steady trickle out. Excess spills over.

**Sliding Window = Moving Average**
Think of it like a rolling 60-second average on a dashboard — the window slides forward with time, always looking back exactly one period.

**Mnemonic for choosing:**
- User-facing API → **Token** (users are bursty)
- Protecting downstream → **Leaky** (smooth the flow)
- Accuracy + low memory → **Sliding Window Counter**

---

## References

- [RFC 2697 — A Single Rate Three Color Marker](https://datatracker.ietf.org/doc/html/rfc2697) — token bucket specification
- [Redis Rate Limiting Patterns](https://redis.io/learn/develop/dotnet/aspnetcore/rate-limiting/sliding-window) — sliding window with Redis sorted sets
- Alex Xu, *System Design Interview Vol. 1*, Chapter 4 — Rate Limiter
- [Cloudflare Blog: How we built rate limiting](https://blog.cloudflare.com/counting-things-a-lot-of-different-things/) — sliding window counter in production