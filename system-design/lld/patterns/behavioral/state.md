# State Pattern

## Table of Contents

- [What It Is](#what-it-is)
- [Why It Exists](#why-it-exists)
- [Structure](#structure)
- [Go Implementation](#go-implementation)
- [State vs. Strategy](#state-vs-strategy)
- [Problems It Brings](#problems-it-brings)
- [When to Use It](#when-to-use-it)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [How to Remember It](#how-to-remember-it)
- [References](#references)

---

## What It Is

State is a **behavioural design pattern** that allows an object to alter its behaviour when its internal state changes. From the outside, the object appears to change its class.

It is a direct implementation of a **finite state machine** in object-oriented code. There is a fixed set of states; behaviour differs per state; events cause transitions between states.

---

## Why It Exists

Consider a vending machine. Without State:

```go
func (v *VendingMachine) InsertCoin() {
    switch v.state {
    case Idle:
        // accept coin
    case HasCoin:
        // error: coin already inserted
    case OutOfStock:
        // error: machine empty
    }
}

func (v *VendingMachine) Dispense() {
    switch v.state {
    case Idle:
        // error: no coin
    case HasCoin:
        // dispense item
    case OutOfStock:
        // error: machine empty
    }
}
```

Every method repeats the same switch. Every new state requires touching every method. The logic for a single state is scattered across the entire file.

State fixes this by giving each state its own type. The `HasCoin` state knows how to handle all events when a coin has been inserted, and it is the only place where that logic lives. Adding a new state means adding a new file — existing states are untouched.

---

## Why It's Called "State"

Named for exactly what it encapsulates: the **state** of an object, with the behaviour that belongs to that state attached directly to it. Each concrete state class is a citizen of the state machine, not just a constant in an enum.

---

## Structure

```
+-----------------------+          +--------------------+
|       Context         |--------->|   State            | <<interface>>
|-----------------------|          |--------------------|
|  - state State        |          |  + InsertCoin(ctx) |
|  + SetState(s State)  |          |  + Eject(ctx)      |
|  + InsertCoin()       |          |  + Dispense(ctx)   |
|  + Eject()            |          +--------------------+
|  + Dispense()         |                    ^
+-----------------------+          __________|__________
                                  |          |          |
                            +-----+---+ +----+----+ +---+--------+
                            | Idle    | |HasCoin  | |OutOfStock  |
                            +---------+ +---------+ +------------+
```

- **Context** — holds a reference to the current state object; delegates all event calls to it; exposes `SetState` so states can trigger transitions
- **State interface** — declares all events the machine must handle
- **Concrete states** — implement the interface with logic specific to that state; trigger transitions by calling `ctx.SetState(nextState)`

The key insight: **concrete state objects call `SetState` on the context to transition**. The context itself rarely initiates transitions — it just delegates.

---

## Go Implementation

```go
package state

import "fmt"

// State interface — all events the machine handles
type VendingState interface {
    InsertCoin(m *VendingMachine)
    Eject(m *VendingMachine)
    Dispense(m *VendingMachine)
}

// Context
type VendingMachine struct {
    state VendingState
    items int
}

func NewVendingMachine(items int) *VendingMachine {
    m := &VendingMachine{items: items}
    if items > 0 {
        m.state = &IdleState{}
    } else {
        m.state = &OutOfStockState{}
    }
    return m
}

func (m *VendingMachine) SetState(s VendingState) { m.state = s }
func (m *VendingMachine) InsertCoin()             { m.state.InsertCoin(m) }
func (m *VendingMachine) Eject()                  { m.state.Eject(m) }
func (m *VendingMachine) Dispense()               { m.state.Dispense(m) }

// IdleState — no coin inserted
type IdleState struct{}

func (s *IdleState) InsertCoin(m *VendingMachine) {
    fmt.Println("Coin inserted.")
    m.SetState(&HasCoinState{})
}
func (s *IdleState) Eject(m *VendingMachine)    { fmt.Println("No coin to eject.") }
func (s *IdleState) Dispense(m *VendingMachine) { fmt.Println("Insert a coin first.") }

// HasCoinState — coin inserted, ready to dispense
type HasCoinState struct{}

func (s *HasCoinState) InsertCoin(m *VendingMachine) { fmt.Println("Coin already inserted.") }
func (s *HasCoinState) Eject(m *VendingMachine) {
    fmt.Println("Coin returned.")
    m.SetState(&IdleState{})
}
func (s *HasCoinState) Dispense(m *VendingMachine) {
    m.items--
    fmt.Println("Item dispensed.")
    if m.items == 0 {
        m.SetState(&OutOfStockState{})
    } else {
        m.SetState(&IdleState{})
    }
}

// OutOfStockState — no items remaining
type OutOfStockState struct{}

func (s *OutOfStockState) InsertCoin(m *VendingMachine) {
    fmt.Println("Machine is out of stock. Coin not accepted.")
}
func (s *OutOfStockState) Eject(m *VendingMachine)    { fmt.Println("Machine is out of stock.") }
func (s *OutOfStockState) Dispense(m *VendingMachine) { fmt.Println("Machine is out of stock.") }
```

**Usage:**

```go
m := NewVendingMachine(2)
m.InsertCoin()  // Coin inserted.
m.Dispense()    // Item dispensed. (1 item left, transitions to Idle)
m.InsertCoin()  // Coin inserted.
m.Dispense()    // Item dispensed. (0 items left, transitions to OutOfStock)
m.InsertCoin()  // Machine is out of stock. Coin not accepted.
```

### Go Notes

- Go's implicit interface satisfaction means `IdleState` satisfies `VendingState` without declaring it — idiomatic and clean
- Passing `m *VendingMachine` into each method (rather than storing a back-reference in the state) avoids a circular dependency and keeps state objects stateless and reusable
- There is no inheritance; the pattern is achieved purely through composition and interface delegation

---

## State vs. Strategy

These two patterns are structurally identical: a context holds an interface reference and delegates method calls to it. The distinction is **intent and direction of control**:

| Dimension | State | Strategy |
|---|---|---|
| Who changes the implementation? | The state objects themselves (self-transition) | The caller / client |
| Do implementations know about each other? | Yes — `HasCoinState` knows `IdleState` exists | No — strategies are independent |
| Does the implementation change at runtime autonomously? | Yes — driven by events | No — explicitly swapped by caller |
| Analogy | FSM where transitions happen automatically | Pluggable algorithm chosen by the user |

A simple test: if the object changes its own behaviour in response to internal events, it is State. If an external caller decides which algorithm to use, it is Strategy.

---

## Problems It Brings

- **Class explosion** — N states × M events → many files. For simple machines (2–3 states), a plain switch is cleaner and more readable
- **Inter-state coupling** — concrete state classes import each other to perform transitions; changing one state can affect others
- **Scattered transition logic** — the full picture of the state machine is distributed across classes rather than visible in a single transition table; harder to audit
- **Increased indirection** — straightforward logic like "if state == X do Y" becomes multiple method calls through two layers of indirection

---

## When to Use It

Use it when:

- An object's behaviour changes substantially based on its state, and there are many states
- You have repeated conditional branches checking the same state variable across multiple methods
- Transitions have non-trivial logic (validation, side effects) beyond just `state = next`
- The set of states is bounded and known upfront (true FSM)

Avoid it when the state machine is trivial — two or three states with simple transitions. A plain enum and switch table is far easier to read and reason about.

---

## Interview Cheat Sheet

**Strong signal phrases:**

- "The State pattern is a direct OO encoding of a finite state machine."
- "Each concrete state owns the logic for its transitions; the context just delegates."
- "State looks identical to Strategy structurally, but in State the objects transition themselves, whereas in Strategy the caller swaps the implementation."
- "Passing the context into each state method avoids storing a back-reference and keeps state objects stateless and reusable."
- "For a simple FSM with 2–3 states, I'd prefer a switch over the State pattern — the indirection isn't worth it."

**Red flags to avoid:**

- Confusing State and Strategy — be able to articulate the structural similarity and the intent difference precisely
- Storing context state in the state objects themselves (makes them non-reusable and introduces subtle bugs)
- Using State when a simple enum + switch would be clearer
- Forgetting that the context must expose `SetState` (or equivalent) so state objects can drive transitions

**Common interview scenarios:**

- Vending machine, ATM, traffic light, order lifecycle (created → paid → shipped → delivered → returned)
- TCP connection (Closed → Listen → SYN-Received → Established → …)
- Document workflow (Draft → Review → Approved → Published)

---

## How to Remember It

**State = FSM as objects.** Each state is a citizen, not a constant. When the machine transitions, it swaps citizens.

The switch-statement smell is the tell: if you see the same state variable being checked in multiple methods with the same case structure, the State pattern is the refactoring.

**Mnemonic:** STATES → **S**wap **T**he **A**ctual **T**ype to **E**xpress **S**tate.

---

## References

- [Refactoring.Guru — State Pattern](https://refactoring.guru/design-patterns/state)
- [GoF — Design Patterns: Elements of Reusable Object-Oriented Software](https://en.wikipedia.org/wiki/Design_Patterns) (Gamma et al., 1994)
- [Go Patterns — State](https://golangbyexample.com/state-design-pattern-go/)