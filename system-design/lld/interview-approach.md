# How To Approach An LLD Interview

## Table of Contents

- [What Is LLD, And Why Does It Exist As An Interview](#what-is-lld-and-why-does-it-exist-as-an-interview)
- [What Interviewers Actually Evaluate](#what-interviewers-actually-evaluate)
- [The Five-Step Framework](#the-five-step-framework)
  - [Step 1: Clarify Requirements (5 min)](#step-1-clarify-requirements-5-min)
  - [Step 2: Identify Entities and Actors (5 min)](#step-2-identify-entities-and-actors-5-min)
  - [Step 3: Define Interfaces and Contracts (5 min)](#step-3-define-interfaces-and-contracts-5-min)
  - [Step 4: Design the Class Structure (10 min)](#step-4-design-the-class-structure-10-min)
  - [Step 5: Code and Iterate (15 min)](#step-5-code-and-iterate-15-min)
- [The Go-Specific LLD Mindset](#the-go-specific-lld-mindset)
- [Relationships Between Entities](#relationships-between-entities)
- [Recognising and Applying Design Patterns](#recognising-and-applying-design-patterns)
- [Common LLD Problems and Entity Maps](#common-lld-problems-and-entity-maps)
- [Worked Mini-Example: Parking Lot](#worked-mini-example-parking-lot)
- [What To Say When You're Stuck](#what-to-say-when-youre-stuck)
- [Anti-Patterns That Kill Interviews](#anti-patterns-that-kill-interviews)
- [Interview Cheat Sheet](#interview-cheat-sheet)

---

## What Is LLD, And Why Does It Exist As An Interview

**LLD (Low-Level Design)** is the act of designing the internal structure of a system — the classes (or structs), interfaces, their relationships, and the code that implements the core behaviour. It sits below HLD (which handles distribution, services, and infrastructure) and above pure coding challenges (which test algorithmic thinking).

**Why it exists as an interview format:** Senior engineers spend the majority of their time making structural decisions — how to represent domain concepts in code, how to make components extensible without modifying them, where to draw abstraction boundaries. LLD interviews test whether you think like a senior engineer even before you're one.

**Why it's called "low-level":** Relative to system design (HLD), which talks about services, databases, and network topology, LLD zooms in to the code structure within a single service or module. "Low-level" here means granularity, not simplicity.

---

## What Interviewers Actually Evaluate

Understanding this changes how you approach the interview.

**Structured thinking:** Can you break an ambiguous problem into a clean, manageable set of classes and responsibilities? They're watching whether you have a process, not just an answer.

**Abstraction and encapsulation:** Do you separate what something does (interface) from how it does it (implementation)? Do you hide state behind methods rather than exposing raw fields?

**Extensibility without rewrites:** Can new requirements be added by writing new code, not by changing existing code? This is OCP from SOLID — the single most tested principle in LLD.

**Correct use of relationships:** Composition over inheritance. Association vs. aggregation vs. composition. Interviewers notice when you reach for inheritance where composition would be cleaner.

**Design pattern awareness:** You don't need to name-drop patterns unnecessarily. But when a creational, structural, or behavioural pattern naturally fits, applying it correctly signals experience.

**Coding quality in the given language:** In Go — small interfaces, constructor injection, no exported struct fields where avoidable, methods on pointer receivers, table-driven tests.

**Communication:** Thinking out loud, asking the right clarifying questions, defending your choices when challenged, and knowing which decisions to defer.

---

## The Five-Step Framework

Use this as a mental checklist, not a rigid script. Adapt timing based on problem complexity.

```
  ┌──────────────────────────────────────────────────────────────┐
  │  Total time: ~40 minutes                                     │
  │                                                              │
  │  [5 min]  Step 1: Clarify Requirements                       │
  │  [5 min]  Step 2: Identify Entities & Actors                 │
  │  [5 min]  Step 3: Define Interfaces & Contracts              │
  │  [10 min] Step 4: Design Class Structure                     │
  │  [15 min] Step 5: Code Key Components                        │
  └──────────────────────────────────────────────────────────────┘
```

### Step 1: Clarify Requirements (5 min)

Never start designing without asking questions. The goal here is to bound the problem and discover implicit constraints.

**Functional requirements — what the system must do:**
- What are the core use cases? ("A user can park a vehicle, pay, and exit.")
- Who are the actors? (User, Admin, System, ExternalService)
- What data needs to be stored? What needs to be retrieved?
- Are there different types of the same entity? (VehicleType: car, bike, truck)

**Non-functional constraints — what the design must accommodate:**
- Is this a single-instance system or distributed? (Affects whether you need thread safety)
- Do we care about persistence, or is in-memory fine for this exercise?
- Any performance constraints? (Real-time vs. batch)
- How many users/transactions? (Often not relevant for LLD, but can surface edge cases)

**Scope control:**
- "Should I handle payment processing in detail, or assume a `PaymentService` interface?"
- "Should I model the database layer, or assume a repository abstraction?"

Interviewers want to see you ask these questions. It demonstrates that you understand requirements before jumping to solutions.

### Step 2: Identify Entities and Actors (5 min)

Read the requirements and extract the **nouns**. Each significant noun is a candidate entity (struct in Go). Don't over-engineer at this stage — just list them and think about what they represent.

```
  Example: Parking Lot system

  Nouns: ParkingLot, Floor, Slot, Vehicle, Ticket, Payment, 
         EntryGate, ExitGate, FeeCalculator
  Actors: Driver (external), Admin (external), System (internal)
```

For each entity, ask:
- What data does it hold? (fields)
- What does it know how to do? (methods)
- What does it depend on? (relationships)
- Is it a role (interface) or a concrete thing (struct)?

**Tip:** Actors are not entities in your class design. They interact with your system through use cases. An `Admin` actor doesn't become an `Admin` struct — they drive operations via an `AdminService` or a command handler.

### Step 3: Define Interfaces and Contracts (5 min)

Before designing concrete types, define the interfaces. This is the most important step for demonstrating Go idiom and SOLID principles.

**What is an interface here?** The contract that a component must satisfy to be used by another. Defined by the *consumer*, not the provider — a key Go idiom.

```go
// Defined where it's consumed, not where it's implemented.
// Small, focused on one concern.

type FeeCalculator interface {
    Calculate(ticket Ticket) (Money, error)
}

type SlotAllocator interface {
    Allocate(v Vehicle) (Slot, error)
    Release(slot Slot) error
}

type PaymentProcessor interface {
    Process(amount Money, method PaymentMethod) error
}
```

**Questions to ask while defining interfaces:**
- What does component A need from component B? That need becomes an interface.
- Can I define it with a single method? (Go prefers small interfaces.)
- Is this testable? (An interface seam means you can mock it in tests.)

### Step 4: Design the Class Structure (10 min)

Now fill in the structs, their fields, and relationships. Draw a class diagram on a whiteboard or verbally walk through it.

**Template for each struct:**
- Name (PascalCase noun)
- Fields (with types, unexported where possible)
- Methods (the behaviours the struct owns)
- Relationships (what it holds, what it receives via constructor)

**Class diagram notation (ASCII):**

```
  ┌─────────────────────────────┐
  │ ParkingLot                  │
  ├─────────────────────────────┤
  │ - floors []*Floor           │
  │ - allocator SlotAllocator   │  ← interface, injected
  │ - calculator FeeCalculator  │  ← interface, injected
  ├─────────────────────────────┤
  │ + Park(v Vehicle) (Ticket)  │
  │ + Exit(t Ticket) (Money)    │
  └──────────────┬──────────────┘
                 │ contains (composition)
         ┌───────▼────────┐
         │ Floor          │
         ├────────────────┤
         │ - level int    │
         │ - slots []*Slot│
         └───────┬────────┘
                 │ contains (composition)
         ┌───────▼────────┐
         │ Slot           │
         ├────────────────┤
         │ - id string    │
         │ - type SlotType│
         │ - occupied bool│
         └────────────────┘
```

**Relationship guide:**

```
  A ──────────▶ B   Association: A has a reference to B
  A ◆──────────▶ B  Composition: A owns B; B cannot exist without A
  A ◇──────────▶ B  Aggregation: A contains B; B can exist independently
  A ─ ─ ─ ─ ─▶ B   Dependency: A uses B (e.g., passed as a parameter)
  A ───────────▷ B  Implementation: A implements interface B
```

### Step 5: Code and Iterate (15 min)

Don't try to code everything. Pick the **two or three most interesting or core components** and implement them completely. Leave the rest as stubs or explain them verbally.

**Priority order for what to code:**
1. Core domain struct and its primary method (the most important use case)
2. The interface your design hinges on (e.g., `FeeCalculator`)
3. One concrete implementation of that interface
4. Constructor functions for each struct (never ask the caller to initialise fields directly)

**Template for a Go struct:**

```go
// Always unexported fields.
// Constructor function (not a bare struct literal) is the only way in.
// Interface injected, not concrete type.

type ParkingLot struct {
    floors    []*Floor
    allocator SlotAllocator
    calc      FeeCalculator
}

func NewParkingLot(floors []*Floor, a SlotAllocator, c FeeCalculator) *ParkingLot {
    return &ParkingLot{
        floors:    floors,
        allocator: a,
        calc:      c,
    }
}

func (p *ParkingLot) Park(v Vehicle) (Ticket, error) {
    slot, err := p.allocator.Allocate(v)
    if err != nil {
        return Ticket{}, fmt.Errorf("park: %w", err)
    }
    return Ticket{
        SlotID:    slot.ID,
        VehicleID: v.ID,
        EntryTime: time.Now(),
    }, nil
}

func (p *ParkingLot) Exit(t Ticket) (Money, error) {
    fee, err := p.calc.Calculate(t)
    if err != nil {
        return Money{}, fmt.Errorf("exit: %w", err)
    }
    if err := p.allocator.Release(t.SlotID); err != nil {
        return Money{}, fmt.Errorf("exit: release: %w", err)
    }
    return fee, nil
}
```

After coding core components, discuss extensions:
- "With more time, I'd implement `HourlyFeeCalculator` and `DailyFeeCalculator` as two implementations of `FeeCalculator`."
- "I'd add a `ParkingLotRepository` interface to abstract persistence."
- "For concurrency, I'd add a mutex around `Allocate` and `Release` in the allocator."

---

## The Go-Specific LLD Mindset

LLD interviews are implicitly testing your fluency in the language you use. In Go, this means:

**Interfaces are consumer-defined.** Don't define a giant `IVehicle` interface on the `vehicle` package and implement it everywhere. Define `type Measurable interface { Size() int }` in the package that needs sizing — and only put what that package actually uses.

**Prefer small interfaces.** `io.Reader` is one method. `io.Writer` is one method. Your `FeeCalculator` should be one method. The moment an interface has five methods, ask if it should be split.

**No inheritance — use struct embedding and composition.** If `ElectricVehicle` needs everything `Vehicle` has plus battery info, embed `Vehicle` in `ElectricVehicle`. Don't reach for a base class.

```go
type Vehicle struct {
    ID       string
    Type     VehicleType
    LicensePlate string
}

type ElectricVehicle struct {
    Vehicle                    // embedded — promotes all Vehicle fields and methods
    BatteryCapacityKWh float64
}
```

**Constructor injection, always.** Dependencies are passed into `NewXxx()` functions. Never have a struct call `NewFeeCalculator()` internally — that creates a hidden dependency.

**Error wrapping, not panics.** Use `fmt.Errorf("operation: %w", err)` for wrappable errors. Use sentinel errors (`var ErrSlotFull = errors.New("slot full")`) for things callers need to check.

**Exported types, unexported fields.** `ParkingLot` is exported (others need to use it). `floors`, `allocator` are unexported (only `ParkingLot`'s methods should touch them).

---

## Relationships Between Entities

Getting these right is a signal of design maturity.

**Composition** ("has-a, and owns it")

```go
type ParkingLot struct {
    floors []*Floor   // ParkingLot owns Floor; Floor doesn't exist without ParkingLot
}
```

**Aggregation** ("has-a, but doesn't own it")

```go
type Ticket struct {
    vehicle *Vehicle  // Ticket references Vehicle; Vehicle exists independently
}
```

**Association** ("uses a")

```go
func (p *ParkingLot) Park(v Vehicle) (Ticket, error) {
    // Uses Vehicle but doesn't store it
}
```

**Dependency via interface** ("needs something that does X")

```go
type ParkingLot struct {
    calc FeeCalculator  // depends on the FeeCalculator contract, not a concrete type
}
```

---

## Recognising and Applying Design Patterns

You don't need to namedrop patterns. You need to reach for them instinctively when they fit.

**Creational Patterns**

Strategy before reaching: when you have multiple algorithms for the same operation.

| Pattern | When to Reach For It |
|---------|----------------------|
| Factory Method / Factory Function | When the caller shouldn't know which concrete type they're getting. `NewFeeCalculator(strategy)` returns the right impl. |
| Builder | When constructing a complex object requires many optional steps. `TicketBuilder.WithDiscount(...).WithEntry(...).Build()` |
| Singleton | When exactly one instance must exist (e.g., `ParkingLotConfig`). Use sparingly — it creates global state. |

**Structural Patterns**

| Pattern | When to Reach For It |
|---------|----------------------|
| Adapter | When you need to make an existing interface fit a new contract. Wrapping a third-party payment SDK behind your `PaymentProcessor` interface. |
| Decorator | When you want to add behaviour to an existing interface without modifying it. A `LoggingFeeCalculator` wraps any `FeeCalculator` and logs before/after. |
| Facade | When a subsystem is complex and you want to expose a simple surface. `ParkingLot.Park()` hides allocation, ticketing, gate control. |

**Behavioural Patterns**

| Pattern | When to Reach For It |
|---------|----------------------|
| Strategy | When you have interchangeable algorithms. `FeeCalculator` with `HourlyStrategy`, `DailyStrategy`, `FlatRateStrategy`. |
| Observer | When state changes in one object should notify others. `SlotOccupied` event → `DisplayBoard` updates. |
| Command | When you want to encapsulate a request as an object (queue, undo). `ParkCommand`, `ExitCommand`. |
| State | When an object's behaviour changes based on its internal state. `Ticket` with states: `Active`, `Paid`, `Expired`. |
| Template Method | When an algorithm's steps are fixed but some steps vary. Base `FeeCalculator.Calculate()` calls `rateForHour()` which subclasses override — but in Go, use the Strategy pattern instead via interfaces. |

---

## Common LLD Problems and Entity Maps

Quick mental model for each classic problem. Use this as a starting scaffold.

**Parking Lot**
Core entities: `ParkingLot`, `Floor`, `Slot`, `Vehicle`, `Ticket`, `Payment`
Key interfaces: `SlotAllocator`, `FeeCalculator`, `PaymentProcessor`
Key pattern: Strategy (fee calculation), Facade (ParkingLot surface)

**ATM**
Core entities: `ATM`, `Card`, `Account`, `Transaction`, `CashDispenser`, `Receipt`
Key interfaces: `AuthService`, `TransactionProcessor`, `CashDispenserPort`
Key pattern: State (ATM states: Idle → CardInserted → PinVerified → Dispensing → Done), Command (Withdraw, Deposit, CheckBalance)

**Chess**
Core entities: `Board`, `Square`, `Piece`, `Player`, `Move`, `Game`
Key interfaces: `MoveValidator`, `PieceMovementStrategy`
Key pattern: Strategy (movement per piece type — Rook, Bishop, Knight all implement `Mover`), Observer (board change notifies UI)

**Splitwise**
Core entities: `User`, `Group`, `Expense`, `Split`, `Balance`, `Settlement`
Key interfaces: `SplitStrategy`, `NotificationService`
Key pattern: Strategy (EqualSplit, ExactSplit, PercentageSplit), Observer (notify users on expense creation)

**Elevator System**
Core entities: `Building`, `Elevator`, `Floor`, `Request`, `ElevatorController`
Key interfaces: `DispatchStrategy`
Key pattern: Strategy (dispatch algorithm — nearest car, SCAN), State (elevator states: Idle, Moving, DoorsOpen)

**Library Management**
Core entities: `Library`, `Book`, `BookItem`, `Member`, `Librarian`, `Loan`, `Fine`
Key interfaces: `SearchCatalog`, `NotificationService`
Key pattern: Observer (overdue notification), Factory (creating BookItem copies)

---

## Worked Mini-Example: Parking Lot

A condensed demonstration of all five steps applied to a single problem.

**Step 1 — Clarify:**
- Multiple floors, multiple slot types (bike, car, truck)
- One entry and one exit gate per floor (simplified)
- Fee: hourly rate based on vehicle type
- In-memory, single instance, no persistence needed

**Step 2 — Entities:**
`ParkingLot`, `Floor`, `Slot`, `Vehicle`, `Ticket`, `Money`

**Step 3 — Interfaces:**

```go
type SlotAllocator interface {
    Allocate(v Vehicle) (Slot, error)
    Release(slotID string) error
}

type FeeCalculator interface {
    Calculate(t Ticket) (Money, error)
}
```

**Step 4 — Structure:**

```
  ParkingLot
    ├── floors      []*Floor        (composition)
    ├── allocator   SlotAllocator   (injected interface)
    └── calculator  FeeCalculator   (injected interface)

  Floor
    ├── level  int
    └── slots  []*Slot             (composition)

  Slot
    ├── id       string
    ├── slotType SlotType
    └── occupied bool

  Ticket
    ├── id        string
    ├── slotID    string
    ├── vehicle   Vehicle
    └── entryTime time.Time

  HourlyFeeCalculator (implements FeeCalculator)
    └── ratePerHour map[VehicleType]float64

  FirstAvailableAllocator (implements SlotAllocator)
    └── floors []*Floor
```

**Step 5 — Core code:**

```go
package parking

import (
    "errors"
    "fmt"
    "time"
)

type VehicleType int

const (
    Bike VehicleType = iota
    Car
    Truck
)

type SlotType = VehicleType

var ErrNoSlotAvailable = errors.New("no slot available")

// --- Domain types ---

type Vehicle struct {
    ID   string
    Type VehicleType
}

type Slot struct {
    ID       string
    Type     SlotType
    occupied bool
}

func (s *Slot) Occupy() error {
    if s.occupied {
        return fmt.Errorf("slot %s already occupied", s.ID)
    }
    s.occupied = true
    return nil
}

func (s *Slot) Free() {
    s.occupied = false
}

type Ticket struct {
    ID        string
    SlotID    string
    Vehicle   Vehicle
    EntryTime time.Time
}

type Money struct {
    Amount   float64
    Currency string
}

// --- Interfaces ---

type SlotAllocator interface {
    Allocate(v Vehicle) (Slot, error)
    Release(slotID string) error
}

type FeeCalculator interface {
    Calculate(t Ticket) (Money, error)
}

// --- ParkingLot ---

type ParkingLot struct {
    floors    []*Floor
    allocator SlotAllocator
    calc      FeeCalculator
}

func NewParkingLot(floors []*Floor, a SlotAllocator, c FeeCalculator) *ParkingLot {
    return &ParkingLot{floors: floors, allocator: a, calc: c}
}

func (p *ParkingLot) Park(v Vehicle) (Ticket, error) {
    slot, err := p.allocator.Allocate(v)
    if err != nil {
        return Ticket{}, fmt.Errorf("park: %w", err)
    }
    return Ticket{
        ID:        newID(),
        SlotID:    slot.ID,
        Vehicle:   v,
        EntryTime: time.Now(),
    }, nil
}

func (p *ParkingLot) Exit(t Ticket) (Money, error) {
    fee, err := p.calc.Calculate(t)
    if err != nil {
        return Money{}, fmt.Errorf("exit: calculate: %w", err)
    }
    if err := p.allocator.Release(t.SlotID); err != nil {
        return Money{}, fmt.Errorf("exit: release: %w", err)
    }
    return fee, nil
}

// --- HourlyFeeCalculator ---

type HourlyFeeCalculator struct {
    ratePerHour map[VehicleType]float64
}

func NewHourlyFeeCalculator(rates map[VehicleType]float64) *HourlyFeeCalculator {
    return &HourlyFeeCalculator{ratePerHour: rates}
}

func (h *HourlyFeeCalculator) Calculate(t Ticket) (Money, error) {
    rate, ok := h.ratePerHour[t.Vehicle.Type]
    if !ok {
        return Money{}, fmt.Errorf("no rate for vehicle type %v", t.Vehicle.Type)
    }
    hours := time.Since(t.EntryTime).Hours()
    if hours < 1 {
        hours = 1 // minimum 1 hour
    }
    return Money{Amount: rate * hours, Currency: "INR"}, nil
}
```

**What to say after coding:**
- "I'd add a mutex in `FirstAvailableAllocator.Allocate` and `Release` to handle concurrent access."
- "The `FeeCalculator` strategy makes it easy to add `DailyFeeCalculator` or `FlatRateFeeCalculator` without touching `ParkingLot`."
- "I'd extract `newID()` into a `IDGenerator` interface to make it mockable in tests."

---

## What To Say When You're Stuck

**If you don't know which pattern to use:**
"I can see this has multiple implementations of the same operation — my instinct is Strategy here. Does that align with what you had in mind, or would you like me to explore something else?"

**If the requirements are ambiguous:**
"I'm going to assume X for now and design around it — let me know if you want me to revisit." Then proceed. Waiting for perfect clarity is a time sink.

**If you run out of time:**
"I've implemented the core allocator and fee logic. Given more time, I'd implement the payment processor and the gate controller. Want me to walk through what those would look like at a higher level?"

**If challenged on a design choice:**
"The reason I chose composition over inheritance here is that `ElectricVehicle` would need to override methods if we used embedding, which Go doesn't support. Keeping them as separate structs with a shared `VehicleInfo` field keeps them independently changeable." Have a reason for every choice.

---

## Anti-Patterns That Kill Interviews

**Jumping to code immediately.** Interviewers see through it. Spending 3 minutes asking clarifying questions signals seniority; diving into code signals anxiety.

**Giant interfaces.** `Vehicle interface { ID() string; Type() VehicleType; LicensePlate() string; Owner() User; Color() string }` — this is a Java habit. Break it. Ask: what does the consumer actually need?

**Inheritance chains.** `CarSlot` extends `Slot` extends `ParkingEntity`. Go doesn't have inheritance. Even in other languages, prefer composition. A `SlotType` field on `Slot` is cleaner than a type hierarchy.

**Exported fields.** `type Slot struct { ID string; Occupied bool }`. Now every caller can set `Occupied = false` without going through your logic. Make fields unexported; expose behaviour through methods.

**No constructor functions.** `slot := Slot{ID: "A1", Type: Car}` — callers now know internal field names and initialise them directly. Use `NewSlot(id string, t SlotType) *Slot`.

**Over-engineering on the first pass.** Don't build a plugin system for fee calculation before you've built fee calculation. Get the core working, then discuss extensions.

**Silence.** If you're not talking, the interviewer has no signal. Think out loud. "I'm considering two options here..."

---

## Interview Cheat Sheet

**Opening line:** "Before I start designing, I'd like to ask a few clarifying questions."

**Requirement question templates:**
- "What are the core use cases we need to support?"
- "Are there multiple types of X, or is it homogeneous?"
- "Is this a single-instance system or does it need to be thread-safe?"
- "Should I handle [edge case] or can I treat it as out of scope?"

**Design narration templates:**
- "The entities I see here are... Let me explain why each one exists."
- "I'm defining this as an interface because the consumer only needs to know X — the implementation can vary."
- "I'm choosing composition here over inheritance because..."
- "This maps naturally to the Strategy pattern because we have multiple algorithms for the same operation."

**Extension narration templates:**
- "With more time, I'd add X to handle Y."
- "For concurrency, I'd protect this method with a mutex / use a channel-based queue."
- "To make this testable, I'd inject a clock interface instead of calling `time.Now()` directly."

**Red flags to self-correct:**
- "Am I coding before I have a clear entity map?" → Stop, draw the structure first.
- "Is this interface growing beyond two or three methods?" → Split it.
- "Am I exporting this field because it's convenient?" → Make it unexported; add a method.
- "Have I been silent for more than 30 seconds?" → Narrate your thinking.

---

## References

- [Effective Go — Interfaces and Methods](https://go.dev/doc/effective_go#interfaces_and_types)
- [Go Blog — Laws of Reflection (on interfaces internally)](https://go.dev/blog/laws-of-reflection)
- [Go Blog — Errors Are Values](https://go.dev/blog/errors-are-values)
- [Go Standard Library — io package (canonical small interface example)](https://pkg.go.dev/io)
- [Design Patterns: Elements of Reusable Object-Oriented Software — GoF Book](https://en.wikipedia.org/wiki/Design_Patterns)
- [Refactoring Guru — Design Patterns — excellent visual walkthroughs of all GoF patterns](https://refactoring.guru/design-patterns)
- [SOLID Principles (original papers, Robert C. Martin)](https://web.archive.org/web/20150202200348/http://www.objectmentor.com/resources/articles/Principles_and_Patterns.pdf)
- [Dave Cheney — Practical Go: Real World Advice for Writing Maintainable Go Programs](https://dave.cheney.net/practical-go/presentations/gophercon-singapore-2019.html)
- [Dave Cheney — SOLID Go Design](https://dave.cheney.net/2016/08/20/solid-go-design)