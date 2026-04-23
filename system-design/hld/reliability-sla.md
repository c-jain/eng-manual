# SLAs, SLOs, and SLIs — Reliability Engineering Basics

## Table of Contents

- [Why This Exists](#why-this-exists)
- [The Three Terms](#the-three-terms)
  - [SLI — Service Level Indicator](#sli--service-level-indicator)
  - [SLO — Service Level Objective](#slo--service-level-objective)
  - [SLA — Service Level Agreement](#sla--service-level-agreement)
- [How They Relate](#how-they-relate)
- [Error Budget](#error-budget)
- [Availability and the Nines](#availability-and-the-nines)
- [Common SLIs by Service Type](#common-slis-by-service-type)
- [Choosing Good SLIs](#choosing-good-slis)
- [Trade-offs and Pitfalls](#trade-offs-and-pitfalls)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

---

## Why This Exists

As systems scale, "the service should be reliable" is not an actionable statement. Teams need a shared, precise language to define what reliability means, measure it, and communicate it — both internally and with customers.

Google's SRE (Site Reliability Engineering) practice, codified in the [SRE Book](https://sre.google/sre-book/), introduced a formal vocabulary: **SLI → SLO → SLA**. The idea is to treat reliability as a feature with measurable targets, not a vague aspiration.

This matters for system design because every architectural decision — replication factor, retry logic, circuit breakers, data durability guarantees — should ultimately trace back to an SLO.

---

## The Three Terms

### SLI — Service Level Indicator

**What it is:** A quantitative measurement of some aspect of your service's behavior. It is a ratio, expressed as a fraction of good events over total events, usually as a percentage.

**Why "indicator":** An indicator is something that shows the current state. An SLI is the raw signal — the thermometer reading, not the target temperature.

```
SLI = (good events / total events) × 100
```

Examples:
- Availability: `(successful HTTP requests / total HTTP requests) × 100`
- Latency: percentage of requests served in under 200 ms
- Error rate: `(5xx responses / total responses) × 100`
- Throughput: requests served per second
- Durability: percentage of written objects successfully read back

**What it brings:** An SLI is only useful if it correlates with user experience. A bad SLI (e.g., CPU utilization) can be green while users are suffering. Choosing the right SLI is the hard part.

---

### SLO — Service Level Objective

**What it is:** A target value or range for an SLI, over a defined time window. It is the internal goal your team commits to.

**Why "objective":** An objective is an internal goal — something you aim for but have not promised to a customer yet.

```
SLO = SLI target + time window

Example: 99.9% of requests succeed, measured over a rolling 30-day window
```

The time window matters significantly:
- A **rolling window** (last 30 days) gives continuous feedback and penalises sustained degradation
- A **calendar window** (this month) can mask problems at month boundaries and makes planning easier

SLOs should be set tighter than SLAs. If your SLA promises 99.9%, your internal SLO might be 99.95% — so you get warned before you breach the customer contract.

**What it brings:** SLOs force a conversation about how reliable is "reliable enough". Too tight → expensive over-engineering. Too loose → poor user experience. The right SLO is the least reliable your users will still be happy with.

---

### SLA — Service Level Agreement

**What it is:** A legal/business contract between a service provider and a customer that includes consequences for not meeting agreed availability or performance targets.

**Why "agreement":** An agreement is bilateral — both parties sign it. There are consequences for breach (credits, refunds, contract termination).

Example SLA clause: "If monthly uptime drops below 99.9%, customers receive a 10% service credit."

SLAs are almost always looser than internal SLOs. If the SLA is 99.9%, the internal SLO should be 99.95% or higher — you want your own alerting to fire well before you owe anyone a credit.

**What it brings:** SLAs create business accountability, but they are a lagging signal. By the time you breach an SLA, users have already had a bad experience. This is why SLOs are the primary operational tool, not SLAs.

---

## How They Relate

```
[Raw Measurement]       [Internal Target]     [External Contract]
     SLI          --->       SLO         --->       SLA

"What we measure"    "What we aim for"    "What we promised"
```

The flow is always from measurement → goal → contract. You cannot define a meaningful SLO without a well-defined SLI to back it. You cannot sign a credible SLA without an SLO you are already meeting.

---

## Error Budget

**What it is:** The amount of unreliability you are allowed before breaching your SLO.

```
Error budget = 1 - SLO (expressed over the measurement window)
```

For a 30-day window:

| SLO      | Allowed Downtime / Month |
|----------|--------------------------|
| 99%      | ~7.3 hours               |
| 99.9%    | ~43.8 minutes            |
| 99.95%   | ~21.9 minutes            |
| 99.99%   | ~4.38 minutes            |
| 99.999%  | ~26.3 seconds            |

**Why it matters:** The error budget turns reliability into a shared team resource. It answers two critical questions:

1. **Can we ship this risky change?** → Only if there is budget remaining
2. **Why are we moving slowly?** → Because we burned the budget and must restore reliability first

When the error budget is exhausted:
- New feature releases freeze
- Engineering focus shifts entirely to reliability improvements
- This prevents the classic conflict between dev velocity and ops stability

**Example policy:**
- Budget > 50% remaining → Ship freely, run experiments
- Budget between 10–50% → Ship with extra caution, no risky experiments
- Budget < 10% → Freeze releases, all hands on reliability

Error budgets make the reliability vs. velocity tension explicit and policy-driven instead of a recurring argument.

---

## Availability and the Nines

Availability is the most commonly cited SLI. "Five nines" is shorthand for 99.999%.

```
Availability (%) = (Uptime / (Uptime + Downtime)) × 100
```

Achieving higher nines gets disproportionately expensive:

```
99%     → tolerate ~87.6 hours/year of downtime
99.9%   → tolerate ~8.76 hours/year
99.99%  → tolerate ~52.6 minutes/year   ← requires redundancy + fast failover
99.999% → tolerate ~5.26 minutes/year   ← extremely costly, rare outside critical infra
```

The reason five nines is rare: planned maintenance windows alone can consume a 99.99% budget. Achieving 99.999% means zero-downtime deploys, multi-region active-active replication, sub-second failover, and no single point of failure anywhere in the stack.

---

## Common SLIs by Service Type

Different services have different meaningful SLIs. Choosing the wrong one (one that does not reflect user pain) is a common mistake.

**User-facing request/response services (APIs, web servers):**
- Availability: fraction of successful requests
- Latency: fraction of requests served within a threshold (p50, p95, p99)
- Error rate: fraction of requests returning 5xx

**Storage systems (databases, object stores):**
- Durability: fraction of written objects that can be successfully read back
- Availability: fraction of reads/writes that succeed
- Latency: read/write p99

**Data pipelines (streaming, batch):**
- Freshness: how recently was the data updated (staleness)
- Correctness: fraction of records processed without errors
- Coverage: fraction of expected data that arrived
- Throughput: events processed per second

**Why percentiles over averages for latency:**

Averages hide tail latency. A service with a p99 of 5 seconds is harming 1% of users — at 1 million requests/day that is 10,000 bad experiences. Always use percentiles (p50, p95, p99, p99.9) for latency SLIs.

```
Request distribution (not to scale):

p50 ──────────────────────|   200ms   ← typical user
p95 ──────────────────────────────|   800ms
p99 ──────────────────────────────────────|   2000ms  ← tail latency
```

---

## Choosing Good SLIs

A good SLI:
- **Correlates with user happiness** — if the SLI is green, users are happy; if it degrades, users notice
- **Is measurable** — can be instrumented reliably at the service boundary
- **Is actionable** — a breach triggers a known remediation path
- **Avoids gaming** — hard to make look good by excluding legitimate bad events

A common anti-pattern is choosing an SLI that is easy to measure but does not reflect user experience. CPU utilization and request rate are easy to measure, but neither directly tells you whether users are succeeding.

The "happy path" framing: write the SLI as "fraction of interactions where the user got what they wanted within a reasonable time."

---

## Trade-offs and Pitfalls

**Tight SLOs are expensive.** Going from 99.9% to 99.99% may require active-active multi-region deployment, dramatically increasing cost and operational complexity. The marginal cost of each additional nine grows non-linearly.

**Wrong SLI selection.** Measuring availability at the load balancer level misses database timeouts experienced by the application. Measure as close to the user as possible.

**Ignoring tail latency.** An SLO of "p50 latency < 100ms" leaves 50% of users undefined. The SLO should cover p99 or p99.9 for user-facing services.

**SLO windows that are too long.** A 90-day window makes the error budget feel infinite early in the period and creates false confidence. 28–30 day rolling windows are typical.

**Alert fatigue from noisy SLOs.** If the SLO fires on minor blips, engineers stop responding to alerts. Use multi-window, multi-burn-rate alerting (e.g., alert only when both 1-hour and 6-hour burn rates are elevated).

**SLA without an internal SLO.** Signing an SLA without a tracked internal SLO means you have no early warning before breaching the customer contract.

---

## Interview Cheat Sheet

**"Design a system with 99.99% availability"**
- 99.99% = ~4.38 minutes/month of allowed downtime
- This requires: redundancy at every layer, no single point of failure, health checks + auto-failover, zero-downtime deploys, multi-AZ or multi-region replication
- Ask: is this the SLA or the internal SLO? What is the error budget policy?

**"What is an error budget?"**
- Error budget = 1 - SLO. It is the allowed amount of unreliability.
- It is spent when incidents happen. When exhausted, feature releases freeze.
- It aligns dev velocity (wants to ship) with SRE (wants stability) by making the trade-off explicit and policy-driven.

**"What is the difference between SLO and SLA?"**
- SLO is an internal target; SLA is an external contract with consequences.
- SLO should always be tighter than SLA — it is your early warning system.

**"How do you choose SLIs?"**
- Start with the user's perspective: what does a "good" interaction look like?
- Common categories: availability, latency (percentiles), error rate, durability, freshness
- Avoid SLIs that can be gamed or that do not correlate with user experience

**"What percentile should latency SLOs use?"**
- p99 or p99.9 for user-facing services. p50 (median) hides tail latency that affects real users.

**"What happens when an error budget runs out?"**
- Engineering focus shifts to reliability. No new feature releases until the budget recovers.
- The policy should be defined in advance, not decided during an incident.

---

## References

- [Google SRE Book — Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
- [Google SRE Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [The Art of SLOs (Google Cloud)](https://cloud.google.com/blog/products/management-tools/the-art-of-slos)
- [SLO vs SLA vs SLI — Atlassian](https://www.atlassian.com/incident-management/kpis/sla-vs-slo-vs-sli)