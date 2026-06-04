# Monolith vs Microservices — Decision Framework

## Table of Contents

- [What They Are](#what-they-are)
- [Core Trade-offs](#core-trade-offs)
- [Decision Framework](#decision-framework)
- [Conway's Law](#conways-law)
- [The Modular Monolith — the Middle Ground](#the-modular-monolith--the-middle-ground)
- [Migration Strategy: Strangler Fig](#migration-strategy-strangler-fig)
- [Problems Microservices Introduce](#problems-microservices-introduce)
- [Operational Prerequisites for Microservices](#operational-prerequisites-for-microservices)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

---

## What They Are

### Monolith

A monolith is an application deployed as a single, unified unit. All modules — auth, orders,
payments, notifications — live in one codebase, share one process, and are deployed together.
They typically share a single database.

The name comes from "monolithic" (Greek: *monolithos* — single stone). The whole system is one
solid block; you cannot deploy one part without deploying all of it.

```
         Monolith Process
┌──────────────────────────────┐
│  Auth   │ Orders │ Payments  │
│  Users  │ Search │  Notify   │
└──────────────────────────────┘
                │
          Single Database
```

**How to remember:** one codebase → one deploy → one failure domain → one scaling unit.

### Microservices

A microservice architecture decomposes the application into independently deployable services.
Each service:
- owns one business domain (bounded context)
- owns its own data store
- communicates over the network (HTTP/REST, gRPC, or message queues)
- is deployed and scaled independently

The "micro" refers to *scope of responsibility*, not lines of code. A service can be
large — what matters is that it has a single, well-bounded domain concern.

```
         API Gateway / BFF
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
Auth Svc   Order Svc  Payment Svc
(own DB)   (own DB)   (own DB)
```

---

## Core Trade-offs

| Dimension | Monolith | Microservices |
|---|---|---|
| Deployment | All-or-nothing, single unit | Independent per service |
| Scaling | Scale the whole app | Scale individual bottleneck services |
| Dev velocity (small team) | Fast — no cross-service coordination | Slow — infra, contracts, versioning overhead |
| Dev velocity (large team) | Slow — coordination, merge conflicts | Fast — teams own services end-to-end |
| Data consistency | ACID transactions trivially available | Distributed transactions required (sagas) |
| Latency | In-process function calls (nanoseconds) | Network hops (milliseconds) |
| Failure isolation | One bug can take down the whole app | Faults contained to the failing service |
| Observability | One log stream, simple debugging | Requires distributed tracing across services |
| Testing | Integration tests straightforward | End-to-end tests span multiple deployable units |
| Operational complexity | Low — one deploy pipeline | High — k8s, service mesh, service discovery |

---

## Decision Framework

The question is never "monolith vs microservices." The question is:
*"What are the actual constraints, and which architecture fits them?"*

Work through these five lenses:

### 1. Team Size and Ownership Model

- **< 10 engineers:** Default to monolith. The overhead of microservices (separate CI/CD per
  service, on-call rotation per service, inter-service API contracts) destroys small teams.
- **Multiple domain teams (10+ engineers):** Microservices enable teams to own a service
  end-to-end and deploy independently, without stepping on each other.
- **Rule of thumb:** one team should own one (or a few closely related) services. If one team
  owns 15 services, that's not microservices — that's a distributed monolith with extra steps.

### 2. Domain Complexity and Stability

- Are the domain boundaries natural and stable? (Orders vs. Payments vs. Inventory — yes.)
- If the domain is still being discovered (pre-PMF), boundaries are wrong until proven otherwise.
  Splitting prematurely locks in incorrect service boundaries, which are expensive to refactor
  across service lines (now a cross-team, cross-deployment migration instead of a code move).
- Tightly coupled domains that share data constantly are better kept together. Splitting them
  introduces distributed joins and saga complexity with no real benefit.

### 3. Scale Profile

- Every component needs the same scale? → Monolith behind a load balancer handles this.
- One component is the bottleneck? → Extract *just that component* (strangler fig). Do not
  decompose everything because one service is hot.
- Multiple independent scale profiles across domains? → Microservices pay off.

### 4. Operational Maturity

Microservices require infrastructure that most early teams do not have:

```
Microservices prerequisite stack:
─────────────────────────────────
  Distributed tracing (Jaeger, OpenTelemetry)
  Centralised log aggregation (ELK, Loki)
  Service discovery (Consul, k8s DNS)
  Container orchestration (Kubernetes)
  Automated CI/CD per service
  Circuit breakers + retries + timeouts
  API gateway / BFF
  Health checks + on-call per service
```

If these are not in place, microservices will cost more in operational incidents than they save
in deployment flexibility.

### 5. Product Stage

- **Pre-product-market-fit:** Monolith. Move fast. Domain boundaries aren't stable. The cost of
  getting boundaries wrong in microservices is high.
- **Post-PMF, growing team, stable domain:** Begin incremental extraction (strangler fig).
- **Established product, multiple teams:** Full microservices with clear ownership.

### Decision Summary

```
Is the team small (< 10 engineers)?
  YES → Monolith (or modular monolith)
  NO  ↓
Is the domain stable and naturally divisible?
  NO  → Modular monolith; revisit when boundaries clarify
  YES ↓
Do you have operational maturity (tracing, CI/CD, orchestration)?
  NO  → Modular monolith; build the platform first
  YES ↓
Are there independent scale or deployment requirements across domains?
  NO  → Modular monolith
  YES → Microservices
```

---

## Conway's Law

> *"Any organisation that designs a system will produce a design whose structure is a copy of
> the organisation's communication structure."* — Melvin Conway, 1967

This is not an opinion — it is an observed empirical tendency. Communication overhead between
teams produces API boundaries in the system. A company with one team produces a monolith. A
company with five domain teams gravitates toward five services.

**Inverse Conway Manoeuvre:** deliberately design your team topology to produce the architecture
you want. If you want microservices, first create independent teams with clear domain ownership.
The architecture follows the org.

**Interview relevance:** when asked "would you use microservices here?", asking about team
structure is not a dodge — it is the correct first question.

---

## The Modular Monolith — the Middle Ground

A modular monolith is a single deployable unit internally structured as loosely coupled modules
with explicit boundaries. Modules communicate through defined interfaces, not direct package
imports across domain lines.

```
Single Deployment Unit
┌──────────────────────────────────────┐
│ ┌──────────┐   ┌──────────┐          │
│ │  Orders  │──▶│Payments  │ (via     │
│ │  module  │   │  module  │  interface│
│ └──────────┘   └──────────┘  not DB) │
└──────────────────────────────────────┘
         │
   Single Database
   (schema-per-module
    or strict ownership)
```

Benefits:
- Fast to develop (in-process)
- Easy to extract to microservices later — the module boundary already matches the service boundary
- Domain boundaries enforced at the code level, not the infrastructure level

This is the recommended starting point for most new systems.

---

## Migration Strategy: Strangler Fig

Named after a vine (strangler fig) that grows around a host tree and eventually replaces it.
You do not rewrite the monolith. You extract services incrementally, routing traffic through a
proxy/gateway that decides which requests go to the monolith and which go to new services.

```
Phase 1: Baseline monolith
──────────────────────────
Requests ──▶ Monolith (all traffic)

Phase 2: First extraction
─────────────────────────
Requests ──▶ Gateway
               │
        ┌──────┴────────────┐
        ▼                   ▼
   New Auth Svc        Monolith (minus auth)

Phase 3: Further extractions
─────────────────────────────
Requests ──▶ Gateway
          ┌──────┼──────┐
          ▼      ▼      ▼
      Auth Svc  Order Svc  Monolith (residual)
```

**Extract in priority order:**
1. Services with independent scale requirements
2. Services with independent deployment cadences
3. Services where team ownership is clear

Avoid extracting services that share heavy transactional coupling with the monolith early —
the distributed transaction cost will bite immediately.

---

## Problems Microservices Introduce

Framing these correctly in an interview is what separates senior from junior candidates.
Microservices shift complexity from the codebase into the infrastructure and operational layer —
they do not eliminate it.

### Distributed Data Management

Services own their data. You cannot do a SQL JOIN across service boundaries. Solutions:
- **Denormalisation:** each service stores the data it needs (duplicated, eventually consistent)
- **API composition:** API gateway or BFF calls multiple services and stitches results
- **Event-driven reads:** services subscribe to domain events and build local read models (CQRS)

### Distributed Transactions

ACID transactions do not span service boundaries. The replacement is the **Saga pattern**:

```
Saga: Choreography style
────────────────────────
Order Svc                Payment Svc            Inventory Svc
  │                           │                       │
  │── OrderCreated event ────▶│                       │
  │                           │── PaymentProcessed ──▶│
  │                           │   event               │
  │                           │        (or)           │
  │                           │── PaymentFailed ─────▶│
  │                           │   event (compensate)  │
```

If any step fails, prior steps must issue **compensating transactions** (e.g., cancel the order,
refund the payment). Sagas are correct but hard to test and reason about.

### Network Unreliability

In-process calls don't fail; network calls do. Every inter-service call needs:
- Timeouts (don't wait forever)
- Retries with exponential back-off (but only for idempotent operations)
- Circuit breakers (stop calling a failing service to prevent cascade)

### Observability Gap

A single request may touch 5 services. Without **distributed tracing** (OpenTelemetry, Jaeger,
Zipkin), you cannot reconstruct what happened. This is not optional in microservices — it is
load-bearing infrastructure.

### Eventual Consistency

After splitting, services don't update atomically. There's always a window where state is
inconsistent across services. Systems must be designed to tolerate and handle this. Users may
see stale data; UIs must handle partial failures gracefully.

### API Versioning and Contracts

Services evolve independently. A breaking change in one service's API must be versioned or it
breaks all consumers. Strategies: URL versioning (`/v2/...`), header versioning, consumer-driven
contract testing (Pact).

---

## Operational Prerequisites for Microservices

Do not recommend microservices without checking these. This signals experience.

| Prerequisite | Why it matters |
|---|---|
| Distributed tracing | Debugging cross-service request paths |
| Centralised logging | Correlating logs across services |
| Container orchestration (k8s) | Running many services without manual ops |
| Service discovery | Services finding each other dynamically |
| Automated CI/CD per service | Independent deployment is the whole point |
| Circuit breakers | Preventing cascade failures |
| API gateway | Single entry point, auth, rate limiting |
| Health checks + alerting | Knowing when a service is down |
| On-call per domain team | Ownership of service reliability |

---

## Interview Cheat Sheet

**Opening frame:** "Before choosing an architecture, I'd ask: what's the team size, how stable
is the domain, and what's our operational maturity?"

**Conway's Law:** "Architecture mirrors org structure. If we have one team, a monolith fits
naturally. Microservices make sense when we have independent domain teams."

**Microservices costs:** "Microservices shift complexity from the codebase to the infrastructure.
We gain deployment independence; we pay with distributed data management, sagas, tracing
requirements, and operational overhead."

**Starting point:** "I'd almost always start with a modular monolith — clean internal
boundaries, single deployment. Extract when there's a concrete reason: scale, team ownership,
or deployment independence."

**Migration:** "Strangler fig pattern — extract incrementally through a gateway, starting with
services that have clear scale or ownership requirements."

**Red flag to avoid:** Saying "microservices scale better" without qualification. Every monolith
can be horizontally scaled behind a load balancer. Microservices give you *selective* scaling,
not inherently more scale.

**Common follow-up questions:**
- "How would you handle transactions across services?" → Saga pattern, compensating transactions
- "What if two services need the same data?" → Denormalise, or event-driven read models
- "How do you debug a failure that spans 5 services?" → Distributed tracing with correlation IDs
- "When would you merge microservices back?" → When the boundary was wrong, or when teams merged

---

## References

- Martin Fowler — [Microservices](https://martinfowler.com/articles/microservices.html)
- Martin Fowler — [Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html)
- Martin Fowler — [MonolithFirst](https://martinfowler.com/bliki/MonolithFirst.html)
- Sam Newman — *Building Microservices* (O'Reilly)
- Melvin Conway — [Conway's Law (original paper)](https://www.melconway.com/Home/Conways_Law.html)
- Team Topologies — Matthew Skelton & Manuel Pais