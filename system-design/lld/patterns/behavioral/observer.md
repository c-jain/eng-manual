# Observer Pattern

## Table of Contents

- [What It Is](#what-it-is)
- [Why It Exists](#why-it-exists)
- [Why "Observer"](#why-observer)
- [Structure](#structure)
- [Push vs Pull Model](#push-vs-pull-model)
- [Go Implementation](#go-implementation)
- [Thread Safety](#thread-safety)
- [Problems It Brings](#problems-it-brings)
- [Observer vs Pub/Sub](#observer-vs-pubsub)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [How to Remember It](#how-to-remember-it)
- [References](#references)

## What It Is

Observer is a behavioural design pattern that defines a **one-to-many dependency** between objects: when one object (the *subject*) changes state, all registered dependents (the *observers*) are notified and updated automatically.

It is a behavioural pattern — it governs communication between objects, not their construction or composition.

## Why It Exists

Without Observer, an object that changes state must directly call every other object that cares about that change. That creates tight coupling: the subject must know each of its consumers by name and type. Adding a new consumer requires modifying the subject.

Observer flips this. Consumers register themselves with the subject via an interface. The subject knows only the interface, not the concrete implementations. Adding a new consumer means writing a new type that implements the interface and registering it — the subject is never touched.

## Why "Observer"

Dependents *observe* the subject for state changes — like watching for events. The subject is the observable thing; its dependents are the observers.

The GoF book uses the names Subject and Observer. The pattern also appears as:
- Publisher/Subscriber (when a broker is involved — see [Observer vs Pub/Sub](#observer-vs-pubsub))
- EventEmitter (Node.js)
- Reactive streams (RxGo, RxJS) — Observer is the foundational pattern beneath all reactive programming

## Structure

```
«interface»                      «interface»
Observer                         Subject
+----------+                     +---------------+
| OnEvent()|                     | Register()    |
+----------+                     | Deregister()  |
      ^                          | Notify()      |
      |                          +---------------+
  implements                           ^
      |                           implements
+----------------+                     |
| AlertObserver  |             +----------------+
| LogObserver    |<--holds-----| StockStore     |
+----------------+             | observers []   |
                               +----------------+
                                      |
                                  calls OnEvent()
                                  on each observer
```

The subject holds a slice of `Observer` interface values. It does not know the concrete types. This is the source of the decoupling.

## Push vs Pull Model

**Push model** — the subject passes relevant data directly in the notification call. Observers receive what they need immediately.

```
subject.Notify(Event{Topic: "AAPL", Data: 205.0})
// Observer receives the data in OnEvent(e Event)
```

Simpler for observers. The subject must decide what data to include.

**Pull model** — the subject notifies observers that something changed, and observers pull what they need from the subject.

```
subject.Notify()  // no data
// Observer calls subject.GetPrice("AAPL") inside OnEvent
```

More flexible — observers fetch only what they need. Requires observers to hold a reference to the subject, which introduces a circular dependency risk.

Go idiom favours the push model: pass data through the notification call. It avoids back-references and keeps observer logic self-contained.

## Go Implementation

```go
// event.go

package observer

// Event carries the change data from subject to observer.
// A struct keeps the interface stable — new fields can be added
// without changing the Observer interface signature.
type Event struct {
    Topic string
    Data  any
}

// Observer is defined in the consumer package (Go idiom).
// Any type with an OnEvent method satisfies it implicitly.
type Observer interface {
    OnEvent(e Event)
}

// compile-time check (optional, useful in larger codebases)
// var _ Observer = (*AlertObserver)(nil)

// Subject manages registration and notification.
type Subject interface {
    Register(o Observer)
    Deregister(o Observer)
    Notify(e Event)
}
```

```go
// store.go — concrete subject

package observer

import "sync"

type StockStore struct {
    mu        sync.RWMutex
    observers []Observer
    prices    map[string]float64
}

func NewStockStore() *StockStore {
    return &StockStore{prices: make(map[string]float64)}
}

func (s *StockStore) Register(o Observer) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.observers = append(s.observers, o)
}

func (s *StockStore) Deregister(o Observer) {
    s.mu.Lock()
    defer s.mu.Unlock()
    filtered := s.observers[:0]
    for _, obs := range s.observers {
        if obs != o {
            filtered = append(filtered, obs)
        }
    }
    s.observers = filtered
}

func (s *StockStore) Notify(e Event) {
    s.mu.RLock()
    // Snapshot observers before releasing the lock.
    // This prevents holding the lock during callbacks,
    // which would risk deadlock if OnEvent re-enters the subject.
    snapshot := make([]Observer, len(s.observers))
    copy(snapshot, s.observers)
    s.mu.RUnlock()

    for _, o := range snapshot {
        o.OnEvent(e)
    }
}

// SetPrice updates state and triggers notification.
func (s *StockStore) SetPrice(ticker string, price float64) {
    s.mu.Lock()
    s.prices[ticker] = price
    s.mu.Unlock()

    s.Notify(Event{Topic: ticker, Data: price})
}
```

```go
// observers.go — concrete observers

package observer

import "fmt"

// AlertObserver fires when a price exceeds a threshold.
type AlertObserver struct {
    Ticker    string
    Threshold float64
}

func (a *AlertObserver) OnEvent(e Event) {
    if e.Topic != a.Ticker {
        return // ignore events for other tickers
    }
    price, ok := e.Data.(float64)
    if !ok {
        return
    }
    if price > a.Threshold {
        fmt.Printf("[ALERT] %s crossed %.2f — current: %.2f\n",
            a.Ticker, a.Threshold, price)
    }
}

// LogObserver records every price change regardless of ticker.
type LogObserver struct{}

func (l *LogObserver) OnEvent(e Event) {
    fmt.Printf("[LOG] %s → %v\n", e.Topic, e.Data)
}
```

```go
// main.go

package main

import "observer"

func main() {
    store := observer.NewStockStore()

    logger := &observer.LogObserver{}
    alert  := &observer.AlertObserver{Ticker: "AAPL", Threshold: 200.0}

    store.Register(logger)
    store.Register(alert)

    store.SetPrice("AAPL", 195.0) // log only — below threshold
    store.SetPrice("AAPL", 205.0) // log + alert
    store.SetPrice("GOOG", 180.0) // log only — alert ignores non-AAPL

    store.Deregister(logger)
    store.SetPrice("AAPL", 210.0) // alert only
}
```

**Output:**
```
[LOG] AAPL → 195
[LOG] AAPL → 205
[ALERT] AAPL crossed 200.00 — current: 205.00
[LOG] GOOG → 180
[ALERT] AAPL crossed 200.00 — current: 210.00
```

## Thread Safety

The snapshot pattern in `Notify` above is deliberate and important:

```go
s.mu.RLock()
snapshot := make([]Observer, len(s.observers))
copy(snapshot, s.observers)
s.mu.RUnlock()

for _, o := range snapshot {
    o.OnEvent(e)  // called without holding the lock
}
```

Why not hold the read lock during the loop?

- An observer's `OnEvent` call might try to call `Register` or `Deregister` on the same subject (e.g., a one-shot observer that removes itself after firing)
- That would attempt to acquire a write lock while the read lock is held → deadlock
- The snapshot ensures observers added or removed during notification take effect on the next `Notify` call, not the current one

## Problems It Brings

- **The lapsed listener problem** — an observer registers but never deregisters. The subject holds a reference to it, preventing garbage collection even after the observer's owner is done with it. Explicit lifecycle management (call `Deregister` in cleanup/finaliser logic) is required.
- **Undefined notification order** — observers are notified in registration order in a slice-based implementation, but this is an internal detail. Never write observer logic that depends on a particular notification order.
- **Cascading updates** — an observer's `OnEvent` triggers a state change in the subject, which calls `Notify` again, creating an infinite loop. Be careful when observers modify the subject they are observing.
- **Synchronous blocking** — by default, `Notify` calls each observer sequentially. A slow observer delays all subsequent observers and blocks the goroutine that called `SetPrice`. For high-throughput scenarios, notify each observer in its own goroutine — but then observers must synchronise their own shared state.
- **Implicit coupling on event data** — observers that type-assert `e.Data.(float64)` break silently if the event data type changes. Using typed event structs or a discriminated union approach makes this explicit.

## Observer vs Pub/Sub

These are related but distinct:

- **Observer** — same-process, direct coupling between subject and observer via an interface. The subject calls observers synchronously (or in goroutines it manages). No broker.
- **Pub/Sub** — cross-process (or cross-goroutine) with a message broker in between. Publisher emits to a topic; broker stores and delivers; subscriber consumes asynchronously. Publisher and subscriber are decoupled in time and space — the publisher does not know subscribers exist and does not block on delivery.

Observer is appropriate when:
- Components are in the same process
- You want the subject to control notification lifecycle
- You need synchronous or near-synchronous reactions

Pub/Sub (Kafka, RabbitMQ, NATS) is appropriate when:
- Components are separate services or processes
- Durability and replay of events are needed
- Publisher and subscriber must be independently scalable

## Interview Cheat Sheet

- **Pattern category** — behavioural (communication between objects)
- **Core intent** — one-to-many notification without the subject knowing its observers
- **Go idiom** — `Observer` interface defined in the consumer package; implicit satisfaction; no registration annotation needed
- **Thread safety** — snapshot the observer slice before notifying; never hold a lock during callbacks
- **Push vs pull** — push is idiomatic in Go (pass data in the event); pull risks circular references
- **Lapsed listener** — the memory-leak case; always provide and call `Deregister`
- **vs pub/sub** — Observer is in-process, direct; pub/sub is cross-process, broker-mediated, decoupled in time
- **Cascading updates** — dangerous; be careful if observers can modify the subject

## How to Remember It

Think of a **YouTube channel subscription**:
- The channel is the subject — it publishes state changes (new video)
- Subscribers are observers — they are notified when the channel uploads
- Subscribers can unsubscribe (deregister) at any time
- The channel does not know or care who is subscribed — it just notifies whoever is registered

The lapsed listener problem maps to: a viewer deletes their account but YouTube still tries to send them notifications (holds a dangling reference).

Observer is also the foundation of Node.js's `EventEmitter`, Go's channel-based fan-out patterns, and reactive programming libraries (RxGo, RxJS).

## References

- Gamma, Helm, Johnson, Vlissides — *Design Patterns: Elements of Reusable Object-Oriented Software* (GoF), Observer pattern
- Refactoring Guru — [Observer](https://refactoring.guru/design-patterns/observer)
- Dave Cheney — [Practical Go](https://dave.cheney.net/practical-go/presentations/gophercon-israel.html) (interface design principles)
- Go standard library — `sync` package for `RWMutex`
- `sony/gobreaker` — example of Observer-adjacent notification in Go infrastructure libraries
- RxGo — [Observer pattern in reactive Go](https://github.com/ReactiveX/RxGo)