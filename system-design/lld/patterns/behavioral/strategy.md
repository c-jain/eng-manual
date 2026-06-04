# Strategy Pattern

## Table of Contents

- [What It Is](#what-it-is)
- [Why It Exists](#why-it-exists)
- [Structure](#structure)
- [Go Implementation](#go-implementation)
- [Function Type Variant (Idiomatic Go)](#function-type-variant-idiomatic-go)
- [Real-World Examples](#real-world-examples)
- [Strategy vs Related Patterns](#strategy-vs-related-patterns)
- [Problems It Introduces](#problems-it-introduces)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

---

## What It Is

The Strategy pattern defines a family of algorithms, encapsulates each one as a type, and makes
them interchangeable at runtime. The client selects which algorithm to use; the algorithm's code
is not hardwired into the client.

**GoF definition:** *"Define a family of algorithms, encapsulate each one, and make them
interchangeable. Strategy lets the algorithm vary independently from clients that use it."*

**Why it is called Strategy:** Think of a general choosing a *battle strategy* — siege, flanking
attack, feigned retreat. The army (context) is the same; only the chosen approach changes. The
general doesn't rebuild the army to switch strategy — they issue a different order. In software:
the context doesn't change; only the plugged-in algorithm does.

**How to remember:** Strategy = *swappable algorithm*. One interface, many implementations,
client picks at runtime. It is Open/Closed Principle applied to algorithms.

---

## Why It Exists

Without Strategy, algorithm variation lives as a `switch` or `if/else` block inside a method:

```go
// Without Strategy — bad
func (s *Sorter) Sort(data []int) []int {
    switch s.algorithm {
    case "bubble":
        // bubble sort logic
    case "quick":
        // quick sort logic
    case "merge":
        // merge sort logic
    }
}
```

Problems:
- Adding a new algorithm requires modifying the `Sort` method — violates OCP
- Each algorithm is hidden inside a large function — hard to test in isolation
- The `Sorter` struct grows unboundedly as algorithms are added
- Cannot swap algorithms at runtime without exposing implementation detail

Strategy replaces the switch with a plugged-in type. Each algorithm lives in its own struct.
Adding a new one means adding a new type — zero changes to existing code.

---

## Structure

```
«interface»
 Strategy
─────────────────
 Execute(input)
       ▲
       │ implements
 ┌─────┴──────────────────┐
 │                        │
ConcreteStrategyA    ConcreteStrategyB
 Execute(input)       Execute(input)


      Context
────────────────────────
 strategy  Strategy    ◀── holds reference to interface
 SetStrategy(Strategy)
 DoWork()              ── delegates to strategy.Execute(...)
```

Roles:
- **Strategy interface** — the contract all algorithms must satisfy. Context only knows this type.
- **Concrete Strategy** — one type per algorithm. Each encapsulates one behaviour.
- **Context** — holds a reference to a Strategy and delegates algorithm work to it. Context is
  oblivious to which concrete strategy is in use.

The Context and Strategy are **composed**, not inherited. This is a classic "favour composition
over inheritance" pattern.

---

## Go Implementation

```go
package main

import "fmt"

// Strategy interface — the contract
type SortStrategy interface {
    Sort(data []int) []int
}

// Concrete Strategy A: Bubble Sort
type BubbleSort struct{}

func (b BubbleSort) Sort(data []int) []int {
    d := append([]int{}, data...) // defensive copy
    n := len(d)
    for i := 0; i < n-1; i++ {
        for j := 0; j < n-1-i; j++ {
            if d[j] > d[j+1] {
                d[j], d[j+1] = d[j+1], d[j]
            }
        }
    }
    return d
}

// Concrete Strategy B: Quick Sort
type QuickSort struct{}

func (q QuickSort) Sort(data []int) []int {
    if len(data) <= 1 {
        return data
    }
    pivot := data[len(data)/2]
    var left, right, mid []int
    for _, v := range data {
        switch {
        case v < pivot:
            left = append(left, v)
        case v > pivot:
            right = append(right, v)
        default:
            mid = append(mid, v)
        }
    }
    result := q.Sort(left)
    result = append(result, mid...)
    result = append(result, q.Sort(right)...)
    return result
}

// Context
type Sorter struct {
    strategy SortStrategy
}

func (s *Sorter) SetStrategy(strategy SortStrategy) {
    s.strategy = strategy
}

func (s *Sorter) Sort(data []int) []int {
    return s.strategy.Sort(data)
}

func main() {
    data := []int{5, 3, 8, 1, 9, 2}

    sorter := &Sorter{}

    sorter.SetStrategy(BubbleSort{})
    fmt.Println("Bubble:", sorter.Sort(data)) // [1 2 3 5 8 9]

    sorter.SetStrategy(QuickSort{})
    fmt.Println("Quick: ", sorter.Sort(data)) // [1 2 3 5 8 9]
}
```

**What makes this Strategy and not just polymorphism:** the *context* exists separately from the
strategies and can have its own state. The strategy is *delegated to*, not *extended from*.

---

## Function Type Variant (Idiomatic Go)

When the Strategy interface has only one method, Go lets you express it more lightly with a
function type. This is idiomatic for simple cases.

```go
// Instead of a single-method interface, use a function type
type SortFunc func([]int) []int

type Sorter struct {
    sortFn SortFunc
}

func (s *Sorter) SetStrategy(fn SortFunc) {
    s.sortFn = fn
}

func (s *Sorter) Sort(data []int) []int {
    return s.sortFn(data)
}

// Usage — any func with the right signature satisfies the type
bubbleSort := func(data []int) []int {
    // ... implementation
    return data
}

sorter := &Sorter{}
sorter.SetStrategy(bubbleSort)
```

**When to use a function type vs a full interface:**
- Function type: strategy has a single action, no internal state needed, simple composability
- Interface: strategy needs multiple methods, or concrete strategies need to carry state between
  calls (e.g., a strategy that keeps a cache internally)

---

## Real-World Examples

**Payment processing**
```go
type PaymentStrategy interface {
    Pay(amount float64) error
}

// Implementations: StripeStrategy, PayPalStrategy, CryptoStrategy
// Context: Checkout — calls strategy.Pay(total) without knowing which provider
```

**Compression**
```go
type CompressionStrategy interface {
    Compress(data []byte) ([]byte, error)
}
// Implementations: GzipStrategy, ZstdStrategy, BrotliStrategy
// Context: FileArchiver — user selects compression level; strategy is plugged in
```

**Authentication middleware**
```go
type AuthStrategy interface {
    Authenticate(r *http.Request) (*User, error)
}
// Implementations: JWTStrategy, APIKeyStrategy, OAuthStrategy
// Context: middleware chain — selects strategy based on request header
```

**Navigation routing**
- Fastest route, shortest route, avoid tolls, avoid highways — each is a strategy
- Context: Navigator — user selection determines which strategy is active

---

## Strategy vs Related Patterns

| Pattern | What it encapsulates | Who picks the variant | Key distinction |
|---|---|---|---|
| Strategy | Algorithm / behaviour | The client (externally) | Behaviour swapped at runtime by the caller |
| Template Method | Algorithm skeleton | Defined in base class; steps overridden in subclass | Uses inheritance; overall shape is fixed |
| State | State-specific behaviour | The object itself (on transition) | Object transitions between strategies automatically |
| Command | A request / action | The client | Encapsulates *what to do*, not *how to compute* |

**Strategy vs State (the common confusion):**

```
Strategy: client controls                State: object controls
─────────────────────────                ──────────────────────
client.SetStrategy(QuickSort{})          (internal)
client.Sort(data)                        obj.handleEvent() → transitions state
                                         obj.doWork() → delegates to current state
```

In State, the context transitions itself between behaviours based on internal conditions.
In Strategy, the context never changes its own strategy — only the caller does.

**Strategy vs Template Method:**
- Template Method: "Here is the algorithm skeleton; subclasses fill in the blanks."
- Strategy: "Here is a slot; plug in the whole algorithm."
- Template Method uses inheritance. Strategy uses composition.

---

## Problems It Introduces

**Client must know the strategies.** The caller is responsible for choosing between
`BubbleSort{}` and `QuickSort{}`. Strategy moves the decision to the caller; it does not
eliminate the decision.

**Class proliferation.** Each algorithm becomes a type. Ten algorithms = ten types. For
simple cases in Go, function types avoid this cost.

**Stateful strategies need care.** If a strategy carries internal state, sharing one instance
across concurrent goroutines is a data race. Either make strategies stateless (preferred), use
`sync.Mutex`, or instantiate a fresh strategy per use.

**Overkill for simple variation.** If there are only two algorithms and they will never change,
a simple boolean parameter or two methods is clearer than a full Strategy setup.

---

## Interview Cheat Sheet

**When to suggest Strategy:**
- "The algorithm or behaviour needs to vary at runtime."
- "I keep adding cases to a switch block — each case should be its own type."
- "I want to be able to test each algorithm in isolation."

**Core signal phrases:**
- "I'd model this as a Strategy — the algorithm varies independently from the context."
- "In Go, if the interface has one method and strategies are stateless, I'd use a function type
  instead of a full interface — it's lighter and more idiomatic."
- "Strategy is OCP in action — adding a new algorithm means a new type, zero changes to
  existing code."
- "The difference between Strategy and State: in Strategy the caller chooses the behaviour;
  in State the object transitions itself."

**Common follow-up questions:**
- "How is this different from just calling different functions?" → Strategy provides a uniform
  interface; the context doesn't know which function it's calling, enabling runtime substitution.
- "What if I have 20 strategies?" → That's fine architecturally, but question whether all 20
  are needed — some may be parameterisable variants of fewer base strategies.
- "How do you test this?" → Test each concrete strategy independently with unit tests. Test
  the context with a mock/stub strategy that records calls.
- "What's the Go-idiomatic version?" → Function type when single-method and stateless; full
  interface when multiple methods or strategies carry state.

**SOLID connections:**
- OCP: adding strategies never modifies the context
- LSP: all concrete strategies are substitutable for the interface
- DIP: context depends on the abstract Strategy, not on concrete implementations

---

## References

- GoF — *Design Patterns: Elements of Reusable Object-Oriented Software* (Strategy, p.315)
- Refactoring Guru — [Strategy Pattern](https://refactoring.guru/design-patterns/strategy)
- Dave Cheney — [Practical Go: Interfaces](https://dave.cheney.net/practical-go/presentations/qcon-china.html)
- Effective Go — [Interfaces and other types](https://go.dev/doc/effective_go#interfaces_and_types)