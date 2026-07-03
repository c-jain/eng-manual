---
Status: 🌳 Evergreen
Created: 2026-07-03
Last Updated: 2026-07-03
---

# Design A Rate Limiter

## Table Of Contents

1. [What It Is And Why It Exists](#what-it-is-and-why-it-exists)
2. [Why It Is Called "Rate Limiter"](#why-it-is-called-rate-limiter)
3. [Step 1 — Clarify Requirements](#step-1--clarify-requirements)
4. [Step 2 — Estimate Scale](#step-2--estimate-scale)
5. [Step 3 — High-Level Design](#step-3--high-level-design)
6. [Step 4 — Deep Dive: Race Conditions And Atomicity](#step-4--deep-dive-race-conditions-and-atomicity)
7. [Step 4 — Deep Dive: Synchronization Across Instances](#step-4--deep-dive-synchronization-across-instances)
8. [Step 4 — Deep Dive: Rules Engine And Configuration](#step-4--deep-dive-rules-engine-and-configuration)
9. [Step 4 — Deep Dive: Multi-Data-Center Deployment](#step-4--deep-dive-multi-data-center-deployment)
10. [Step 5 — Trade-Offs](#step-5--trade-offs)
11. [Step 6 — Wrap-Up](#step-6--wrap-up)
12. [Edge Cases And Extensions](#edge-cases-and-extensions)
13. [Problems This Design Brings](#problems-this-design-brings)
14. [How To Remember This](#how-to-remember-this)
15. [What Interviewers Actually Look For](#what-interviewers-actually-look-for)
16. [Interview Cheat Sheet](#interview-cheat-sheet)
17. [References](#references)

## What It Is And Why It Exists

"Design a rate limiter" asks you to build the **component** that decides, for every incoming request, whether to let it through or reject it — correctly, at scale, without becoming a bottleneck itself.

This is a different question from "what algorithm do I use to rate limit?" — that question is already fully answered in `hld/rate-limiting.md` (token bucket, leaky bucket, sliding window, the Redis+Lua atomic pattern). This case study assumes you already know the algorithm and asks the harder systems question that sits around it: where does this component live in the request path, how does it stay correct when there are dozens of instances of it running at once, how are its rules configured and changed without a redeploy, and what happens when traffic spans multiple data centers.

The problems this system solves are the same ones the fundamentals file names — abuse prevention, resource fairness, cost control — but the case study's real test is whether you can design a **correct, horizontally-scaled, configurable** system around whichever algorithm you pick, not just whether you know the algorithm.

## Why It Is Called "Rate Limiter"

Covered in full in `hld/rate-limiting.md` — "rate" (requests per unit time) and "limiting" (enforcing a ceiling on that rate), from traffic-engineering and telephony terminology. Not re-derived here. One naming nuance specific to this case study: in production systems this component is also called a **throttler**, and "throttling" specifically emphasizes the action of slowing a client down rather than the ceiling itself — the two terms are used interchangeably in most interviews.

## Step 1 — Clarify Requirements

**Functional:**

- Accurately throttle excessive requests to a protected resource (an API, a login endpoint, a third-party call).
- Support **configurable rules** — different limits per user tier, per endpoint, per API key, per IP.
- Communicate throttling clearly to the client (`429` status, rate-limit headers — full definitions in `hld/rate-limiting.md`, just consumed here).

**Non-functional:**

- **Low added latency** — this component sits on the hot path of *every* request; it cannot become a bottleneck itself.
- **High availability** — the limiter must never be the reason the protected system goes down.
- **Horizontally scalable** — must stay correct as the number of rate-limiter instances grows, not just as traffic grows.
- **Accurate enough** — the more accurate the count needs to be, the more coordination it costs. Checking one shared, authoritative counter (Redis) on every request gives the most accurate answer, but adds a network round-trip to every single check. Two cheaper alternatives trade some accuracy for speed, each for a different reason: keeping the counter local to each rate-limiter instance avoids a network call entirely, but only stays correct if one instance sees all of a given user's traffic (Deep Dive B); replicating counters between data centers asynchronously instead of synchronously keeps per-request checks fast, but lets a user briefly exceed the *global* limit while replication catches up (Deep Dive D). The real test is naming this spectrum explicitly, rather than assuming zero latency and zero error are both free at once.

**Explicitly out of scope** (state this out loud in an interview):

> "I'll focus on the rate limiter's own architecture — placement, distributed correctness, rule configuration, multi-region behavior. I'll treat the specific throttling algorithm (token bucket vs. sliding window) as already solved — happy to go deep on that separately — and I'll assume client identity (user ID, API key, IP) is already resolved by the time a request reaches the limiter, unless you'd like me to design that too."

## Step 2 — Estimate Scale

The number that matters here isn't raw QPS in the abstract — it's **how many rate-limiter instances are concurrently checking the same key**, because that number is the direct cause of the two hardest problems in this design.

```
API Gateway fleet size           : 100 instances (behind a load balancer)
Total incoming QPS               : 1,000,000 req/sec across the fleet
Distinct rate-limited keys       : ~10,000 active users/API-keys per second
Target added latency per check   : < 5 ms
```

**What this forces:**

- `1,000,000 QPS ÷ 100 instances ≈ 10,000 QPS per instance` — each instance independently decides "allow or reject" for requests that may share a key with requests hitting *other* instances at the exact same instant.
- If each instance kept its own local, in-memory counter, the same user could get `limit × 100` requests through — once per instance — before any single counter noticed. This is the **synchronization problem** (Step 4, Deep Dive B) — the reason "distributed counters" is the concept the roadmap calls out for this case study.
- Even with one shared store, two instances can both read "9 requests so far, limit is 10" at the same instant, both decide "allow," and both increment — 11 requests get through on a limit of 10. This is the **race condition problem** (Step 4, Deep Dive A) — a separate failure mode from synchronization, and commonly confused with it in interviews.

## Step 3 — High-Level Design

### Where The Limiter Lives

| Placement | How It Works | Problem It Brings |
|---|---|---|
| Client-side | App/browser self-throttles before sending | Trivially bypassable — a modified client ignores it entirely. Useful only as a UX courtesy, never as the actual control. |
| Middleware per service | Each service instance checks its own requests in-process | Every service reimplements (and can drift on) the same logic; no single place to change a rule. |
| API Gateway | One centralized interception point in front of all services | Single point every request must pass through — but that's exactly the point: one place to enforce and one place to change rules. |
| Sidecar (service mesh, e.g. Envoy) | A co-located process per instance, language-agnostic | Adds an operational component (the mesh) per instance; overkill if there's no mesh already in place. |

This design places the limiter at the **API Gateway** — the standard production answer, and the one that satisfies "one place to enforce, one place to reconfigure" from Step 1.

### Architecture

```
API Gateway instances (horizontally scaled, N of them)
        |
        v
Identity Resolver   -->  produces the key to check (user/API-key/IP)
        |
        v
Rules Engine        -->  matches the request path to a configured Rule
        |
        v
Rate Limiter (Allow?) -->  atomic check-and-increment against shared Redis
        |
        v
   allow or reject   (two outcomes — see scenarios below)
```

**Scenario A — Allowed:**

```
Rate Limiter says "allow"
        |
        v
Request forwarded to the upstream service, unmodified
```

**Scenario B — Rejected:**

```
Rate Limiter says "reject"
        |
        v
Gateway returns 429 directly to the client
(upstream service is never contacted — this is the whole
point: reject before spending any backend resource)
```

The four boxes in the main flow — Identity Resolver, Rules Engine, Rate Limiter, and the shared store behind it — are exactly the four things this case study needs to design. Each gets its own deep dive below.

## Step 4 — Deep Dive: Race Conditions And Atomicity

**What it is:** a race condition here is two requests each reading the counter, both deciding independently, and both writing — with no guarantee either saw the other's write. This has nothing to do with which gateway instance handles which request — it happens identically whether both requests land on the same instance (e.g. two goroutines racing) or on two different instances entirely, because the race is on the shared Redis key, not on any particular gateway process.

```
t0:  Request A reads count -> 9
t1:  Request B reads count -> 9         (same value — A hasn't written yet)
t2:  Request A decides: 9 < 10, allow
t3:  Request A writes count -> 10
t4:  Request B decides: 9 < 10, allow   (based on the stale read from t1)
t5:  Request B writes count -> 11       <- limit of 10 violated
```

Both requests read the same stale value because "read" and "write" are two separate round-trips with a gap between them, and nothing stops a second request from landing in that gap.

**The fix:** collapse the read, compare, and write into a **single atomic operation** — no gap for another request to land in. `hld/rate-limiting.md` already derives the Lua-script approach for this (Redis executes a Lua script as one indivisible command); it's reused verbatim here, not re-derived:

```go
// fixedWindowScript performs an atomic check-and-increment inside Redis so
// that "read the counter" and "increment the counter" cannot be split
// across two requests racing each other. See hld/rate-limiting.md for the
// full derivation of this script; it is reused here unmodified.
const fixedWindowScript = `
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local count = redis.call("INCR", key)
if count == 1 then
  redis.call("EXPIRE", key, window)
end
if count > limit then
  return 0
end
return 1
`
```

The Go side of this — the part an interviewer could plausibly ask to see, since it's the actual integration surface between the gateway and Redis:

```go
// RateLimiter abstracts the allow/reject decision so the request-handling
// code never needs to know whether the backing store is Redis, an
// in-memory map (for tests), or something else entirely.
type RateLimiter interface {
    // Allow reports whether the request identified by key is permitted
    // under the configured limit. It returns an error only on
    // infrastructure failure (e.g. Redis unreachable) — never as the
    // normal signal for "over the limit," which is communicated via
    // the bool.
    Allow(ctx context.Context, key string) (bool, error)
}

// RedisRateLimiter is the RateLimiter used when multiple app-server
// instances must share one source of truth for a key's request count.
type RedisRateLimiter struct {
    client     *redis.Client
    script     *redis.Script
    limit      int
    windowSecs int
}

func NewRedisRateLimiter(client *redis.Client, limit, windowSecs int) *RedisRateLimiter {
    return &RedisRateLimiter{
        client:     client,
        script:     redis.NewScript(fixedWindowScript),
        limit:      limit,
        windowSecs: windowSecs,
    }
}

// Allow runs the Lua script as a single atomic Redis command — the
// increment and the limit check happen without any other client's
// command interleaving between them.
func (r *RedisRateLimiter) Allow(ctx context.Context, key string) (bool, error) {
    redisKey := fmt.Sprintf("ratelimit:%s", key)
    result, err := r.script.Run(ctx, r.client, []string{redisKey}, r.limit, r.windowSecs).Int()
    if err != nil {
        return false, fmt.Errorf("rate limiter script: %w", err)
    }
    return result == 1, nil
}
```

A test worth naming for its teaching value: spin up many goroutines that all call `Allow` on the *same* key at the same time, with a low limit (e.g. 10 allowed out of 100 concurrent attempts), and assert that exactly `limit` of them succeed. That test is the direct, executable statement of the race condition this whole section exists to close.

**An alternative worth naming — `WATCH`/`MULTI`/`EXEC`:** this is Redis's optimistic-locking alternative to a Lua script. It has four *steps*, but not four round-trips — one of the four is purely local. ("Client" below means the gateway, acting as Redis's client — not the end user's browser or app, which never talks to Redis directly.)

1. `WATCH key` — the gateway tells Redis "notify me if this key changes before I'm done." *(1 network round-trip)*
2. `GET key` — the gateway reads the current count back into its own code. *(1 network round-trip)*
3. The gateway decides, in its own application logic, whether to allow the request and computes the new value. *(0 round-trips — purely local computation)*
4. `MULTI` ... `INCR` ... `EXEC` — the gateway queues the write and asks Redis to commit it, but only if the watched key hasn't changed since step 1. *(1 round-trip if the client library pipelines these together, which most do; 3 if it sends each separately)*

So the real total is **3 network round-trips in the common case, or up to 5 if nothing is pipelined** — either way, several, not one. If another client (i.e. another gateway instance) modified the key between steps 1 and 4, `EXEC` comes back empty and the entire sequence must be retried from scratch. Under a hot key with real concurrency, that's expensive twice over: every attempt costs several round-trips instead of one, and retries pile up exactly when the key is busiest.

A Lua script avoids this entirely by moving the read, the check, and the write *inside* Redis itself, executed as one indivisible unit in a single round-trip — there's no separate watch-read-decide-write sequence for another client to land in the middle of, so there's nothing to retry.

## Step 4 — Deep Dive: Synchronization Across Instances

**What it is:** even with atomicity solved per-check, if instance A and instance B each keep their *own* counter for the same key, correctness still fails — not because of a race, but because the two counters never talk to each other at all. A user could exhaust the limit on every single instance independently.

**Why it's a different problem from the race condition:** atomicity is about *one* shared counter being updated safely. Synchronization is about *making sure there is only one counter in the first place*. Fixing one does not fix the other — a common interview mix-up worth naming explicitly.

`hld/rate-limiting.md` already lists the three standard options (centralized store, local approximate limiting, sticky routing). Applying them to this specific design, with the numbers from Step 2:

- **Centralized store (Redis)** — chosen here. A single shared Redis (or Redis Cluster for its own scaling) means every instance, however many there are, checks the same counter. Sub-millisecond Redis latency fits comfortably inside the 5 ms budget from Step 2.
- **Sticky routing (consistent hashing on the identity key)** — routes each user consistently to one gateway instance, which can then keep a correct *local* counter with zero coordination. See `hld/consistent-hashing.md` for the mechanism. Attractive because it removes the Redis dependency entirely, but it fails to survive node failure gracefully: if the instance a user is stuck to goes down, that user's rate-limit state goes with it.
- **Local approximate limiting** — each instance enforces `limit / N`. Removes the need to coordinate *across* instances, but breaks down badly the moment traffic isn't evenly spread across instances (which it usually isn't — a legitimate user's connections aren't guaranteed to land evenly across the fleet). It also doesn't remove Deep Dive A's problem, just relocates it: the local counter is still shared across every goroutine handling concurrent requests *within* that one instance, so it still needs its own atomicity (`sync/atomic` or a mutex), just without the network round-trip.

For this design, centralized Redis wins because the latency budget comfortably absorbs it and it's the only option of the three that stays correct regardless of how unevenly traffic happens to distribute across the 100 gateway instances.

## Step 4 — Deep Dive: Rules Engine And Configuration

**What it is:** a rule maps a request to a limit — e.g., "`/api/v1/payments/*` → 5 requests per 60 seconds," "everything else under `/api/v1` → 100 per 60 seconds." The rules engine is the piece that, given an incoming request, picks the single rule that applies.

**Why it exists:** hardcoding limits into the gateway's binary means changing a limit requires a redeploy. A rules engine backed by an external config (a file, or a config service) lets limits change live — critical when a limit turns out to be wrong under real traffic and there's no time to wait for a deploy pipeline.

**Internals — the part worth showing code for:** rules can overlap (a specific path prefix and a broader one both matching the same request), so the engine needs an unambiguous tiebreak. **Longest matching prefix wins** — the more specific rule always takes precedence over the broader one.

```go
// Rule is one configured limit: N requests per WindowSecs, scoped to
// requests matching PathPrefix. Rules are typically loaded from a config
// file or config service, not hardcoded — that's what makes them
// changeable without a redeploy.
type Rule struct {
    PathPrefix string
    Limit      int
    WindowSecs int
}

// RuleProvider abstracts where rules come from (static file, config
// service, database) so the matching logic below never needs to change
// when the storage backend does.
type RuleProvider interface {
    Rules() []Rule
}

// RulesEngine picks the single most specific rule that applies to a
// given request path. "Most specific" is resolved by longest matching
// prefix — /api/v1/payments should match a payments-specific rule
// instead of falling through to a blanket /api/v1 rule.
type RulesEngine struct {
    provider RuleProvider
}

// Match returns the longest-prefix-matching rule for path, and false if
// no configured rule applies (caller decides the default: allow, or fall
// back to a global default rule).
func (e *RulesEngine) Match(path string) (Rule, bool) {
    var best Rule
    found := false
    for _, r := range e.provider.Rules() {
        if strings.HasPrefix(path, r.PathPrefix) {
            if !found || len(r.PathPrefix) > len(best.PathPrefix) {
                best = r
                found = true
            }
        }
    }
    return best, found
}
```

**Worked example:**

```
Rules configured:
  "/api/v1"           -> limit 100 / 60s
  "/api/v1/payments"   -> limit 5   / 60s

Request path: "/api/v1/payments/charge"
  Matches "/api/v1"            (prefix, length 8)
  Matches "/api/v1/payments"   (prefix, length 17)  <- longer, wins

Result: limit 5 / 60s applies, not 100 / 60s
```

**Problems it brings:**

- Overlapping rules are only safe because of the explicit longest-prefix tiebreak — an engine that used "first rule defined" instead would silently depend on config ordering, a common source of production surprises when someone reorders a config file.
- Rules must be refreshed (polled or pushed) without restarting the gateway process; a naive implementation that re-reads a file on every request pays a filesystem or network cost on the hot path — cache the parsed rules in memory and refresh on an interval or via pub/sub instead.
- A bad rule (e.g., limit accidentally set to 0) can lock out all traffic to a path instantly across every instance the moment it propagates — this is why staged/canary rollout of rule changes matters in practice, not just the matching logic itself.

## Step 4 — Deep Dive: Multi-Data-Center Deployment

**What it is:** once the gateway fleet spans multiple regions/data centers for latency reasons, a globally-distributed user's requests can land in different data centers across a short span of time. If each data center has its own Redis, the single-DC synchronization fix from Deep Dive B doesn't extend across DCs — the same user could get `limit × number_of_DCs` through, for exactly the reason described in Deep Dive B, just one level up.

**Options:**

- **One global Redis instance** — perfectly correct, but every data center now pays a cross-region round-trip on every single request, undoing the very reason multi-DC deployment exists: keeping traffic local to a region for low latency.
- **Per-DC Redis with asynchronous replication** — each DC checks its own local Redis (fast), and counters replicate to other DCs in the background (eventually consistent). A user can briefly exceed the *global* limit by the amount of traffic that lands during replication lag, bounded by that lag.
- **Geo-sticky routing** — route each identity's *entire* traffic (not just the rate-limit check) consistently to one "home" DC — typically the lowest-latency one for that user, which isn't always the physically nearest — via geo-DNS or anycast at the global load balancer. Because everything for that identity lands in the same region, the rate-limit check is naturally local too, with no extra cross-region hop. Same trade-off as single-DC sticky routing: no coordination needed, but fragile to that DC's availability.

**The trade-off to name explicitly:** most production systems choose per-DC Redis with eventual consistency. A brief, bounded overage during replication lag is a far smaller cost than adding cross-region latency to every request, forever, to prevent it. This mirrors the accuracy-vs-latency trade-off named back in Step 1 — multi-DC is that same trade-off recurring at a larger scale.

## Step 5 — Trade-Offs

| Decision | Option A | Option B | This Design Chooses |
|---|---|---|---|
| Placement | Client-side (fast, spoofable) | API Gateway (centralized, one place to reconfigure) | API Gateway |
| Atomicity | `WATCH`/`MULTI`/`EXEC` (client retry loop) | Lua script (single round-trip, no retry) | Lua script |
| Synchronization | Sticky routing (no shared store, fragile to node failure) | Centralized Redis (correct regardless of traffic distribution) | Centralized Redis |
| Multi-DC | Single global Redis (correct, adds cross-region latency) | Per-DC Redis + eventual consistency (fast, bounded overage) | Per-DC + eventual consistency |
| Rule storage | Hardcoded in binary (simple, needs redeploy to change) | Config service, cached + refreshed (changeable live) | Config service |

## Step 6 — Wrap-Up

This design places rate limiting at the API Gateway, resolves each request's identity and applicable rule, and enforces the limit with a single atomic Redis operation shared across every gateway instance — closing both the race-condition gap (atomicity) and the synchronization gap (one shared counter, not one per instance). Rules live in an external, hot-reloadable config rather than the binary, so limits can change without a redeploy. The biggest production risk is Redis itself becoming a single dependency every request now relies on — I'd address that with Redis Cluster for its own horizontal scaling and a documented fail-open policy for the rare case it's unreachable. Given more time, I'd add per-rule monitoring (rejection rate, near-limit warnings) so limits that are wrong in practice get caught before they cause a support incident.

## Edge Cases And Extensions

Every item named as out of scope in Step 1 gets a concrete answer here.

**Client identity resolution:** Step 1 deferred "how do we know which key to rate-limit on." This is a small, single-purpose interface an interviewer could reasonably ask to see written out:

```go
// Request is the minimal slice of an incoming HTTP request the identity
// resolver needs. Kept separate from net/http.Request so this package
// has zero framework dependencies.
type Request struct {
    APIKey string
    UserID string
    IP     string
}

// IdentityResolver decides which key a request is rate-limited on.
type IdentityResolver interface {
    Resolve(req Request) string
}

// PriorityIdentityResolver prefers the most specific, least spoofable
// identity available: an authenticated API key first, then a logged-in
// user ID, and only falls back to IP address for fully anonymous traffic
// (the easiest identity for an abuser to rotate, so it's the last resort).
type PriorityIdentityResolver struct{}

func (PriorityIdentityResolver) Resolve(req Request) string {
    if req.APIKey != "" {
        return "apikey:" + req.APIKey
    }
    if req.UserID != "" {
        return "user:" + req.UserID
    }
    return "ip:" + req.IP
}
```

The priority order itself is the interesting design decision, not the code: API key and user ID are both authenticated, so an abuser can't easily shed them the way they can rotate an IP address — which is why IP is the last resort, not the default.

**Throttling algorithm internals:** Step 1 deferred token bucket vs. leaky bucket vs. sliding window entirely. Fully covered in `hld/rate-limiting.md`, including the internals, diagrams, and the algorithm comparison table — intentionally not re-derived here to keep this file about the system around the algorithm, not the algorithm itself.

**Redis becomes unreachable:** already covered as fail-open vs. fail-closed in `hld/rate-limiting.md`'s Trade-Offs section — referenced, not re-derived. Worth restating which side this design picks: fail open, since an unthrottled window is preferable to taking down every upstream service the gateway protects.

## Problems This Design Brings

- **Redis is now a critical dependency for every request** — mitigated with Redis Cluster/replication for its own availability, and a fail-open policy for the rare total outage.
- **A misconfigured rule can lock out (or fully open) traffic across every instance the moment it propagates** — mitigate with staged/canary rollout of rule changes, not just correct matching logic.
- **Cross-DC eventual consistency permits a bounded overage** — an explicit, accepted trade against the alternative (a slower request on every single call).
- **Every legitimate request now pays a Redis round-trip forever** — the cost of correctness is paid on the happy path too, not just when someone is actually being throttled.

## How To Remember This

**Race condition vs. synchronization — two different gaps:** a race condition is a gap *within* one check (read and write split into two steps). Synchronization is a gap *between* checks happening on different machines that never talk to each other. Closing one does not close the other.

**Atomicity = "ask and act in the same breath."** A Lua script never lets another request interleave between the read and the write — the two happen as one indivisible statement, so there's no gap left for a race to exploit.

**Rules Engine = "more specific always wins."** Whenever two rules could both apply, the longest matching prefix is the tiebreak — never rule definition order, which is fragile to config reordering.

**Multi-DC = the Step 1 trade-off, one level up.** The same "accuracy costs latency" trade from Step 1 reappears at a larger scale: a single global store is accurate but slow; per-region stores are fast but eventually consistent.

## What Interviewers Actually Look For

| Signal | Green Flag | Red Flag |
|---|---|---|
| Placement | Names API Gateway/middleware, explains why client-side is spoofable | Assumes client-side self-throttling is sufficient |
| Concurrency | Unprompted, names the race condition and picks an atomic operation | Proposes a plain read-then-write without noticing the gap |
| Distribution | Explicitly separates the race-condition problem from the synchronization problem | Treats "use Redis" as solving both, without explaining why |
| Configurability | Mentions rules must change without a redeploy | Hardcodes limits and treats that as a detail, not a design decision |
| Multi-region | If asked, offers eventual consistency as an acceptable, bounded trade-off | Insists on a single global store without acknowledging the latency cost, or panics and has no answer |

## Interview Cheat Sheet

```
Placement:        API Gateway — one enforcement point, one reconfiguration point
                  Client-side alone is never sufficient — it's spoofable

Race condition:   read-then-write split into two steps -> two requests
                  can both pass on stale data. Fix: atomic op (Lua script,
                  one round-trip, no retry loop) beats WATCH/MULTI/EXEC
                  (client-side retry loop) under real contention.

Synchronization:  per-instance local counters don't sum correctly across
                  N instances. Fix: centralized shared store (Redis) —
                  correct regardless of traffic distribution across
                  instances. Alternative: sticky routing (no shared
                  store, fragile to node failure).

Rules engine:     rules external to the binary -> changeable without a
                  redeploy. Overlapping rules -> longest-prefix-match
                  wins, never definition order.

Multi-DC:         same accuracy-vs-latency trade as Step 1, one level up.
                  Per-DC store + eventual consistency (bounded overage) is
                  the standard production answer over one global store.

Failure mode:     fail open (allow traffic) — the limiter should never be
                  why the entire system goes down.
```

## References

- Alex Xu, *System Design Interview* Vol. 1, Chapter 4 — Design a Rate Limiter
- `hld/rate-limiting.md` — algorithm internals, headers, fail-open/fail-closed trade-off (referenced throughout, not re-derived)
- `hld/consistent-hashing.md` — the mechanism behind sticky routing
- [Redis: EVAL and Scripting](https://redis.io/docs/latest/develop/interact/programmability/eval-intro/) — atomicity guarantee for Lua scripts
- ByteByteGo Blog — https://blog.bytebytego.com