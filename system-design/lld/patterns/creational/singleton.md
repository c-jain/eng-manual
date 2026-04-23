# Singleton Pattern

## Table of Contents

- [What It Is and Why It Exists](#what-it-is-and-why-it-exists)
- [Why It Is Called "Singleton"](#why-it-is-called-singleton)
- [The Core Problem It Solves](#the-core-problem-it-solves)
- [Problems Singleton Brings](#problems-singleton-brings)
- [Implementing Singleton in Go](#implementing-singleton-in-go)
  - [Naive Implementation (Broken)](#naive-implementation-broken)
  - [Mutex-Based Implementation](#mutex-based-implementation)
  - [sync.Once — The Idiomatic Go Way](#synconce--the-idiomatic-go-way)
  - [Package-Level Initialization](#package-level-initialization)
- [How sync.Once Works Internally](#how-synconce-works-internally)
- [Singleton vs Dependency Injection](#singleton-vs-dependency-injection)
- [When to Use Singleton](#when-to-use-singleton)
- [When Not to Use Singleton](#when-not-to-use-singleton)
- [What Interviewers Actually Look For](#what-interviewers-actually-look-for)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

---

## What It Is and Why It Exists

**Singleton** is a creational design pattern that ensures a type has **exactly one instance** for the lifetime of a program, and provides a **single, well-known access point** to that instance.

It exists because some resources are inherently singular — there is physically or logically only one of them:

- A database connection pool (you don't want 50 goroutines each creating their own pool)
- A configuration registry loaded from a file at startup
- A logger shared across an entire application
- A hardware interface (there is one GPU, one serial port)

Without a controlled mechanism, naive code creates multiple instances, wastes resources, and introduces inconsistency (two different "configs" with potentially different values).

---

## Why It Is Called "Singleton"

"Singleton" comes from mathematics: a **singleton set** is a set with exactly one element, written `{x}`. The pattern borrows this directly — you have a type that can only ever produce one member of itself. The name was formalised by the "Gang of Four" (GoF) in *Design Patterns: Elements of Reusable Object-Oriented Software* (1994).

---

## The Core Problem It Solves

Consider a database pool without Singleton:

```go
// Called in three different packages:
db1 := database.NewPool(cfg)  // opens 20 connections
db2 := database.NewPool(cfg)  // opens another 20 connections
db3 := database.NewPool(cfg)  // opens another 20 connections
// Total: 60 connections when you wanted 20
```

This is wasteful and can exhaust database connection limits. Singleton prevents the second and third `NewPool` calls from doing any work — they just return the same object as the first call.

---

## Problems Singleton Brings

Singleton is also one of the most criticised patterns. Understanding why is as important as knowing how to implement it.

**Global mutable state.** A Singleton is, by nature, a global variable in disguise. Any part of the program can reach in and read or modify it, making it hard to reason about who changed what and when.

**Testing difficulty.** You cannot swap a Singleton for a fake in a unit test without changing the package-level code itself. This couples your tests to the real implementation.

```
Scenario: you want to test a function that reads from a config Singleton.
Problem:  the Singleton reads from disk. In a test environment, there may be no config file.
Result:   the test panics or reads wrong values.
```

**Hidden dependency.** A function that calls `GetInstance()` hides its dependency on the Singleton from its signature. Callers don't know what the function depends on just by reading its type.

```go
// Bad: hidden dependency
func ProcessOrder(id string) {
    db := database.GetInstance() // invisible to callers
    ...
}

// Good: explicit dependency (dependency injection)
func ProcessOrder(id string, db *database.Pool) {
    ...
}
```

**Concurrency subtleties.** If initialization is lazy (on first use) and multiple goroutines call `GetInstance()` simultaneously, you have a race condition unless protected correctly. This is where most Singleton bugs live.

---

## Implementing Singleton in Go

Go has no classes, no constructors, and no access modifiers. This changes the implementation strategy significantly compared to Java/C++.

### Naive Implementation (Broken)

```go
package database

import "fmt"

type Pool struct {
    dsn string
}

var instance *Pool

// GetInstance is NOT thread-safe.
func GetInstance(dsn string) *Pool {
    if instance == nil {         // goroutine A checks: nil → true
        instance = &Pool{dsn}   // goroutine B also checks: nil → true
    }                           // both goroutines create separate instances
    return instance
}
```

**Why it breaks:** Between the `nil` check and the assignment, another goroutine can pass the same `nil` check. Both goroutines create an instance. The second write overwrites the first. Whichever goroutine returns second may hand back the overwritten pointer, and the object the first goroutine is already using is now unreferenced.

This is a **data race**. The Go race detector (`go test -race`) will catch it.

### Mutex-Based Implementation

```go
package database

import "sync"

type Pool struct {
    dsn string
}

var (
    instance *Pool
    mu       sync.Mutex
)

func GetInstance(dsn string) *Pool {
    mu.Lock()
    defer mu.Unlock()

    if instance == nil {
        instance = &Pool{dsn: dsn}
    }
    return instance
}
```

**Correct, but suboptimal.** Every call to `GetInstance` acquires a mutex, even after initialization is complete. If `GetInstance` is called frequently (e.g., in a hot path), this becomes a bottleneck.

The classic fix is **double-checked locking**:

```go
func GetInstance(dsn string) *Pool {
    if instance != nil {     // fast path: no lock if already initialized
        return instance
    }
    mu.Lock()
    defer mu.Unlock()
    if instance == nil {     // re-check after acquiring lock
        instance = &Pool{dsn: dsn}
    }
    return instance
}
```

This is technically valid in Go because of its memory model, but it is verbose and easy to get wrong. There is a better way.

### sync.Once — The Idiomatic Go Way

`sync.Once` is a type in the standard library built for exactly this purpose: run an initialization function exactly once, safely, across any number of goroutines.

```go
package database

import "sync"

type Pool struct {
    dsn string
}

var (
    instance *Pool
    once     sync.Once
)

// GetInstance returns the singleton Pool.
// It is safe to call from multiple goroutines.
func GetInstance(dsn string) *Pool {
    once.Do(func() {
        instance = &Pool{dsn: dsn}
    })
    return instance
}
```

**Usage:**

```go
func main() {
    db := database.GetInstance("postgres://localhost/mydb")
    // ... use db

    db2 := database.GetInstance("postgres://localhost/mydb")
    // db2 == db (same pointer)
    fmt.Println(db == db2) // true
}
```

**What `sync.Once` guarantees:**
- The function passed to `Do` is called at most once
- If multiple goroutines call `Do` simultaneously, only one executes the function; the rest block until it completes
- After `Do` returns for all callers, the initialized value is visible to all of them (memory barrier)

### Package-Level Initialization

For resources that are always needed and whose initialization cannot fail with a meaningful error, Go's `init()` or package-level `var` is the simplest Singleton:

```go
package database

var defaultPool = &Pool{dsn: "postgres://localhost/mydb"}

func DefaultPool() *Pool {
    return defaultPool
}
```

This is initialized by the runtime before `main()` runs. It is safe, simple, and the Go runtime handles the "exactly once" guarantee automatically. The downside: no lazy initialization, and the DSN is baked in (not configurable at runtime).

---

## How sync.Once Works Internally

Understanding the internals helps you answer follow-up questions in interviews.

```
sync.Once struct:
┌─────────────────────────────────┐
│  done  uint32   (0 = not done)  │
│  m     Mutex                    │
└─────────────────────────────────┘
```

When `Do(f)` is called:

```
Scenario A: First call (done == 0)
─────────────────────────────────
Goroutine checks done via atomic load → 0
Goroutine acquires mutex
Goroutine checks done again → still 0
Goroutine calls f()
Goroutine sets done = 1 via atomic store
Goroutine releases mutex
Returns.

Scenario B: Subsequent calls (done == 1)
────────────────────────────────────────
Goroutine checks done via atomic load → 1
Returns immediately (no lock acquired).
```

The fast path (Scenario B) uses an **atomic read** — no mutex acquisition — making subsequent calls after initialization essentially free. This is why `sync.Once` is preferred over a plain mutex check.

**Important nuance:** If `f()` panics, `sync.Once` still marks it as done. Future calls to `Do` will not re-run `f`. This is intentional but can be surprising. If initialization can fail and you need retries, you need a different mechanism (a custom struct with a flag and error field).

---

## Singleton vs Dependency Injection

In production Go code, explicit **dependency injection** (DI) is almost always preferred over Singleton for application-level objects. The two are not mutually exclusive — you can create one instance and inject it everywhere.

```
Singleton approach:
  db := database.GetInstance(dsn)   ← global, hidden
  handler.Process(orderID)          ← secretly uses db

Dependency injection approach:
  db := database.NewPool(dsn)       ← one instance, created in main()
  handler := handler.New(db)        ← explicitly receives db
  handler.Process(orderID)          ← uses the injected db
```

Both use one `Pool` instance. The difference: DI makes the dependency explicit, mockable in tests, and replaceable. Singleton hides it.

**Rule of thumb for Go:** Use `sync.Once` Singleton for truly global infrastructure (logger, metrics client, feature flags). Use dependency injection for anything your business logic depends on.

---

## When to Use Singleton

- **Connection pools** (database, HTTP client with a transport layer)
- **Configuration** loaded once from env/file at startup
- **Logger** or metrics client shared across the app
- **Cache client** (Redis, Memcached)
- **Feature flag client** (LaunchDarkly, etc.)
- Hardware or OS resource handles (serial port, GPU context)

---

## When Not to Use Singleton

- **Domain objects** (a `User`, `Order`, or `Product` should never be a Singleton)
- **Services under unit test** — prefer DI so you can inject mocks
- **Anything that varies by context** (per-request database transaction, per-tenant config)
- When you find yourself writing `GetInstance` in business logic rather than infrastructure setup

---

## What Interviewers Actually Look For

**In an LLD interview:**

- Can you implement thread-safe Singleton in Go without being asked? The expected answer uses `sync.Once`.
- Do you know *why* the naive implementation is wrong? (data race, not just "it's not thread-safe")
- Can you articulate the trade-offs — testability problems, global state risks?
- Do you proactively suggest dependency injection as an alternative and explain when each is appropriate?

**Common follow-up questions:**

- *"What happens if the initialization function panics?"* — `sync.Once` marks it done; future calls skip it. If you need retry-on-failure, roll a custom implementation.
- *"How would you test code that uses a Singleton?"* — Option 1: accept an interface, inject a mock. Option 2: provide a `Reset()` function for tests (test-only, not for production use). Option 3: refactor away from Singleton to DI.
- *"Is Singleton the same as a global variable?"* — Functionally yes, with controlled construction. The pattern adds guaranteed single initialization and a controlled access point, but does not eliminate the global state problem.
- *"Can you have multiple Singletons of the same type?"* — Conceptually no (that would not be a Singleton). In practice, you can have multiple `sync.Once` variables for different "singleton" instances of the same struct (e.g., a primary DB pool and a replica DB pool — two Singletons of the same type with different configs).

---

## Interview Cheat Sheet

```
Pattern:        Singleton (Creational)
Intent:         One instance, global access point
Go mechanism:   sync.Once (preferred), init(), package-level var

Thread safety:
  Naive check-then-set    → data race (BROKEN)
  Mutex around every call → correct, hot-path overhead
  sync.Once               → correct, atomic fast path after init

sync.Once internals:
  done uint32 + mutex
  Fast path: atomic load of done
  Slow path: mutex lock + re-check + call f() + atomic set done

Problems with Singleton:
  1. Global state → hard to reason about
  2. Hidden dependencies → tight coupling
  3. Hard to test → can't inject mocks
  4. Concurrency bugs if implemented naively

Prefer over Singleton:
  Dependency injection — create one instance in main(), pass explicitly

Use Singleton for:
  Logger, config, DB pool, cache client, metrics

Do NOT use for:
  Domain objects, per-request state, anything mockable in tests
```

---

## References

- *Design Patterns: Elements of Reusable Object-Oriented Software* — Gang of Four (1994)
- [sync.Once — Go standard library source](https://cs.opensource.google/go/go/+/main:src/sync/once.go)
- [Effective Go — Initialization](https://go.dev/doc/effective_go#initialization)
- [Go Memory Model](https://go.dev/ref/mem) — explains why atomic operations in sync.Once provide the right visibility guarantees
- *100 Go Mistakes and How to Avoid Them* — Teiva Harsanyi, Mistake #74 (copying a sync type — covers sync.Once, sync.Mutex, and other sync primitives that must not be copied after first use)