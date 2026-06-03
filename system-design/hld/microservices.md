# Microservices Architecture

## Table of Contents

- [What It Is](#what-it-is)
- [Why It Exists — The Monolith Problem](#why-it-exists--the-monolith-problem)
- [Core Principles](#core-principles)
- [Decomposition Strategies](#decomposition-strategies)
- [Inter-Service Communication](#inter-service-communication)
- [Distributed Transactions and the Saga Pattern](#distributed-transactions-and-the-saga-pattern)
- [Failure Handling](#failure-handling)
- [Pitfalls](#pitfalls)
- [Service Mesh](#service-mesh)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [How to Remember It](#how-to-remember-it)
- [References](#references)

## What It Is

Microservices architecture structures an application as a **collection of small, independently deployable services**. Each service:

- owns a single business capability
- runs in its own process
- communicates with others via network calls (HTTP/gRPC/events)
- owns its own data store

The "micro" in microservices refers to **scope of responsibility**, not lines of code. A service is micro if it can be reasoned about, deployed, and scaled independently. It is the spiritual successor to SOA, but with lighter-weight protocols and stricter data ownership rules.

## Why It Exists — The Monolith Problem

Monolithic applications start simple but develop compounding problems as they grow:

- **Deployment coupling** — one team's change requires the entire application to redeploy, even if unrelated
- **Scaling ceiling** — the only option is to scale the entire application, not just the component under load
- **Technology lock-in** — all teams must share the same language, framework, and database
- **Large blast radius** — a bug in payments can bring down unrelated features like user profiles
- **Team bottlenecks** — all teams modify the same codebase, creating merge conflicts and coordination overhead

Conway's Law: organisations produce systems that mirror their communication structures. Microservices align service boundaries with team boundaries, so teams can own and evolve their services independently.

## Core Principles

- **Bounded context** — each service maps to a bounded context from Domain-Driven Design. The `Order` concept inside `inventory-service` and inside `billing-service` can be different models. Services do not share domain objects.
- **Database per service** — the most important and most violated rule. Each service owns its own data store exclusively. No service reads another's database directly. This is what enables true independence.
- **Communicate over the network** — services interact via APIs or events, never via shared memory or a shared database.
- **Design for failure** — your dependency will go down. Every synchronous call needs a timeout; every critical dependency needs a circuit breaker.
- **Smart endpoints, dumb pipes** — business logic lives in services, not in the communication layer. Message brokers route; they do not transform.
- **Decentralised governance** — teams choose their own tech stack independently.

## Decomposition Strategies

**By business capability**
Map services to what the business does: `user-service`, `payment-service`, `notification-service`. Intuitive and widely used.

**By subdomain (DDD)**
Model the business domain, identify bounded contexts, and map each context to a service. More rigorous. Useful when business capabilities are large and overlapping.

**Strangler Fig Pattern**
The migration path from monolith to microservices. New functionality is built as services alongside the living monolith. Traffic is gradually rerouted from the monolith to services until the monolith can be retired.

```
Phase 1                 Phase 2                 Phase 3
+----------+           +----------+            +----------+
| Monolith |           | Monolith |            | Monolith |
|  (all)   |           | (shrunk) |            | (retired)|
+----------+           +----------+            +----------+
                       +----------+            +----------+
                       | svc-A    |            | svc-A    |
                       +----------+            +----------+
                       +----------+            +----------+
                       | svc-B    |            | svc-B    |
                       +----------+            +----------+
                                               +----------+
                                               | svc-C    |
                                               +----------+

Traffic: 100% monolith  Traffic: split          Traffic: 100% services
```

## Inter-Service Communication

**Synchronous (request/response)**

- REST over HTTP — simple, widely understood, good for external-facing APIs
- gRPC — binary (Protobuf), strongly typed, efficient for internal service-to-service calls; supports streaming
- GraphQL — useful when clients need flexible queries, typically at the API Gateway layer

**Asynchronous (event-driven)**

- Services publish events to a broker (Kafka, RabbitMQ, SQS)
- Consumers process at their own pace
- Naturally decoupled; consumers can be added without modifying the publisher
- Supports replay (especially Kafka with its durable log)

Prefer async when:
- The operation is long-running
- One event should fan out to multiple consumers
- Durability and replay capability are required
- The caller does not need an immediate result

## Distributed Transactions and the Saga Pattern

In a monolith, a transaction is atomic — rollback on failure, ACID guaranteed. In microservices, one logical operation spans multiple services with separate databases. Two-phase commit across services is too slow and too brittle for production use.

**Saga Pattern** — break the operation into a sequence of local transactions. Each step publishes an event that triggers the next. On failure, compensating transactions undo completed steps.

**Choreography Saga** (event-driven, decentralised)

```
order-svc          payment-svc         inventory-svc
    |                    |                    |
    |--OrderCreated----->|                    |
    |                    |--PaymentDone------>|
    |                    |                    |--StockReserved-->

Failure path:
    |<--PaymentFailed----|
    | (compensate: cancel order)
```

Simple to implement. Hard to trace and reason about as the number of steps grows.

**Orchestration Saga** (centralised coordinator)

```
          saga-orchestrator
                |
    +-----------+-----------+
    |           |           |
order-svc  payment-svc  inventory-svc
```

The orchestrator tells each service what to do and handles compensation centrally. Easier to trace and debug; adds a coordinator service as a dependency.

## Failure Handling

**Circuit Breaker**

Named after the electrical circuit breaker. Monitors calls to a dependency. When the failure rate exceeds a threshold, the breaker "trips" and subsequent calls fail immediately without touching the network.

```
                failures > threshold
  +---------+ -----------------------> +----------+
  |  Closed |                          |   Open   |
  | (normal)|                          |(fail fast)|
  +---------+                          +----------+
       ^                                     |
       |                              timeout elapsed
       |                                     |
       |                              +------------+
       |---success--------------------|  Half-Open  |
                                      | (probe one) |
                                      +------------+
```

- Closed — calls go through normally
- Open — calls fail immediately (no network call attempted)
- Half-Open — one probe call is allowed through; success resets to Closed, failure returns to Open

Go library: `sony/gobreaker`

**Bulkhead**

Isolate resources (goroutines, connection pools) per downstream dependency. A slow dependency consumes only its allocated pool, not the shared pool of the entire service.

**Retry with Exponential Backoff**

Never retry in a tight loop — it hammers a degraded dependency. Back off exponentially with jitter: `wait = base * 2^attempt + random_jitter`.

**Timeouts**

Set explicit timeouts on every network call. Without a timeout, a goroutine can block indefinitely, exhausting the goroutine pool.

## Pitfalls

- **Network as a dependency** — what was a function call is now a network call. Network calls fail, are slow, and introduce latency. Every hop is a failure point.
- **Eventual consistency** — database-per-service means no global ACID. Accept that reads may be stale; design your system around this with idempotency and compensating transactions.
- **Testing complexity** — integration testing requires running many services simultaneously. Contract testing (Pact) helps verify API compatibility without full end-to-end deployments.
- **Operational overhead** — 50 services means 50 deployments, 50 health checks, 50 logging pipelines, and a need for container orchestration (Kubernetes).
- **Distributed tracing is non-negotiable** — a single user request fans out across many services. Without trace propagation (OpenTelemetry, Jaeger, Zipkin) and correlation IDs, debugging production failures is nearly impossible.
- **The nano-service trap** — services that are too fine-grained create chatty inter-service communication and heavy coordination overhead. If two services always change and deploy together, they are likely one service.
- **Cascading failures** — `A → B → C`. Slowness in C causes B to back up, which causes A to back up. Circuit breakers and bulkheads prevent this.
- **Shared databases** — the most common and damaging mistake. A shared database couples services at the data layer, eliminating independence.

## Service Mesh

As you scale to dozens of services, cross-cutting infrastructure concerns — retries, mTLS, metrics, tracing, circuit breaking — get copy-pasted into every service. A service mesh extracts these into a dedicated infrastructure layer, invisible to application code.

**The Sidecar Pattern**

Every service pod gets a proxy (typically Envoy) injected alongside it. All network traffic, inbound and outbound, is transparently intercepted by the proxy. Application code makes a plain network call; the proxy handles the rest.

```
Pod A                                  Pod B
+---------------------------+          +---------------------------+
|  app-container            |          |  app-container            |
|  (business logic only)    |          |  (business logic only)    |
|          |                |          |          |                |
|          v                |          |          v                |
|  [envoy sidecar]          |--------->|  [envoy sidecar]          |
+---------------------------+          +---------------------------+
           ^                                       ^
           |                                       |
           +----------- control plane -------------+
                             (Istiod)
                   pushes config to all proxies
```

**Data plane** — all Envoy sidecar proxies collectively. They perform routing, load balancing, mTLS enforcement, and telemetry collection.

**Control plane** — Istiod (in Istio). You write high-level policies (VirtualService, DestinationRule); the control plane compiles them into Envoy config and pushes to all data-plane proxies. You never configure Envoy directly.

**What a service mesh gives you**

- **mTLS** between all services, with automatic certificate rotation — zero-trust networking by default
- **Traffic management** — canary deployments, A/B testing, traffic splitting by percentage, all via config not code
- **Observability** — every sidecar emits metrics, logs, and distributed traces automatically, without code changes
- **Retries, timeouts, circuit breaking** — configured once in the mesh policy, enforced uniformly

**Costs**

- Added latency — two extra network hops per call (egress through local sidecar, ingress through remote sidecar)
- Operational complexity — Istio in particular has a steep learning curve and large control-plane footprint
- Kubernetes-centric — service meshes are designed for containerised, orchestrated workloads

**Options**

- **Istio** — most feature-rich; most complex; widest adoption
- **Linkerd** — simpler, lighter, Rust-based proxies; less operational burden
- **Consul Connect** — works outside Kubernetes; useful for hybrid VM + container environments

## Interview Cheat Sheet

- **When to not use microservices** — small team, early-stage product, unclear domain boundaries. Start with a modular monolith and extract services when pain points are clear.
- **How to define service boundaries** — DDD bounded contexts. A service should change for one business reason, not multiple.
- **How to handle distributed transactions** — Saga pattern. Choreography for simple flows, orchestration for complex ones where traceability matters.
- **How to handle cascading failures** — circuit breakers, bulkheads, timeouts, and retry with exponential backoff + jitter.
- **What a service mesh solves** — cross-cutting concerns (mTLS, observability, traffic management) extracted from application code into infrastructure.
- **Biggest microservices mistake** — shared databases between services. It destroys independence at the data layer.
- **Monolith vs microservices trade-off** — microservices trade simplicity for scalability, team autonomy, and independent deployability. The operational cost is real.

## How to Remember It

Think of a city of specialised shops rather than a single department store. Each shop (service) owns its own inventory (database), has its own staff (team), and can open and close independently. Shops communicate by calling each other (synchronous) or leaving messages (async/events). The roads between shops are the network. A bad road (slow network) affects every delivery. A shop that catches fire (failure) should not burn down the whole city — circuit breakers are fire doors.

The service mesh is like a city-wide traffic management system operating at the road level — it handles routing, tolls (auth), and road quality monitoring without each shop needing to manage its own traffic lights.

## References

- Martin Fowler — [Microservices](https://martinfowler.com/articles/microservices.html)
- Martin Fowler — [Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html)
- Martin Fowler — [Saga](https://martinfowler.com/articles/patterns-of-distributed-systems/saga.html)
- Sam Newman — *Building Microservices* (O'Reilly)
- Chris Richardson — [Microservices Patterns](https://microservices.io/patterns/index.html)
- Netflix Tech Blog — [Hystrix](https://netflixtechblog.com/making-the-netflix-api-more-resilient-a8ec62159c2d)
- Istio — [Architecture](https://istio.io/latest/docs/ops/deployment/architecture/)
- Linkerd — [Architecture](https://linkerd.io/2.15/reference/architecture/)
- `sony/gobreaker` — [GitHub](https://github.com/sony/gobreaker)
- OpenTelemetry — [Distributed Tracing](https://opentelemetry.io/docs/concepts/signals/traces/)