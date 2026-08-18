---
Status: 🌳 Evergreen
Created: 2026-08-18
Last Updated: 2026-08-18
---

# ATM Machine

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

An ATM is a self-service terminal that lets a bank customer authenticate with a card and PIN and then perform a small set of financial operations — withdraw cash, deposit cash, check balance, transfer between accounts — against their bank account. It looks like a hardware problem at first glance, but the interesting design lives in two places: the strict order of operations (you can't select a transaction before authenticating; you can't authenticate without a card), and the fact that the ATM is a *client* of the bank, not the bank itself. That combination — a state machine at the interaction layer plus a clean boundary between the machine and the bank — is what makes it a reliable interview problem.

---

## Requirements

*~5 minutes. The interviewer gives you a one-line prompt: "Design an ATM." Your job is to ask the right clarifying questions to pin down scope. Each question below is one I'd ask — with the honest reason I'm asking, which is usually "I want to earn the right to introduce a specific design decision."*

**Questions I'd ask, and why I'm asking each one:**

*"What transactions does the ATM support?"*
Surface purpose: scope. Real purpose: how many operation types do we need, and do they share enough shape to benefit from a common interface? If the answer is "withdraw, deposit, balance inquiry, transfer" — four types with a shared structure — I've earned the right to introduce Command. If it's "just withdraw," a plain method on ATM is fine and Command would be pattern-dropping.

*"Does the ATM hold physical cash, or does it request it from the bank on demand?"*
Surface purpose: scope. Real purpose: if the ATM holds cash (which it does — that's what makes it an ATM), then `CashInventory` becomes a first-class entity with atomic operations and a dispensing algorithm. If the interviewer said "assume infinite cash," I'd skip that whole subsystem.

*"Does the ATM authenticate against the bank, or store PINs locally?"*
Surface purpose: clarify auth flow. Real purpose: earning the external boundary. If auth is against the bank, `BankService` becomes an injected interface with `AuthenticatePIN(cardNumber, pin) → accountID` — and by the time I introduce it, the interviewer has already agreed the bank is external. Declaring "I'll inject BankService" without this question first looks like a decision imposed on the design; asking makes it feel derived from the requirements.

*"What denominations, and can the customer request specific ones?"*
Surface purpose: clarify UX. Real purpose: if the customer requests an amount (not specific notes), some algorithm has to pick which notes to dispense. That earns `DispenseStrategy` as an interface — and the follow-up conversation about greedy vs DP is a natural place to demonstrate depth.

*"What if the network to the bank fails mid-withdrawal?"*
Surface purpose: edge case. Real purpose: two-fold. First, it lets me flag idempotency keys as a design concern before writing any code — so when the `BankService` interface has `idempotencyKey` in the signature, it looks obvious rather than surprising. Second, it lets me proactively scope out full network-failure recovery: *"I'll require idempotency keys at the interface level — that's the design decision that matters — but assume the bank call itself either succeeds or returns a clear error. Full recovery is out of scope."* Naming what's *not* in scope is as important as naming what is.

*"What's explicitly out of scope?"*
Don't wait to be told — offer it. *"Cheque deposit, multi-currency, hardware fault handling (jammed cash, torn cards), physical security, and bank-side ledger consistency are all out. I'll mock the bank behind a `BankService` interface and treat cash inventory as in-memory. Does that scope work?"* Offering the boundary yourself signals that you know what a 45-minute design can and can't cover.

**Final scope:**

| In | Out |
|---|---|
| Card + PIN authentication (with retry limit) | Cheque or document deposit |
| Cash withdrawal (multi-denomination) | Multi-currency |
| Cash deposit | Hardware fault recovery |
| Balance inquiry | Physical security / camera |
| Transfer between customer's own accounts | Bank-side ledger, double-entry accounting |
| In-memory cash inventory | Actual network protocol to the bank switch |
| Per-session transaction audit log | PIN change, mini-statement (extensions) |

---

## Core Entities

*~5 minutes. Sketch the structure before writing any code — communicates intent to the interviewer and catches modeling mistakes early.*

What I'd put on the whiteboard:

```
Concrete types             Interfaces (behaviour that varies)
─────────────────          ────────────────────────────────
ATM                        ATMState          ← per-state behaviour
Card                       BankService       ← external bank system
CashInventory              Transaction       ← Command pattern
Session                    DispenseStrategy  ← note-picking algorithm
WithdrawalTxn              ReceiptPrinter
DepositTxn
TransferTxn
BalanceInquiryTxn
```

Ownership and dependencies — what I'd draw as arrows:

```
ATM         owns    CashInventory, Session          (I create them — &Session{} on InsertCard)
ATM         uses    BankService, DispenseStrategy,
                    ReceiptPrinter                   (injected — built outside, handed in)
ATM         has-a   ATMState                         (State — swaps itself)
Session     refs    Card                             (inserted by the customer)
Transaction refs    Session, amount, TxnContext      (Command; executed against context)
```

> **owns vs uses:** *owns* = this type is what calls `&Session{...}`; the object didn't exist before and this type is responsible for it. *uses* = the object was created elsewhere and handed in via constructor; the ATM just holds a pointer.

**The key insight to say out loud:**

*"The most important modeling decision is that the ATM is a **client** of the bank, not a substitute for it. The ATM owns physical cash and the user session; the bank owns accounts and balances. `BankService` is the boundary between them. This distinction drives everything: the ATM tracks 'what's in my cash tray'; the bank tracks 'what's in the customer's account'. Withdrawal touches both — debit at the bank, dispense from the tray — and only succeeds if both succeed."*

**Why the State pattern fits here:** the ATM has four states with genuinely different legal operations per state — Idle (only `InsertCard`), HasCard (only `EnterPIN` or `EjectCard`), Authenticated (`SelectTransaction` or `EjectCard`), TransactionInProgress (only `ExecuteTransaction`). Three or more states with meaningfully different behaviour is the trigger.

**Concrete implementers** (worth naming now so the interviewer sees the full picture):
- `ATMState`: `idleState`, `hasCardState`, `authenticatedState`, `transactionInProgressState`
- `Transaction`: `WithdrawalTxn`, `DepositTxn`, `TransferTxn`, `BalanceInquiryTxn`
- `DispenseStrategy`: `greedyDispenser`
- `BankService`: `MockBank` (in tests), real implementation talks to the bank switch

---

## Class Design

*~10-15 minutes. Go entity by entity, orchestrator first. For each: state (what it holds), then behaviour (what it does). Explain decisions as you write them.*

### ATM

```go
type ATM struct {
    ID         string
    location   string
    inventory  *CashInventory
    bank       BankService
    dispenser  DispenseStrategy
    printer    ReceiptPrinter
    mu         sync.Mutex
    state      ATMState
    session    *Session // nil when idle
    nextTxnSeq int
}
```

**While writing the mutex:** *"An ATM serves one physical customer at a time, so contention isn't the reason for the mutex — customer transactions don't compete with each other. What the mutex is actually protecting against is rare concurrent access from other threads: a maintenance thread refilling inventory when a technician opens the machine, or a monitoring thread reading counters to report cash levels to a central dashboard. Without a mutex, a monitor reading `inventory.counts` mid-transaction could observe a half-updated map. Single mutex on the whole ATM is the simplest correct answer at this scope. I'd flag it aloud but not over-engineer it — if maintenance became frequent, I'd split into per-resource mutexes, which is why inventory already has its own."*

> **On contention:** when two or more threads want the same resource at the same time and one has to wait. A mutex has two possible jobs — **traffic cop** (manage heavy contention under load) or **safety rail** (prevent rare corruption even when traffic is light). Same code, different justification. The ATM mutex is a safety rail, not a traffic cop.

**On `session *Session`:** *"I keep session as a pointer — `nil` means no card in the machine. This is cleaner than a boolean plus scattered fields, because session-related data (card, authenticated account, per-session transaction history) lives in one place and gets cleared together on eject. Fewer invariants to maintain."*

**On injecting `BankService` rather than embedding bank logic:** *"Two reasons. Testability — my tests don't need a running bank; a `MockBank` returns whatever the test needs. More importantly, correctness at boundaries — real ATMs occasionally run in 'stand-in' mode when the bank switch is unreachable, processing small transactions offline. Injection makes that swap a one-line change without touching any ATM logic."*

Methods: `InsertCard`, `EnterPIN`, `SelectTransaction`, `ExecuteTransaction`, `EjectCard`. Each is a thin wrapper that locks and delegates to the current state:

```go
func (a *ATM) SelectTransaction(kind TransactionKind, params TransactionParams) (Transaction, error) {
    a.mu.Lock()
    defer a.mu.Unlock()
    return a.state.SelectTransaction(a, kind, params)
}

func (a *ATM) ExecuteTransaction(txn Transaction) error {
    a.mu.Lock()
    defer a.mu.Unlock()
    return a.state.ExecuteTransaction(a, txn)
}
```

**On the lock held across `ExecuteTransaction`:** *"I'm holding the ATM mutex for the whole call, which includes the slow bank round-trip and physical dispense inside `txn.Execute`. Fine at this scope — one customer per ATM at a time, so no customer contention to worry about. If monitoring or maintenance had strict SLAs, I'd release the mutex before `txn.Execute` and re-acquire only for the state transition back to Authenticated. I'm choosing the simpler version because writing the release-reacquire pattern costs whiteboard time that Verification and Extensions need more — naming the trade-off verbally is what earns the senior signal, not typing the extra lines."*

---

### ATMState — the state machine

```go
type ATMState interface {
    Name() string
    InsertCard(a *ATM, card Card) error
    EnterPIN(a *ATM, pin string) error
    SelectTransaction(a *ATM, kind TransactionKind, params TransactionParams) (Transaction, error)
    ExecuteTransaction(a *ATM, txn Transaction) error
    EjectCard(a *ATM) error
}
```

The state machine in compact form:

```
[Idle]                  --InsertCard(c)---------------------> [HasCard]

[HasCard]               --EnterPIN(p), valid-----------------> [Authenticated]
[HasCard]               --EnterPIN(p), invalid, attempts<3---> [HasCard]   (attempts++)
[HasCard]               --EnterPIN(p), invalid, attempts==3--> [Idle]      (card retained, session cleared)
[HasCard]               --EjectCard()-----------------------> [Idle]

[Authenticated]         --SelectTransaction(k, p)-----------> [TransactionInProgress]
[Authenticated]         --EjectCard()-----------------------> [Idle]

[TransactionInProgress] --ExecuteTransaction(t), success---> [Authenticated]
[TransactionInProgress] --ExecuteTransaction(t), failure---> [Authenticated]   (error surfaced to caller)

Any                     --anything else--> ErrInvalidOperation
```

**On the concrete states being unexported:** *"Same reasoning as any State-pattern use in Go — the four states are a closed set. If I exported them, caller code could construct `atmpkg.AuthenticatedState{}` and hand-build an ATM in that state with no card and no session, bypassing the entire authentication flow. The interface is exported because it's the contract; the four concrete states belong entirely inside this package."*

To illustrate — `idleState` and `authenticatedState`:

```go
type idleState struct{}

func (idleState) Name() string { return "Idle" }

func (idleState) InsertCard(a *ATM, card Card) error {
    a.session = &Session{Card: card, InsertedAt: time.Now()}
    a.transition(hasCardState{})
    return nil
}
func (idleState) EnterPIN(*ATM, string) error                                       { return errInvalidOperation }
func (idleState) SelectTransaction(*ATM, TransactionKind, TransactionParams) (Transaction, error) {
    return nil, errInvalidOperation
}
func (idleState) ExecuteTransaction(*ATM, Transaction) error                        { return errInvalidOperation }
func (idleState) EjectCard(*ATM) error                                              { return errInvalidOperation }

type authenticatedState struct{}

func (authenticatedState) Name() string { return "Authenticated" }

func (authenticatedState) SelectTransaction(a *ATM, kind TransactionKind, params TransactionParams) (Transaction, error) {
    a.nextTxnSeq++
    txnID := fmt.Sprintf("%s-T-%d", a.ID, a.nextTxnSeq)
    var txn Transaction
    switch kind {
    case TxnWithdrawal:
        txn = &WithdrawalTxn{id: txnID, amount: params.Amount, ts: time.Now()}
    case TxnDeposit:
        txn = &DepositTxn{id: txnID, notes: params.DepositedNotes, ts: time.Now()}
    case TxnTransfer:
        txn = &TransferTxn{id: txnID, toAccount: params.ToAccount, amount: params.Amount, ts: time.Now()}
    case TxnBalanceInquiry:
        txn = &BalanceInquiryTxn{id: txnID, ts: time.Now()}
    default:
        return nil, errUnknownTransactionKind
    }
    a.transition(transactionInProgressState{})
    return txn, nil
}
func (authenticatedState) EjectCard(a *ATM) error {
    a.session = nil
    a.transition(idleState{})
    return nil
}
// remaining methods return errInvalidOperation
```

`hasCardState` and `transactionInProgressState` follow the same shape — same interface, different subset of operations permitted.

*"In production these `errInvalidOperation` and `errUnknownTransactionKind` values would be named sentinels declared once in a shared errors file, so callers can use `errors.Is`. I'm writing them referenced-only here to keep the class design moving."*

**Why `SelectTransaction` doesn't `Execute`:** *"`SelectTransaction` builds the Transaction object and transitions to TransactionInProgress, but doesn't run it. Execute is a separate call. That separation matters because the UI typically shows a confirmation screen — 'you're about to withdraw ₹5000, proceed?' — between selection and execution. The state machine treats those as two events, which is what the real user flow needs."*

---

### Transaction — Command pattern

```go
type Transaction interface {
    ID() string
    Kind() TransactionKind
    Timestamp() time.Time
    Execute(ctx TxnContext) error
}

type TxnContext struct {
    Bank      BankService
    Inventory *CashInventory
    Dispenser DispenseStrategy
    Session   *Session
}
```

**On Command over a switch in ATM.Execute:** *"Each transaction does a different combination of bank calls, inventory changes, and side effects. Withdrawal calls Debit and Take; Deposit calls Credit and Add; Transfer calls Debit-then-Credit atomically; BalanceInquiry calls Balance and does nothing to inventory. If I put a switch inside `ATM.Execute` it grows every time we add a transaction type. As Command, adding 'pay bill' means one new struct implementing Transaction — no switch touched. As a bonus, each transaction is a data object I can serialize into an audit log."*

> **On "switch on type":** a `switch` statement that branches on a type or kind, where each case does substantially different work. It's a **code smell** — code that runs correctly but signals a deeper design problem — because every new type forces edits to the same central function. That function keeps growing, reviewers must re-verify it on every change, and bugs cluster in the busiest file. Command inverts this: each type owns its own behaviour in its own struct, and the caller is one line (`txn.Execute(ctx)`) that never grows. The principle behind the fix is **Open/Closed**: code should be open for extension (add new types) but closed for modification (existing code unchanged).

---

### CashInventory

```go
type Denomination int

const (
    Note100  Denomination = 100
    Note200  Denomination = 200
    Note500  Denomination = 500
    Note2000 Denomination = 2000
)

type CashInventory struct {
    mu     sync.Mutex
    counts map[Denomination]int
}

func (ci *CashInventory) Snapshot() map[Denomination]int { /* returns a defensive copy */ }
func (ci *CashInventory) Take(picks map[Denomination]int) error   { /* all or nothing */ }
func (ci *CashInventory) Add(deposits map[Denomination]int)       { /* refill or deposit */ }
```

**On the separate mutex:** *"Inventory has its own mutex so refill and dispense operations serialize on the inventory itself, independent of the ATM's state-machine lock. Even if the ATM mutex is held across a slow bank call, a maintenance thread refilling cash contends only with the brief window when `Take` or `Add` is actually mutating the map — milliseconds, not seconds. The point of fine-grained locking here isn't zero blocking; it's that inventory has genuinely different concurrency partners (customer vs maintainer) from the state machine, and separating the locks lets each partner wait only on what actually protects its data."*

> **On lock granularity:** the mental model — keep critical sections short (only the code that mutates shared state), do slow work outside the lock when possible, and use separate mutexes for resources that have different concurrency partners. Same code correctness, very different tail latencies. When time doesn't allow writing the release-reacquire pattern, naming it verbally ("I'd release before the slow call") captures the same senior signal at a fraction of the whiteboard cost.

**On `Take` being atomic:** *"Partial dispense is the single worst failure mode of an ATM: customer's account debited, only some cash in the tray. `Take` is all-or-nothing — either every requested note leaves the tray or none do. The check-and-decrement happens under the mutex. This is a small function that carries a huge amount of design weight."*

**On persistence:** *"Inventory is in-memory only in this design. Real ATMs write to non-volatile storage after every mutation so a power cycle doesn't lose track of what's in the tray. I'd flag this as a production gap, not a design flaw — the interface (`Take`, `Add`, `Snapshot`) doesn't change; the implementation swaps in a durable backing store. Not modeling it now to keep the class design tight."*

---

### DispenseStrategy — greedy vs DP

```go
type DispenseStrategy interface {
    Pick(amount int, available map[Denomination]int) (map[Denomination]int, error)
}

type greedyDispenser struct {
    denoms []Denomination // sorted descending
}
```

**While writing this:** *"Greedy works for canonical denomination sets — ₹100, ₹200, ₹500, ₹2000 or $20/$50/$100 — because the largest note that fits always leaves a remainder solvable by smaller notes. For an arbitrary denomination set (say a hypothetical ₹300 note existed) greedy can fail: you'd need coin-change dynamic programming. I'd name that trade-off aloud but ship greedy, because real currencies are canonical by design — central banks pick denominations specifically so greedy works."*

**Why this is Strategy and not a helper function:** *"If the ATM operator wants to bias toward dispensing small notes (customer preference, or to burn down small-note inventory), that's a different algorithm with the same interface. Strategy makes it swappable per-ATM. A helper function would force every caller to change."*

---

### Session

```go
type Session struct {
    Card          Card
    InsertedAt    time.Time
    Authenticated bool
    AccountID     string        // populated after successful PIN
    PINAttempts   int
    TxnHistory    []Transaction // this session only
}
```

**On `TxnHistory` being per-session and not persistent:** *"The session-local history exists so the customer can print a mini-statement of what they just did. Cross-session history belongs to the bank, not the ATM — it's the bank's ledger. Keeping session history purely transient makes the ATM stateless across customers, which is what you want."*

---

### BankService

```go
type BankService interface {
    AuthenticatePIN(cardNumber, pin string) (accountID string, err error)
    Balance(accountID string) (int, error)                                    // minor units (paise/cents)
    Debit(accountID string, amount int, idempotencyKey string) error
    Credit(accountID string, amount int, idempotencyKey string) error
    LinkedAccounts(cardNumber string) ([]string, error)
}
```

**On idempotency keys being in the interface signature:** *"This is non-negotiable on Debit and Credit, and it's a design decision worth defending. If the ATM sends 'debit ₹5000' and the response is lost in the network, retrying without a key would double-charge. With a key, the bank sees the duplicate request and returns the original result. A candidate who writes `Debit(accountID, amount) error` has never thought about the failure mode. Putting the key in the interface signature makes it impossible for a caller to forget."*

**On `Balance` returning `int` in minor units, not float:** *"Same rule as anywhere else in financial code — money is integer paise or integer cents. Float multiplication produces rounding errors that will eventually cost someone real money. I'll show all amounts in the code as integers in the smallest unit."*

**On the synchronous interface:** *"All methods here return in-line — call, wait, get result or error. Real inter-bank calls queue behind an ISO 8583 switch and can take seconds or fail with 'pending' rather than success or error. A production interface might return a request token and expose `PollTransaction(token)` for the async cases. I'm assuming synchronous because it's the cleanest way to show the design decisions that matter (idempotency, boundary, injection); the async variant is a straight refinement of the same shape."*

---

## Implementation

*~10-15 minutes. Implement the 2-3 most interesting methods. Start with the happy path, then name the edge cases.*

### Withdraw — the most interesting method

Three-step atomic operation: verify inventory can cover the request, debit the bank, dispense cash. Rollback on failure.

```go
func (t *WithdrawalTxn) Execute(ctx TxnContext) error {
    // 1. Can inventory cover this amount? Local check — no bank call.
    picks, err := ctx.Dispenser.Pick(t.amount, ctx.Inventory.Snapshot())
    if err != nil {
        return fmt.Errorf("cannot dispense %d: %w", t.amount, err)
    }

    // 2. Debit the bank. If this fails, no cash leaves the machine.
    idemKey := fmt.Sprintf("%s-%s", ctx.Session.Card.Number, t.id)
    if err := ctx.Bank.Debit(ctx.Session.AccountID, t.amount, idemKey); err != nil {
        return fmt.Errorf("debit failed: %w", err)
    }

    // 3. Dispense. If this fails after the debit succeeded, reverse the debit.
    if err := ctx.Inventory.Take(picks); err != nil {
        if rbErr := ctx.Bank.Credit(ctx.Session.AccountID, t.amount, "rev-"+idemKey); rbErr != nil {
            // Debited but couldn't dispense AND couldn't roll back.
            // Real ATM: this pages a human and files a reconciliation ticket.
            return fmt.Errorf("dispense failed and rollback failed: dispense=%w rollback=%v", err, rbErr)
        }
        return fmt.Errorf("dispense failed, debit rolled back: %w", err)
    }

    t.dispensedNotes = picks
    return nil
}
```

**Three things to narrate while writing this:**

*"The order matters. I check inventory first — a local map lookup, no network. If inventory can't cover the amount, I never call the bank. This saves a round-trip on rejected requests and avoids a spurious debit-then-refund pair showing up on the customer's statement."*

*"I debit the bank **before** dispensing cash. The reverse order — dispense then debit — feels safer intuitively but is wrong: if the debit fails after cash has been handed out, you're chasing the customer for money. Debit-first means the worst case is 'we owe them a refund', which the reversal path handles cleanly."*

*"The dispense-fails-after-debit-succeeds path is where senior candidates get separated from mid-level. `Inventory.Take` is atomic — it either fully succeeds or fails without touching the tray. If it fails, I reverse the debit with a related idempotency key (`rev-`-prefixed) so the bank can distinguish an original from a reversal. If the reversal itself fails, I return a compound error and — in a real system — that pages a human. I won't pretend I can solve dispense-failed-AND-refund-failed inside a single function."*

> **On "pages a human":** to *page* someone is to send an urgent alert (via PagerDuty, Opsgenie, or similar) to whichever engineer is on-call, waking them up if necessary. Term survives from the old pager devices. Saying "this would page a human" in a design is a deliberate signal: the code detects an impossible state and escalates it rather than guessing at a fix. Senior instinct — know what your code *can't* solve, and hand off cleanly.

---

### EnterPIN — attempts and card retention

```go
const maxPINAttempts = 3

func (hasCardState) EnterPIN(a *ATM, pin string) error {
    accountID, err := a.bank.AuthenticatePIN(a.session.Card.Number, pin)
    if err == nil {
        a.session.Authenticated = true
        a.session.AccountID = accountID
        a.transition(authenticatedState{})
        return nil
    }

    a.session.PINAttempts++
    if a.session.PINAttempts >= maxPINAttempts {
        a.retainCard() // physical action — card captured inside the machine
        a.session = nil
        a.transition(idleState{})
        return errCardRetained
    }
    return errIncorrectPIN
}
```

**The rule to say out loud:** *"After N failed attempts, the ATM keeps the card and returns to idle. This is not just 'log out' — the physical card is pulled past the slot and dropped into an internal retention bin, freeing the slot for the next customer. The retained card sits in the bin until a bank technician opens the machine (usually during the next cash refill) and forwards it to the issuing bank. Modeled here as a hook `retainCard()` that in production would trigger the hardware capture mechanism. The state cleanup is important: I null the session pointer, not just its fields — the next `InsertCard` builds a fresh session with no chance of stale data leaking through."*

**On `maxPINAttempts` as a global constant:** *"I've hardcoded 3 as a constant. Real bank policies vary — some banks want 5, some want per-card overrides for corporate accounts. The clean fix is to have `BankService.SessionPolicy(cardNumber)` return the limit at session start. Not doing it now because a global constant is the honest default and the swap is one interface call away."*

> **On the retention bin:** the internal bin that holds captured cards has finite capacity. Production ATMs track its fill level and either refuse further retentions (degrading security by returning cards on failed auth) or go out of service until a technician empties it. Not modeled in the LLD — it's an operational concern, not a class-design one — but worth knowing if the interviewer probes "what if the bin fills up?"

---

## Verification Walkthrough

*~2-3 minutes. Trace one non-trivial scenario out loud to prove the design holds together.*

Scenario: Customer inserts card, enters wrong PIN twice, then correct PIN, withdraws ₹5000. Inventory starts at 5×₹2000, 10×₹500, 20×₹100.

```
t0  state=Idle              session=nil                     inventory={2000:5, 500:10, 100:20}

t1  InsertCard(C-1234)
    state=HasCard           session={card:C-1234, attempts:0}

t2  EnterPIN("wrong")       bank.AuthenticatePIN → error
    state=HasCard           session.PINAttempts=1           returns ErrIncorrectPIN

t3  EnterPIN("wrong")
    state=HasCard           session.PINAttempts=2           returns ErrIncorrectPIN

t4  EnterPIN("1234")        bank.AuthenticatePIN → accountID="A-9"
    state=Authenticated     session.AccountID="A-9"

t5  SelectTransaction(Withdrawal, 5000)
    state=TransactionInProgress
    returns txn=W-1 (built, not yet executed)

t6  ExecuteTransaction(W-1)
    ├─ Dispenser.Pick(5000) → {2000:2, 500:2}       ✓ inventory can cover
    ├─ Bank.Debit(A-9, 5000, key="C-1234-W-1") → OK
    └─ Inventory.Take({2000:2, 500:2}) → OK,        inventory now {2000:3, 500:8, 100:20}
    state=Authenticated     session.TxnHistory=[W-1]

t7  EjectCard()
    state=Idle              session=nil                     inventory unchanged
```

What each tick confirms:

- `t1`: State transition creates the session — session data has no meaningful existence when Idle.
- `t2-t3`: Wrong PIN in HasCard state increments `PINAttempts` without transitioning. `authenticatedState` never sees these calls — the state machine enforces the guard, not scattered `if` statements.
- `t4`: Successful auth transitions AND populates `AccountID`. The ATM never stored the card→account mapping; the bank returned it. Boundary respected.
- `t5`: Selection builds the Transaction object but doesn't execute — leaves room for a confirmation screen. Transitioning to TransactionInProgress locks out further user input during the async execute.
- `t6`: The three-step atomic sequence — inventory check, bank debit, physical dispense. Inventory decrements only after both prior steps succeed. Idempotency key derived from card and transaction ID means a retried Execute wouldn't double-charge.
- `t7`: `EjectCard` clears the session pointer, not just fields. Next `InsertCard` builds a fresh session; stale data can't leak.

---

## Trade-Offs

*Most trade-offs are named inline at the point of decision — mutex granularity, in-memory inventory, greedy dispensing, synchronous bank calls, `maxPINAttempts` as a constant. What's collected here is scope-level: the things I've explicitly chosen not to model, so the interviewer sees the boundary of the design.*

- **No hardware failure modeling** — jammed cash trays, receipt-printer paper out, torn cards, retention bin full. Each would need its own error path or state. Out of scope by choice; the design has hooks (`retainCard`, the `CardReader` extension) where they'd attach.
- **Network-failure recovery scoped out.** Idempotency keys in the `BankService` interface capture the design decision that matters; full stand-in mode, transaction replay queues, and recovery orchestration are punted to a decorator (see the offline extension below).
- **In-memory throughout.** Cash inventory, session history, and the state machine all live in RAM. A power cycle loses everything. Production would persist inventory and audit log to non-volatile storage; the interfaces don't change, only the implementations.

---

## Extensions

*~5 minutes if time allows. For each follow-up, say where it hooks in and what changes — no need to write full code.*

**"What if the bank returns success on Debit but the response is lost in transit?"**
The idempotency key handles it. Client retries `Debit(A-9, 5000, "C-1234-W-1")`; bank sees the key, returns the same successful result without double-charging. The design decision was making the interface require the key — the retry logic itself is a small loop around the call.

**"How would you handle a torn or unreadable card?"**
`InsertCard` returns an error and stays in Idle without creating a session. Hardware layer decides whether to eject or retain the physical card; a `CardReader` interface would hide that decision from ATM logic. Not modeled here because it's a hardware abstraction, not a class-design concern.

**"How would you support fund transfer to a different bank?"**
`BankService.Transfer(fromAccount, toAccount, amount, idemKey)` becomes `Transfer(fromAccount, toRoutingInfo, amount, idemKey)` where routing info carries a bank identifier. ATM code doesn't change — only the interface and the concrete `BankService` implementation. If real-time inter-bank isn't available, the transaction becomes async: the ATM returns "submitted" instead of "completed" and a separate polling call reports final status.

**"How would you support offline / stand-in mode when the bank is unreachable?"**
A `StandInBankService` decorator wraps the real one. It caches recent balances and permits withdrawals up to a configurable per-card limit while the network is down, queueing debits for replay when connectivity returns. Because `BankService` is an interface, the ATM has no idea whether it's talking to a live switch or a stand-in cache. This is the payoff of injection.

**"How would you audit every transaction across reboots?"**
Every Transaction already has `ID()`, `Timestamp()`, and `Kind()`. Add a `TransactionLog` interface with `Append(txn Transaction)`. Call it from `ATM.ExecuteTransaction` after Execute returns (success or failure). The Command pattern gave us this almost free — every transaction is a serializable data object.

**"How would you handle a maintenance mode where the ATM refuses customer transactions but accepts cash refills?"**
Fifth state: `maintenanceState`. Its `InsertCard` returns "out of service"; an admin path can transition it back to Idle. The state machine gets a new node but the shape of the code doesn't change — that's exactly what the State pattern is for.

---

## References

- `lld/patterns/behavioral/state.md` — State pattern; applied here to `ATMState`
- `lld/patterns/behavioral/command.md` — Command pattern; applied here to `Transaction`
- `lld/patterns/behavioral/strategy.md` — Strategy pattern; applied here to `DispenseStrategy`
- `lld/design-library-management.md` — Sibling case study; State pattern application and injected external-boundary reasoning mirror this file
- `lld/design-parking-lot.md` — Sibling case study; state machine driving a physical resource parallels the ATM's dispense flow