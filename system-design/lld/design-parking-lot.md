---
Status: 🌳 Evergreen
Created: 2026-07-02
Last Updated: 2026-07-02
---

# Parking Lot

## Table Of Contents

1. [What It Is And Why It Exists](#what-it-is-and-why-it-exists)
2. [Step 1 — Clarify Requirements](#step-1--clarify-requirements)
3. [Step 2 — Identify Entities And Actors](#step-2--identify-entities-and-actors)
4. [Step 3 — Define Interfaces](#step-3--define-interfaces)
5. [Class Diagram](#class-diagram)
6. [Go Implementation](#go-implementation)
7. [Why Slot Occupancy Stays A Bool But Ticket Gets The Full State Pattern](#why-slot-occupancy-stays-a-bool-but-ticket-gets-the-full-state-pattern)
8. [Edge Cases And Extensions](#edge-cases-and-extensions)
9. [Patterns Used In This Design](#patterns-used-in-this-design)
10. [Problems This Design Brings](#problems-this-design-brings)
11. [Trade-Offs](#trade-offs)
12. [How To Remember This](#how-to-remember-this)
13. [What Interviewers Actually Look For](#what-interviewers-actually-look-for)
14. [Interview Cheat Sheet](#interview-cheat-sheet)
15. [References](#references)

## What It Is And Why It Exists

Parking Lot is the classic "first" LLD interview problem: design the classes for a multi-floor parking facility that can park a vehicle, charge a fee, and let it exit.

**Why this exact problem is used as a teaching vehicle:** everyone already understands how a parking lot works, so zero domain research is needed — the entire interview budget goes toward *design*, not requirements archaeology. It also happens to contain, in miniature, almost every recurring LLD ingredient: a natural ownership hierarchy (composition), categorization that varies in a fixed set of ways (enums), an object whose behavior genuinely changes over its lifetime (state), and a pricing rule that should be swappable without touching anything else (strategy). That combination is exactly why the roadmap lists "OOP modeling, enums, state pattern" as the key concepts here.

A condensed version of this problem already lives in `lld/interview-approach.md` as a five-step worked example. This file goes further: a richer slot/vehicle compatibility model, a properly implemented State pattern for the ticket lifecycle (rather than a one-line stub), and a concurrency-safe allocator that handles 200 goroutines competing for 50 slots correctly, not just a promise that it would.

## Step 1 — Clarify Requirements

**Functional:**

- A vehicle can be parked, if and only if a compatible slot is free.
- A driver pays before exiting; payment is required before the slot is released.
- The lot has multiple floors, each with multiple slots of different sizes.
- Fee is computed hourly, varying by vehicle type.

**Non-functional:**

- Single-instance, in-process system — everything lives in memory in one Go process; no database, no separate servers, no distribution.
- Concurrent-safe: two vehicles arriving at the same instant must never be handed the same slot. This matters even though the system is single-process, because "single-process" is not the same as "single-threaded" — many goroutines (e.g. one per incoming request) can call `Park()` at the same moment within that one process. A mutex is needed here for exactly that reason: it protects against concurrent goroutines racing on shared state, not against multiple machines.

**Scope control, said out loud:**

> "I'll model parking, payment, and exit as the core flow, with a pluggable fee strategy. I'll treat payment processing itself as an injected `interface` rather than implementing a payment gateway, and I'll mention entry/exit gates and a live availability display as extensions rather than building them out, unless you'd like me to go there."

## Step 2 — Identify Entities And Actors

```
Nouns (candidate entities):
  ParkingLot, Floor, Slot, Vehicle, Ticket, Money,
  SlotAllocator, FeeCalculator

Actors (drive the system, are not structs):
  Driver  -- triggers Park / Pay / Exit
  Admin   -- out of scope here (would configure rates, add floors)
```

For each entity: what it holds, what it does, what it depends on.

| Entity | Holds | Knows How To | Depends On |
|---|---|---|---|
| `ParkingLot` | floors, ticket registry | Park, Pay, Exit | `SlotAllocator`, `FeeCalculator` (both interfaces) |
| `Floor` | level, slots | — (pure data holder) | — |
| `Slot` | id, size, occupied | Occupy, Free | — |
| `Vehicle` | id, type | — | — |
| `Ticket` | id, slot id, vehicle, timestamps, fee, **state** | Pay, CheckOut, Status | `TicketState` (interface, swaps itself) |

## Step 3 — Define Interfaces

Defined where they're consumed (`ParkingLot` needs them), not where they're implemented — the Go idiom already established in `lld/interview-approach.md`.

```go
type SlotAllocator interface {
    Allocate(v Vehicle) (*Slot, error)
    Release(slotID string) error
}

type FeeCalculator interface {
    Calculate(t Ticket) (Money, error)
}
```

Both are single-purpose, single-method-family interfaces — easy to mock in tests, easy to swap (a `NearestSlotAllocator` or a `DailyFeeCalculator` are drop-in replacements that never require touching `ParkingLot`).

## Class Diagram

```
  ParkingLot
    - floors      []*Floor          (composition: owns Floor)
    - allocator   SlotAllocator     (dependency: injected interface)
    - calc        FeeCalculator     (dependency: injected interface)
    - tickets     map[string]*Ticket
    + Park(v Vehicle) (*Ticket, error)
    + Pay(ticketID string) (Money, error)
    + Exit(ticketID string) error
         |
         | composition
         v
  Floor
    - level  int
    - slots  []*Slot                (composition: owns Slot)
         |
         | composition
         v
  Slot
    - id        string
    - size      SlotSize
    - occupied  bool

  Ticket                            (separate object graph — references, doesn't own, a Slot)
    - id, slotID, vehicle, entryTime, exitTime, fee
    - state  TicketState            (State pattern — see below)
```

`ParkingLot → Floor → Slot` is composition all the way down: a `Slot` cannot meaningfully exist outside a `Floor`, nor a `Floor` outside a `ParkingLot`. `Ticket` only *references* a slot by ID (aggregation) — the slot exists independently of any one ticket's lifetime.

## Go Implementation

### File Organization

Interviewers who ask for real code — onsite coding rounds, take-homes, or a direct "how would you structure this as a project?" — are watching whether you group code by cohesion, not just correctness. Dumping every type into one file reads as inexperience with real codebases.

The principle: **each file holds one cohesive responsibility** — either a single behavioral unit, or a full pattern (interface plus every implementer of it), so a reader never has to jump across files to see one abstraction end to end.

```
types.go            Vehicle, Money, VehicleType, SlotSize, sentinel errors.
                     No behavior — just domain vocabulary everything else
                     depends on. Read this file first.

slot.go              Slot + Floor. Kept together because Floor owns Slots
                     (composition) — neither is meaningful alone.

ticket.go            Ticket + the full State pattern: TicketState interface
                     and all three concrete states in one file. Splitting
                     activeState/paidState/exitedState across files would
                     force a reader to hunt for one FSM's pieces.

allocator.go          SlotAllocator interface + FirstAvailableAllocator.
                     Interface and its implementation live together.

fee.go               FeeCalculator interface + HourlyFeeCalculator +
                     FlatFeeCalculator + the round2 helper they share.

parkinglot.go         ParkingLot — the Facade. Depends on every file
                     above, so it sits last in the dependency order.

gate.go              EntryGate + ExitGate. Thin wrappers around the
                     Facade for the multi-gate extension (see Edge
                     Cases And Extensions) — kept out of parkinglot.go
                     since they're an optional extension, not core.

persistence.go       TicketRepository interface + in-memory
                     implementation — the storage boundary for the
                     Persistence extension.

payment.go           PaymentProcessor interface + mock implementation —
                     the Payment Gateway extension.

observer.go           LotObserver interface + DisplayBoard — the
                     Observer extension for the Lot Full sign.

parkinglot_test.go    Core Facade tests. Standard Go convention:
                     _test.go suffix, same package, compiled only
                     during `go test`.

gate_test.go          Gate-specific tests, kept separate from
                     parkinglot_test.go for the same reason gate.go
                     is kept separate from parkinglot.go.

fee_test.go, persistence_test.go, payment_test.go, observer_test.go
                     One test file per extension file, same pairing
                     convention as the rest of the package.
```

**The rule to say out loud:** organize by cohesion, not by type — one file per pattern or behavioral unit, interfaces living beside their implementations, and shared behavior-less value types collected into a `types.go` that everything else can depend on without circular imports.

### Types And Enums

```go
type VehicleType int

const (
    Bike VehicleType = iota
    Car
    Truck
)

// SlotSize is ordered so a larger size can always host anything a smaller
// size can host — Large >= Medium >= Small as integers.
type SlotSize int

const (
    Small  SlotSize = iota // fits bikes only
    Medium                 // fits bikes, cars
    Large                  // fits bikes, cars, trucks
)

// minSlotSize is the smallest slot size that legally hosts a vehicle type.
var minSlotSize = map[VehicleType]SlotSize{
    Bike:  Small,
    Car:   Medium,
    Truck: Large,
}

// Fits reports whether a vehicle of type v can park in a slot of size s.
func Fits(v VehicleType, s SlotSize) bool {
    return s >= minSlotSize[v]
}

type Vehicle struct {
    ID   string
    Type VehicleType
}

// Money is a minimal value type for fees. A real system would use an
// integer minor-unit (cents) representation to avoid float rounding —
// kept as float64 here for teaching clarity.
type Money struct {
    Amount   float64
    Currency string
}

var (
    ErrNoSlotAvailable        = errors.New("no slot available for vehicle type")
    ErrSlotAlreadyOccupied    = errors.New("slot already occupied")
    ErrSlotNotFound           = errors.New("slot not found")
    ErrInvalidStateTransition = errors.New("invalid ticket state transition")
    ErrTicketNotFound         = errors.New("ticket not found")
)
```

Sentinel errors are declared once, here, and referenced by name everywhere else in the design — this is what makes `errors.Is(err, ErrInvalidStateTransition)` in the tests below possible, instead of comparing fragile string messages.

This is a deliberate upgrade over the condensed mini-example, which used `type SlotType = VehicleType` as a direct 1:1 alias. That simplification quietly assumes a bike can only ever park in a bike-sized slot — wrong in any real lot, where a bike happily fits in a car spot. Modeling `SlotSize` as its own ordered enum with a `Fits` compatibility rule is the kind of detail an interviewer notices.

### Slot And Floor

```go
type Slot struct {
    ID       string
    Size     SlotSize
    occupied bool
}

func (s *Slot) IsOccupied() bool { return s.occupied }

func (s *Slot) occupy() error {
    if s.occupied {
        return fmt.Errorf("slot %s: %w", s.ID, ErrSlotAlreadyOccupied)
    }
    s.occupied = true
    return nil
}

func (s *Slot) free() { s.occupied = false }

type Floor struct {
    Level int
    Slots []*Slot
}
```

### Ticket And The State Pattern

A `Ticket` has exactly three states with materially different allowed behavior: **Active** (just parked, can pay, cannot exit), **Paid** (fee settled, can exit, cannot pay again), and **Exited** (terminal — nothing further is legal). That's the textbook trigger condition for the State pattern: more than two states, with behavior that genuinely differs per state. (Full pattern theory in `lld/patterns/behavioral/state.md` — applied here, not re-derived.)

```go
type TicketState interface {
    Name() string
    Pay(t *Ticket, calc FeeCalculator) error
    CheckOut(t *Ticket) error
}

type Ticket struct {
    ID        string
    SlotID    string
    Vehicle   Vehicle
    EntryTime time.Time
    ExitTime  time.Time
    Fee       Money
    state     TicketState
}

func NewTicket(id, slotID string, v Vehicle) *Ticket {
    return &Ticket{ID: id, SlotID: slotID, Vehicle: v, EntryTime: time.Now(), state: activeState{}}
}

func (t *Ticket) Status() string               { return t.state.Name() }
func (t *Ticket) Pay(calc FeeCalculator) error  { return t.state.Pay(t, calc) }
func (t *Ticket) CheckOut() error               { return t.state.CheckOut(t) }
func (t *Ticket) transition(s TicketState)      { t.state = s }

// Each concrete state is a zero-size struct — no per-instance data, just
// behavior. This is the idiomatic Go shape: stateless value receivers,
// no allocation needed beyond the assignment in transition().

type activeState struct{}

func (activeState) Name() string { return "Active" }

func (activeState) Pay(t *Ticket, calc FeeCalculator) error {
    fee, err := calc.Calculate(*t)
    if err != nil {
        return fmt.Errorf("pay: %w", err)
    }
    t.Fee = fee
    t.transition(paidState{})
    return nil
}

func (activeState) CheckOut(t *Ticket) error {
    return fmt.Errorf("checkout from Active: %w", ErrInvalidStateTransition)
}

type paidState struct{}

func (paidState) Name() string { return "Paid" }

func (paidState) Pay(t *Ticket, calc FeeCalculator) error {
    return fmt.Errorf("pay from Paid: %w", ErrInvalidStateTransition)
}

func (paidState) CheckOut(t *Ticket) error {
    t.ExitTime = time.Now()
    t.transition(exitedState{})
    return nil
}

type exitedState struct{}

func (exitedState) Name() string                                   { return "Exited" }
func (exitedState) Pay(t *Ticket, calc FeeCalculator) error {
    return fmt.Errorf("pay from Exited: %w", ErrInvalidStateTransition)
}
func (exitedState) CheckOut(t *Ticket) error {
    return fmt.Errorf("checkout from Exited: %w", ErrInvalidStateTransition)
}
```

**State transition diagram:**

```
  [Active] --Pay()--> [Paid] --CheckOut()--> [Exited]
     |                   |
     | CheckOut()        | Pay()
     v                   v
   error               error
  (must pay first)   (already paid)

  Exited is terminal: both Pay() and CheckOut() from Exited return an error.
```

### Concurrency-Safe Allocator

```go
type SlotAllocator interface {
    Allocate(v Vehicle) (*Slot, error)
    Release(slotID string) error
}

type FirstAvailableAllocator struct {
    mu     sync.Mutex
    floors []*Floor
}

func NewFirstAvailableAllocator(floors []*Floor) *FirstAvailableAllocator {
    return &FirstAvailableAllocator{floors: floors}
}

func (a *FirstAvailableAllocator) Allocate(v Vehicle) (*Slot, error) {
    a.mu.Lock()
    defer a.mu.Unlock()

    for _, f := range a.floors {
        for _, s := range f.Slots {
            if !s.IsOccupied() && Fits(v.Type, s.Size) {
                _ = s.occupy() // safe: still holding the lock, cannot race
                return s, nil
            }
        }
    }
    return nil, ErrNoSlotAvailable
}

func (a *FirstAvailableAllocator) Release(slotID string) error {
    a.mu.Lock()
    defer a.mu.Unlock()

    for _, f := range a.floors {
        for _, s := range f.Slots {
            if s.ID == slotID {
                s.free()
                return nil
            }
        }
    }
    return ErrSlotNotFound
}
```

The mutex wraps the entire scan-and-occupy sequence, not just the `occupy()` call — locking only the write would still let two goroutines both pass the `!s.IsOccupied()` check on the same slot before either claims it. The lock has to cover the *check* and the *claim* together. The scenario worth testing here: 200 concurrent goroutines competing for 50 slots should produce exactly 50 successes and zero duplicates.

The real allocator in this package also has a `Subscribe` method and calls a `notifyLocked` helper at the end of `Allocate`/`Release`, for the Observer extension (see Edge Cases And Extensions — Lot Full) — omitted from the snippet above to keep the core invariant readable on its own. That notification hook could *not* have been bolted on from outside as a decorator wrapping `SlotAllocator`: a decorator would need to scan `Slot.IsOccupied()` after delegating, without holding `a.mu` — an unsynchronized read of a field that `occupy()`/`free()` write while holding that same lock. That is a real data race, not just an appearance of one. Some extensions can live outside the lock (gates, persistence, payment); this one can't, because it needs to read the exact state the lock protects.

### Fee Calculation (Strategy)

```go
type FeeCalculator interface {
    Calculate(t Ticket) (Money, error)
}

// ratePerHour is guarded by mu because it can be replaced at runtime via
// UpdateRates (see Edge Cases And Extensions — Admin) while Calculate is
// reading it concurrently. Same class of Go map race the allocator
// already guards against, same fix: a mutex.
type HourlyFeeCalculator struct {
    mu          sync.RWMutex
    ratePerHour map[VehicleType]float64
}

func NewHourlyFeeCalculator(rates map[VehicleType]float64) *HourlyFeeCalculator {
    return &HourlyFeeCalculator{ratePerHour: rates}
}

func (h *HourlyFeeCalculator) Calculate(t Ticket) (Money, error) {
    h.mu.RLock()
    rate, ok := h.ratePerHour[t.Vehicle.Type]
    h.mu.RUnlock()
    if !ok {
        return Money{}, fmt.Errorf("no rate configured for %s", t.Vehicle.Type)
    }
    hours := math.Ceil(time.Since(t.EntryTime).Hours())
    if hours < 1 {
        hours = 1
    }
    return Money{Amount: round2(rate * hours), Currency: "USD"}, nil
}

// FlatFeeCalculator always returns the same amount, regardless of duration.
// Used for the lost-ticket edge case: if EntryTime is unreliable, duration
// pricing is meaningless, so a flat penalty is substituted at the call site —
// a clean demonstration of swapping the Strategy without touching ParkingLot.
type FlatFeeCalculator struct {
    Flat Money
}

func (f *FlatFeeCalculator) Calculate(t Ticket) (Money, error) {
    return f.Flat, nil
}

// round2 rounds to 2 decimal places — needed because float64 multiplication
// (rate * hours) produces values like 15.000000000000002, which is not a
// valid currency amount. Money.Amount must always be a clean 2-decimal value.
func round2(f float64) float64 {
    return math.Round(f*100) / 100
}
```

### The ParkingLot Facade

```go
type ParkingLot struct {
    floors    []*Floor
    allocator SlotAllocator
    calc      FeeCalculator

    mu      sync.Mutex
    tickets map[string]*Ticket
    nextID  int
}

func NewParkingLot(floors []*Floor, a SlotAllocator, c FeeCalculator) *ParkingLot {
    return &ParkingLot{floors: floors, allocator: a, calc: c, tickets: make(map[string]*Ticket)}
}

func (p *ParkingLot) Park(v Vehicle) (*Ticket, error) {
    slot, err := p.allocator.Allocate(v)
    if err != nil {
        return nil, fmt.Errorf("park: %w", err)
    }
    p.mu.Lock()
    p.nextID++
    t := NewTicket(fmt.Sprintf("T-%04d", p.nextID), slot.ID, v)
    p.tickets[t.ID] = t
    p.mu.Unlock()
    return t, nil
}

func (p *ParkingLot) Pay(ticketID string) (Money, error) {
    t, err := p.lookup(ticketID)
    if err != nil {
        return Money{}, err
    }
    if err := t.Pay(p.calc); err != nil {
        return Money{}, fmt.Errorf("pay: %w", err)
    }
    return t.Fee, nil
}

func (p *ParkingLot) Exit(ticketID string) error {
    t, err := p.lookup(ticketID)
    if err != nil {
        return err
    }
    if err := t.CheckOut(); err != nil {
        return fmt.Errorf("exit: %w", err)
    }
    return p.allocator.Release(t.SlotID)
}

func (p *ParkingLot) lookup(ticketID string) (*Ticket, error) {
    p.mu.Lock()
    defer p.mu.Unlock()
    t, ok := p.tickets[ticketID]
    if !ok {
        return nil, fmt.Errorf("ticket %s: %w", ticketID, ErrTicketNotFound)
    }
    return t, nil
}
```

A caller only ever sees three methods — `Park`, `Pay`, `Exit`. Allocation strategy, fee strategy, and ticket state transitions are entirely hidden behind that surface. That's the Facade.

## Why Slot Occupancy Stays A Bool But Ticket Gets The Full State Pattern

This is worth stating explicitly in an interview, because reaching for State *everywhere* is itself a smell. `Slot.occupied` only ever has two values, with one trivial transition rule (occupied ⇄ free, both directions always legal given the right caller). A `bool` and two guarded methods (`occupy`/`free`) say everything that needs saying — wrapping that in a full `SlotState` interface with `AvailableState`/`OccupiedState` types would be ceremony with no payoff: no third state is coming, and the transition logic isn't going to grow.

`Ticket`, by contrast, has **three** states with genuinely different legal operations per state, and a real terminal state (`Exited`) where *everything* becomes illegal. That asymmetry — more than two states, materially different behavior, a terminal state — is the actual trigger condition for reaching for State. Knowing the difference, and saying so, is a stronger signal than mechanically applying every pattern you know.

## Edge Cases And Extensions

**Lost ticket:** if a ticket's `EntryTime` can't be trusted (the physical ticket was lost and re-issued at the booth), duration-based pricing is meaningless. Swap in `FlatFeeCalculator` for that one ticket at the point of payment — no change to `ParkingLot`, `Ticket`, or the allocator required. This is the Strategy pattern paying for itself.

**Lot full:** `Allocate` returns `ErrNoSlotAvailable`; the natural extension is surfacing this to a physical "LOT FULL" sign — the Observer pattern. Full pattern theory lives at `lld/patterns/behavioral/observer.md`; this is the applied version, kept intentionally small: the allocator is the subject, `DisplayBoard` is one possible observer, and neither knows anything about the other beyond the `LotObserver` interface.

```go
// LotObserver is notified whenever occupancy changes. FirstAvailableAllocator
// is the subject — it doesn't know or care what a DisplayBoard (or anything
// else) does with the notification, it just reports occupied/total.
type LotObserver interface {
    OnOccupancyChanged(occupied, total int)
}

// Subscribe registers an observer. Purely additive to the allocator —
// existing callers that never subscribe anything see no behavior change.
func (a *FirstAvailableAllocator) Subscribe(o LotObserver) {
    a.mu.Lock()
    defer a.mu.Unlock()
    a.observers = append(a.observers, o)
}

// notifyLocked is called from inside Allocate/Release while a.mu is
// already held — it never needs a lock of its own.
func (a *FirstAvailableAllocator) notifyLocked() {
    occupied, total := 0, 0
    for _, f := range a.floors {
        for _, s := range f.Slots {
            total++
            if s.IsOccupied() {
                occupied++
            }
        }
    }
    for _, o := range a.observers {
        o.OnOccupancyChanged(occupied, total)
    }
}

// DisplayBoard is a concrete observer — the "LOT FULL" sign itself. It only
// tracks fullness; it knows nothing about allocation logic.
type DisplayBoard struct {
    Full bool
}

func (d *DisplayBoard) OnOccupancyChanged(occupied, total int) {
    d.Full = occupied >= total
}
```

The scenario worth testing here: a single-slot lot driven empty → full → empty, checking the board's state at each transition.

**Multiple entry/exit gates:** out of scope here by the Step 1 scoping statement — but "out of scope" should still mean you can write the code if asked, not just describe it. Both gate types are thin wrappers around the existing Facade; they hold no state of their own and need no additional locking, because concurrency safety already lives inside `ParkingLot`/`FirstAvailableAllocator`.

```go
// EntryGate represents one physical entry terminal. A lot can have several —
// all of them share the same underlying ParkingLot, so a vehicle entering
// through Gate 2 competes for the same slot pool as one entering through
// Gate 1.
type EntryGate struct {
    ID  string
    Lot *ParkingLot
}

func (g *EntryGate) Enter(v Vehicle) (*Ticket, error) {
    t, err := g.Lot.Park(v)
    if err != nil {
        return nil, fmt.Errorf("entry gate %s: %w", g.ID, err)
    }
    return t, nil
}

// ExitGate represents one physical exit terminal. Pay-then-Exit is
// sequenced here rather than inside ParkingLot, because a real gate might
// insert a manual override or a receipt-printing step between the two.
type ExitGate struct {
    ID  string
    Lot *ParkingLot
}

func (g *ExitGate) Exit(ticketID string) (Money, error) {
    fee, err := g.Lot.Pay(ticketID)
    if err != nil {
        return Money{}, fmt.Errorf("exit gate %s: %w", g.ID, err)
    }
    if err := g.Lot.Exit(ticketID); err != nil {
        return Money{}, fmt.Errorf("exit gate %s: %w", g.ID, err)
    }
    return fee, nil
}
```

Worth testing here: one scenario driving the full `Enter` → `Exit` flow through both gate types, and a second proving the shared-pool property that justifies the whole design — a lot with exactly one slot rejects a second vehicle entering through a *different* gate, confirming gates don't each get private capacity, they compete for the same pool the Facade already protects.

**Persistence:** swapping the in-memory ticket map for a real database means defining the storage boundary as an interface, so `ParkingLot` never has to change when the underlying store does.

```go
// TicketRepository abstracts ticket storage. Swapping the in-memory map
// for a real database means implementing this interface once; nothing
// in ParkingLot, Ticket, or the allocator needs to change.
type TicketRepository interface {
    Save(t *Ticket) error
    Get(ticketID string) (*Ticket, error)
}

// inMemoryTicketRepository is what ParkingLot uses today, expressed
// through the interface boundary instead of a bare map field.
type inMemoryTicketRepository struct {
    tickets map[string]*Ticket
}

func (r *inMemoryTicketRepository) Save(t *Ticket) error {
    r.tickets[t.ID] = t
    return nil
}

func (r *inMemoryTicketRepository) Get(ticketID string) (*Ticket, error) {
    t, ok := r.tickets[ticketID]
    if !ok {
        return nil, fmt.Errorf("ticket %s: %w", ticketID, ErrTicketNotFound)
    }
    return t, nil
}
```

Worth testing here: a compile-time assertion (`var _ TicketRepository = (*inMemoryTicketRepository)(nil)`) that the in-memory type actually satisfies the interface — the cheapest possible check for "does this implementation match this contract" — plus the ordinary save-then-get and not-found cases.

**Payment gateway:** `Pay()` currently only computes a fee via `FeeCalculator` — it never actually charges a card. `FeeCalculator` answers "how much?"; a separate `PaymentProcessor` answers "how do we actually collect it?" — two different concerns, two different injected interfaces, following the same pattern already used everywhere else in this design.

```go
type PaymentMethod int

const (
    Card PaymentMethod = iota
    Cash
)

// PaymentProcessor abstracts actually collecting money. A real
// implementation wraps a third-party gateway (Stripe-like) with retries,
// using idempotencyKey so a retried request never double-charges.
type PaymentProcessor interface {
    Charge(amount Money, method PaymentMethod, idempotencyKey string) error
}

// mockPaymentProcessor stands in for a real gateway in tests.
type mockPaymentProcessor struct {
    charged []Money
}

func (m *mockPaymentProcessor) Charge(amount Money, method PaymentMethod, idempotencyKey string) error {
    if amount.Amount <= 0 {
        return fmt.Errorf("charge: amount must be positive, got %.2f", amount.Amount)
    }
    m.charged = append(m.charged, amount)
    return nil
}
```

Worth testing here: a rejection case (a zero-amount charge should return an error) alongside the ordinary success case, plus a compile-time interface-satisfaction check. Wiring this into the real `Pay()` flow means calling `Charge` after the fee is computed but before the ticket's state transitions to `Paid` — that ordering matters, because a failed charge should leave the ticket `Active`, not silently mark it `Paid` with nothing collected.

**Admin — rate configuration and floor changes:** named out of scope in Step 1 as an actor who would "configure rates, add floors." `HourlyFeeCalculator` (above) already carries the mutex needed for this — the admin extension is just the write side:

```go
// UpdateRates replaces the entire rate table atomically. A config service
// calls this after an operator changes pricing — no change needed to
// ParkingLot or Ticket.
func (h *HourlyFeeCalculator) UpdateRates(newRates map[VehicleType]float64) {
    h.mu.Lock()
    defer h.mu.Unlock()
    h.ratePerHour = newRates
}
```

The scenario worth testing here: 50 goroutines calling `Calculate` concurrently against 50 goroutines calling `UpdateRates` — proving the lock is both necessary and sufficient.

Adding a floor is simpler still: append to the `[]*Floor` slice the allocator already scans. Floor layout changes are a rare operational event, not a hot-path operation, so this can happen behind the allocator's existing mutex, or even just at process restart with an updated config — no need to build live-reconfiguration unless specifically asked for.

## Patterns Used In This Design

| Pattern | Where | Why It Fits Here |
|---|---|---|
| Facade | `ParkingLot`'s three-method surface | Hides allocator, fee strategy, and ticket internals behind `Park`/`Pay`/`Exit` |
| Strategy | `FeeCalculator` (`HourlyFeeCalculator`, `FlatFeeCalculator`) | Pricing rule swaps without touching `ParkingLot` or `Ticket` |
| State | `TicketState` (`activeState`, `paidState`, `exitedState`) | Three states, materially different legal operations per state |
| (Not used) Observer | Would back a live availability display | Deliberately left as a named extension — see `lld/patterns/behavioral/observer.md` |

## Problems This Design Brings

- **Mutex granularity:** `FirstAvailableAllocator`'s single mutex serializes *all* allocation across the entire lot, even across floors that have nothing to do with each other. Fine at this scale; would become a real bottleneck at very high concurrent arrival rates, where a per-floor lock (or a lock-free structure) would be the next step.
- **Linear scan allocation:** `Allocate` is O(floors × slots) in the worst case (lot nearly full). Acceptable for a lot with hundreds of slots; a large multi-level garage might index free slots by size in a separate structure to avoid the scan.
- **In-memory only:** every ticket and slot state vanishes on process restart — explicitly scoped out in Step 1, but worth naming as the first thing that would need to change for production use.
- **Float-based `Money`:** used here for teaching clarity; real systems use integer minor units (cents) to avoid floating-point rounding errors in financial calculations.

## Trade-Offs

| Decision | Option A | Option B | This Design Chooses |
|---|---|---|---|
| Slot occupancy | Full State pattern | Plain `bool` + guarded methods | `bool` — only two states, no payoff from the pattern |
| Ticket lifecycle | Plain `bool`/enum field + switch statements | Full State pattern | State pattern — three states, real per-state behavior differences |
| Allocation strategy | Single global mutex (simple, fully serialized) | Per-floor locks (faster, more complex) | Single mutex — simplest correct answer at stated scale |
| Slot/vehicle fit | 1:1 type alias (simpler, wrong in practice) | Ordered `SlotSize` + `Fits()` compatibility rule | Compatibility rule — matches how real lots work |

## How To Remember This

**"Composition owns, interfaces decide."** `ParkingLot` *owns* its `Floor`s and `Slot`s (composition — they don't outlive it); it *decides* how to allocate and price entirely through injected interfaces (`SlotAllocator`, `FeeCalculator`) it doesn't own. Any time you're unsure whether a relationship should be a struct field of a concrete type or an interface, ask: "does the parent own this thing's lifecycle, or does it just need this thing to do something?" Ownership → composition. Delegation → interface.

**For when State earns its keep:** *"Two states and a toggle, skip it. Three states and a terminal state, build it."*

## What Interviewers Actually Look For

| Signal | Green Flag | Red Flag |
|---|---|---|
| Entity modeling | Distinguishes `SlotSize` from `VehicleType` and defines a `Fits` rule | Hardcodes a 1:1 vehicle-to-slot mapping without questioning it |
| Pattern judgment | Explains *why* State fits Ticket but not Slot | Applies State (or any pattern) reflexively, everywhere |
| Concurrency awareness | Locks the check-and-claim sequence together, not just the write | Locks only `occupy()`, leaving a race between check and claim |
| Interface design | `SlotAllocator`/`FeeCalculator` are small, swappable, consumer-defined | One large `ParkingLotService` interface with everything on it |
| Extensibility | Names Observer/persistence/gates as clean extension points without over-building them now | Either ignores extensibility entirely, or builds every extension unprompted |

## Interview Cheat Sheet

```
Entities:    ParkingLot, Floor, Slot, Vehicle, Ticket, Money
Interfaces:  SlotAllocator { Allocate, Release }
             FeeCalculator { Calculate }

Relationships:
  ParkingLot ◆--> Floor ◆--> Slot     composition (owns, doesn't outlive parent)
  Ticket  ---->  Slot (by ID)         aggregation (slot exists independently)
  ParkingLot ---->  SlotAllocator     dependency via interface (injected)

State trigger rule:  3+ states AND per-state behavior differs  -> use State pattern
                      2 states, simple toggle                  -> bool + guarded methods is enough

Concurrency rule:    lock the check-AND-claim together, never just the claim

Pattern map:  Facade    -> ParkingLot's narrow public surface
              Strategy  -> FeeCalculator (swap pricing without touching callers)
              State     -> Ticket lifecycle (Active -> Paid -> Exited)
```

## References

- Refactoring.Guru — State Pattern: https://refactoring.guru/design-patterns/state
- Refactoring.Guru — Strategy Pattern: https://refactoring.guru/design-patterns/strategy
- `lld/interview-approach.md` — five-step LLD framework this case study applies
- `lld/patterns/behavioral/state.md`, `lld/patterns/behavioral/strategy.md`, `lld/patterns/structural/facade.md` — full pattern theory, referenced not re-derived