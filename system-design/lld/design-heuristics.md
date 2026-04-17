# Design Heuristics — DRY, KISS, YAGNI

## Table of Contents

- [Overview](#Overview)
- [DRY — Don't Repeat Yourself](#dry--dont-repeat-yourself)
- [KISS — Keep It Simple, Stupid](#kiss--keep-it-simple-stupid)
- [YAGNI — You Aren't Gonna Need It](#yagni--you-arent-gonna-need-it)
- [The heuristics in tension](#The-heuristics-in-tension)
- [Interview lens](#Interview-lens)
- [References](#References)

---

## Overview

DRY, KISS, and YAGNI are **design heuristics** — rules of thumb for navigating common engineering failure modes, not laws. They are most useful as a shared team vocabulary: shorthand for calling out recurring anti-patterns in code reviews, design discussions, and architecture decisions.

Each heuristic targets a different failure mode:

- **DRY** combats *duplication* — the same knowledge encoded in multiple places
- **KISS** combats *accidental complexity* — solutions more complicated than the problems they solve
- **YAGNI** combats *speculation* — features built for imagined future requirements

These three heuristics frequently conflict with each other. Understanding the tension between them — and knowing which one to apply when — is what separates a junior engineer who has memorized the terms from a senior engineer who uses them as a thinking tool.

---

## DRY — Don't Repeat Yourself

### What it is

From *The Pragmatic Programmer* (Hunt & Thomas, 1999):

> "Every piece of knowledge must have a single, unambiguous, authoritative representation within a system."

The word that matters is **knowledge** — not code, not text. DRY is about the representation of a logical concept, business rule, or decision. Two code blocks that look identical but represent fundamentally different concepts are *not* a DRY violation. Two code blocks that implement the same validation rule *are*.

### Why it exists

Duplication is the root cause of a specific class of bugs: **update anomalies**. When the same logic exists in multiple places, a change to that logic must be applied everywhere it appears. Miss one location, and you have an inconsistency. At scale, this becomes the kind of bug that takes days to find because "we fixed that last month."

Common forms of duplication:

- Validation logic duplicated across the API handler, the service layer, and the database constraint
- Database schema knowledge encoded both in Go struct tags and in raw SQL query strings
- Configuration values hardcoded in multiple places instead of coming from a single config source
- Error messages defined in multiple languages/layers instead of one authoritative error catalog
- Business rules like "an order is eligible for refund if it is less than 30 days old" scattered across multiple services

### in Go

```go
// ❌ DRY violation — email validation duplicated across two handlers
func createUserHandler(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    json.NewDecoder(r.Body).Decode(&req)

    // Same validation logic appears again in updateUserHandler
    if len(req.Email) == 0 || !strings.Contains(req.Email, "@") {
        http.Error(w, "invalid email", http.StatusBadRequest)
        return
    }
    // ...
}

func updateUserHandler(w http.ResponseWriter, r *http.Request) {
    var req UpdateUserRequest
    json.NewDecoder(r.Body).Decode(&req)

    // Duplicated — if the rule changes, must change both
    if len(req.Email) == 0 || !strings.Contains(req.Email, "@") {
        http.Error(w, "invalid email", http.StatusBadRequest)
        return
    }
    // ...
}

// ✓ DRY — single authoritative representation
var ErrInvalidEmail = errors.New("invalid email address")

func validateEmail(email string) error {
    if len(email) == 0 || !strings.Contains(email, "@") {
        return ErrInvalidEmail
    }
    return nil
}

// Both handlers call validateEmail — one rule, one place to change
```

### The wrong abstraction — the most important DRY pitfall

DRY is the most commonly *over-applied* heuristic. Developers see two similar-looking functions and immediately reach for abstraction. This creates what Sandi Metz calls **the wrong abstraction**, which is worse than duplication.

```go
// You notice CreateOrder and CreateSubscription look similar
// and "DRY them up" into a combined function:

func createPurchasable(
    ctx context.Context,
    purchasableType string,
    userID string,
    amount int,
    metadata map[string]interface{},
) error {
    if purchasableType == "order" {
        // ... order logic
        if v, ok := metadata["shipping_address"]; ok {
            // ... shipping validation
        }
    } else if purchasableType == "subscription" {
        // ... subscription logic
        if v, ok := metadata["billing_cycle"]; ok {
            // ... billing cycle handling
        }
    } else if purchasableType == "gift_card" {
        // added 3 months later
        // ...
    }
    // 6 months later: 15 branches, each fighting each other
    // Abstraction now owns complexity instead of eliminating it
    return nil
}
```

The abstraction was created because the functions looked similar, not because they represented the same knowledge. They were coincidentally similar — they would have diverged naturally over time. Forcing them into a shared abstraction coupled them permanently.

**Sandi Metz's rule:** Duplication is far cheaper than the wrong abstraction.

**The rule of three:** A practical heuristic — wait for the *third* instance before abstracting. One is unique. Two is a coincidence. Three is a pattern worth abstracting.

### Coupling through shared code

DRY enforced via shared libraries or packages creates coupling. A shared `validation` package used by 10 services means a bug fix in that package requires coordinated deploys across all 10. Sometimes a little duplication is the right trade-off to preserve service independence.

```
Service A ──► shared/validation ◄── Service B
Service C ──►                   ◄── Service D

Change to shared/validation requires:
  - Review by teams owning A, B, C, D
  - Coordinated deploy of all four services
  - Risk of breaking any of the four
```

### DRY in system design

- **Single source of truth for configuration** — environment variables or a config service as the authoritative source; never duplicate the same config value across multiple services
- **Schema as the contract** — protobuf `.proto` files or OpenAPI specs as the authoritative definition; generate code from them rather than maintaining separate hand-written models
- **Event sourcing** — the event log as the single authoritative record of what happened; projections and read models are derived from events, not independently maintained
- **Single ownership of a domain** — in microservices, each piece of domain logic (pricing, eligibility, discount calculation) lives in exactly one service; other services call it rather than duplicating the logic

---

## KISS — Keep It Simple, Stupid

### What it is

Originally a design principle from the US Navy (attributed to Kelly Johnson, 1960): design systems so simple that even someone with basic knowledge can repair them under adverse conditions. Applied to software, the core claim is:

> Simplicity should be an explicit, first-class design goal — not an afterthought.

Simple does not mean simplistic. A simple system handles complexity gracefully and clearly. Simple means minimizing *incidental complexity* (complexity introduced by the solution) while accepting *essential complexity* (complexity inherent to the problem).

### Why it exists

Complexity is the primary enemy of:

- **Reliability** — more moving parts means more failure modes, more edge cases, more ways for things to go wrong under load
- **Debuggability** — complex systems are harder to reason about at 2am when something is on fire
- **Onboarding** — new engineers cannot contribute until they understand the system; complexity extends that ramp-up time
- **Maintainability** — complexity begets more complexity; a complex codebase accumulates new complexity faster than a simple one

### What simple looks like in Go

```go
// ❌ Over-engineered — unnecessary abstraction for simple CRUD with one implementation
type UserRepositoryFactoryProviderInterface interface {
    GetRepositoryFactory() UserRepositoryFactory
}
type UserRepositoryFactory interface {
    CreateRepository(cfg Config) UserRepository
}
type UserRepository interface {
    Find(id string) (*User, error)
    Save(user *User) error
}
type postgresUserRepository struct { db *sql.DB }
type postgresUserRepositoryFactory struct {}
type postgresUserRepositoryFactoryProvider struct {}

// ✓ KISS — just a function; add the interface when you have a second implementation
type UserRepo struct {
    db *sql.DB
}

func NewUserRepo(db *sql.DB) *UserRepo {
    return &UserRepo{db: db}
}

func (r *UserRepo) Find(ctx context.Context, id string) (*User, error) {
    var u User
    err := r.db.QueryRowContext(ctx,
        "SELECT id, name, email FROM users WHERE id = $1", id,
    ).Scan(&u.ID, &u.Name, &u.Email)
    return &u, err
}
```

The over-engineered version might be appropriate if you genuinely have multiple repository backends and need to swap them at runtime. The KISS version is appropriate for the 95% case where you have one database.

### Complexity categories

- **Essential complexity** — arises from the problem domain itself; cannot be eliminated, only managed. A billing system is inherently complex because billing rules are complex.
- **Accidental complexity** — introduced by the solution; can be reduced. Using five abstraction layers to solve a two-layer problem is accidental complexity.

KISS is specifically about minimizing *accidental* complexity. Don't mistake it for an argument against tackling hard problems.

### Go anti-patterns that violate KISS

```go
// ❌ Over-generic: "flexible" function that's harder to use than five specific ones
func ProcessEvent(ctx context.Context, eventType string, data interface{}, opts ...Option) error

// ✓ KISS: specific functions that are easy to read and test
func ProcessOrderCreated(ctx context.Context, event OrderCreatedEvent) error
func ProcessPaymentFailed(ctx context.Context, event PaymentFailedEvent) error

// ❌ Clever: regexp for something a strings.Contains solves in one line
var emailDomainRe = regexp.MustCompile(`@([^@]+)$`)
domain := emailDomainRe.FindStringSubmatch(email)[1]

// ✓ KISS: readable and obviously correct
parts := strings.SplitN(email, "@", 2)
domain := parts[1]
```

### Problems KISS brings

- **False economy** — skipping an abstraction now to keep things simple can mean paying a much higher complexity price later when you need to change the behavior in five places
- **Subjective** — "simple" is relative to the reader's experience level; what is simple to a senior engineer can be opaque to a new hire
- **Tension with DRY** — the simplest solution often contains some duplication; adding the DRY abstraction adds indirection. KISS asks you to pay the cost of that indirection only when the abstraction genuinely earns it

### KISS in system design

- **Prefer a monolith first** — microservices add operational complexity (deployment, service discovery, distributed tracing, network failures); start with a well-structured monolith and extract services when you have a specific, demonstrated pain point
- **Choose boring technology** — a well-understood PostgreSQL database with an index is simpler to operate than a custom caching layer with a message queue; don't introduce novel technology without a concrete problem it solves
- **Synchronous before asynchronous** — a direct HTTP call is simpler than a message queue; introduce async only when you need decoupling or when the latency of the synchronous call is a measured problem
- **Avoid premature optimization** — a simple query with a proper index is almost always faster to ship, operate, and understand than a custom caching layer built speculatively

---

## YAGNI — You Aren't Gonna Need It

### What it is

From Extreme Programming (XP), attributed to Ron Jeffries and Kent Beck:

> "Always implement things when you actually need them, never when you just foresee that you need them."

YAGNI pushes back on the engineering instinct to generalize. Developers are pattern-matching machines — we see a use case and immediately imagine the generalized version. YAGNI says: wait for the reality before building for the generalization.

### Why it exists

Speculative features fail in three ways:

1. **Predicted requirements are often wrong** — the feature you imagined needing six months from now often does not materialize, or materializes in a different form than you anticipated
2. **Unused features still have maintenance cost** — dead code has to be understood, tested, updated with dependency upgrades, and worked around by future engineers who don't know it's dead
3. **Speculative abstractions are often wrong** — a generalization built before the second use case exists is frequently the wrong generalization; the second real use case reveals what the correct abstraction should be

### in Go

```go
// ❌ YAGNI violation — building a plugin system because "we might need extensibility"
type Processor interface {
    Process(ctx context.Context, payload Payload) error
}

type ProcessorChain struct {
    processors []Processor
    hooks      map[string][]Hook
    middleware []func(Processor) Processor
    registry   map[string]ProcessorFactory
}

func (c *ProcessorChain) Register(name string, factory ProcessorFactory) { ... }
func (c *ProcessorChain) Use(mw func(Processor) Processor) { ... }
// ... 200 lines of infrastructure for the one processor that exists

// ✓ YAGNI — just handle the current use case directly
func processPayment(ctx context.Context, p Payment) error {
    if err := validatePayment(p); err != nil {
        return fmt.Errorf("validation: %w", err)
    }
    if err := chargeCard(ctx, p); err != nil {
        return fmt.Errorf("charge: %w", err)
    }
    return sendReceipt(ctx, p)
}
// When a second payment type appears, THEN extract the interface
```

```go
// ❌ YAGNI: configuration struct with fields you don't need yet
type Config struct {
    DSN              string
    MaxConns         int
    MaxIdleConns     int
    ConnMaxLifetime  time.Duration
    ConnMaxIdleTime  time.Duration
    ReplicaSetName   string        // we're not using replication
    ShardKey         string        // we're not sharding
    ReadPreference   string        // we're not doing read replicas
    RetryOnFailover  bool          // we're not in a failover scenario
}

// ✓ YAGNI: start with what you actually configure
type Config struct {
    DSN      string
    MaxConns int
}
```

### The YAGNI line: known vs speculative

YAGNI is not an argument against all forward-thinking design. The distinction that matters:

- **Known, near-term requirement** → design for it now. If you are building a multi-tenant SaaS and you have signed contracts with three tenants, multi-tenancy is not speculative.
- **Speculative, distant requirement** → do not build for it. If you imagine you might need multi-tenancy someday, that is YAGNI territory.

A useful framing: **design for 10x your current load, not 100x.** If you have 1,000 users today and growth is the explicit goal, designing for 10,000 is reasonable forward-thinking. Designing for 1,000,000 is speculation.

### Problems YAGNI brings

- **Retrofitting cost** — some decisions are cheap to add later (a new field on a struct), and some are expensive (adding multi-tenancy to a system designed for a single tenant, or adding sharding to a schema that didn't plan for a shard key). YAGNI works best when you know which category you're in.
- **Misapplied as laziness** — "YAGNI" is sometimes used to justify not thinking ahead at all. YAGNI is about *features*; it does not apply to *quality* (error handling, observability, graceful degradation).
- **Tension with DRY** — sometimes you need to build the abstraction before you have two concrete uses (e.g., designing a plugin interface before the first plugin exists, because the interface affects the project's external API). The two heuristics pull in opposite directions here.

### YAGNI in system design

- **Don't prematurely shard** — start with a single database; shard when you have measured a bottleneck that vertical scaling and read replicas cannot solve
- **Don't add a cache speculatively** — caches add consistency complexity; add one when you have measured a hot read path that the database cannot serve at the required latency
- **Don't build a platform** — "let's make this a platform" is the YAGNI anti-pattern at system scale; generic platforms are expensive to build, hard to evolve, and rarely used in the way their creators imagined
- **Don't go async prematurely** — a synchronous HTTP call is simpler; add a message queue when you have a concrete requirement for decoupling, fan-out, or durability, not because "event-driven is the modern way"
- **Start with a monolith** — this is YAGNI applied to service boundaries; microservices solve problems (independent deployability, team autonomy, technology heterogeneity) that a monolith does not yet have; extract services when the pain of the monolith is concrete and measured

---

## The heuristics in tension

The most important thing to understand about DRY, KISS, and YAGNI is that they frequently conflict, and the right answer depends on context:

```
DRY ◄─────────────────────────────────► YAGNI
  (abstract now, single               (don't build until
  representation)                      you need it)
         ▲
         │
        KISS
  (is this abstraction
   earning its cost?)
```

**DRY vs YAGNI:** DRY says to avoid duplication; YAGNI says don't build the abstraction until you have two real uses. The reconciliation: tolerate the duplication until you have the second (or third) real instance; then the abstraction is no longer speculative.

**DRY vs KISS:** DRY removes duplication by adding abstraction; KISS questions whether the abstraction is worth its indirection cost. The reconciliation: does the abstraction make the system *more* understandable overall, or less? If the shared function is clearer than the duplicated code, DRY wins. If you need to read three files to understand what the abstraction does, KISS wins.

**KISS vs YAGNI:** KISS says keep things simple; YAGNI says don't build what you don't need. These usually align — speculative features add complexity. They conflict when the simplest current solution forecloses a known future requirement (e.g., designing a data model without a `tenant_id` column when multi-tenancy is on the roadmap).

A practical ordering for making decisions:

1. **YAGNI first** — is this feature/abstraction actually needed now? If not, don't build it.
2. **KISS second** — of the approaches that satisfy the current need, which is the simplest?
3. **DRY third** — given the simplest approach, is there duplication that introduces real maintenance risk?

---

## Interview lens

### what interviewers actually look for

In system design interviews, these heuristics show up not as trivia but as *justification patterns*. The interviewer is watching whether you design just enough, or over-design. They are listening for whether you can articulate *why* you made a choice.

**YAGNI signals that impress:**

- "I'd start with a single PostgreSQL instance; add read replicas when we have a measured read bottleneck, and shard when we exceed what a single primary can handle."
- "I wouldn't add a cache yet — I don't know the access patterns well enough to know what to cache, and adding it speculatively means we need to solve cache invalidation for no measured benefit."
- "A monolith is the right starting point; we'd extract a service when we have a team autonomy or scaling problem a monolith can't solve."

**KISS signals that impress:**

- "I'd pick PostgreSQL over a more exotic data store here — the team knows it, the operational tooling is mature, and I don't see a problem it can't solve."
- "A cron job polling the database is simpler and sufficient for this use case; a message queue would give us more scalability headroom but at the cost of operational complexity we don't need yet."

**DRY signals that impress:**

- "Auth logic would live in a gateway middleware — I wouldn't duplicate token validation in every service. But I'd be careful about what else goes in a shared library; coupling services through shared code has real coordination costs."
- "The pricing rules belong in the Pricing service, not in every service that needs to display a price. Other services call Pricing; they don't duplicate the logic."

**Anti-patterns that concern interviewers:**

- Reaching for microservices, Kubernetes, Kafka, and Redis from day one for a startup problem
- "We might need X later" as a justification for adding complexity now
- Abstracting at the first sign of two similar things without waiting for the pattern to emerge
- Treating YAGNI/KISS as excuses to skip error handling, observability, or graceful degradation

**The answer that wins almost every time:** Start simple. Define the trigger conditions that would cause you to change the approach. Know what the next step would look like. This demonstrates you have thought about the evolution of the system, not just its current state.

```
Interviewer: "What if you need to handle 10x the traffic?"
Weak answer: "We'd scale up."
Strong answer: "At 10x, the first bottleneck is likely the database. I'd add read replicas
               for read-heavy paths first. If writes are the bottleneck, I'd look at
               write-ahead log shipping to a secondary or partitioning the writes by
               customer shard. I'd only reach for a distributed database after those
               options are exhausted, because they introduce significant operational
               complexity."
```

---

## References

- [The Pragmatic Programmer — Hunt & Thomas (1999)](https://pragprog.com/titles/tpp20/the-pragmatic-programmer-20th-anniversary-edition/) — origin of DRY
- [Extreme Programming Explained — Kent Beck (2000)](https://www.oreilly.com/library/view/extreme-programming-explained/0201616416/) — origin of YAGNI
- [Sandi Metz — The Wrong Abstraction (2016)](https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction)
- [Rich Hickey — Simple Made Easy (Strange Loop 2011)](https://www.infoq.com/presentations/Simple-Made-Easy/) — essential vs accidental complexity
- [Dan North — CUPID — for joyful coding](https://dannorth.net/cupid-for-joyful-coding/) — modern take on design heuristics
- [Martin Fowler — Yagni (2015)](https://martinfowler.com/bliki/Yagni.html)
- [Martin Fowler — BeckDesignRules](https://martinfowler.com/bliki/BeckDesignRules.html) — Kent Beck's four rules of simple design