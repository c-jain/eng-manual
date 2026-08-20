---
Status: 🌳 Evergreen
Created: 2026-08-20
Last Updated: 2026-08-20
---

# Vending Machine

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

A vending machine is a self-service dispenser: the customer inserts money, selects a product by slot code, and the machine dispenses the product plus any change. It's superficially simple and often dismissed as a toy problem — the reason it survives as an interview staple is that the states are *physically real*. The machine is literally in one mode at a time and each mode allows a different set of inputs: Idle accepts coins but ignores product selection; HasMoney accepts more coins or a selection; Dispensing ignores everything until the mechanical action finishes. That isn't a modeling choice imposed on the design — it's how the hardware actually behaves, which makes it the cleanest possible motivation for the State pattern.

---

## Requirements

*~5 minutes. The interviewer gives you a one-line prompt: "Design a vending machine." Ask the questions that earn the design decisions.*

**Questions I'd ask, and why I'm asking each one:**

*"What's the purchase flow — money first then select, or select first then pay?"*
Surface purpose: clarify UX. Real purpose: pin down the state machine before drawing it. Money-first is the classical mechanical design (older machines can't reserve a product without collecting payment first) and it maps cleanly to Idle → HasMoney → Dispensing. Select-first requires a "ProductSelected" state that waits for enough money, which loops until sufficient — legal but adds an edge case. I'll ship money-first because it matches the physical machines most interviewers picture, and I'll flag the alternative aloud.

*"Does the machine track products *and* coins for change?"*
Surface purpose: scope. Real purpose: earning two distinct inventory subsystems. If the answer is "yes, both" — which it always is for a real machine — then `ProductInventory` and `CoinInventory` become first-class entities with their own atomic operations. Conflating them is the classic modeling mistake: the coin count and the chocolate-bar count have nothing in common except that both decrement. Naming both up front prevents the interviewer from having to prompt me for the second one.

*"Can the customer request a specific bill breakdown, or does the machine pick the change denominations?"*
Surface purpose: clarify UX. Real purpose: earning `ChangeStrategy` as an interface. If the machine picks, some algorithm decides which coins to dispense — that's Strategy, and the greedy-vs-DP conversation is a natural follow-up worth showing depth on. If the customer specifies, the strategy is trivial and I'd skip it.

*"Can the customer buy multiple items with a single insertion?"*
Surface purpose: scope. Real purpose: state machine simplicity. If yes, HasMoney becomes a loop with a "buy more?" branch after Dispensing. If no (standard), one purchase per money-insert cycle keeps the state graph a straight line. I'll assume no.

*"What if the machine can't make exact change — take a loss or refuse the sale?"*
Surface purpose: edge case. Real purpose: define the failure path. Modern machines refuse the sale and refund the exact coins the customer inserted. That earns me two things: an "escrow" concept (inserted coins are held separately until commit) and a clean rollback path in Dispense. Deciding this before writing code means my `doDispense` doesn't get retrofitted with a refund branch as an afterthought.

*"Would you add new modes later like maintenance or service?"*
Surface purpose: scope. Real purpose: cementing that State is the right choice rather than a `switch` on an enum. If yes, adding `MaintenanceState` is one new struct implementing `VMState`; with a switch, every operation has to grow a new case. I want the interviewer to agree the extension will happen before I invest whiteboard time in the State setup.

*"What's explicitly out of scope?"*
Don't wait to be told — offer it. *"Card and mobile payments, multi-item purchases, restocking workflows, remote telemetry, product expiry, and physical hardware faults (jammed dispenser, stuck coin, torn note) are all out. I'll treat both inventories as in-memory. Does that scope work?"*

**Final scope:**

| In | Out |
|---|---|
| Money-first purchase flow (Idle → HasMoney → Dispensing → Idle) | Card / mobile / digital payments |
| Coin and note insertion (fixed denominations) | Multi-item purchases in one session |
| Product selection by slot code (A1, B2, …) | Restocking workflow / admin UI |
| Change dispensing (greedy strategy) | Remote telemetry, IoT reporting |
| Cancel / refund of inserted coins | Product expiry dates |
| Out-of-stock and no-change failure paths (with refund) | Physical hardware faults (jams, stuck coins) |
| Two in-memory inventories: products and coins | Multi-currency / dynamic pricing |

---

## Core Entities

*~5 minutes. Sketch structure before writing code. Communicates intent and catches modeling mistakes early.*

What I'd put on the whiteboard:

```
Concrete types             Interfaces (behaviour that varies)
─────────────────          ────────────────────────────────
VendingMachine             VMState           ← per-state behaviour
Product                    ChangeStrategy    ← change-making algorithm
Slot
ProductInventory
CoinInventory
Session
```

Ownership and dependencies — what I'd draw as arrows:

```
VendingMachine  owns   Session                        (I create it — newSession() on first coin)
VendingMachine  uses   ProductInventory, CoinInventory,
                       ChangeStrategy                  (injected — built outside, handed in)
VendingMachine  has-a  VMState                         (State — swaps itself)
Session         holds  Escrow[Denomination]int,
                       SelectedSlot                    (per-purchase scratch space)
ProductInventory owns  map[SlotCode]*Slot              (creates Slots on Load)
CoinInventory    owns  map[Denomination]int
```

> **owns vs uses:** *owns* = this type is what constructs the object; the object didn't exist before and this type is responsible for it. *uses* = the object was built elsewhere and passed in via constructor; the vending machine just holds a pointer.

**The key insight to say out loud:**

*"The most important modeling decision is that there are **two** inventories, not one. `ProductInventory` tracks how many cans of Cola are behind slot A1; `CoinInventory` tracks how many ₹5 coins are in the change reservoir. These decrement on different events — product on a successful sale, coins on both change-out and cancel-refund — and they fail for different reasons. Collapsing them into a single 'inventory' would force every operation to know about both concerns, and would hide the two most interesting failure modes (out-of-stock vs can't-make-change) behind one abstraction."*

**Why the State pattern fits here:** three states with genuinely different legal operations — Idle (only `InsertCoin`), HasMoney (`InsertCoin`, `SelectProduct`, `Cancel`), Dispensing (only `Dispense`). Three or more states with meaningfully different behaviour is the trigger.

**Why I'm *not* using Command:** the ATM had four transaction types with the same shape (`Execute(ctx)`) that varied in the combination of bank calls and inventory changes — Command earned itself. A vending machine really has one primary operation, `Purchase`, plus lifecycle actions (`InsertCoin`, `Cancel`) that are direct verbs on the machine. Wrapping `Purchase` in a Command adds a type with no siblings — pattern-dropping. Naming this aloud is worth as much as reaching for the pattern would be, because it shows the interviewer I choose patterns by fit, not by reflex.

**Concrete implementers** (worth naming now so the interviewer sees the full picture):
- `VMState`: `idleState`, `hasMoneyState`, `dispensingState`
- `ChangeStrategy`: `greedyChanger`

---

## Class Design

*~10-15 minutes. Go entity by entity, orchestrator first. State first, then behaviour. Explain decisions as you write them.*

*Trivial data holders (`Product`, `Slot`) get stubs only — no methods, no decisions to narrate. The two inventories are grouped under one heading because their design story is shared (separate mutex, atomic `Take`, `Add` as the inverse) and telling it twice would be duplication.*

### VendingMachine

```go
type VendingMachine struct {
    ID       string
    mu       sync.Mutex
    products *ProductInventory
    coins    *CoinInventory
    changer  ChangeStrategy
    state    VMState
    session  *Session // nil when Idle
}
```

**While writing the mutex:** *"A vending machine serves one physical customer at a time — there's one coin slot and one hand — so customer transactions don't contend with each other. What the mutex is actually protecting against is rare concurrent access from other threads: a maintenance thread refilling inventory when a technician opens the machine, or a monitoring thread reading counters for a central dashboard. Without it, a monitor reading `products.slots` mid-transaction could observe a half-updated map. Single mutex on the whole machine is the simplest correct answer at this scope — I'd flag this aloud rather than over-engineer. If maintenance became frequent, I'd split into per-resource mutexes, which is why each inventory already has its own."*

> **On contention:** when two or more threads want the same resource at the same time and one has to wait. A mutex has two possible jobs — **traffic cop** (manage heavy contention under load) or **safety rail** (prevent rare corruption even when traffic is light). Same code, different justification. The VM mutex is a safety rail, not a traffic cop.

**On `session *Session`:** *"I keep session as a pointer — `nil` means the machine is Idle. This is cleaner than a boolean plus scattered fields because per-purchase data (inserted coins in escrow, selected slot) lives in one place and gets cleared together on completion or cancel. Fewer invariants to maintain."*

**On injecting inventories and strategy rather than embedding them:** *"Testability — my tests spin up a machine with a pre-loaded `ProductInventory` and a small `CoinInventory` without touching global state. More importantly, extensibility — if the change algorithm needs to swap to DP for a non-canonical currency, that's a one-line change in the constructor with zero edits to VM logic."*

Methods: `InsertCoin`, `SelectProduct`, `Dispense`, `Cancel`. Each is a thin wrapper that locks and delegates to the current state:

```go
func (v *VendingMachine) SelectProduct(code SlotCode) error {
    v.mu.Lock()
    defer v.mu.Unlock()
    return v.state.SelectProduct(v, code)
}

func (v *VendingMachine) Dispense() (Product, map[Denomination]int, error) {
    v.mu.Lock()
    defer v.mu.Unlock()
    return v.state.Dispense(v)
}
```

**On the lock held across `Dispense`:** *"I'm holding the VM mutex for the whole call, which includes the physical dispense inside `doDispense`. Fine at this scope — one customer per machine, no customer contention to worry about. If monitoring or maintenance had strict SLAs, I'd release the mutex before the mechanical action and re-acquire only for the state transition back to Idle. Naming the trade-off verbally captures the senior signal at a fraction of the whiteboard cost."*

---

### VMState — The State Machine

```go
type VMState interface {
    Name() string
    InsertCoin(v *VendingMachine, d Denomination) error
    SelectProduct(v *VendingMachine, code SlotCode) error
    Dispense(v *VendingMachine) (Product, map[Denomination]int, error)
    Cancel(v *VendingMachine) (map[Denomination]int, error)
}
```

The state machine — happy path first:

```
[Idle]        --InsertCoin(d)------------------> [HasMoney]     (session created, escrow[d]++)

[HasMoney]    --InsertCoin(d)------------------> [HasMoney]     (escrow[d]++)
[HasMoney]    --SelectProduct(c), enough $$----> [Dispensing]   (record slot + price)
[HasMoney]    --Cancel()----------------------> [Idle]          (return escrow coins as-is)

[Dispensing]  --Dispense(), success-----------> [Idle]          (product + change out, session cleared)
```

And the two failure scenarios, labelled separately so each is unambiguous:

```
Scenario A — Selection with insufficient funds
[HasMoney]    --SelectProduct(c), inserted < price--> [HasMoney]   (ErrInsufficientFunds returned; state unchanged)

Scenario B — Dispense fails (out-of-stock or no-change)
[Dispensing]  --Dispense(), out-of-stock-------> [Idle]         (product not decremented; escrow refunded to caller)
[Dispensing]  --Dispense(), no-change----------> [Idle]         (product rolled back; escrow refunded to caller)
```

**On the concrete states being unexported:** *"Same reasoning as any State-pattern use in Go — the three states are a closed set. If I exported them, caller code could construct `vmpkg.DispensingState{}` and hand-build a machine in that state with no selected slot and no escrow, bypassing the entire purchase flow. The interface is exported because it's the contract; the concrete states belong entirely inside this package."*

To illustrate — `idleState` and `hasMoneyState`:

```go
type idleState struct{}

func (idleState) Name() string { return "Idle" }

func (idleState) InsertCoin(v *VendingMachine, d Denomination) error {
    v.session = newSession()
    v.session.Escrow[d]++
    v.transition(hasMoneyState{})
    return nil
}
func (idleState) SelectProduct(*VendingMachine, SlotCode) error                     { return errInvalidOperation }
func (idleState) Dispense(*VendingMachine) (Product, map[Denomination]int, error)   { return Product{}, nil, errInvalidOperation }
func (idleState) Cancel(*VendingMachine) (map[Denomination]int, error)              { return nil, errInvalidOperation }

type hasMoneyState struct{}

func (hasMoneyState) Name() string { return "HasMoney" }

func (hasMoneyState) InsertCoin(v *VendingMachine, d Denomination) error {
    v.session.Escrow[d]++
    return nil
}
func (hasMoneyState) SelectProduct(v *VendingMachine, code SlotCode) error {
    prod, err := v.products.Get(code)
    if err != nil {
        return err
    }
    if v.session.Inserted() < prod.Price {
        return fmt.Errorf("%w: need %d, have %d", errInsufficientFunds, prod.Price, v.session.Inserted())
    }
    v.session.SelectedSlot = code
    v.session.SelectedPrice = prod.Price
    v.transition(dispensingState{})
    return nil
}
func (hasMoneyState) Dispense(*VendingMachine) (Product, map[Denomination]int, error) {
    return Product{}, nil, errInvalidOperation
}
func (hasMoneyState) Cancel(v *VendingMachine) (map[Denomination]int, error) {
    refund := v.session.Escrow
    v.session = nil
    v.transition(idleState{})
    return refund, nil
}
```

`dispensingState` follows the same shape — only `Dispense` is legal; everything else returns `errInvalidOperation`.

*"In production these `errInvalidOperation` and `errInsufficientFunds` values would be named sentinels declared once in a shared errors file, so callers can use `errors.Is`. I'm writing them referenced-only here to keep the class design moving."*

**Why `SelectProduct` doesn't dispense:** *"Selection validates the choice (slot exists, funds sufficient) and transitions to Dispensing, but doesn't run the mechanical action. Dispense is a separate call. Real machines show a brief confirmation ('dispensing…') between the button press and the actual drop — the state machine models that as two events. It also means the interviewer can ask 'what if the customer walks away between selection and dispense?' and the answer is 'timeout in Dispensing state, refund escrow' rather than 'we already committed everything'."*

---

### Product And Slot

```go
type Product struct {
    Name  string
    Price int // minor units — paise/cents
}

type SlotCode string // e.g. "A1", "B2" — matches the physical row/column label

type Slot struct {
    Code    SlotCode
    Product Product
    Count   int // units remaining in this slot
}
```

**On `Price` being an int in minor units:** *"Same rule as anywhere else in code that touches money — never floats. Float multiplication accumulates rounding errors that will eventually cost someone real money or produce a machine that dispenses ₹0.99 change when it should have been ₹1. Integers in the smallest unit (paise here, cents elsewhere) sidestep the whole class of bugs."*

**On `SlotCode` being a string type, not just `string`:** *"Cheap typedef, real benefit — a function taking `SlotCode` can't be accidentally called with a `Product.Name` or any other string. The compiler catches the mistake. Costs nothing at runtime; adds a small amount of type safety that pays off the first time you refactor a signature."*

---

### ProductInventory And CoinInventory

Separate types on purpose — different failure modes, different concurrency partners.

```go
type ProductInventory struct {
    mu    sync.Mutex
    slots map[SlotCode]*Slot
}

func (p *ProductInventory) Take(code SlotCode) error   { /* decrement one; err if unknown or empty */ }
func (p *ProductInventory) Add(code SlotCode) error    { /* increment one; err if unknown */ }
func (p *ProductInventory) Get(code SlotCode) (Product, error)

type CoinInventory struct {
    mu     sync.Mutex
    counts map[Denomination]int
}

func (c *CoinInventory) Snapshot() map[Denomination]int          { /* defensive copy */ }
func (c *CoinInventory) Take(picks map[Denomination]int) error   { /* all or nothing */ }
func (c *CoinInventory) Add(coins map[Denomination]int)          { /* commit or refill */ }
```

**On the separate mutexes:** *"The VM has its own lock for the state machine; each inventory has its own lock for its map. Three locks, not one. The reason is that different threads care about different data. A customer's `Dispense` holds `VM.mu` while it walks the state machine, but only touches `CoinInventory.mu` for the microseconds it takes to actually mutate the coin map. A maintenance thread topping up coins doesn't care about the state machine at all — it only needs `CoinInventory.mu`. If I collapsed everything into one VM-wide lock, the maintenance thread would block on the whole customer transaction to touch a map the customer barely uses. Splitting the locks means each waiter only waits on the data it actually needs."*

> **On lock granularity:** the mental model — keep critical sections short (only the code that mutates shared state), do slow work outside the lock when possible, and use separate mutexes for resources with different concurrency partners. Same code correctness, very different tail latencies. When time doesn't allow writing the release-reacquire pattern, naming it verbally captures the same senior signal.

**On `CoinInventory.Take` being atomic:** *"Partial change is the second-worst failure of a vending machine (first being partial dispense of a product). `Take` checks all requested denominations under the mutex, and only decrements if all are available. This is a small function that carries huge design weight — the entire correctness of the change-out path depends on it."*

**On `ProductInventory.Add` as the rollback hook:** *"`Add` is the inverse of `Take` — same signature, opposite direction. Its current caller is the rollback path in `doDispense`, but the shape generalises: when restocking lands as a real feature it'll call `Add` too. Modeling this as a first-class method — not a raw `s.Count++` inline in the VM — keeps the invariant ('slot count only changes through the inventory') intact. In production this would also emit a metric so 'unusually high rollback rate' becomes an observable signal of a coin-inventory misconfiguration."*

**On persistence:** *"Both inventories are in-memory in this design. Real machines write to non-volatile storage after every mutation so a power cycle doesn't lose track of stock. The interfaces (`Take`, `Add`, `Snapshot`) don't change; the implementation swaps in a durable backing store. Flagged as a production gap, not a design flaw."*

---

### ChangeStrategy — Greedy Vs DP

```go
type ChangeStrategy interface {
    Pick(amount int, available map[Denomination]int) (map[Denomination]int, error)
}

type greedyChanger struct {
    denoms []Denomination // sorted descending
}
```

**While writing this:** *"Greedy works for canonical currency sets — ₹1, ₹2, ₹5, ₹10, ₹20, ₹50, ₹100 — because the largest denomination that fits always leaves a remainder solvable by smaller ones. For an arbitrary set (say a hypothetical ₹3 and ₹7 with no ₹1) greedy can fail: you'd need coin-change dynamic programming. I'd name that trade-off aloud but ship greedy — real currencies are canonical by design, because central banks pick denominations specifically so greedy works. Deploying a machine in a country with non-canonical currency is the trigger to swap in a DP implementation, one constructor line."*

**Why this is Strategy and not a helper function:** *"If the operator wants to bias toward dispensing small coins (to burn down small-coin inventory that's about to overflow the reservoir), that's a different algorithm with the same signature. Strategy makes it swappable per-machine. A helper function would force every caller to change."*

**Subtle correctness point:** *"The strategy takes the coin inventory *plus* the coins the customer just inserted as its available pool — because those inserted coins are eligible for use as change. Otherwise the machine would refuse a perfectly valid transaction: customer pays ₹50 for a ₹25 item, machine has zero ₹25-worth of change but the customer just handed over a ₹25 as part of the payment. The commit path in `doDispense` handles the plumbing; the strategy sees the combined snapshot and doesn't need to know the difference."*

---

### Session

```go
type Session struct {
    Escrow        map[Denomination]int // inserted coins, not yet committed to inventory
    SelectedSlot  SlotCode
    SelectedPrice int
    StartedAt     time.Time
}

func newSession() *Session         { /* fresh session with empty escrow, StartedAt=now */ }
func (s *Session) Inserted() int   { /* sum of denom × count across escrow */ }
```

**On the tiny method surface:** *"Session is mostly a data holder — the VM and state methods read and write its fields directly because they're all in the same package. The two methods that do exist earn their place: `newSession` is the constructor (keeps escrow-map initialisation in one place so no caller ever forgets and hits a nil-map panic on first insert), and `Inserted()` computes the running total across denominations, which the state methods need in three places (funds check on select, change calculation on dispense, refund total on cancel). Anything else — direct field access is fine at this scope."*

**On the escrow being separate from `CoinInventory`:** *"This is the modeling decision that makes cancel-refund clean. When the customer inserts coins, they land in `Escrow` — not the coin reservoir. On successful purchase, `Escrow` merges into `CoinInventory` and change is picked from the combined pool. On cancel, `Escrow` is returned to the customer as-is — same denominations they put in, no algorithmic re-picking. This mirrors how real machines physically work: inserted coins sit in a temporary holding area until the transaction commits."*

---

## Implementation

*~10-15 minutes. Implement the most interesting method. Start with happy path, then name the edge cases.*

### Dispense — The Atomic Three-Step

**Before the code — why there are two methods:** the public `Dispense()` shown in Class Design is a thin wrapper that locks `VM.mu` and delegates to the current state's `Dispense`. When the state is `dispensingState`, that state's method turns around and calls a private worker on the VM — `doDispense()` — which holds all the actual purchase logic. Three layers: public wrapper (locking + state dispatch) → state method (permission check: is this state allowed to dispense?) → private worker (the atomic three-step below). The split matters because every public method on the VM (`InsertCoin`, `SelectProduct`, `Dispense`, `Cancel`) has to look the same — lock, delegate, unlock — so the state pattern stays clean; and the state method can't call the public `Dispense` back, or it would re-enter the state dispatch and loop forever. Putting the real work in `doDispense` breaks the cycle.

> **On the `doX` naming convention:** Go idiom for "the private worker called by a public wrapper that has already handled the framing concerns" — locking, permission checks, retries, or state guards. The standard library uses it in several places (`net/http`, `database/sql`). When you see `foo()` and `doFoo()` side by side in Go code, the `do`-prefixed one is almost always the actual implementation and the un-prefixed one is a thin wrapper that guarantees preconditions before calling in. The convention signals *"if you're inside the package and the preconditions already hold, call `doFoo` directly and skip the wrapper's overhead."*

`doDispense` is where every design decision above comes together. Three-step atomic operation: reserve the product, compute and reserve change (against the *combined* pool of vault plus escrow), commit inserted coins and take change.

```go
func (v *VendingMachine) doDispense() (Product, map[Denomination]int, error) {
    if v.session == nil || v.session.SelectedSlot == "" {
        return Product{}, nil, errNoSelection
    }
    slot := v.session.SelectedSlot
    price := v.session.SelectedPrice
    changeDue := v.session.Inserted() - price

    // 1. Reserve the product.
    if err := v.products.Take(slot); err != nil {
        refund := v.session.Escrow
        v.session = nil
        v.transition(idleState{})
        return Product{}, refund, err
    }

    // 2. Compute change against the *combined* pool — vault plus inserted coins.
    available := v.coins.Snapshot()
    for d, n := range v.session.Escrow {
        available[d] += n
    }
    change, err := v.changer.Pick(changeDue, available)
    if err != nil {
        _ = v.products.Add(slot) // rollback the product
        refund := v.session.Escrow
        v.session = nil
        v.transition(idleState{})
        return Product{}, refund, err
    }

    // 3. Commit: inserted coins → vault; take change from vault.
    v.coins.Add(v.session.Escrow)
    if err := v.coins.Take(change); err != nil {
        // Shouldn't happen — Pick used the same snapshot — but keep rollback honest
        // in case a maintenance thread mutated coins between Snapshot and Take.
        _ = v.products.Add(slot)
        for d, n := range v.session.Escrow {
            _ = v.coins.Take(map[Denomination]int{d: n})
        }
        refund := v.session.Escrow
        v.session = nil
        v.transition(idleState{})
        return Product{}, refund, err
    }

    product, _ := v.products.Get(slot)
    v.session = nil
    v.transition(idleState{})
    return product, change, nil
}
```

**Three things to narrate while writing this:**

*"The order matters. I reserve the product first — a local map decrement, no expensive work. If out-of-stock, I never touch the coin inventory and the customer's escrow is returned exactly as they inserted it. This is faster and cleaner than the reverse order, where a coin-side failure would force me to un-take coins that shouldn't have been touched."*

*"Change is computed against the **combined** pool — vault plus inserted coins — because the customer's own inserted coins are eligible for use as change. This is the non-obvious correctness point. If I computed change only from the vault, the machine would refuse valid transactions where the customer's own denominations would have made the math work. The commit step then merges escrow into the vault before actually calling `Take` — so the strategy's plan matches the physical reality."*

*"The 'commit-then-take-fails' branch shouldn't happen — the strategy computed against a snapshot that already accounted for the escrow — but I still write the rollback path. Why? Because between `Snapshot` and `Take`, a maintenance thread could have removed coins for a cash-out. I'd rather return a compound error and refund the escrow than assume the invariant holds. Senior instinct: don't trust a check-then-act sequence to be atomic across separately-locked resources."*

> **On check-then-act:** a pattern where code reads a value, decides based on that value, then writes — with no lock spanning both operations. Between the read and the write another thread can mutate the shared state, invalidating the decision. Classic source of bugs. The fix is either to hold a lock across both operations (coarse) or to design the write step to be atomic and self-validating (fine — what `CoinInventory.Take` does by re-checking all denominations under its own mutex before decrementing).

---

## Verification Walkthrough

*~2-3 minutes. Trace one non-trivial scenario out loud to prove the design holds together.*

**Scenario:** Customer inserts ₹50, then ₹5, selects slot A1 (Cola, price ₹25), receives Cola and ₹30 change. Coin inventory starts at 5×₹10, 5×₹5, 5×₹2, 5×₹1. Product inventory starts at A1×3.

```
t0  state=Idle          session=nil                              coins={10:5, 5:5, 2:5, 1:5}   A1=3

t1  InsertCoin(50)
    state=HasMoney      session={escrow:{50:1}, selected:""}     coins unchanged                 A1=3

t2  InsertCoin(5)
    state=HasMoney      session={escrow:{50:1, 5:1}}             coins unchanged                 A1=3

t3  SelectProduct("A1")
    ├─ products.Get("A1") → {Cola, price=25}
    ├─ inserted=55 >= 25 ✓
    └─ session.selected="A1", session.selectedPrice=25
    state=Dispensing    session={escrow:{50:1, 5:1}, selected:"A1"}

t4  Dispense()
    ├─ changeDue = 55 - 25 = 30
    ├─ products.Take("A1") → OK                                                                   A1=2
    ├─ available = coins.Snapshot() ⊕ escrow = {50:1, 10:5, 5:6, 2:5, 1:5}
    ├─ changer.Pick(30, available) → {10:3}    ✓ (greedy: three ₹10)
    ├─ coins.Add({50:1, 5:1})                                                    coins={50:1, 10:5, 5:6, 2:5, 1:5}
    └─ coins.Take({10:3})                                                        coins={50:1, 10:2, 5:6, 2:5, 1:5}
    returns (Cola, {10:3})
    state=Idle          session=nil                                              A1=2
```

What each tick confirms:

- `t1`: State transition to HasMoney creates the session — escrow has no meaningful existence when Idle. The first insertion is treated identically to any subsequent one from the caller's perspective; the state layer handles the setup.
- `t2`: Additional coin lands in HasMoney's `InsertCoin`, which just increments the escrow map. No state transition — HasMoney → HasMoney is a self-loop for repeated inserts.
- `t3`: Selection validates two things (slot exists, funds sufficient) and records the choice. It does *not* touch either inventory. `authenticatedState`-equivalent — we've earned the right to dispense, but haven't yet.
- `t4`: The atomic three-step. Product decrements first (cheap check). Change computed against `vault ⊕ escrow`, not vault alone — this is how a ₹5 the customer just inserted becomes eligible for use in the change calculation (though in this scenario greedy picks three ₹10 without needing the escrow ₹5). Vault then absorbs the escrow, and change comes out of the vault. Final coin state reflects both flows: the ₹50 and ₹5 the customer inserted are now in the vault; three ₹10 have left it.
- Post `t4`: Session is cleared. Next `InsertCoin` starts a fresh session — no chance of stale escrow leaking into the next customer.

---

## Trade-Offs

*Most trade-offs are named inline at the point of decision — mutex granularity, in-memory inventories, greedy change strategy, single mutex on the VM, `Purchase` not modeled as a Command. What's collected here is scope-level: the things I've explicitly chosen not to model.*

- **No hardware failure modeling** — jammed dispenser, stuck coin in the mechanism, torn note in the acceptor, cash box full. Each would need its own error path and possibly a new state. Out of scope by choice; the design has hooks (each inventory operation returns an error, `ProductInventory.Add` is the rollback anchor) where they'd attach.
- **No persistence.** Both inventories, the state, and session data all live in RAM. Power cycle loses everything. Production would persist inventory to non-volatile storage on every mutation; the interfaces don't change, only the backing store.
- **Single-payment method.** Only cash. Real machines accept cards, mobile wallets, campus cards, and increasingly QR codes. Adding these is an extension (below), not a redesign — but the current design assumes payment == physical money in escrow.
- **One purchase per session.** Simplifies the state graph. Multi-item ("buy another?") would loop HasMoney after Dispensing and requires deciding whether change from purchase 1 is available for purchase 2 or refunded at the end.
- **Restocking modeled as inventory `Add`, not a workflow.** Real restocking involves a physical door, an admin auth, itemized top-ups, and audit logging. Out of scope; `ProductInventory.Load` and `CoinInventory.Add` are the seams where a real restocking flow would plug in.

---

## Extensions

*~5 minutes if time allows. For each follow-up, say where it hooks in and what changes — no need to write full code.*

**"What if the machine can't make exact change?"**
Already handled — the third step of `doDispense` catches `errCannotMakeChange` from the strategy, rolls back the product decrement via `Add`, and returns the untouched escrow as the refund. The design decision that made this clean was keeping escrow separate from the vault until commit — refund is literally handing back the same coins the customer inserted.

**"How would you support card payment?"**
Introduce a `PaymentMethod` interface — `Charge(amount, idempotencyKey) error` and `Refund(idempotencyKey) error`. Cash becomes `CashPayment` (wrapping the current escrow logic); cards become `CardPayment` (talking to a payment gateway). The session holds a `PaymentMethod` rather than a raw escrow map, and `doDispense` calls `Charge` in step 3 instead of committing coins. State machine unchanged. This is the payoff of already treating inserted money as a session-scoped concept.

**"How would you support buying multiple items in one session?"**
Add a state transition Dispensing → HasMoney after a successful dispense if the customer indicates "buy more" (physical button or timeout branch). The escrow re-populates from the change that would have been dispensed. Complexity to name aloud: the change strategy now has to consider that "change" might become "credit toward next item," which is a different UX and a different failure mode when the customer eventually cancels.

**"How would you handle a timeout — customer walks away mid-purchase?"**
A background goroutine per session checks `time.Since(session.StartedAt) > timeout` and, if in HasMoney or Dispensing, triggers `Cancel()`. The refund goes to a "returned coins" tray for the next customer to collect. Design-wise this is a cross-cutting concern; the cleanest hook is a `sessionTimeoutMonitor` started when the session is created and stopped on any successful terminal transition.

**"How would you add a maintenance mode?"**
Fourth state: `maintenanceState`. Its `InsertCoin` returns "out of service"; a separate admin path (with auth) transitions it to and from Idle. Restocking and coin cash-out both happen while in this state, so `ProductInventory.Load` and `CoinInventory.Add` become the admin-only operations. State machine gets one new node but the shape of the code doesn't change — exactly what State is for.

**"How would you track inventory across power cycles?"**
Every mutation on either inventory writes to durable storage (WAL, or a small embedded KV store). On startup, replay the log to reconstruct state. The `Take`/`Add` methods become the sync boundary — write to disk *before* returning success. Small perf hit per operation, but this is a vending machine — throughput is measured in transactions per hour, not per second.

**"How would you send low-stock alerts?"**
Observer pattern hooked onto `ProductInventory.Take`. When a slot count crosses a configured threshold, publish a `LowStockEvent`. A separate service subscribes and pushes to whichever alerting channel the operator uses (SMS, email, dashboard). No changes to VM logic — the observation is entirely inside `ProductInventory`.

---

## References

- `lld/patterns/behavioral/state.md` — State pattern; applied here to `VMState`
- `lld/patterns/behavioral/strategy.md` — Strategy pattern; applied here to `ChangeStrategy`
- `lld/design-atm.md` — Sibling case study; two-inventory pattern (product vs coin here, cash vs bank there), atomic rollback in the primary operation, and Strategy for denomination selection all mirror this file