---
Status: 🌳 Evergreen
Created: 2026-06-09
Last Updated: 2026-06-09
---

# Chain of Responsibility

## Table of Contents

1. [Intent](#intent)
2. [Why It Exists](#why-it-exists)
3. [Why Called "Chain of Responsibility"](#why-called-chain-of-responsibility)
4. [Problems It Brings](#problems-it-brings)
5. [GoF Structure](#gof-structure)
6. [Two Variants](#two-variants)
7. [Implementation — Go (Classical GoF)](#implementation--go-classical-gof)
8. [Implementation — Go (Idiomatic Middleware)](#implementation--go-idiomatic-middleware)
9. [Real-World Analogies](#real-world-analogies)
10. [Comparison with Related Patterns](#comparison-with-related-patterns)
11. [Interview Cheat Sheet](#interview-cheat-sheet)
12. [Mnemonics](#mnemonics)
13. [References](#references)

---

## Intent

> Pass a request along a chain of handlers. Each handler decides either to process the request or to pass it to the next handler in the chain.

The sender does not know which handler will process the request. The chain can be assembled, reordered, and extended at runtime.

**Category:** Behavioural | **GoF Book:** Chapter 5

---

## Why It Exists

Without CoR, dispatching a request to the right handler requires a monolithic `if-else` or `switch` in one central place:

```go
func handle(r Request) {
    if r.Level < 5 {
        monkeyHandler(r)
    } else if r.Level < 10 {
        dogHandler(r)
    } else {
        catHandler(r)
    }
}
```

Adding a new handler requires modifying this function — violating the **Open/Closed Principle**. Chain of Responsibility externalises the routing so:
- New handlers can be added without touching existing code
- The chain can be reconfigured at runtime (different chains for different contexts)
- Each handler is a self-contained unit that only knows about its own condition

---

## Why Called "Chain of Responsibility"

Each link in the chain is given the opportunity to **take responsibility** for the request. If it can handle it, it does and the chain stops. If not, it passes responsibility to the next link. The word "responsibility" is precise — it's not just routing; each handler genuinely *owns* the request if it handles it.

---

## Problems It Brings

- **No guaranteed handling:** A request can reach the end of the chain with no handler taking it. This must be handled explicitly — a terminal handler, a default response, or a returned error.
- **Hard to debug:** The routing logic is distributed across handlers. Without logging at each link, tracing which handler processed what is difficult.
- **Silent misconfiguration:** A broken link (e.g., `SetNext` not called, `next` is nil) silently drops requests.
- **Performance in long chains:** Every handler is checked in sequence. Rarely a problem in practice, but worth noting for hot paths.
- **Implicit control flow:** Less discoverable than an explicit `switch`. Requires reading every handler to understand routing logic.

---

## GoF Structure

```
        «interface»
          Handler
  ──────────────────────
  Handle(r) string
  SetNext(Handler) Handler
            △
            │ implements
      BaseHandler (struct)
  ──────────────────────
  - next: Handler
  ──────────────────────
  SetNext(h) Handler       <- stores next, returns it (enables fluent chaining)
  Handle(r) string         <- passes to next if next != nil; returns "unhandled" otherwise
            △
       ┌────┴────┐
   HandlerA   HandlerB
  ──────────  ──────────
  Handle(r)   Handle(r)
  [own logic, [own logic,
  falls through falls through
  to BaseHandler] to BaseHandler]
```

**Request flow — three labelled scenarios:**

```
Scenario A — HandlerA handles:
  Client ──request──► HandlerA ──[handled]──► done

Scenario B — HandlerA passes, HandlerB handles:
  Client ──request──► HandlerA ──[pass]──► HandlerB ──[handled]──► done

Scenario C — No handler (request falls through chain):
  Client ──request──► HandlerA ──[pass]──► HandlerB ──[pass]──► nil (unhandled)
```

---

## Two Variants

| | Pure CoR (GoF) | Pipeline / Middleware |
|-|----------------|----------------------|
| Each handler | Handles OR passes — not both | Runs its logic, then always delegates to next (unless short-circuiting) |
| Chain stops when | First handler that handles | Explicit return / short-circuit by a handler |
| Request processed by | Exactly one handler (or none) | All handlers, in sequence |
| Use case | Ticket escalation, command routing, permission levels | Logging, auth, rate limiting, compression |
| Go example | Custom routing logic | `net/http` middleware |

---

## Implementation — Go (Classical GoF)

```go
package cor

import "fmt"

// Handler is the chain interface.
type Handler interface {
    Handle(request int) string
    SetNext(h Handler) Handler
}

// BaseHandler provides default pass-through behaviour.
// Concrete handlers embed this to inherit SetNext and the nil-safe default Handle.
type BaseHandler struct {
    next Handler
}

func (b *BaseHandler) SetNext(h Handler) Handler {
    b.next = h
    return h // return h enables fluent chaining: a.SetNext(b).SetNext(c)
}

// Handle passes to next if present; returns sentinel if chain is exhausted.
func (b *BaseHandler) Handle(request int) string {
    if b.next != nil {
        return b.next.Handle(request)
    }
    return "unhandled"
}

// MonkeyHandler handles requests where request < 5.
type MonkeyHandler struct {
    BaseHandler
}

func (m *MonkeyHandler) Handle(request int) string {
    if request < 5 {
        return fmt.Sprintf("MonkeyHandler: handled request %d", request)
    }
    return m.BaseHandler.Handle(request) // pass to next
}

// DogHandler handles requests where request < 10.
type DogHandler struct {
    BaseHandler
}

func (d *DogHandler) Handle(request int) string {
    if request < 10 {
        return fmt.Sprintf("DogHandler: handled request %d", request)
    }
    return d.BaseHandler.Handle(request)
}

// CatHandler is the terminal handler — handles everything remaining.
type CatHandler struct {
    BaseHandler
}

func (c *CatHandler) Handle(request int) string {
    return fmt.Sprintf("CatHandler: handled request %d", request)
}

// BuildChain assembles and returns the head of the chain.
func BuildChain() Handler {
    monkey := &MonkeyHandler{}
    dog := &DogHandler{}
    cat := &CatHandler{}
    monkey.SetNext(dog).SetNext(cat) // fluent: returns dog, then sets cat on dog
    return monkey
}
```

```go
// Usage
chain := cor.BuildChain()
fmt.Println(chain.Handle(3))  // MonkeyHandler: handled request 3
fmt.Println(chain.Handle(7))  // DogHandler: handled request 7
fmt.Println(chain.Handle(15)) // CatHandler: handled request 15
```

**Go embedding note:** `MonkeyHandler` embeds `BaseHandler` as an anonymous field, which promotes `SetNext` and the default `Handle` method onto `MonkeyHandler`. When `MonkeyHandler.Handle` calls `m.BaseHandler.Handle(request)`, it explicitly invokes the promoted method — which in turn calls `b.next.Handle(request)`. This is the Go substitute for abstract class inheritance.

---

## Implementation — Go (Idiomatic Middleware)

Go's `net/http` uses function composition rather than struct chains. Each middleware is a function that wraps an `http.Handler`. This is the **pipeline variant** of CoR — every middleware runs unless one short-circuits by returning early.

```go
package middleware

import (
    "log"
    "net/http"
)

// LoggingMiddleware logs every request, then always delegates — pipeline behaviour.
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Printf("%s %s", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
    })
}

// AuthMiddleware checks for an Authorization header.
// Short-circuits (pure CoR behaviour) if missing — chain stops here.
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Header.Get("Authorization") == "" {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return // chain does not continue
        }
        next.ServeHTTP(w, r)
    })
}

// Chain composes middleware left-to-right.
// Chain(A, B)(final) produces: A → B → final
func Chain(middlewares ...func(http.Handler) http.Handler) func(http.Handler) http.Handler {
    return func(final http.Handler) http.Handler {
        for i := len(middlewares) - 1; i >= 0; i-- {
            final = middlewares[i](final)
        }
        return final
    }
}
```

```go
// Usage
mux := http.NewServeMux()
mux.HandleFunc("/api", func(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("hello"))
})

handler := middleware.Chain(
    middleware.LoggingMiddleware,
    middleware.AuthMiddleware,
)(mux)
// Request path: LoggingMiddleware → AuthMiddleware → mux handler

http.ListenAndServe(":8080", handler)
```

The functional approach is more idiomatic Go: handlers are closures, the chain is function composition, no `SetNext`, no struct hierarchy.

---

## Real-World Analogies

| Analogy | Handlers | Stopping Condition |
|---------|----------|--------------------|
| Help desk escalation | L1 → L2 → L3 support | Issue resolved at current tier |
| HTTP middleware stack | logging → auth → rate-limit → handler | Middleware returns early (e.g., 401, 429) |
| DOM event bubbling | child → parent → document → window | `event.stopPropagation()` |
| Exception handling | inner `try-catch` → outer `try-catch` | Matching `catch` clause found |
| Logging framework | DEBUG → INFO → WARN → ERROR | Level matches threshold; log or discard |
| Airport security | document check → baggage scan → body scan | Flagged at any checkpoint |

---

## Comparison with Related Patterns

| | Chain of Responsibility | Decorator | Command |
|-|------------------------|-----------|---------|
| Purpose | Route request to one handler | Add behaviour around all handlers | Encapsulate request as an object |
| Chain executes | Stops at first handler that handles it | Always runs all handlers | No chain concept |
| Each handler | May or may not process | Always adds its behaviour, always delegates | Executes its own action |
| Sender knows handler? | No — sender only knows the chain head | Yes — wraps directly | Yes — invokes via invoker |
| Example | Auth middleware stopping on 401 | Logging + caching wrappers | Undo/redo queue |

**CoR vs Decorator — the key distinction:**
The structural diagram is nearly identical — both pattern objects hold a reference to a `next` object and delegate. The difference is **intent**:
- **Decorator** always delegates — the full chain always runs — it is about *adding behaviour* around all components.
- **CoR** may stop the chain — it is about *routing* a request to the one appropriate handler.

---

## Interview Cheat Sheet

**Signal Phrases (Demonstrate Depth)**
- "CoR lets me add new middleware without modifying existing handlers — that is the Open/Closed Principle in practice."
- "Go's `net/http` middleware is a pipeline variant of CoR — every handler runs unless one short-circuits with an early return."
- "I'd distinguish pure CoR (one handler handles, chain stops) from a pipeline (every handler runs) — the choice depends on whether handlers are mutually exclusive or composable."
- "The risk with CoR is a request falling off the end of the chain — I'd always add a terminal handler or an explicit not-found/default response."
- "CoR and Decorator look structurally identical but serve different purposes — CoR routes, Decorator decorates. The distinction is whether the full chain always runs."

**Red Flags**
- Confusing CoR with Decorator (same structure, different intent — this distinction is a common interview probe)
- Not handling the "no handler" case — the request silently disappears
- Hardcoding the chain order inside handlers — handlers should not know about each other
- Using CoR where a simple Strategy or switch would be clearer

**Common Interview Probes**
- "How is Chain of Responsibility different from a Decorator?"
- "Where have you seen this pattern in production Go code?"
- "What happens if no handler in the chain processes the request?"
- "How would you make the chain order configurable at runtime?"
- "When would you use the pipeline variant vs the pure GoF variant?"

---

## Mnemonics

**What CoR does:** "Hot potato — pass it down the chain until someone catches it."

**Pure vs Pipeline:**
- Pure CoR: **One and done** — first handler that can handle it, does.
- Pipeline: **Everyone runs** — each handler adds something; chain only stops on explicit short-circuit.

**Structure:** `Handler interface → BaseHandler (holds next, default pass-through) → ConcreteA | ConcreteB (override Handle with own condition)`

**CoR vs Decorator:** "CoR **routes** (one handles); Decorator **wraps** (all run)."

---

## References

- *Design Patterns: Elements of Reusable Object-Oriented Software* — GoF, Chain of Responsibility (p. 223)
- Refactoring.Guru — [Chain of Responsibility](https://refactoring.guru/design-patterns/chain-of-responsibility)
- Go standard library — [`net/http`](https://pkg.go.dev/net/http) (middleware composition in practice)