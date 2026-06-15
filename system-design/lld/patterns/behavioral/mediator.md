---
Status: 🌳 Evergreen
Created: 2026-06-15
Last Updated: 2026-06-15
---

# Mediator Pattern

## Table of Contents

- [What It Is](#what-it-is)
- [Why It Exists](#why-it-exists)
- [Why It's Called "Mediator"](#why-its-called-mediator)
- [Structure](#structure)
- [Go Implementation](#go-implementation)
- [Trade-offs and Problems It Brings](#trade-offs-and-problems-it-brings)
- [Mediator vs Observer vs Facade](#mediator-vs-observer-vs-facade)
- [Mediator at the Architecture Level: Orchestration vs Choreography](#mediator-at-the-architecture-level-orchestration-vs-choreography)
- [How to Remember This](#how-to-remember-this)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

## What It Is

Mediator is a behavioural GoF pattern that defines an object — the mediator — that encapsulates how a set of other objects (colleagues) interact with each other. Colleagues never hold direct references to one another; they only know the mediator, through an interface, and all interaction is routed through it.

It's a behavioural pattern because it governs communication between objects, not their construction (creational concern) or their structural composition (structural concern).

## Why It Exists

Consider a set of objects that need to coordinate: chat participants, GUI widgets in a dialog box, aircraft sharing a runway. Without a mediator, the natural approach is for each object to hold direct references to every other object it needs to talk to.

For N colleagues, that's up to N(N-1) direct relationships. Every new colleague potentially requires changes to every existing colleague it interacts with, and similar coordination logic ends up duplicated across all of them.

The Mediator pattern centralises that interaction logic in one place. Each colleague holds a single reference — to the mediator. The mediator holds references to (or a registry of) the colleagues and decides how messages and events flow between them. Adding a new colleague means registering it with the mediator; existing colleagues are untouched.

### Scenario 1: Without a Mediator (Direct References)

```
        Colleague A
         ^       ^
         |       |
         v       v
Colleague B <--> Colleague C
```

Three colleagues, three direct relationships. This already isn't trivial to keep straight, and it gets worse fast: with four colleagues that all need to talk to each other, that's six direct relationships; with five, ten. Every colleague's code has to know about, and handle messages from, every other colleague it relates to.

### Scenario 2: With a Mediator (Hub and Spoke)

```
+-------------+   +-------------+   +-------------+
| Colleague A |   | Colleague B |   | Colleague C |
+-------------+   +-------------+   +-------------+
       |                 |                 |
       +--------+--------+--------+--------+
                |
                v
       +-----------------------+
       |        Mediator         |
       |   (Concrete Mediator)    |
       +-----------------------+
```

Now there are exactly three relationships — each colleague to the mediator. The mediator holds the coordination logic; colleagues hold none of it. Adding Colleague D means connecting it to the mediator and updating the mediator's logic if needed — Colleagues A, B, and C are untouched.

## Why It's Called "Mediator"

A mediator, in everyday language — legal, diplomatic, or interpersonal — is a neutral third party who facilitates communication between parties who don't, shouldn't, or can't deal with each other directly. The GoF pattern borrows this meaning exactly: the Mediator object sits between the Colleagues and manages their interaction, so the colleagues don't need to know about each other at all.

The canonical analogy used in the GoF book, and in most write-ups, is an air traffic control tower: pilots don't communicate plane-to-plane about runway access — they all talk to the tower, which coordinates takeoffs, landings, and routing.

## Structure

In GoF terms:

- **Mediator** (interface) — declares the communication methods Colleagues use to talk to it.
- **ConcreteMediator** — implements the coordination logic and holds references to the Colleagues it manages.
- **Colleague** — each Colleague holds a reference to the Mediator interface, not to other Colleagues, and calls into it to communicate.

In Go, which has no inheritance, this maps cleanly onto interfaces and structs: the Mediator is an interface, the ConcreteMediator is a struct that implements it, and each Colleague is a struct holding a field of the Mediator interface type.

## Go Implementation

### Example 1: Chat Room (Broadcast Coordination)

The classic illustrative example — a chat room mediates messages between users so that no `User` needs a reference to any other `User`.

```go
package mediator

import "fmt"

// ChatRoom is the Mediator interface. Colleagues (Users) only
// ever interact through this interface.
type ChatRoom interface {
    Register(u *User)
    SendMessage(from *User, message string)
}

// chatRoom is the concrete mediator. It owns the registry of
// users and decides how a message is routed.
type chatRoom struct {
    users []*User
}

func NewChatRoom() ChatRoom {
    return &chatRoom{}
}

func (c *chatRoom) Register(u *User) {
    c.users = append(c.users, u)
}

// SendMessage is the coordination logic: broadcast to everyone
// except the sender, in registration order.
func (c *chatRoom) SendMessage(from *User, message string) {
    for _, u := range c.users {
        if u.Name == from.Name {
            continue
        }
        u.Receive(from.Name, message)
    }
}

// User is a Colleague. It knows only the ChatRoom interface —
// never another User directly.
type User struct {
    Name string
    room ChatRoom
}

func NewUser(name string, room ChatRoom) *User {
    u := &User{Name: name, room: room}
    room.Register(u)
    return u
}

func (u *User) Send(message string) {
    u.room.SendMessage(u, message)
}

func (u *User) Receive(from, message string) {
    fmt.Printf("[%s] %s: %s\n", u.Name, from, message)
}
```

```go
// main.go

package main

import "mediator"

func main() {
    room := mediator.NewChatRoom()

    alice := mediator.NewUser("Alice", room)
    bob := mediator.NewUser("Bob", room)
    carol := mediator.NewUser("Carol", room)

    alice.Send("Hey everyone!")
    bob.Send("Hi Alice!")
    carol.Send("Hi all!")
}
```

**Output:**
```
[Bob] Alice: Hey everyone!
[Carol] Alice: Hey everyone!
[Alice] Bob: Hi Alice!
[Carol] Bob: Hi Alice!
[Alice] Carol: Hi all!
[Bob] Carol: Hi all!
```

`Alice`, `Bob`, and `Carol` never reference each other. All three hold a `ChatRoom` and call `Send` — the mediator decides who receives what.

### Example 2: Air Traffic Control (Stateful Coordination)

The chat room example only broadcasts — it doesn't make decisions. A more realistic mediator often holds shared state and actively arbitrates access to it. This is the air traffic control analogy made concrete: multiple aircraft (colleagues) want to use a single runway (a shared resource), and the tower (mediator) decides who gets it and when.

```go
package atc

import (
    "fmt"
    "sync"
)

// Tower is the Mediator interface.
type Tower interface {
    RequestRunway(a *Aircraft) bool
    ReleaseRunway(a *Aircraft)
}

// controlTower is the concrete mediator. It owns the shared
// resource (the runway) and arbitrates access to it.
type controlTower struct {
    mu       sync.Mutex
    occupant string
}

func NewControlTower() Tower {
    return &controlTower{}
}

// RequestRunway grants the runway if it's free, otherwise denies it.
// This is the coordination logic — colleagues don't decide this
// themselves.
func (t *controlTower) RequestRunway(a *Aircraft) bool {
    t.mu.Lock()
    defer t.mu.Unlock()

    if t.occupant != "" {
        fmt.Printf("Tower: %s, hold — runway occupied by %s\n", a.CallSign, t.occupant)
        return false
    }

    t.occupant = a.CallSign
    fmt.Printf("Tower: %s, cleared for runway\n", a.CallSign)
    return true
}

func (t *controlTower) ReleaseRunway(a *Aircraft) {
    t.mu.Lock()
    defer t.mu.Unlock()

    if t.occupant == a.CallSign {
        fmt.Printf("Tower: %s has cleared the runway\n", a.CallSign)
        t.occupant = ""
    }
}

// Aircraft is a Colleague. It never talks to other aircraft —
// only to the Tower.
type Aircraft struct {
    CallSign string
    tower    Tower
}

func NewAircraft(callSign string, tower Tower) *Aircraft {
    return &Aircraft{CallSign: callSign, tower: tower}
}
```

```go
// main.go

package main

import "atc"

func main() {
    tower := atc.NewControlTower()

    ua100 := atc.NewAircraft("UA100", tower)
    dl200 := atc.NewAircraft("DL200", tower)

    tower.RequestRunway(ua100) // granted
    tower.RequestRunway(dl200) // denied — UA100 still on the runway

    tower.ReleaseRunway(ua100)
    tower.RequestRunway(dl200) // granted — runway now free
}
```

**Output:**
```
Tower: UA100, cleared for runway
Tower: DL200, hold — runway occupied by UA100
Tower: UA100 has cleared the runway
Tower: DL200, cleared for runway
```

Neither `Aircraft` knows the other exists. The `controlTower` is the only place that knows about runway state and decides who can use it — that's the mediator doing real coordination, not just message-passing.

## Trade-offs and Problems It Brings

**Benefits:**

- Reduces up to N(N-1) potential direct relationships to N relationships — each colleague to the mediator.
- New colleagues are added by registering with the mediator; existing colleagues are untouched.
- Interaction and coordination logic lives in one place instead of being duplicated across colleagues.

**Problems introduced:**

- **God object risk** — as more colleague types and interaction rules accumulate, the mediator absorbs more and more logic. It can grow into a large, hard-to-test, hard-to-change class that violates the Single Responsibility Principle. Mitigation: split into multiple smaller mediators by domain or concern, and keep the mediator focused on *coordination*, delegating actual business logic back to colleagues or separate services.
- **Single point of contention** — if the mediator owns shared mutable state, like the runway above, and needs locking, it can become a concurrency bottleneck under high load. The lock scope and contention pattern need the same care as any shared-resource design.
- **Indirection obscures flow** — tracing "what happens when Colleague A does X" now requires understanding the mediator's internal logic too. For very simple, two-object relationships, this indirection is pure overhead.
- **Over-engineering for small N** — with only two colleagues and a simple relationship, a Mediator is usually unnecessary ceremony. The pattern earns its keep once there are three or more colleagues with genuinely many-to-many or non-trivial coordination needs.

## Mediator vs Observer vs Facade

These three are frequently confused because all three sit "between" objects in some sense.

**Mediator vs Observer:**

- **Observer** is one-to-many notification: a single subject notifies many independent observers when its state changes. Observers generally don't talk back through the subject in a coordinated way, and the subject doesn't make decisions about how observers interact with each other.
- **Mediator** is many-to-many coordination among roughly equal-status peers — "colleagues." The mediator actively makes decisions about how an interaction should proceed; it holds coordination logic, not just a notification list.
- Rule of thumb: **Observer notifies. Mediator decides.**
- They often combine: a mediator's internal communication with colleagues can be implemented using Observer, where colleagues subscribe to the mediator's events. But conceptually they solve different problems. Observer answers "who needs to know when something changes?"; Mediator answers "how should these objects' interactions be coordinated?"

**Mediator vs Facade:**

- **Facade** is one-directional: a client calls the facade, the facade calls into a subsystem, and the subsystem objects don't know the facade exists and never call back into it.
- **Mediator** is bidirectional and many-to-many: colleagues actively call into the mediator, and the mediator calls back into colleagues. Colleagues are aware of, and hold a reference to, the mediator.
- Rule of thumb: **Facade simplifies a one-way call into a subsystem. Mediator coordinates two-way conversations between peers who all know about it.**

## Mediator at the Architecture Level: Orchestration vs Choreography

The same idea reappears one level up, at the service/architecture level — and drawing this connection is a strong way to demonstrate depth in an interview.

- At the object level (LLD), Mediator centralises communication between colleague objects.
- At the service level (HLD), the equivalent is **orchestration**: a central orchestrator service calls and coordinates other services in a defined sequence — for example, a Saga orchestrator coordinating Order, Payment, and Inventory services for a single business transaction. Architecturally, the orchestrator *is* a Mediator.
- The opposite architectural style, **choreography**, is closer to Observer/pub-sub: each service reacts to events from other services without any central coordinator.

The trade-offs mirror the LLD-level ones almost exactly: orchestration gives one place to see and control the whole workflow, at the risk of the orchestrator becoming a "god service"; choreography is loosely coupled, but the overall flow is harder to trace because no single component has the full picture.

## How to Remember This

- **Air traffic control tower** — pilots don't talk to each other; they talk to the tower, and the tower decides who lands or takes off when. Tower = Mediator, planes = Colleagues.
- **"Mediator coordinates; Observer notifies; Facade simplifies."** — a one-line way to keep all three apart.
- **Hub-and-spoke vs. mesh** — Mediator turns a point-to-point mesh, where every colleague is potentially wired to every other colleague, into a hub-and-spoke, where every colleague is wired only to the mediator.
- **The smell that calls for it** — if you find yourself saying "...and then X also needs to tell Y, and Z, and W..." while describing object interactions, that's the signal a Mediator might help.

## Interview Cheat Sheet

### Signal Phrases

- "Instead of every colleague knowing about every other colleague, they only know the Mediator interface — that's what decouples them."
- "This is essentially the orchestration pattern, applied at the object level."
- "The mediator centralises coordination logic, turning a many-to-many mesh into a hub-and-spoke."

### Red Flags to Avoid

| Mistake | Correct Framing |
|---|---|
| Proposing direct references between many objects | Recognise the O(N²) coupling and maintenance cost as the system grows |
| Dumping all coordination *and* business logic into one mediator | Keep the mediator focused on coordination; push business logic to colleagues or services |
| Confusing Mediator with Observer when asked to differentiate | Observer notifies; Mediator decides and coordinates |
| Confusing Mediator with Facade when asked to differentiate | Facade is one-directional; Mediator is bidirectional and many-to-many |

### Common Interviewer Probes

| Probe | What They're Looking For |
|---|---|
| "What happens as the mediator grows?" | God object risk; split into multiple smaller mediators by concern, keep business logic out of the mediator |
| "How is this different from Observer?" | Observer = one-to-many notification without coordination logic; Mediator = many-to-many coordination where the mediator decides outcomes |
| "How is this different from Facade?" | Facade = one-directional simplification of a subsystem; Mediator = bidirectional coordination between peers aware of the mediator |
| "Where have you seen this at the architecture level?" | Saga orchestrators and workflow engines (e.g., AWS Step Functions, Temporal) coordinating microservices — an architectural Mediator, contrasted with event-driven choreography |

## References

- Gamma, Helm, Johnson, Vlissides — *Design Patterns: Elements of Reusable Object-Oriented Software* (GoF), Mediator pattern
- Refactoring Guru — [Mediator](https://refactoring.guru/design-patterns/mediator)
- This repo: `observer.md` (Observer vs Pub/Sub), `facade.md` (Facade vs Adapter/Proxy/Decorator), `message-queues.md` / `microservices.md` (orchestration vs choreography context)