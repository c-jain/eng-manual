# UML Diagrams

## Table of Contents

- [Overview](#overview)
- [Class Diagrams](#class-diagrams)
- [Sequence Diagrams](#sequence-diagrams)
- [Activity Diagrams](#activity-diagrams)
- [Choosing the Right Diagram](#choosing-the-right-diagram)
- [Interviewer Angles](#interviewer-angles)
- [References](#references)

---

## Overview

UML (Unified Modeling Language) is a standardised notation for visualising software structure and behaviour. There are 14 diagram types; for engineering interviews, three matter:

- **Class diagrams** — static structure (LLD design sessions)
- **Sequence diagrams** — dynamic interactions over time (HLD and LLD)
- **Activity diagrams** — workflows, business logic (LLD and process design)

You do not need to be a UML expert. You need to be fluent enough to sketch these on a whiteboard or in ASCII during an interview, and precise enough to use the right relationship type.

---

## Class Diagrams

### What It Is

A static structural diagram. Shows: what classes/interfaces exist, what attributes and methods they have, and how they relate to each other.

### Why It Exists

Object-oriented design needs a language-agnostic notation to communicate structure. UML class diagrams are the standard lingua franca — understood across Java, Go, Python, C++, and are part of design documents, architecture reviews, and interview whiteboards.

### Why It Is Called a Class Diagram

Because the primary element is the *class* — a named, encapsulated unit with state (attributes) and behaviour (methods). Even when modelling interfaces or abstract types, the "class" box is the visual primitive.

### Notation

A class is drawn as a box with three compartments:

```
  ┌─────────────────────┐
  │      ClassName      │  ← name (bold, centred)
  ├─────────────────────┤
  │  - privateField: T  │  ← attributes
  │  + PublicField: T   │     - = private
  ├─────────────────────┤     + = public
  │  + Method(): R      │  ← methods
  │  - helper(): void   │     # = protected
  └─────────────────────┘
```

For interfaces, the name compartment shows `«interface»` above the name.

### The Six Relationship Types

This is where most candidates get imprecise. Know all six cold.

```
  Relationship     Symbol           Meaning                  Lifetime of B
  ─────────────    ──────           ───────                  ─────────────
  Dependency       - - - →          A uses B temporarily     independent
  Association      ────             A holds reference to B   independent
  Aggregation      ◇────            A has-a B                B can exist alone
  Composition      ◆────            A owns B                 B dies with A
  Inheritance      ────▷            A is-a B (extends)       —
  Realization      - - -▷           A implements interface B  —
```

**Dependency** — weakest. B appears as a function parameter or local variable in A. A does not store B.

```
  OrderService - - - → PaymentGateway
```

**Association** — A holds a persistent reference to B (a field), but neither owns the other.

```
  User ──── Address
```

**Aggregation** — "has-a" with independent lifetime. Deleting A does not destroy B. Use a hollow diamond on the A side.

```
  Team ◇──── Player
  (a Player can exist without a Team; can join another Team)
```

**Composition** — "owns" with dependent lifetime. Deleting A destroys B. Use a filled diamond on the A side.

```
  House ◆──── Room
  (a Room cannot meaningfully exist without its House)
```

**The aggregation vs composition test:** If you delete the parent, does the child die too? Yes → composition. No → aggregation.

**Inheritance** — "is-a" relationship. B is the parent (superclass). Arrow points to parent; hollow triangle arrowhead.

```
  Dog ────▷ Animal
```

**Realization** — A implements interface B. Dashed line, hollow triangle arrowhead.

```
  Dog - - -▷ «interface» Swimmer
```

### Multiplicity

Written at either end of a relationship line:

```
  User 1────* Order       (one User, many Orders)
  Team 1────1..* Player   (one Team, one or more Players)
```

### Example: Order System (ASCII)

```
  «interface»
  Payable
  ┌────────────────┐          ┌─────────────────────┐
  │  + pay(): bool │◁- - - - -│       Order         │
  └────────────────┘          ├─────────────────────┤
                              │  - id: string       │
                              │  - status: Status   │
                              ├─────────────────────┤
                              │  + addItem(): void  │
                              │  + totalPrice(): int│
                              └──────────┬──────────┘
                                         ◆ 1
                                         │ (composition)
                                         │
                                         │ *
                              ┌──────────┴──────────┐
                              │      OrderItem      │
                              ├─────────────────────┤
                              │  - productID: string│
                              │  - quantity: int    │
                              ├─────────────────────┤
                              │  + subtotal(): int  │
                              └─────────────────────┘
```

### Go Mapping

Go has no classes, but the mapping is direct:

| UML Concept    | Go Equivalent                                                   |
|----------------|-----------------------------------------------------------------|
| Class          | `struct` with methods                                           |
| Interface      | `interface` type                                                |
| Inheritance    | Struct embedding (not true inheritance — know the difference)   |
| Composition    | Struct field of another struct type                             |
| Aggregation    | Struct field holding a pointer to another struct                |
| Realization    | Implicit interface satisfaction (no `implements` keyword)       |

```go
// Realization: PaymentService implicitly satisfies Payable
type Payable interface {
    Pay() bool
}

type PaymentService struct {
    gateway string
}

func (p *PaymentService) Pay() bool {
    // ...
    return true
}

// Composition: Order owns its Items — items are created with Order
type Order struct {
    id    string
    items []OrderItem   // composition — items die with Order
}

// Aggregation: Team references Players that exist independently
type Team struct {
    players []*Player   // aggregation — players exist outside Team
}
```

---

## Sequence Diagrams

### What It Is

A dynamic interaction diagram showing the sequence of messages exchanged between participants over time. Time flows downward; participants are arranged left to right.

### Why It Exists

Class diagrams describe structure — what exists. Sequence diagrams describe behaviour — what happens, and in what order. You need both to fully specify a system.

### Why It Is Called a Sequence Diagram

Because it explicitly models the *sequence* of interactions — the temporal ordering of messages is the primary information being communicated.

### Notation

```
  Client          AuthService      Database
    │                  │               │
    │──login(u,p)─────►│               │
    │                  │─findUser(u)──►│
    │                  │◄─── user ─────│
    │                  │               │
    │◄── JWT token ────│               │
    │                  │               │
```

**Participants** — named boxes at the top. Can be actors (stick figure) or system components (rectangle).

**Lifelines** — dashed vertical lines below each participant. Represent existence over time.

**Activation bars** — thin rectangles on lifelines. Show when a participant is actively processing.

**Message types:**

```
  A ──────────────► B    Synchronous call (blocks until response)
  A ◄ ─ ─ ─ ─ ─ ─ ─ B    Return / response
  A ──────────────> B    Async message (fire and forget)
  A ──────────────► A    Self-call (recursive or internal)
```

### Combined Fragments

These model conditional and looping logic inside sequence diagrams:

```
  ┌─ alt ─────────────────────────────┐
  │  [user authenticated]             │
  │    Client ──► Server: getData()   │
  ├──────────────────────────────────-┤
  │  [else]                           │
  │    Client ◄── Server: 401 Error   │
  └───────────────────────────────────┘
```

| Fragment | Meaning                              | Analogy   |
|----------|--------------------------------------|-----------|
| `alt`    | Mutually exclusive branches          | if/else   |
| `opt`    | Single optional branch               | if        |
| `loop`   | Repeated execution                   | for/while |
| `par`    | Parallel execution                   | goroutines|
| `ref`    | Reference to another sequence diagram| function call |

### Example: User Login Flow

```
Browser        APIGateway        AuthService        Database
   │                │                 │                 │
   │──POST /login──►│                 │                 │
   │                │──validateReq()─►│                 │
   │                │                 │──findUser()────►│
   │                │                 │◄──user record───│
   │                │                 │                 │
   │                │        ┌──────── alt ─────────────┐
   │                │        │ [valid credentials]      │
   │                │◄───────┤ generateJWT()            │
   │◄──200 OK + JWT─│        │                          │
   │                │        ├──────────────────────────┤
   │                │        │ [invalid credentials]    │
   │◄──401 Error────│◄───────┤ return 401               │
   │                │        └──────────────────────────┘
   │                │                 │                 │
```

### When to Sketch This in an Interview

Any time you are describing "what happens when a user does X." Interviewers want to see:
- Who calls whom (coupling)
- What is synchronous vs asynchronous (and why)
- Where retries and timeouts live
- Where failures propagate or are absorbed

---

## Activity Diagrams

### What It Is

A behavioural diagram modelling the flow of control through a process — a workflow, business process, or algorithm. Visually similar to flowcharts, but with added constructs for parallel behaviour and multi-actor workflows.

### Why It Exists

Sequence diagrams show *interaction* between components. Activity diagrams show the *logic flow* within a single process or across multiple roles. They answer "how does this process work?" rather than "who talks to whom?"

### Why It Is Called an Activity Diagram

Because the primary element is an *activity* (an action or step), and the diagram shows how control flows between activities.

### Notation

```
  ●                  Start node (filled circle)
  ⊙                  End node (circle with filled centre)
  ╭──────────────╮   Action / activity step (rounded rectangle)
  │  Action Name │
  ╰──────────────╯
        ◇            Decision node — one input, multiple guarded outputs

      │
   ═══════   ← fork — one input splits into parallel flows (thick bar)
    /   \
   A     B 
     
   A     B
    \   /
   ═══════   ← join — parallel flows merge back into one (thick bar)
      │
```

### Swimlanes

Vertical (or horizontal) columns that assign each action to an actor or system. Useful for multi-role workflows where responsibility boundaries matter.

### Example: Order Checkout Flow

```
                    ●
                    │
          ┌─────────┴──────────┐
          │     Add Items      │  (Customer)
          └─────────┬──────────┘
                    │
          ┌─────────┴──────────┐
          │      Checkout      │  (Customer)
          └─────────┬──────────┘
                    │
          ┌─────────┴──────────┐
          │   Validate Cart    │  (OrderService)
          └─────────┬──────────┘
                    │
                    ◇─── [invalid] ───► ┌──────────────┐
                    │                   │ Return Error │──► ⊙
                 [valid]                └──────────────┘
                    │
        ════════════╪════════════════════════  fork (parallel start)
                    │                    │
          ┌─────────┴──────────┐  ┌──────┴───────────────┐
          │ Reserve Inventory  │  │   Charge Payment     │  (PaymentService)
          └─────────┬──────────┘  └──────┬───────────────┘
                    │                    │
        ════════════╪════════════════════════  join (wait for both)
                    │
          ┌─────────┴──────────────────────┐
          │   Send Confirmation Email      │  (OrderService)
          └─────────┬──────────────────────┘
                    │
                    ⊙
```

### Problems It Exposes

Activity diagrams surface:
- Missing error paths (what happens if payment fails after inventory is reserved?)
- Incorrect sequencing (charging before reserving inventory is a bug)
- Unnecessary sequential steps that could be parallelised

---

## Choosing the Right Diagram

| Question You Are Answering                   | Use              |
|----------------------------------------------|------------------|
| What classes exist and how do they relate?   | Class diagram    |
| What does this class's interface look like?  | Class diagram    |
| What happens when a user does X?             | Sequence diagram |
| How do these two services communicate?       | Sequence diagram |
| How does this business process work?         | Activity diagram |
| What's the flow for this algorithm/workflow? | Activity diagram |

In an LLD interview, you typically start with a class diagram (structure), then draw a sequence diagram for one or two key use cases (behaviour). Activity diagrams come up less often — use them when the interviewer asks about a complex flow with branching or parallelism.

---

## Interviewer Angles

- **"Draw the class diagram for a parking lot system."** — Start by identifying the nouns (entities): ParkingLot, Floor, Slot, Vehicle, Ticket. Then map relationships precisely: ParkingLot *composes* Floors, Floor *composes* Slots (they die with their parent), Slot *associates* with a Vehicle (the vehicle exists independently). Then add interfaces: a Payable interface for different payment methods.

- **"Walk me through what happens when a user sends a message in a chat system."** — Sketch a sequence diagram. Show: client → API gateway → message service → message queue (async) → notification service → recipient client. Highlight what's synchronous (API response to sender) vs async (push to recipient).

- **"What's the difference between aggregation and composition?"** — Lifetime. In composition, the child cannot exist without the parent and is destroyed when the parent is. In aggregation, the child has an independent lifecycle — it can be reassigned or continue to exist after the parent is gone.

- **"Why use a sequence diagram over just describing it in prose?"** — Sequence diagrams force you to be explicit about: the participants involved, the order of operations, what is sync vs async, and where responsibility lies. Prose allows ambiguity; diagrams do not.

- **"How does Go handle the 'implements' relationship in UML?"** — Go uses implicit interface satisfaction. There is no `implements` keyword. If a struct has all the methods an interface declares, it satisfies that interface automatically. This means the "realization" relationship in a Go class diagram is determined structurally, not by declaration — which encourages small, consumer-defined interfaces (the io.Reader pattern).

---

## References

- [UML Class Diagram — Visual Paradigm Guide](https://www.visual-paradigm.com/guide/uml-unified-modeling-language/uml-class-diagram-tutorial/)
- [UML Sequence Diagram — Visual Paradigm Guide](https://www.visual-paradigm.com/guide/uml-unified-modeling-language/what-is-sequence-diagram/)
- [UML Activity Diagram — Visual Paradigm Guide](https://www.visual-paradigm.com/guide/uml-unified-modeling-language/what-is-activity-diagram/)
- [UML Distilled — Martin Fowler](https://martinfowler.com/books/uml.html)
- [PlantUML — text-based UML tool](https://plantuml.com/)