---
Status: 🌳 Evergreen
Created: 2026-08-17
Last Updated: 2026-08-17
---

# Library Management System

## Table Of Contents

1. [Problem Statement](#problem-statement)
2. [Requirements](#requirements)
3. [Core Entities](#core-entities)
4. [Class Design](#class-design)
5. [Implementation](#implementation)
6. [Verification Walkthrough](#verification-walkthrough)
7. [Extensions](#extensions)
8. [Trade-Offs](#trade-offs)
9. [References](#references)

---

## Problem Statement

A Library Management System models the lending lifecycle of a physical library: searching for books, checking them out, returning them, reserving unavailable titles, and renewing loans. It looks simple until you ask one question — *"if the library owns five copies of Clean Code, is that five objects or one?"* — and from there the problem opens into entity modeling, a real object lifecycle, and borrowing rules that vary by member type. That combination makes it a reliable interview problem: the surface is familiar, but the design has enough moving parts to separate shallow from deep.

---

## Requirements

*~5 minutes. The goal is to turn the one-sentence prompt into something concrete enough to design against.*

**Questions worth asking, and what each one is fishing for:**

*"When a member checks out a book, does the system assign a copy, or does the member pick one?"*
In most interview framings the system assigns — which means you need allocation logic and a `CheckoutByISBN` entry point in addition to a `Checkout(copyID)` for kiosk-style barcode scanning.

*"How many books can a member borrow at once — and does that vary by membership type?"*
This is the fishing question for Strategy. If the answer is "it depends on the plan," you introduce `MembershipPlan` as an interface. If it's a fixed global limit, a constant is enough.

*"What happens when someone returns a book late — and does the fine rate vary by plan?"*
Both parts matter. Fine calculation should also be delegated to the plan, not hardcoded in `Return()`.

*"What if all copies of a title are checked out and someone wants it?"*
Opens the reservation queue. The follow-up: *"Do they reserve a specific copy or 'any copy of this title'?"* Almost always: the title, not a barcode. That drives the FIFO queue keyed by ISBN.

*"Can members renew, and are there limits?"*
Gets `maxRenewals` and the rule: *"can't renew if someone else is waiting."*

*"What's explicitly out of scope?"*
Don't wait to be told. Say it yourself: *"I'll treat fine collection and durable storage as injected interfaces — I'll define the boundary but not build a payment gateway or a database. Authentication is upstream. Multi-branch transfers are an extension I can describe if there's time. Does that scope work?"*

**Final scope:**

| In | Out |
|---|---|
| Search by title, author, genre | Payment processing |
| Checkout (by ISBN and by copy ID) | Member authentication |
| Return with fine calculation | Multi-branch transfers |
| FIFO reservation queue | Persistence / database |
| Renewal with restrictions | UI / notifications infrastructure |

---

## Core Entities

*~5 minutes. Sketch the structure before writing any code — communicates intent to the interviewer and catches modeling mistakes early.*

What I'd put on the whiteboard:

```
Concrete types            Interfaces (behaviour that varies)
─────────────────         ────────────────────────────────
Book                      MembershipPlan   ← borrow limits, fine rate
BookCopy                  CopyState        ← per-state behaviour
Member                    NotificationService
Loan
Catalog
LibraryService  ← orchestrator, entry point for all operations
```

Ownership and dependencies — what I'd draw as arrows:

```
LibraryService  owns    Loan                                (I create it — &Loan{} lives in Checkout)
LibraryService  uses    Catalog, NotificationService        (injected — built outside, handed in)
LibraryService  refs    Member                              (registered — built outside, I just hold a pointer)
Catalog         owns    Book, BookCopy
Member          has-a   MembershipPlan                      (Strategy)
BookCopy        has-a   CopyState                           (State — swaps itself)
Loan            refs    BookCopy, Member                    (by ID string, not by pointer)
```

> **owns vs refs:** *owns* = this type is what calls `&Loan{...}`; the object didn't exist before and this type is responsible for it. *refs* = the object was created elsewhere and handed in; `RegisterMember` takes a `*Member` that someone else built. Same distinction as composition vs aggregation in UML.

**The key insight to say out loud:**

*"The most important modeling decision in this design is splitting Book from BookCopy. Book is title-level metadata — one per ISBN. BookCopy is one physical, barcoded object on the shelf. Five copies of the same title means one Book and five BookCopies. This distinction drives everything: search and reservation queues operate on Books (ISBNs), checkout and state transitions operate on BookCopies."*

**Why Loan references by ID, not by pointer:** a copy can be removed from the catalog — lost, damaged, transferred — and you want the loan record to survive as a historical artifact. Reference by ID string means no dangling pointer.

**Concrete implementers** (worth naming now so the interviewer sees the full picture):
- `CopyState`: `availableState`, `checkedOutState`, `reservedState`, `lostState`
- `MembershipPlan`: `RegularPlan`, `PremiumPlan`
- `NotificationService`: `ConsoleNotifier`

---

## Class Design

*~10-15 minutes. Go entity by entity, orchestrator first. For each: state (what it holds), then behaviour (what it does). Explain decisions as you write them — the reasoning is as important as the structure.*

### LibraryService

```go
const (
    maxRenewals    = 1 // per loan
    holdWindowDays = 2 // days a returned copy is held for the next waiter
)
```

```go
type LibraryService struct {
    catalog      *Catalog
    notify       NotificationService
    mu           sync.Mutex
    members      map[string]*Member
    loans        map[string]*Loan
    activeCount  map[string]int                  // memberID → open loan count
    reservations map[string][]reservationRequest // ISBN → FIFO queue
    nextLoanID   int                             // increment on each checkout
}
```

**While writing the mutex:** *"I'm using a single mutex on the whole service. That serializes checkout, return, reserve, and renew across the entire library — even for unrelated titles. At this scope it's the simplest correct answer. I'd flag it proactively: at multi-branch scale, per-branch locking would be the right move. But I won't over-engineer it now."*

**On `activeCount`:** *"I'm maintaining a per-member loan count as a separate field rather than scanning loans on every checkout. Without it, checking the borrow limit means iterating all loans and counting where `MemberID == m AND ReturnDate == nil` — O(n) in total loan history, which only grows over time. With the counter, it's a single map lookup: `activeCount[memberID]`. One extra field, O(1) check forever. Increment on checkout, decrement on return."*

**On not using Singleton for Catalog:** *"I'm injecting Catalog as a dependency rather than making it a global Singleton. Two reasons: testability — a global means every test shares the same state and can't run in isolation. More importantly, correctness — a Singleton encodes the assumption that exactly one Catalog can ever exist. That assumption breaks the moment the library opens a second branch: downtown and uptown each need their own catalog. With injection, you construct one `*Catalog` per branch and hand it to each `LibraryService`. With a Singleton, you'd have to unwind the global at that point, which is painful. Injection costs nothing now."*

Methods: `RegisterMember`, `Checkout`, `CheckoutByISBN`, `Return`, `Reserve`, `Renew`.

---

### Catalog

```go
type Catalog struct {
    mu     sync.RWMutex
    books  map[string]*Book       // ISBN → Book
    copies map[string][]*BookCopy // ISBN → all physical copies
    byID   map[string]*BookCopy   // copyID → BookCopy, O(1) lookup
}
```

**On search:** *"Search is a simple case-insensitive substring scan over titles, authors, and genres. I'd name it honestly: in production with a large catalog, this would be an inverted index — but that's an HLD search-systems problem, not a class-design one. For this scope, linear scan is correct and I won't pretend otherwise."*

Methods: `AddBook`, `FindCopy`, `FirstAvailableCopy`, `SearchByTitle`, `SearchByAuthor`, `SearchByGenre`.

---

### BookCopy and the State Pattern

```go
type BookCopy struct {
    ID           string
    ISBN         string
    state        CopyState  // unexported — transitions only through methods
    checkedOutBy string
    reservedFor  string
    holdUntil    time.Time
}
```

**While writing this:** *"BookCopy has four states with genuinely different legal operations in each — Available, CheckedOut, Reserved, and Lost. Plus a real terminal state. That's the trigger for the State pattern: three or more states, meaningfully different behaviour per state, one terminal state. I'll model each as a small struct implementing a `CopyState` interface."*

```go
type CopyState interface {
    Name() string
    CheckOut(c *BookCopy, memberID string) error
    Return(c *BookCopy) error
    Reserve(c *BookCopy, memberID string, holdUntil time.Time) error
    ExpireHold(c *BookCopy) error
    MarkLost(c *BookCopy) error
}
```

The state machine in compact form:

```
[Available] --CheckOut(m)--> [CheckedOut] --Return()--> [Available]
[Available] --Reserve(m)---> [Reserved]
[Reserved]  --CheckOut(m), only if m == holder--> [CheckedOut]
[Reserved]  --ExpireHold()--> [Available]
Any         --MarkLost()---> [Lost]   ← terminal, no transitions out
Everything not shown above → error
```

**On why the concrete states are lowercase (unexported):** *"In Go, capitalization is visibility — not just style. If I exported these, caller code could construct `lms.CheckedOutState{}` and hand-build a BookCopy in that state, bypassing transition validation entirely. The interface is exported because it's the contract. The four concrete states are a closed set that belongs entirely inside this package."*

To illustrate the pattern — Available and the terminal Lost state:

```go
type availableState struct{}

func (availableState) CheckOut(c *BookCopy, memberID string) error {
    c.checkedOutBy = memberID
    c.transition(checkedOutState{})
    return nil
}
func (availableState) Reserve(c *BookCopy, memberID string, holdUntil time.Time) error {
    c.reservedFor = memberID
    c.holdUntil = holdUntil
    c.transition(reservedState{})
    return nil
}
func (availableState) Return(*BookCopy) error    { return errors.New("invalid transition") }
func (availableState) ExpireHold(*BookCopy) error { return errors.New("invalid transition") }
func (availableState) MarkLost(c *BookCopy) error { c.transition(lostState{}); return nil }

type lostState struct{} // terminal — every operation is an error
func (lostState) CheckOut(*BookCopy, string) error              { return errors.New("invalid transition") }
func (lostState) Return(*BookCopy) error                        { return errors.New("invalid transition") }
func (lostState) Reserve(*BookCopy, string, time.Time) error    { return errors.New("invalid transition") }
func (lostState) ExpireHold(*BookCopy) error                    { return errors.New("invalid transition") }
func (lostState) MarkLost(*BookCopy) error                      { return errors.New("invalid transition") }
```

`checkedOutState` and `reservedState` follow the same pattern — same interface, different subset of operations permitted.

*"In production these error strings would be named sentinel errors — `errors.New(...)` declared once in a shared types file — so callers can use `errors.Is`. I'm writing them inline here to keep the class design moving."*

---

### Member and MembershipPlan

```go
type Member struct {
    ID   string
    Name string
    Plan MembershipPlan
}

type MembershipPlan interface {
    MaxBooksAllowed() int
    LoanDurationDays() int
    CalculateFine(daysOverdue int) Money
}
```

**While writing this:** *"LibraryService never switches on membership type. It just asks the plan. Adding a third tier — say, a Student plan with different limits — means implementing this interface once. Nothing in Checkout, Return, or Renew changes."*

The concrete plans, written with named constants:

```go
const (
    regularMaxBooks  = 3
    regularLoanDays  = 14
    regularFineRate  = 0.50 // per day

    premiumMaxBooks  = 10
    premiumLoanDays  = 28
    premiumFineRate  = 0.20
)

type RegularPlan struct{}
func (RegularPlan) MaxBooksAllowed() int        { return regularMaxBooks }
func (RegularPlan) LoanDurationDays() int       { return regularLoanDays }
func (RegularPlan) CalculateFine(days int) Money {
    return Money{Amount: round2(regularFineRate * float64(days))}
}

type PremiumPlan struct{}
// same shape, different constants
```

**On the named constants:** *"I'm not writing bare 3s and 14s. These are business policy — a product manager could change the Regular fine rate tomorrow. A named constant makes that a one-line diff instead of a grep-and-hope."*

---

### Loan

```go
type Loan struct {
    ID           string
    CopyID       string
    MemberID     string
    CheckoutDate time.Time
    DueDate      time.Time
    ReturnDate   *time.Time // nil = not yet returned
    RenewalCount int
    Fine         Money
}
```

Two things worth calling out:

**On `ReturnDate *time.Time`:** *"`ReturnDate` is a pointer, not a value. `nil` means 'not returned yet' — that's meaningfully different from a zero `time.Time`, which looks like a return that happened at the Unix epoch. The pointer makes the nil case explicit and checkable."*

**On why Loan doesn't get the State pattern:** *"Loan only has two states — open and returned — with one irreversible transition. The entire state machine is `loan.ReturnDate != nil`. Forcing a LoanState interface on this would be ceremony with no payoff. The trigger for State is three or more states with meaningfully different behaviour per state — Loan doesn't clear that bar."*

---

## Implementation

*~10-15 minutes. Implement the 2-3 most interesting methods. Start with the happy path, then name the edge cases.*

### CheckoutByISBN — the walk-up case

The most common path: a member wants "any copy of this title," not a specific barcode.

```go
func (s *LibraryService) CheckoutByISBN(memberID, isbn string) (*Loan, error) {
    s.mu.Lock()
    defer s.mu.Unlock()

    member, ok := s.members[memberID]
    if !ok {
        return nil, errors.New("member not found")
    }
    if s.activeCount[memberID] >= member.Plan.MaxBooksAllowed() {
        return nil, errors.New("borrow limit reached")
    }
    cp, err := s.catalog.FirstAvailableCopy(isbn)
    if err != nil {
        return nil, err
    }
    if err := cp.CheckOut(memberID); err != nil {
        return nil, err
    }

    now := time.Now()
    s.nextLoanID++
    loan := &Loan{
        ID:           fmt.Sprintf("L-%d", s.nextLoanID),
        CopyID:       cp.ID,
        ISBN:         isbn,
        MemberID:     memberID,
        CheckoutDate: now,
        DueDate:      now.AddDate(0, 0, member.Plan.LoanDurationDays()),
    }
    s.loans[loan.ID] = loan
    s.activeCount[memberID]++
    return loan, nil
}
```

**Critical thing to say while writing the lock:** *"I'm locking before the borrow-limit check and the copy claim — not just before the claim. If I locked only the claim, two goroutines could both pass the limit check before either one increments the counter. Classic TOCTOU (Time Of Check To Use — you check a condition, another goroutine changes the world before you act on it, so your action is based on stale information). The check-and-claim have to be atomic under the same lock."*

---

### Return — the most interesting method

`Return` does three things in one: closes the loan, calculates any fine through the member's plan, and hands the copy to the next waiter if the reservation queue is non-empty.

```go
func (s *LibraryService) Return(loanID string) (Money, error) {
    s.mu.Lock()
    defer s.mu.Unlock()

    loan, ok := s.loans[loanID]
    if !ok {
        return Money{}, errors.New("loan not found")
    }
    if loan.ReturnDate != nil {
        return Money{}, errors.New("already returned")
    }

    cp, _ := s.catalog.FindCopy(loan.CopyID) // exists — loan proves it was checked out
    cp.Return()                               // state machine: CheckedOut → Available

    now := time.Now()
    loan.ReturnDate = &now
    s.activeCount[loan.MemberID]--

    var fine Money
    if now.After(loan.DueDate) {
        daysOverdue := int(now.Sub(loan.DueDate).Hours() / 24)
        if daysOverdue < 1 {
            daysOverdue = 1 // any lateness = at least 1 day
        }
        fine = s.members[loan.MemberID].Plan.CalculateFine(daysOverdue)
        loan.Fine = fine
    }

    // If anyone is waiting for this title, hand the copy directly to them
    if queue := s.reservations[loan.ISBN]; len(queue) > 0 {
        next := queue[0]
        s.reservations[loan.ISBN] = queue[1:]
        cp.Reserve(next.MemberID, now.AddDate(0, 0, holdWindowDays))
        s.notify.Notify(next.MemberID, loan.ISBN+" is ready for pickup")
    }

    return fine, nil
}
```

**Three things to narrate while writing this:**

*"Fine calculation goes through `member.Plan.CalculateFine` — LibraryService never switches on membership type. That's Strategy doing real work."*

*"After Return, I check the reservation queue. If it's non-empty I call `cp.Reserve` rather than leaving the copy generally Available. The copy goes Available → Reserved in the same locked section, before Return returns to the caller. The state machine allows this: Return only ever lands on availableState, and availableState accepts Reserve."*

*"Two things share the name Reserve here — worth naming out loud to the interviewer. `LibraryService.Reserve` puts a member on the waiting list for a title; it never touches a physical copy. `BookCopy.Reserve` puts a specific copy on hold for a specific member; it's called by Return once a copy comes back. Queue on the title, hold on the copy."*

---

### Renew — the queue-blocks-renewal rule

```go
func (s *LibraryService) Renew(loanID string) error {
    s.mu.Lock()
    defer s.mu.Unlock()

    loan, ok := s.loans[loanID]
    if !ok {
        return errors.New("loan not found")
    }
    if loan.ReturnDate != nil {
        return errors.New("already returned")
    }
    if loan.RenewalCount >= maxRenewals {
        return errors.New("renewal limit reached")
    }
    if len(s.reservations[loan.ISBN]) > 0 {
        return errors.New("renewal blocked: reservation pending")
    }

    member := s.members[loan.MemberID]
    loan.DueDate = loan.DueDate.AddDate(0, 0, member.Plan.LoanDurationDays())
    loan.RenewalCount++
    return nil
}
```

**The rule to say out loud:** *"Renewing into a non-empty reservation queue means the waiting member waits forever — the book never returns to circulation. Checking the queue before renewing is what keeps the system fair."*

---

## Verification Walkthrough

*~2-3 minutes. Trace one non-trivial scenario out loud to prove the design holds together.*

Scenario: one copy of ISBN-1 (`C-1`). M-1 checks it out. M-2 reserves the title. M-1 returns it 3 days late.

```
t0  C-1=Available    M-1.loans=0  M-2.loans=0  queue=[]

t1  M-1 checks out C-1
    C-1=CheckedOut   M-1.loans=1  M-2.loans=0  queue=[]

t2  M-2 reserves ISBN-1     ← joins queue, no copy touched
    C-1=CheckedOut   M-1.loans=1  M-2.loans=0  queue=[M-2]

t3  3 days pass, loan is overdue

t4  M-1 returns C-1
    fine = $1.50  (0.50 × 3 days, from RegularPlan.CalculateFine)
    queue non-empty → C-1 reserved for M-2, M-2 notified
    C-1=Reserved     M-1.loans=0  M-2.loans=0  queue=[]

t5  M-1 tries to check C-1 out immediately
    → rejected: copy reserved for another member
    C-1=Reserved (hold intact — rejected attempt doesn't degrade the hold)

t6  M-2 checks C-1 out
    C-1=CheckedOut   M-1.loans=0  M-2.loans=1  queue=[]
```

What each tick confirms:

- `t1`: borrow-limit check and copy claim are atomic — no TOCTOU.
- `t2`: queue grows without touching the physical copy — reservations are per-title, not per-copy.
- `t4`: fine comes from the plan (not from a switch in LibraryService); copy lands in Reserved not Available (hand-to-waiter path fires); notification fires before Return returns.
- `t5`: `reservedState.CheckOut` checks `memberID == reservedFor` before transitioning — wrong member gets `ErrReservedForAnotherMember`, hold stays intact.
- `t6`: correct holder transitions Reserved → CheckedOut through the normal checkout path.

---

## Extensions

*~5 minutes if time allows. For each follow-up the interviewer might raise, say where it hooks in and what changes — no need to write full code.*

**"How would you persist loans across restarts?"**
Define a `LoanRepository` interface — `Save(loan)` and `Get(loanID)`. `LibraryService` talks to the interface, not to the in-memory map directly. Swap in a Postgres implementation without touching any lending logic.

**"How would you collect fines, not just calculate them?"**
`FinePaymentProcessor` interface — `Charge(amount, method, idempotencyKey)`. `Return` already knows the amount via `Plan.CalculateFine`. Inject the processor alongside `notify`. Two separate injected interfaces: "how much" (MembershipPlan) and "how to collect" (FinePaymentProcessor) are two different concerns.

**"How would you notify members when a new title they'd like arrives?"**
Observer on `Catalog`. `CatalogObserver` interface with `OnNewArrival(book)`. Members subscribe to genres. `AddBook` notifies observers when it sees a title for the first time — adding a second copy of an existing title doesn't re-fire.

**"How would you support multiple branches?"**
This is the payoff of not using Singleton. `Branch` is a struct with an ID and a `*Catalog`. Transfer a copy by calling `RemoveCopy` on the source catalog and `AddCopy` on the destination. Each branch has its own `LibraryService` with its own catalog — reservations and loan counts stay per-branch already.

**"What if a copy goes missing?"**
`MarkLost()` transitions the copy to `lostState` — terminal. That's deliberate: losing a book is an administrative anomaly, not part of the normal lending lifecycle. If the copy turns up, a librarian calls `ResolveFoundCopy` — an admin function that bypasses the state machine and puts the copy back to Available. Members can never call it.

**"What if a member never picks up their hold?"**
`ExpireHold()` already exists as the primitive — it transitions `Reserved → Available` and clears `reservedFor`. The open question is who calls it and when. That's a scheduling/infrastructure decision (a cron job scanning Reserved copies past `holdUntil`, a delayed queue message, a DB TTL trigger) — not a class-design one. Name it, flag the mechanism as TBD, and move on.

---

## Trade-Offs

*Name these proactively — before the interviewer asks. It signals you've thought past the happy path.*

- **Single mutex on LibraryService** serializes every operation across the whole library. Simplest correct answer at this scale. Multi-branch or high-concurrency: per-branch or per-title locking.
- **Linear search** (`SearchByTitle` etc.) is O(titles). Correct for one branch's catalog. Large union catalog: inverted index — covered in `hld/search-systems.md`.
- **Float-based Money** is convenient for illustration. Production: integer minor units (cents) — float multiplication produces rounding errors in financial calculations.
- **`maxRenewals` is a global constant**, not a per-plan field. A real system would likely want the renewal cap to vary by tier, the same way borrow limit and fine rate already do.
- **No hold-expiry scheduler**. `ExpireHold()` exists; something needs to call it. Named in Extensions above — infrastructure decision, not modeled here.

---

## References

- `lld/patterns/behavioral/state.md` — State pattern theory; applied here to `CopyState`
- `lld/patterns/behavioral/strategy.md` — Strategy pattern theory; applied here to `MembershipPlan`
- `lld/patterns/behavioral/observer.md` — Observer pattern theory; applied in the new-arrival extension
- `lld/patterns/creational/singleton.md` — Singleton analysis; specifically why it's rejected for `Catalog`
- `hld/search-systems.md` — Inverted-index search; the right answer for catalog-at-scale
- `lld/design-parking-lot.md` — Sibling case study; concurrency locking argument mirrors `LibraryService`