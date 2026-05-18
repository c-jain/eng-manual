# Builder Pattern

## Table of Contents

1. [What It Is](#what-it-is)
2. [Why It Exists](#why-it-exists)
3. [Why It's Called "Builder"](#why-its-called-builder)
4. [Structure](#structure)
5. [Go Implementation](#go-implementation)
6. [Problems It Introduces](#problems-it-introduces)
7. [Builder vs Related Patterns](#builder-vs-related-patterns)
8. [Interview Cheatsheet](#interview-cheatsheet)
9. [References](#references)

---

## What It Is

**Builder** is a creational design pattern that constructs a complex object step by step. Instead of one giant constructor call, you chain setter-like methods on a builder object, then call `Build()` (or a constructor with variadic options) to get the finished product.

```
NewBuilder(required).
    WithA(x).
    WithB(y).
    WithC(z).
    Build()
```

---

## Why It Exists

### The Problem: Telescoping Constructor Anti-Pattern

When a struct has many fields — most of them optional — constructors grow unwieldy:

```go
// Hard to read: what does each value mean?
NewServer("localhost", 8080, 30, 100, true, 3, "info", false, nil)
```

This is the **telescoping constructor anti-pattern**: the function signature grows with every new option. Callers must pass values for everything even when they care about only a few fields.

### Alternative Considered: Bare Struct Literal

```go
// Works, but:
// - Exposes unexported fields if same package only
// - No validation before object is used
// - Requires knowing all field names at call site
s := Server{Host: "localhost", Port: 8080}
```

### What Builder Gives You

| Problem | Builder Solution |
|---|---|
| Long constructor signatures | Required fields in constructor, optional in `With*` methods |
| No validation | `Build()` can validate the whole config before constructing |
| Unclear call sites | Method names document intent: `WithTLS()`, `WithTimeout(10s)` |
| Open/closed for extension | Add a new `With*` method without changing existing callers |

---

## Why It's Called "Builder"

The metaphor is construction: a **director** (the caller) tells a **builder** (the builder object) what to assemble. The builder knows how to put the pieces together; the director just specifies the desired configuration. The output is the **product** (the finished struct). Just like a construction foreman telling workers to add a roof, then windows, then paint — each step adds to the structure until `Build()` signs off on the completed object.

---

## Structure

```
┌──────────────────┐         ┌───────────────────────┐
│ Director         │  uses   │ Builder               │
│ (caller)         │────────►│  - field A            │
│                  │         │  - field B            │
└──────────────────┘         │  + WithA() *Builder   │
                             │  + WithB() *Builder   │
                             │  + Build() (*Product, error)
                             └───────────┬───────────┘
                                         │ returns
                                         ▼
                                   ┌───────────┐
                                   │  Product  │
                                   └───────────┘
```

---

## Go Implementation

Go doesn't have a canonical class hierarchy, so Builder appears in two idioms.

### Flavour 1 — Explicit Builder Struct (Classic)

```go
package main

import (
    "fmt"
    "time"
)

// Product
type HTTPClient struct {
    baseURL    string
    timeout    time.Duration
    maxRetries int
    headers    map[string]string
    tlsEnabled bool
}

// Builder
type HTTPClientBuilder struct {
    baseURL    string
    timeout    time.Duration
    maxRetries int
    headers    map[string]string
    tlsEnabled bool
    err        error // accumulate errors; checked at Build()
}

func NewHTTPClientBuilder(baseURL string) *HTTPClientBuilder {
    return &HTTPClientBuilder{
        baseURL:    baseURL,
        timeout:    30 * time.Second, // sensible defaults
        maxRetries: 3,
        headers:    make(map[string]string),
    }
}

func (b *HTTPClientBuilder) WithTimeout(d time.Duration) *HTTPClientBuilder {
    if d <= 0 {
        b.err = fmt.Errorf("timeout must be positive, got %v", d)
    }
    b.timeout = d
    return b // return self for method chaining
}

func (b *HTTPClientBuilder) WithMaxRetries(n int) *HTTPClientBuilder {
    b.maxRetries = n
    return b
}

func (b *HTTPClientBuilder) WithHeader(key, value string) *HTTPClientBuilder {
    b.headers[key] = value
    return b
}

func (b *HTTPClientBuilder) WithTLS() *HTTPClientBuilder {
    b.tlsEnabled = true
    return b
}

func (b *HTTPClientBuilder) Build() (*HTTPClient, error) {
    if b.err != nil {
        return nil, b.err
    }
    if b.baseURL == "" {
        return nil, fmt.Errorf("baseURL is required")
    }
    return &HTTPClient{
        baseURL:    b.baseURL,
        timeout:    b.timeout,
        maxRetries: b.maxRetries,
        headers:    b.headers,
        tlsEnabled: b.tlsEnabled,
    }, nil
}

// Usage — reads like a sentence
func main() {
    client, err := NewHTTPClientBuilder("https://api.example.com").
        WithTimeout(10 * time.Second).
        WithMaxRetries(5).
        WithHeader("Authorization", "Bearer token").
        WithTLS().
        Build()
    if err != nil {
        panic(err)
    }
    _ = client
}
```

**Key technique — error accumulation:** Validation errors are stored in `b.err` and only surfaced at `Build()`. This keeps the fluent chain clean but means errors are deferred. See trade-offs below.

---

### Flavour 2 — Functional Options (Idiomatic Go)

Popularised by Dave Cheney. Options are functions that mutate a config struct. This is the **preferred Go idiom** — used by gRPC, zap, and many standard libraries.

```go
package main

import (
    "fmt"
    "time"
)

type HTTPClient struct {
    baseURL    string
    timeout    time.Duration
    maxRetries int
    headers    map[string]string
    tlsEnabled bool
}

// Option is a function that configures an HTTPClient and returns an error
type Option func(*HTTPClient) error

func WithTimeout(d time.Duration) Option {
    return func(c *HTTPClient) error {
        if d <= 0 {
            return fmt.Errorf("timeout must be positive, got %v", d)
        }
        c.timeout = d
        return nil
    }
}

func WithMaxRetries(n int) Option {
    return func(c *HTTPClient) error {
        c.maxRetries = n
        return nil
    }
}

func WithHeader(key, value string) Option {
    return func(c *HTTPClient) error {
        c.headers[key] = value
        return nil
    }
}

func WithTLS() Option {
    return func(c *HTTPClient) error {
        c.tlsEnabled = true
        return nil
    }
}

// Constructor applies all options in order
func NewHTTPClient(baseURL string, opts ...Option) (*HTTPClient, error) {
    c := &HTTPClient{
        baseURL:    baseURL,
        timeout:    30 * time.Second, // defaults
        maxRetries: 3,
        headers:    make(map[string]string),
    }
    for _, opt := range opts {
        if err := opt(c); err != nil {
            return nil, err
        }
    }
    return c, nil
}

// Usage
func main() {
    client, err := NewHTTPClient("https://api.example.com",
        WithTimeout(10*time.Second),
        WithMaxRetries(5),
        WithHeader("Authorization", "Bearer token"),
        WithTLS(),
    )
    if err != nil {
        panic(err)
    }
    _ = client
}
```

**Why this is idiomatic Go:**
- Callers only import `Option` type + the `With*` functions they need
- Third parties can define their own `Option` values without modifying the library
- Errors are returned eagerly (each option validates at application time)
- Zero boilerplate for callers who only need defaults: `NewHTTPClient("https://...")`

---

### Flavour Comparison

| | Builder Struct | Functional Options |
|---|---|---|
| Idiomatic Go? | Less idiomatic | ✅ Yes |
| Error surfacing | Deferred to `Build()` | Eager, per option |
| Third-party extensibility | Must export Builder struct | Callers define their own `Option` |
| Staged construction order | ✅ Easy (e.g. SQL query builder) | ❌ Order implicit |
| Used in popular libs? | Rare | ✅ gRPC, zap, AWS SDK v2 |
| When to prefer | Query builders, multi-phase construction | Config objects, most Go libraries |

---

### Staged Construction Example (Use Builder Struct)

When the order of steps matters — e.g. a SQL query builder — the explicit Builder struct shines because methods represent **ordered stages**:

```go
query := NewQueryBuilder("users").
    Select("id", "name", "email").
    Where("active = true").
    OrderBy("created_at DESC").
    Limit(20).
    Build()
// → "SELECT id, name, email FROM users WHERE active = true ORDER BY created_at DESC LIMIT 20"
```

Here you wouldn't use functional options because the stages must run in a meaningful order.

---

## Problems It Introduces

### 1. Boilerplate

Every new field requires a new `With*` method or `Option` function. For simple structs with 2–3 fields, this is overkill.

**Rule of thumb:** If your struct has fewer than 4 optional fields and no validation logic, a struct literal or a small constructor is fine.

### 2. Mutable Intermediate State

The builder holds partially-constructed state. If shared across goroutines, it needs a mutex. Don't share builders across goroutines — construct once, share the product.

### 3. Deferred Error Discovery (Builder Struct Flavour)

With error accumulation, a bad value set deep in a chain isn't discovered until `Build()`. The stack trace points to `Build()`, not to the line that set the bad value.

**Mitigation:** Use functional options (errors surface immediately), or panic-on-error in the `With*` methods if you're building internal-only infrastructure.

### 4. Required Field Ambiguity

It's not obvious from the API which fields are required vs optional.

**Mitigation:** Put required fields in the constructor (`NewHTTPClientBuilder(baseURL)`), never in `With*` methods. Document required fields explicitly.

---

## Builder vs Related Patterns

| Pattern | Intent | Key Difference |
|---|---|---|
| **Builder** | Construct a complex object step by step | Focus on *how* to build incrementally |
| **Factory Method** | Create an object, subclass decides which type | One call, type-dispatch logic |
| **Abstract Factory** | Create families of related objects | Multiple factories for a product family |
| **Prototype** | Clone an existing object | Copies state rather than constructing fresh |

**Interview distinction — Builder vs Factory:**
- Factory: "Give me *some* Server" — one call, often type-based dispatch
- Builder: "Give me a Server configured *exactly like this*" — incremental, validation-aware construction

---

## Interview Cheatsheet

| Question | Key Point |
|---|---|
| Why not a struct literal? | No validation; exposes internals; unclear intent at call site |
| Builder vs Factory? | Factory = one-call type dispatch. Builder = incremental construction with validation |
| Required vs optional fields? | Required → constructor. Optional → `With*` methods |
| Thread-safe? | Builder is not; share the *product*, not the builder |
| Idiomatic Go? | Functional options over a builder struct for config objects |
| When to use builder struct? | Staged/ordered construction (e.g. query builders, test data builders) |
| Real-world Go examples? | `grpc.Dial(..., grpc.WithInsecure())`, `zap.NewLogger(...)`, AWS SDK v2 client |

---

## References

- [Dave Cheney — Functional Options for Friendly APIs](https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis)
- [Go Patterns — Builder](https://github.com/tmrts/go-patterns/blob/master/creational/builder.md)
- [Refactoring Guru — Builder](https://refactoring.guru/design-patterns/builder)
- [Gang of Four — Design Patterns (Gamma et al.)](https://www.oreilly.com/library/view/design-patterns-elements/0201633612/)
- [Rob Pike — Self-Referential Functions (original functional options post)](https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html)