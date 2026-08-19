---
Status: 🌳 Evergreen
Created: 2026-08-18
Last Updated: 2026-08-19
---

# Elevator System

## Table Of Contents

1. [Problem Statement](#problem-statement)
2. [Requirements](#requirements)
3. [Core Entities](#core-entities)
4. [Class Design](#class-design)
5. [Implementation](#implementation)
6. [Verification Walkthrough](#verification-walkthrough)
7. [Trade-Offs](#trade-offs)
8. [Extensions](#extensions)
9. [References](#references)

---

## Problem Statement

An elevator system moves people between the floors of a building. A single elevator is a state machine — it is Idle, or it is moving Up or Down toward a set of pending stops. A building with multiple elevators adds a second concern that has nothing to do with any individual car: when a passenger presses an Up button on the third floor, *which* car should answer? That split — one type modeling the car's motion, another type modeling how the fleet is coordinated — is the design insight the whole problem is built around. Miss it and the code turns into a single class that knows everything; catch it and every extension (add cars, swap the scheduling algorithm, add destination dispatch) lands in exactly one place.

---

## Requirements

*~5 minutes. The interviewer says "Design an elevator system." Your job is to pin down scope with questions that each earn the right to introduce a specific design decision later.*

**Questions I'd ask, and why I'm asking each one:**

*"One elevator or multiple?"*
Surface purpose: scope. Real purpose: this is the fork between a single state-machine problem and a scheduling problem. If it's multiple — which is the interesting case — I've earned the right to introduce a `Controller` type that sits above the cars and a `DispatchStrategy` interface for how it picks between them. If it's one, the controller collapses to a queue and Strategy is pattern-dropping.

*"Where do requests come from — floor buttons outside the elevator, floor buttons inside, or both?"*
Surface purpose: input model. Real purpose: two request shapes fall out of this. A **hall call** carries a floor *and* a direction (the passenger pressed Up on floor 3). A **car call** carries only a destination (the passenger inside car #2 pressed 7). Same interface, different data — that's Command. And the direction on hall calls is what makes senior-level dispatch (LOOK / SCAN) possible; without direction, all we can do is FCFS or nearest-car.

*"How does time work in this system — real time, or a `tick()` model where I advance the simulation one step at a time?"*
Surface purpose: clarify the execution model. Real purpose: earning permission to skip concurrency. A `tick()` abstraction — one call = one time step, the elevator moves one floor — lets me focus the interview on state and scheduling instead of goroutines and mutexes. In production the loop runs on real time and each car is its own actor; in a 45-minute interview `tick()` is the honest simplification. Naming this upfront prevents the interviewer thinking I forgot about concurrency.

*"What scheduling behaviour is expected — first-come-first-serve, nearest car, or something smarter?"*
Surface purpose: pick an algorithm. Real purpose: earning the Strategy pattern. If more than one candidate algorithm is on the table — and there always is — Strategy is the right shape. I'll default to a LOOK-style policy (continue in the current direction, service every stop along the way, only reverse when there's nothing left ahead) because it's what real elevators do and it's the answer that signals I've thought beyond nearest-car.

*"What if a hall call comes in for a floor the elevator is already passing in the wrong direction?"*
Surface purpose: edge case. Real purpose: earning the direction-aware dispatch scoring. This is the question that reveals whether the candidate knows that a car moving Up past floor 5 with a pending stop at 8 should *not* answer a Down hall call at floor 4 immediately — it should finish the up-sweep first. If the interviewer engages, LOOK is the natural answer. If they say "just take it," it's a signal that FCFS-with-nearest is enough.

*"What's explicitly out of scope?"*
Don't wait to be told — offer it. *"Hardware drivers (motors, sensors, door mechanisms), fire-service override, maintenance mode, energy optimization, and true concurrent execution are all out. I'll model time with `tick()` and treat door open/close as instantaneous. Does that scope work?"* Offering the boundary yourself signals you know what a 45-minute design can and can't cover.

**Final scope:**

| In | Out |
|---|---|
| Multiple elevators in one building | Hardware drivers (motors, sensors, doors) |
| Hall calls (floor + direction) | Fire-service override, maintenance mode |
| Car calls (destination floor) | Real-time concurrency (use `tick()` instead) |
| LOOK-style dispatch, pluggable via strategy | Destination dispatch (hall-side floor entry) |
| Per-car state machine: Idle / MovingUp / MovingDown | Energy optimization, zone assignment |
| Per-car capacity limit | Real-world door-timing, boarding delays |
| Emergency stop per car | Multi-building coordination |

---

## Core Entities

*~5 minutes. Sketch the structure before writing code — communicates intent and catches modeling mistakes early.*

What I'd put on the whiteboard:

```
Concrete types             Interfaces (behaviour that varies)
─────────────────          ────────────────────────────────
Building                   LiftState         ← per-state motion
Elevator                   Request           ← Command
Controller                 DispatchStrategy  ← scheduling algorithm
HallCall
CarCall
```

Ownership and dependencies — what I'd draw as arrows:

```
Building    owns    Controller, []*Elevator      (constructed at boot)
Controller  uses    DispatchStrategy             (injected)
Controller  refs    []*Elevator                  (routes calls to them)
Elevator    has-a   LiftState                    (State — swaps itself)
Elevator    owns    upStops, downStops           (pending destinations)
Request     is-a    HallCall | CarCall           (Command)
```

> **owns vs uses:** *owns* = this type is what calls `&Controller{...}`; the object didn't exist before and this type is responsible for its lifecycle. *uses* = the object was created elsewhere and handed in via constructor; the holder just keeps a pointer.

**The key insight to say out loud:**

*"A single elevator is a state machine. Multiple elevators are a scheduling problem. Those are two different concerns and they belong in two different types — `Elevator` and `Controller`. The Elevator has no idea other elevators exist; the Controller has no idea how an elevator physically moves."*

Everything else — Strategy for dispatch, Command for request shapes, State for motion — falls out of that split cleanly. Miss it and the code turns into one god-class where scheduling logic sits next to floor-tick arithmetic and every extension has to touch both.

---

## Class Design

*~10 minutes. Per-class Go stubs — state fields plus method signatures. Reasoning stated inline at the point of each decision, not batched into a separate justifications section.*

### `LiftState` — The Spine

Per-state motion behaviour. The Elevator holds a `LiftState` and calls `Tick(e)` on it once per time step; the state decides where the car moves and, when appropriate, swaps itself to a different state. This is textbook State pattern — the alternative is a nested `switch` inside `Elevator.Tick()` that grows a case for every new state (arriving, doors open, maintenance) and turns unreadable by the third one.

```go
type LiftState interface {
    Tick(e *Elevator)
    Name() string
}

// State structs are unexported — only Elevator constructs them, and callers
// interact through the LiftState interface. Go's convention for closed sets
// of implementations.
type idleState struct{}
type movingUpState struct{}
type movingDownState struct{}
```

Why the state structs carry no fields: all mutable data (current floor, pending stops) lives on `Elevator`. The state is *behaviour*, not data — so shared singletons would work, but per-tick allocation is cheap and the interface stays uniform. I'd mention this trade-off aloud rather than optimize it.

### `Elevator` — The State Machine

Owns its current floor, its pending stops (split into "above me" and "below me"), and its current state. It exposes `Tick()` (drive one time step) and `AddStop(floor)` (add a destination — used by the Controller when routing a call to this car).

```go
type Elevator struct {
    id           int
    currentFloor int
    state        LiftState
    upStops      map[int]bool  // floors above currentFloor with pending stops
    downStops    map[int]bool  // floors below currentFloor with pending stops
    capacity     int           // max passengers
    load         int           // current passenger count
}

func NewElevator(id, startFloor, capacity int) *Elevator { /* ... */ }
func (e *Elevator) Tick()                                 { /* delegates to state */ }
func (e *Elevator) AddStop(floor int)                     { /* routes to up/down bucket */ }
func (e *Elevator) CurrentFloor() int                     { return e.currentFloor }
func (e *Elevator) State() string                         { return e.state.Name() }
```

Why two stop buckets instead of one `map[int]bool`: it makes the LOOK reversal logic — "no more stops in my current direction, so flip" — a one-line check per tick. With a single set you'd re-scan every entry on every tick to decide whether to reverse. The redundancy is bounded (each floor appears in at most one bucket at a time) and the readability win is large.

### `Request` — The Command

The Controller's public surface receives one of two request shapes. Rather than two unrelated methods (`HandleHallCall`, `HandleCarCall`) with duplicated dispatch plumbing, I'd model them as a common interface — same trick as Transaction in the ATM.

```go
type Request interface {
    Execute(c *Controller) error
}

type Direction int
const (
    DirNone Direction = iota
    DirUp
    DirDown
)

type HallCall struct {
    Floor     int
    Direction Direction // Up or Down — the button pressed outside the car
}

type CarCall struct {
    ElevatorID int
    Floor      int // destination pressed inside a specific car
}

func (h HallCall) Execute(c *Controller) error  { /* pick car via strategy */ }
func (cc CarCall) Execute(c *Controller) error  { /* route to specific car */ }
```

Why this earns its keep: hall and car calls flow through the same audit queue, the same rate-limit hook, the same logging. Add a `PriorityHallCall` for accessibility later — one new type, no plumbing changes.

### `DispatchStrategy` — Which Car Answers

The scoring function that turns *"a hall call arrived at floor 5 going Up"* into *"give it to car #2."* Interface-first because there is more than one reasonable algorithm and the interviewer will ask about the others.

```go
type DispatchStrategy interface {
    Choose(cars []*Elevator, hc HallCall) *Elevator
}

// LOOK: prefer cars already moving in the call's direction and positioned
// to reach the call floor on their current sweep. Penalize (but don't
// exclude) cars that would have to reverse or finish another sweep first.
type LookStrategy struct{}

// Alternatives to mention aloud, not implement:
//   FCFSStrategy      — hand to whichever car has been idle longest
//   NearestCarStrategy — pure |floor - currentFloor|, ignores direction
//   ZoneStrategy      — each car owns a floor range
```

Why LOOK is the default: it's what real elevators do, and the direction-aware scoring is the piece an interviewer waits to see. Naming NearestCar as the "obvious wrong answer" — the one that thrashes in busy buildings — is a chance to demonstrate depth without writing extra code.

### `Controller` — The Coordinator

Holds the fleet, holds the strategy, receives every external request, drives every tick. The Elevator has no reference to the Controller and no reference to sibling cars — this is the boundary that keeps the two concerns separated.

```go
type Controller struct {
    cars     []*Elevator
    strategy DispatchStrategy
}

func NewController(cars []*Elevator, s DispatchStrategy) *Controller { /* ... */ }
func (c *Controller) Submit(r Request) error                          { return r.Execute(c) }
func (c *Controller) Tick()                                           { /* tick every car */ }
```

Why `Submit(Request)` instead of typed methods: it's the single entry point for the Command interface to pay off. All calls — hall, car, and any future request type — funnel through one method that can add cross-cutting concerns (metrics, throttling, request logging) in one place. In an interview I'd verbalise: *"In production I'd wrap this with a queue and a per-request span; here I'll keep it synchronous."*

> [!NOTE]
> **What "queue" and "per-request span" mean.** *Queue*: `Submit` becomes a two-step handoff — shove the request onto a buffered channel and return immediately; a background goroutine pulls from the channel and executes. Wins: backpressure under bursty load, one-at-a-time ordering (no races), throttling, panic isolation. Cost: goroutine lifecycle, shutdown semantics, and `Submit` can no longer return an execution error since it returns before execution. *Per-request span*: distributed-tracing vocabulary. Every incoming request gets a unique trace ID; each method that touches it records a "span" (start time, end time, tagged attributes) via OpenTelemetry or similar. All spans stitch into a timeline in tools like Datadog / Honeycomb / Jaeger, so when a passenger complains "the elevator took forever at 10 AM," you can pull up that exact hall call's trace and see whether time went into dispatch, sweep, or the reversal. Both are named aloud in the interview and not coded — the point is signalling that you know sync-in-memory is a scoped-down version of a production shape you've built before.

### `Building` — The Boot Assembly

The outermost type that wires everything together at construction time. Interviewers often skip this, but naming it explicitly is a five-second signal that you know where dependency injection lives.

```go
type Building struct {
    controller *Controller
    numFloors  int
}
```

---

## Implementation

*~15 minutes. Two methods carry the interview: `Elevator.Tick` via the moving states (SCAN reversal is the meat), and `LookStrategy.Choose` (direction-aware scoring is the second meat). I'd code these two and describe the rest aloud.*

Errors here are inline strings, not named sentinels. I'd mention verbally that production code would define `ErrInvalidFloor`, `ErrNoCarAvailable` etc. — that ceremony doesn't earn its place in a 45-minute interview.

> [!NOTE]
> **Runtime model — who calls `Tick` and how the system bootstraps.** Nothing inside the system schedules itself. An outer driver (`main`, a test, or in a real building the firmware's timer-interrupt handler) constructs the fleet, wires the `Controller`, and then drives two kinds of calls: `controller.Submit(req)` when a passenger presses a button, and `controller.Tick()` to advance the simulation one time step. `Submit` only *records* a request (adds a floor to some car's stop set) — it never moves anything. All motion happens inside `Tick`, which delegates to each car's `state.Tick(elevator)`. Between two `Tick` calls the world is frozen; requests that arrive in that gap just appear in the next tick's stop sets. Bootstrap chain: `main → []Elevator + Strategy → Controller → for loop calling Tick`. Skeleton:
>
> ```go
> cars := []*Elevator{NewElevator(1, 1, 10), NewElevator(2, 5, 10)}
> controller := NewController(cars, LookStrategy{})
> controller.Submit(HallCall{Floor: 3, Direction: DirUp})
> for t := 0; t < 10; t++ {
>     controller.Tick()  // one call = one time step
> }
> ```
>
> This is the same shape as game loops (`processInput` then `updatePhysics(dt)`) and React (state updates queue, render commits them) — events accumulate, ticks execute. Wins: no goroutines, no mutexes, deterministic tests. To swap in real time, replace the `for` loop with `time.Tick(1 * time.Second)`; the types don't change.

> [!NOTE]
> **Sweep, SCAN, LOOK.** A **sweep** is one continuous run in a single direction: the car keeps moving up (or down), stopping at every requested floor along the way, until the direction's stop set is empty. Then it reverses (or goes idle). The word comes from disk-scheduling — the disk head *sweeps* across the platter. **SCAN** always sweeps to the physical end (top or bottom floor) before reversing, even if there are no stops left ahead — like a windshield wiper that always hits the edge. **LOOK** reverses the moment there are no more stops in the current direction, saving the extra travel. Our `movingUpState.Tick` is LOOK: it flips to `movingDownState` as soon as `upStops` is empty, not when it hits the top floor. How to remember: sweep = broom (one direction, pick up everything in the path); SCAN = full scan of the range; LOOK = look ahead first, only go as far as needed.

### `Elevator.Tick` — Delegating To State

```go
func (e *Elevator) Tick() {
    e.state.Tick(e)
}

func (e *Elevator) AddStop(floor int) {
    if floor == e.currentFloor {
        return // already here — doors would open, out of scope
    }
    if floor > e.currentFloor {
        e.upStops[floor] = true
    } else {
        e.downStops[floor] = true
    }
}
```

`Elevator.Tick` is one line because the interesting work has been pushed into the state. That's the payoff of State — the coordinator (`Elevator`) doesn't grow a switch every time a new state is added.

### The Three States — Where SCAN Lives

```go
func (idleState) Name() string { return "Idle" }
func (idleState) Tick(e *Elevator) {
    // Idle just picks a direction if any work exists.
    // Prefer up first — arbitrary tie-break, mention aloud that a real
    // controller might use last-direction or a fairness policy.
    if len(e.upStops) > 0 {
        e.state = movingUpState{}
    } else if len(e.downStops) > 0 {
        e.state = movingDownState{}
    }
}

func (movingUpState) Name() string { return "MovingUp" }
func (movingUpState) Tick(e *Elevator) {
    e.currentFloor++
    if e.upStops[e.currentFloor] {
        delete(e.upStops, e.currentFloor)
        // Doors would open here; treated as instantaneous per scope.
    }
    // SCAN reversal: finish the up-sweep, then flip only if work exists below.
    if len(e.upStops) == 0 {
        if len(e.downStops) > 0 {
            e.state = movingDownState{}
        } else {
            e.state = idleState{}
        }
    }
}

func (movingDownState) Name() string { return "MovingDown" }
func (movingDownState) Tick(e *Elevator) {
    e.currentFloor--
    if e.downStops[e.currentFloor] {
        delete(e.downStops, e.currentFloor)
    }
    if len(e.downStops) == 0 {
        if len(e.upStops) > 0 {
            e.state = movingUpState{}
        } else {
            e.state = idleState{}
        }
    }
}
```

The insight to say aloud on `movingUpState.Tick`: *"The reversal check happens **after** removing a stop, not before. If we're at floor 8 going Up and 8 is our last up-stop, we serve floor 8 first, then this tick decides whether to reverse. That's the semantic difference between LOOK and 'nearest-first' — LOOK never skips a stop it's already at."*

### `LookStrategy.Choose` — Direction-Aware Scoring

```go
func (LookStrategy) Choose(cars []*Elevator, hc HallCall) *Elevator {
    var best *Elevator
    bestScore := math.MaxInt
    for _, c := range cars {
        score := scoreCar(c, hc)
        if score < bestScore {
            bestScore = score
            best = c
        }
    }
    return best
}

func scoreCar(c *Elevator, hc HallCall) int {
    dist := abs(c.currentFloor - hc.Floor)
    switch c.state.(type) {
    case idleState:
        // Idle cars are always fair game; score by raw distance.
        return dist
    case movingUpState:
        // Only "on the way" if we're going up AND the call is above us AND
        // the passenger wants to go up (same direction as our sweep).
        if hc.Direction == DirUp && hc.Floor >= c.currentFloor {
            return dist
        }
        // Otherwise this car must finish its up-sweep and reverse before
        // it can serve the call. Penalize heavily but don't exclude — if
        // every other car is also busy, this is still an answer.
        return dist + 1000
    case movingDownState:
        if hc.Direction == DirDown && hc.Floor <= c.currentFloor {
            return dist
        }
        return dist + 1000
    }
    return math.MaxInt
}

func abs(x int) int { if x < 0 { return -x }; return x }
```

The `+1000` penalty is a magic number and I'd flag it aloud: *"In production I'd model this as expected-arrival-time by simulating the car's remaining sweep — that's the honest cost function. Here I'm using a constant to keep the code readable; the direction check is what matters."*

> [!NOTE]
> **`switch x.(type)` — type switch.** Go's way of asking *"what concrete type is behind this interface value?"* Normal `switch x` matches values; `switch x.(type)` matches the runtime type of `x`. Each `case` names a concrete type — the matching case's body runs when the interface holds that type. Bind the value with `switch s := x.(type)` if you need to read fields on the concrete type. Used here because scoring is a *dispatcher* concern, not a *state* concern — pushing scoring onto `LiftState` as a method would bleed scheduler logic into the states. Downside: not exhaustive — adding a fourth state (e.g. `maintenanceState`) without updating this switch silently falls through to `math.MaxInt`. Sibling operation: outside a switch, `v, ok := x.(T)` asserts a single specific type. Both belong to the family Go calls *type assertions*.

### `HallCall.Execute` — Wiring The Command To The Strategy

```go
func (h HallCall) Execute(c *Controller) error {
    if h.Direction == DirNone {
        return fmt.Errorf("hall call requires a direction")
    }
    car := c.strategy.Choose(c.cars, h)
    if car == nil {
        return fmt.Errorf("no car available")
    }
    car.AddStop(h.Floor)
    return nil
}

func (cc CarCall) Execute(c *Controller) error {
    for _, car := range c.cars {
        if car.id == cc.ElevatorID {
            car.AddStop(cc.Floor)
            return nil
        }
    }
    return fmt.Errorf("unknown elevator %d", cc.ElevatorID)
}
```

Hall calls flow through the strategy; car calls skip it — the passenger is already in a specific car and the destination is that car's problem. Naming this asymmetry aloud is another five-second signal.

### `Controller.Tick`

```go
func (c *Controller) Tick() {
    for _, car := range c.cars {
        car.Tick()
    }
}
```

One line. All the intelligence lives inside each car's state — this is the payoff of the split. If the interviewer asks about concurrency, this is the seam: replace the loop with per-car goroutines and add a channel for `Submit`. The types don't change.

---

## Verification Walkthrough

*~5 minutes. Talk through a concrete scenario to prove the design behaves. This is the moment to close the loop — the interviewer wants to see that your code and your explanation agree.*

**Scenario:** Two cars in a 10-floor building. Car A at floor 1, idle. Car B at floor 5, going up with a pending stop at 8. A passenger presses Up on floor 3.

**Setup:**
```
Car A: floor=1, state=Idle,     upStops={},   downStops={}
Car B: floor=5, state=MovingUp, upStops={8},  downStops={}
```

**t=0 — Hall call arrives: `HallCall{Floor: 3, Direction: Up}`.**

The Controller runs `LookStrategy.Choose`:
- Car A (Idle, at floor 1): score = |1 - 3| = **2**
- Car B (MovingUp, at floor 5, call is Up but floor 3 is *below* B's current position): the "on the way" check fails (`hc.Floor >= c.currentFloor` is `3 >= 5` — false). Penalized: |5 - 3| + 1000 = **1002**.

Car A wins. `car.AddStop(3)` — since 3 > 1, it lands in `A.upStops`. Now:
```
Car A: floor=1, state=Idle,     upStops={3}, downStops={}
Car B: floor=5, state=MovingUp, upStops={8}, downStops={}
```

**t=1 — Both cars tick.**

Car A is Idle with `len(upStops) > 0` → transitions to MovingUp. It does *not* move this tick — the state change happens without floor advancement, matching the "idle wakes up" phase. (Design note to say aloud: some designs would advance a floor on wake-up too; I'm being explicit that the idle→moving transition is a bookkeeping step, not a physical move. Either is defensible.)

Car B ticks in MovingUp: `currentFloor` becomes 6. 6 is not in `upStops`, no delete. `upStops` still has {8}, so no reversal.
```
Car A: floor=1, state=MovingUp, upStops={3}, downStops={}
Car B: floor=6, state=MovingUp, upStops={8}, downStops={}
```

**t=2 — Both tick.**

Car A: `currentFloor` becomes 2. 2 not in stops. `upStops` still has {3}, no reversal.
Car B: `currentFloor` becomes 7. 7 not in stops. `upStops` still has {8}, no reversal.

**t=3 — Both tick.**

Car A: `currentFloor` becomes 3. 3 **is** in `upStops` → delete it. Now `upStops` is empty and `downStops` is empty → transition to Idle. Doors open (out of scope) and the passenger boards.

Car B: `currentFloor` becomes 8. 8 **is** in `upStops` → delete. Both stop sets empty → Idle. Doors open at 8 for whoever pressed that.

```
Car A: floor=3, state=Idle, upStops={}, downStops={}
Car B: floor=8, state=Idle, upStops={}, downStops={}
```

The system settles cleanly. The passenger on floor 3 waited 3 ticks; if we'd sent Car B (the closer car by raw distance), they would have waited for B to finish serving floor 8 first, then reverse and come back — that's the specific pathology LOOK prevents.

---

## Trade-Offs

*Proactive flags — say these before the interviewer asks. The signal is that you know your own design's limits.*

**`tick()` is not real time.** All timing collapses to one-tick-per-floor. In production, cars move at real speed, doors take real seconds, boarding takes variable time. My design happens to work with a real clock — replace the loop in `Controller.Tick` with per-car goroutines and a ticker — but the reasoning I demonstrated is on the state and scheduling, not the timing model.

**The `+1000` penalty is a magic number.** A proper cost function simulates the car's expected time-to-arrival: remaining stops in the current sweep, plus the reversal, plus the distance from the top/bottom of the sweep to the call floor. That's more code than an interview affords. Naming the shortcut aloud is what matters.

**Ties are broken by iteration order.** If two cars score identically, the first in `c.cars` wins — non-deterministic if the slice is reordered. A production dispatcher would tie-break on load (fewer pending stops), then on car ID for stability.

**No starvation protection.** A LOOK dispatcher plus a busy sweep upward can leave a Down call at floor 2 waiting indefinitely if new Up calls keep arriving. The classic fix is aging — after N ticks, promote the request's score. Worth naming even without implementing.

**Passenger load isn't enforced during dispatch.** `Elevator.load` and `capacity` exist as fields but `AddStop` doesn't check whether the car is full. A dispatch that respects load would exclude cars where `load >= capacity` from `Choose` — a five-line addition, worth mentioning.

**Idle wake-up doesn't move this tick.** A passenger who calls an idle car from an adjacent floor waits one tick longer than they would if the state transition also advanced a floor. Cosmetic in a simulation, but a real interviewer with a physical elevator background might notice.

**No persistence.** The stop sets and current floor live in memory. Power cycle and the fleet forgets what it was doing. Real controllers persist state to durable storage per motion cycle — mention as an extension.

---

## Extensions

*~5 minutes. Brief verbal answers to follow-up questions the interviewer is likely to fire in the last five minutes.*

**"How would you add destination dispatch — passengers enter their destination floor at a kiosk before boarding?"**
The `HallCall` shape grows a `Destination` field. `LookStrategy.Choose` gets a richer cost function that can group passengers with common destinations onto the same car. The Elevator's state machine doesn't change at all — this is why the split into two concerns pays off.

**"How would you handle a car breaking down mid-sweep?"**
Add a `maintenanceState` to `LiftState` — a car in maintenance always returns `math.MaxInt` from `scoreCar`, so the dispatcher never picks it. Its pending stops need reassignment: expose an `Evacuate()` method on Elevator that returns its remaining stops as Requests, and have the Controller re-submit each one.

**"Multiple buildings with connected elevator banks?"**
`Controller` per building stays as is; a `Federation` type sits above and holds a strategy that chooses which building's controller to route an inter-building request to. The Elevator type doesn't know Federation exists — same clean layering as adding cars to a single building.

**"How would you notify a display panel on each floor when the elevator arrives?"**
Observer. The Elevator publishes a `FloorArrived(carID, floor)` event on every tick where a stop was served; floor displays subscribe. In Go I'd model this as a channel-per-subscriber pattern, mention it aloud, skip the code.

**"How would you add emergency stop?"**
An `EmergencyStop(carID)` method on the Controller that transitions the target car to a new `stoppedState` and clears both stop sets. The state's `Tick` is a no-op. Reset requires an admin `Reset(carID)` call — never automatic.

**"Priority for accessibility calls?"**
`PriorityHallCall` — a new `Request` type. `Execute` routes through a priority-aware strategy that biases toward faster arrival (e.g. reduces the reversal penalty). One new command, one new strategy, no changes to Elevator. Command + Strategy earning their keep together.

---

## References

- [`system-design/lld/patterns/state.md`](./patterns/state.md) — State pattern deep-dive
- [`system-design/lld/patterns/strategy.md`](./patterns/strategy.md) — Strategy pattern deep-dive (LOOK vs FCFS vs NearestCar is a clean case)
- [`system-design/lld/patterns/command.md`](./patterns/command.md) — Command pattern (HallCall / CarCall share this shape with ATM's Transaction)
- [`system-design/lld/design-atm.md`](./design-atm.md) — ATM machine LLD (compare Command usage, injected boundary, State-as-spine)
- [`system-design/lld/design-parking-lot.md`](./design-parking-lot.md) — Parking Lot LLD (compare Strategy usage on parking allocation)