# Facade Pattern

## Table of Contents

- [What It Is](#what-it-is)
- [Why It Exists](#why-it-exists)
- [The Name](#the-name)
- [Where It Fits Among Structural Patterns](#where-it-fits-among-structural-patterns)
- [Structure](#structure)
- [Go Implementation](#go-implementation)
- [Real-World Go Examples](#real-world-go-examples)
- [Trade-offs and Problems Introduced](#trade-offs-and-problems-introduced)
- [How to Remember It](#how-to-remember-it)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

---

## What It Is

The **Facade** pattern provides a unified, simplified interface to a set of interfaces in a subsystem. It does not replace or wrap the subsystem — the subsystem remains directly accessible. The Facade just offers a high-level entry point that coordinates the subsystem on the caller's behalf for common use cases.

It is a **structural** pattern (GoF classification) because it deals with how objects and classes are composed — specifically, how a client object relates to a set of subsystem objects through an intermediary.

---

## Why It Exists

Large systems accumulate internal complexity:

- Multiple subsystems each have their own API
- Clients must know the correct initialisation order
- Dependencies between subsystems are non-obvious
- The same multi-step sequence is repeated across many callers

Without a Facade, every caller duplicates the coordination logic. Any change to a subsystem propagates to all callers. The Facade centralises that coordination: callers call one method, the Facade handles everything underneath.

**Before Facade — caller must coordinate everything:**

```
Client
  |-- amp.On()
  |-- amp.SetVolume(20)
  |-- projector.On()
  |-- projector.WideScreen()
  |-- lights.Dim(10)
  |-- dvd.On()
  |-- dvd.Play(movie)
```

**After Facade — caller sees one operation:**

```
Client
  |-- theater.WatchMovie(movie)   // Facade handles all of the above
```

---

## The Name

"Facade" comes from the French/Italian word for "face" or "front." In architecture, the **building facade** is the outward-facing front surface — clean, intentional, and presentable — hiding the internal structure of floors, columns, pipes, and wiring.

In software, the pattern is exactly this: a clean front surface over a complex interior.

---

## Where It Fits Among Structural Patterns

These three patterns are frequently confused:

- **Adapter** — solves an *incompatibility* problem: changes an existing interface to match what the client expects. The underlying object's interface is wrong; Adapter fixes it.
- **Facade** — solves a *complexity* problem: simplifies a subsystem that has many correct interfaces. Nothing is incompatible; there is just too much to coordinate.
- **Proxy** — solves an *access control* problem: controls or mediates access to an object (lazy initialisation, logging, auth, remote proxy).
- **Decorator** — solves an *extension* problem: adds responsibilities to an object at runtime without subclassing.

Memory shortcut: Adapter = "wrong shape", Facade = "too complicated", Proxy = "gated access", Decorator = "add features".

---

## Structure

```
                         Facade
                    ┌─────────────────┐
       Client ────> │  HighLevelOp()  │
                    └────────┬────────┘
                             │ coordinates
               ┌─────────────┼─────────────┐
               v             v             v
        [SubsystemA]  [SubsystemB]  [SubsystemC]
         (direct        (direct       (direct
          access         access        access
          still OK)      still OK)     still OK)
```

Key points:
- The Facade holds references to subsystem objects (owns them or receives them)
- The client can still call subsystems directly if it needs fine-grained control
- The Facade does not add new behaviour — it *orchestrates* existing behaviour

---

## Go Implementation

Go has no inheritance, so the Facade pattern fits naturally: a struct whose exported methods coordinate multiple internal structs. Use interface types for subsystem dependencies if testability matters.

### Example 1: Home Theater System

```go
package main

import "fmt"

// ── Subsystems ─────────────────────────────────────────────────

type Amplifier struct{}

func (a *Amplifier) On()             { fmt.Println("Amplifier: on") }
func (a *Amplifier) SetVolume(v int) { fmt.Printf("Amplifier: volume → %d\n", v) }
func (a *Amplifier) Off()            { fmt.Println("Amplifier: off") }

type DVDPlayer struct{}

func (d *DVDPlayer) On()             { fmt.Println("DVD player: on") }
func (d *DVDPlayer) Play(m string)   { fmt.Printf("DVD player: playing '%s'\n", m) }
func (d *DVDPlayer) Stop()           { fmt.Println("DVD player: stopped") }
func (d *DVDPlayer) Off()            { fmt.Println("DVD player: off") }

type Projector struct{}

func (p *Projector) On()             { fmt.Println("Projector: on") }
func (p *Projector) WideScreen()     { fmt.Println("Projector: widescreen mode") }
func (p *Projector) Off()            { fmt.Println("Projector: off") }

type Lights struct{}

func (l *Lights) Dim(level int)      { fmt.Printf("Lights: dimmed to %d%%\n", level) }
func (l *Lights) On()                { fmt.Println("Lights: full brightness") }

// ── Facade ─────────────────────────────────────────────────────

// HomeTheater is the facade. It owns the subsystem objects and
// exposes high-level operations to the caller.
type HomeTheater struct {
    amp       *Amplifier
    dvd       *DVDPlayer
    projector *Projector
    lights    *Lights
}

func NewHomeTheater() *HomeTheater {
    return &HomeTheater{
        amp:       &Amplifier{},
        dvd:       &DVDPlayer{},
        projector: &Projector{},
        lights:    &Lights{},
    }
}

// WatchMovie coordinates all subsystems to prepare for a movie.
// The caller does not need to know anything about individual subsystems.
func (h *HomeTheater) WatchMovie(movie string) {
    fmt.Println("--- Get ready to watch a movie ---")
    h.lights.Dim(10)
    h.projector.On()
    h.projector.WideScreen()
    h.amp.On()
    h.amp.SetVolume(20)
    h.dvd.On()
    h.dvd.Play(movie)
}

// EndMovie shuts everything down in the correct order.
func (h *HomeTheater) EndMovie() {
    fmt.Println("--- Shutting down the home theater ---")
    h.dvd.Stop()
    h.dvd.Off()
    h.amp.Off()
    h.projector.Off()
    h.lights.On()
}

// ── Client ─────────────────────────────────────────────────────

func main() {
    theater := NewHomeTheater()
    theater.WatchMovie("Inception")
    fmt.Println()
    theater.EndMovie()
}
```

Output:
```
--- Get ready to watch a movie ---
Lights: dimmed to 10%
Projector: on
Projector: widescreen mode
Amplifier: on
Amplifier: volume → 20
DVD player: on
DVD player: playing 'Inception'

--- Shutting down the home theater ---
DVD player: stopped
DVD player: off
Amplifier: off
Projector: off
Lights: full brightness
```

### Example 2: Payment Processing Facade (closer to production Go)

This example uses interfaces for each subsystem dependency so the Facade is testable.

```go
package payment

import "fmt"

// ── Subsystem interfaces ────────────────────────────────────────

// FraudChecker checks whether a transaction looks fraudulent.
type FraudChecker interface {
    IsSuspicious(userID string, amount float64) bool
}

// PaymentGateway charges a payment instrument.
type PaymentGateway interface {
    Charge(userID string, amount float64) (txnID string, err error)
}

// LedgerService records a completed transaction.
type LedgerService interface {
    Record(userID, txnID string, amount float64) error
}

// Notifier sends a confirmation to the user.
type Notifier interface {
    Send(userID, message string)
}

// ── Facade ─────────────────────────────────────────────────────

// PaymentService is a facade that coordinates the fraud check,
// payment gateway, ledger, and notification in the correct order.
// Callers interact only with Charge(); they are decoupled from all
// subsystem details.
type PaymentService struct {
    fraud    FraudChecker
    gateway  PaymentGateway
    ledger   LedgerService
    notifier Notifier
}

func NewPaymentService(
    fraud FraudChecker,
    gateway PaymentGateway,
    ledger LedgerService,
    notifier Notifier,
) *PaymentService {
    return &PaymentService{fraud, gateway, ledger, notifier}
}

// Charge is the single entry point for making a payment.
// It runs: fraud check → gateway → ledger → notification.
func (p *PaymentService) Charge(userID string, amount float64) error {
    if p.fraud.IsSuspicious(userID, amount) {
        return fmt.Errorf("payment rejected: suspicious activity for user %s", userID)
    }

    txnID, err := p.gateway.Charge(userID, amount)
    if err != nil {
        return fmt.Errorf("payment gateway error: %w", err)
    }

    if err := p.ledger.Record(userID, txnID, amount); err != nil {
        // Gateway charged but ledger failed — this warrants an alert in production
        return fmt.Errorf("ledger error for txn %s: %w", txnID, err)
    }

    p.notifier.Send(userID, fmt.Sprintf("Payment of %.2f confirmed. TxnID: %s", amount, txnID))
    return nil
}
```

**Why this is the Facade pattern (not just a service):**
The caller is fully decoupled from FraudChecker, PaymentGateway, LedgerService, and Notifier. The caller does not need to know these subsystems exist, what order they run in, or how they interact. `PaymentService.Charge` is the facade method.

---

## Real-World Go Examples

The standard library uses Facade-like structures frequently:

- **`http.Client`** — coordinates DNS resolution, TCP dialling, TLS handshaking, HTTP request serialisation, response parsing. You call `client.Get(url)`; you do not interact with `net.Dialer`, `tls.Conn`, `bufio.Reader`, etc.
- **`os/exec.Cmd`** — coordinates process creation, stdin/stdout pipes, environment setup, signal handling. You call `cmd.Run()`; you do not assemble a `syscall.ProcAttr` manually.
- **`database/sql.DB`** — coordinates connection pooling, driver registration, query preparation, and result scanning. You call `db.QueryContext()`; you do not manage driver connections.

---

## Trade-offs and Problems Introduced

**Benefits:**
- Decouples callers from subsystem details — subsystems can change without callers noticing
- Reduces cognitive load for the most common operations
- Single place to update if subsystem coordination logic changes

**Problems introduced:**

- **God object risk** — if every feature of every subsystem is exposed through the Facade, it accumulates methods indefinitely and becomes a god object (knows too much, does too much). Mitigation: keep the Facade thin; expose only the high-frequency, high-level operations. Rare or advanced operations should be accessed via the subsystems directly.
- **Leaky abstraction** — if the Facade does not cover some subsystem operation that callers need, they must bypass it and call the subsystem directly. Now you have two access patterns in the codebase, which is harder to reason about.
- **Testability** — if subsystems are concrete structs (not interfaces), the Facade cannot be tested in isolation. Inject subsystem interfaces to allow mocking (as shown in Example 2).
- **Hidden complexity** — the Facade makes complex operations *look* simple, which can give callers a false sense of safety. For example, if `WatchMovie` can fail halfway through and the Facade does not handle partial failures, the room is now in an undefined state.

---

## How to Remember It

**Memory hook — the hotel concierge:**

You tell the concierge: "I need dinner at 7, a taxi at 9, and theater tickets for the 10pm show."

The concierge (Facade) calls the restaurant, the taxi company, and the box office (subsystems). You never interact with any of them. You *could* call them yourself, but you do not have to — and most guests don't.

Key properties encoded in this analogy:
- The concierge *coordinates* others; it doesn't do the work itself
- You can bypass the concierge and call the restaurant directly — nothing prevents it
- The concierge knows the order of operations and the dependencies (taxi must be booked after the reservation confirms)
- If the concierge accumulates too many responsibilities, it becomes overwhelmed — the God Object problem

---

## Interview Cheat Sheet

**In one sentence:** "Facade provides a simplified interface to a complex subsystem — it reduces the number of objects a client must interact with for a common operation."

**What interviewers actually probe:**

- "How is Facade different from Adapter?" — Adapter solves incompatibility; Facade solves complexity. The subsystem interfaces in Facade are not wrong — there are just too many of them to coordinate.
- "Does Facade hide the subsystem or encapsulate it?" — It hides, not encapsulates. Clients can still call subsystems directly. The Facade is a convenience layer, not a restriction.
- "What are the risks of using Facade?" — God object accumulation, leaky abstraction when coverage is incomplete, hidden failure modes.
- "Where have you seen Facade in production code?" — `http.Client`, `database/sql.DB`, AWS SDK's service clients, any SDK that wraps a complex API.

**Patterns to contrast (ready-to-use answers):**

- "I'd use Adapter if the subsystem interface is the right behaviour but the wrong shape for my client."
- "I'd use Facade if the subsystem interfaces are all correct but too numerous for the caller to coordinate."
- "I'd use Proxy if I need to add access control, caching, or logging around a single object."
- "I'd use Decorator if I want to add capabilities to an object at runtime without changing its type."

**Signal that shows depth:**
- Mentioning that Go's lack of inheritance makes Facade very natural — the struct-with-methods pattern maps directly to it, and interface injection keeps it testable.
- Pointing out that `http.Client` or `database/sql.DB` are real-world Facades without using the pattern name, then identifying them as such.

---

## References

- Gamma et al., *Design Patterns: Elements of Reusable Object-Oriented Software* (GoF), 1994 — Facade, pp. 185–193
- Refactoring Guru — [Facade Pattern](https://refactoring.guru/design-patterns/facade)
- Go standard library source: [`net/http/client.go`](https://cs.opensource.google/go/go/+/main:src/net/http/client.go), [`database/sql/sql.go`](https://cs.opensource.google/go/go/+/main:src/database/sql/sql.go)