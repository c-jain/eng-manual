# Decorator Pattern

## Table of Contents

- [What It Is](#what-it-is)
- [Why It Exists](#why-it-exists)
- [Why It Is Called Decorator](#why-it-is-called-decorator)
- [How to Remember It](#how-to-remember-it)
- [Structure](#structure)
- [Go Implementation — Coffee Example](#go-implementation--coffee-example)
- [Real-World Go: HTTP Middleware](#real-world-go-http-middleware)
- [Real-World Go: io.Reader Chain](#real-world-go-ioreader-chain)
- [Problems the Pattern Brings](#problems-the-pattern-brings)
- [Decorator vs Related Patterns](#decorator-vs-related-patterns)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

---

## What It Is

The **Decorator** is a structural design pattern that wraps an object in another object implementing the same interface. The wrapper (decorator) adds behaviour before or after delegating the call to the wrapped object. To the caller, the wrapped and unwrapped objects look identical — both satisfy the same interface.

You can stack multiple decorators. Each layer adds exactly one responsibility. The result is a chain of objects where each one does its job and passes control inward (or outward on return).

---

## Why It Exists

**Inheritance** is the naive solution for adding behaviour: subclass and override. But inheritance has two serious problems at scale:

1. **Static composition** — the combination is decided at compile time. You cannot mix and match behaviours at runtime.
2. **Subclass explosion** — with `N` optional behaviours (e.g., logging, auth, caching), you need up to `2^N` subclasses to cover every combination.

Decorator solves both:
- Composition is decided at **runtime** — stack whatever decorators you need when building the object.
- Each behaviour lives in exactly one decorator class — no duplication.

---

## Why It Is Called Decorator

Like decorating a room — you add furniture, lighting, and shelves on top of the base structure without changing the walls or floor. The room (base object) is intact and functional on its own; the decorations add capability without altering the thing they decorate.

---

## How to Remember It

**The onion model:** The base object is the core. Each decorator is a layer around it. A method call travels inward through all layers, reaches the core, and the result propagates back outward. Each layer can act on the inbound call, the outbound result, or both.

This is exactly how HTTP middleware works in Go — which is why middleware is the most common real-world Decorator you encounter.

```
Incoming request
  -> AuthMiddleware (layer 3)
    -> LoggingMiddleware (layer 2)
      -> RateLimitMiddleware (layer 1)
        -> [base handler] (core)
      <- returns response
    <- logs duration
  <- enforces auth header
Response
```

---

## Structure

In the GoF formulation there are four roles:

- **Component** (interface) — defines the operation that both the base and decorators implement.
- **ConcreteComponent** — the base object; the actual implementation.
- **BaseDecorator** — holds a reference to a Component; delegates to it. In Go, this role is absorbed by each concrete decorator directly — no abstract intermediate class is needed.
- **ConcreteDecorator** — extends BaseDecorator; adds its specific behaviour around the delegation.

```
<<interface>>
Component
+ Operation()
     ^
     | implements
+----+----------+        +---------------------------+
| ConcreteComp  |        | ConcreteDecoratorA        |
|---------------|        |---------------------------|
| Operation()   |        | - wrapped: Component      |
+---------------+        |---------------------------|
                         | + Operation()             |
                         |   (adds behaviour,        |
                         |    calls wrapped.Op())    |
                         +---------------------------+
                                    ^
                                    | same structure
                         +---------------------------+
                         | ConcreteDecoratorB        |
                         +---------------------------+
```

In Go, `ConcreteDecoratorA` holds a field of type `Component` (the interface). There is no inheritance; composition via an interface field replaces the GoF `BaseDecorator` class.

---

## Go Implementation — Coffee Example

A classic illustration. A `Beverage` has a cost and a description. Condiments (milk, sugar) are decorators that each add to both.

```go
package main

import "fmt"

// Component interface — the contract shared by base and all decorators.
type Beverage interface {
    Description() string
    Cost() float64
}

// ConcreteComponent — the plain base object.
type Espresso struct{}

func (e *Espresso) Description() string { return "Espresso" }
func (e *Espresso) Cost() float64       { return 1.99 }

// ConcreteDecorator: Milk.
// Holds a reference to *any* Beverage — base or already-decorated.
type MilkDecorator struct {
    beverage Beverage // the wrapped component
}

func (m *MilkDecorator) Description() string {
    return m.beverage.Description() + ", Milk" // delegate then augment
}
func (m *MilkDecorator) Cost() float64 {
    return m.beverage.Cost() + 0.30 // delegate then augment
}

// ConcreteDecorator: Sugar.
type SugarDecorator struct {
    beverage Beverage
}

func (s *SugarDecorator) Description() string {
    return s.beverage.Description() + ", Sugar"
}
func (s *SugarDecorator) Cost() float64 {
    return s.beverage.Cost() + 0.10
}

func main() {
    // Start with the bare base component.
    var b Beverage = &Espresso{}
    fmt.Println(b.Description(), b.Cost()) // Espresso 1.99

    // Wrap with Milk at runtime.
    b = &MilkDecorator{beverage: b}

    // Wrap with Sugar at runtime — stacking a second decorator.
    b = &SugarDecorator{beverage: b}

    fmt.Println(b.Description()) // Espresso, Milk, Sugar
    fmt.Printf("$%.2f\n", b.Cost())  // $2.39
}
```

**Call chain for `b.Cost()` after stacking:**

```
SugarDecorator.Cost()
  calls MilkDecorator.Cost()
    calls Espresso.Cost()  -> 1.99
  returns 1.99 + 0.30      -> 2.29
returns 2.29 + 0.10         -> 2.39
```

Each layer is unaware of the other layers. `MilkDecorator` does not know there is a `SugarDecorator` around it. This is the key property: **layers are independent and composable**.

---

## Real-World Go: HTTP Middleware

Go's `net/http` package is built around the `http.Handler` interface — a direct application of the Decorator pattern.

```go
package main

import (
    "log"
    "net/http"
)

// LoggingMiddleware decorates any http.Handler with request logging.
type LoggingMiddleware struct {
    next http.Handler
}

func (l *LoggingMiddleware) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    log.Printf("-> %s %s", r.Method, r.URL.Path) // before delegating
    l.next.ServeHTTP(w, r)                        // delegate to inner handler
    log.Printf("<- done")                         // after delegating
}

// AuthMiddleware decorates any http.Handler with token validation.
type AuthMiddleware struct {
    next http.Handler
}

func (a *AuthMiddleware) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if r.Header.Get("Authorization") == "" {
        http.Error(w, "unauthorised", http.StatusUnauthorized)
        return // short-circuit: do NOT call next
    }
    a.next.ServeHTTP(w, r)
}

// Usage: stack Auth around Logging around the real handler.
// Auth is outermost — it guards access before logging occurs.
var handler http.Handler = &AuthMiddleware{
    next: &LoggingMiddleware{
        next: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            w.Write([]byte("hello"))
        }),
    },
}
```

Note: a decorator can **short-circuit** — the `AuthMiddleware` returns early without calling `next` if auth fails. This is a legitimate and common variation.

---

## Real-World Go: io.Reader Chain

The standard library's I/O stack is a classic Decorator chain. Each type implements `io.Reader` and wraps another `io.Reader`.

```go
package main

import (
    "bufio"
    "compress/gzip"
    "os"
)

func main() {
    // Layer 0 (core): reads raw bytes from disk.
    f, _ := os.Open("data.gz")
    defer f.Close()

    // Layer 1: decompresses gzip transparently.
    gz, _ := gzip.NewReader(f)
    defer gz.Close()

    // Layer 2: buffers reads for efficiency (reduces syscall count).
    br := bufio.NewReader(gz)

    // The caller works with br — an io.Reader.
    // It has no knowledge of the layers beneath.
    line, _ := br.ReadString('\n')
    _ = line
}
```

This is the exact Decorator structure:
- `os.File` = ConcreteComponent
- `gzip.Reader` = ConcreteDecoratorA
- `bufio.Reader` = ConcreteDecoratorB
- `io.Reader` = Component interface

---

## Problems the Pattern Brings

- **Debugging difficulty:** A call passes through multiple layers. Stack traces can be noisy; it may be non-obvious which decorator is misbehaving.
- **Order sensitivity:** The order of stacking matters. Auth before logging vs logging before auth produces different behaviour and different security properties. There is no automatic enforcement of a correct order.
- **Verbose construction:** Deeply nested struct literals to build the chain are readable for three layers but become awkward at ten. Builder functions or functional options help.
- **Identity loss:** After wrapping, `b` is no longer `*Espresso` — it is `*SugarDecorator`. Type assertions fail. In Go, this is usually not a problem since you program to the interface.

---

## Decorator vs Related Patterns

| Pattern | Same structure? | Different intent |
|---|---|---|
| **Proxy** | Yes | Controls *access* (lazy init, access control, remote proxy). Usually wraps a known concrete type, not composable stacks. |
| **Adapter** | No | *Changes* the interface to match a different one. Decorator preserves the interface. |
| **Chain of Responsibility** | Similar | Any handler in the chain can stop propagation. In Decorator, delegation to `next` is nearly always done (except short-circuit middleware). |
| **Composite** | No | Composes objects into *tree structures*. Decorator wraps a single object. |

---

## Interview Cheat Sheet

**Common prompts and anchor answers:**

- *"What problem does Decorator solve?"* — Avoids subclass explosion when behaviour is optional and combinatorial; enables runtime composition of capabilities.
- *"How does Go's approach differ from GoF Decorator?"* — No abstract BaseDecorator class needed; the Component interface field in each concrete decorator serves the same role. Go's implicit interface satisfaction keeps it clean.
- *"Where have you seen this in real Go code?"* — `io.Reader`/`io.Writer` wrappers in the standard library; `http.Handler` middleware; logging/metrics/auth wrappers in any web framework.
- *"How is Decorator different from Proxy?"* — Same structure; different intent. Proxy controls *access* to an object. Decorator *adds behaviour* to an object. The distinction is conceptual and documented at the design level.
- *"Does the order of decorators matter?"* — Yes. Each layer applies its behaviour in sequence. Auth before logging vs logging before auth has different semantics (and security implications). Order is a design decision, not an accident.
- *"What are the downsides?"* — Hard to debug across many layers, order is not enforced, construction can be verbose.

**Signal phrases that show depth:**

- "same interface, wraps a reference to the same interface — the caller is unaware of wrapping"
- "runtime composition vs compile-time inheritance — that's the key trade-off"
- "HTTP middleware is the canonical Go Decorator; every framework uses it"
- "layers are independent; each adds one responsibility"
- "short-circuit is a valid variation — a decorator may not always call next"

---

## References

- Gamma, E. et al. *Design Patterns: Elements of Reusable Object-Oriented Software* (GoF) — Decorator, pp. 175–184. Addison-Wesley, 1994.
- Go standard library — `bufio`, `compress/gzip`, `net/http` packages: https://pkg.go.dev/std
- [Refactoring Guru — Decorator pattern](https://refactoring.guru/design-patterns/decorator)
- [Go by Example — Interfaces](https://gobyexample.com/interfaces)