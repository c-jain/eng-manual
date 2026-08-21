---
Status: 🌳 Evergreen
Created: 2026-08-21
Last Updated: 2026-08-21
---

# Hotel Booking System

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

A hotel booking system lets a guest search for available rooms over a date range, reserve one, pay, and later cancel or check in and out. What makes it a real design problem (not a CRUD exercise) is that rooms are booked over *intervals*, not moments: availability is a function of `(room, day)`, and the primary correctness question is "did two guests just book the same room for overlapping nights?" That question is answered inside a small, high-contention critical section, and everything else in the design (payment, pricing, cancellation policy) either flows through it or lives outside it. Getting that boundary right is the interview.

---

## Requirements

*~5 minutes. The interviewer says "design a hotel booking system." Ask the questions that shape the design.*

**Questions I'd ask, and why I'm asking each one:**

*"Does a guest book a specific room, or a room type?"*
Surface purpose: clarify UX. Real purpose: shape the availability index. Real hotel chains often search on type (`Deluxe Double`) with specific-room assignment deferred to check-in, because it lets housekeeping optimise. But single-property boutiques and premium suites book specific rooms. I'll ship specific-room booking (simpler, cleaner overlap check per room) and flag type-level as an extension. That framing avoids modeling both simultaneously and keeps the search algorithm boring.

*"Do we need to enforce that check-in and check-out on the same day don't overlap?"*
Surface purpose: an edge case. Real purpose: pin down the interval convention. Hotels use half-open ranges: `[checkIn, checkOut)`. A guest checking out Monday morning and another checking in Monday afternoon share zero nights, so the two bookings must be allowed. If I don't name this, my overlap check will reject valid adjacent bookings and I'll waste time debugging on the whiteboard. Naming it early earns the `DateRange.Overlaps` method its shape.

*"Does the price vary by date, or is it a flat nightly rate?"*
Surface purpose: pricing complexity. Real purpose: earn `PricingStrategy` as an interface. Weekend surcharges, seasonal peaks, occupancy-based dynamic pricing all fit the same signature `Price(room, dates) int`. If the answer is "flat rate," I'd skip the interface. If "yes it varies," Strategy makes each policy swappable without touching the booking flow.

*"Payment: sync while the API call blocks, or async after we hold the room?"*
Surface purpose: UX. Real purpose: the single most important design decision in this whole system. Payment gateways take seconds and occasionally minutes. Holding a mutex on the availability index across that call would serialise every booking behind every payment. The senior-signal answer is two-phase: reserve availability with a short TTL (a "hold"), then process payment outside the lock, then promote the hold to a confirmed booking. I want the interviewer to explicitly agree to this before I invest whiteboard time in the state machine.

*"What's the cancellation policy: full refund, tiered, non-refundable?"*
Surface purpose: business rule. Real purpose: earn a `RefundPolicy` seam (or, at interview scope, decide it's out and pick "full refund" as the demo). The point of asking is to show I'd model it as a policy rather than an `if` block in `Cancel()`, even if the demo implementation is trivial.

> [!NOTE]
> **On the word "seam":** a term from Michael Feathers' *Working Effectively with Legacy Code*. A seam is a point in the code where behaviour can be changed without editing that place. The mental image is a seam in fabric: the join where two pieces meet, and where you cut when you want to alter the garment. Same idea in code: the boundary between orchestration and varying policy is where you'd cut to change behaviour, and turning that accidental boundary into an explicit interface is what makes it a seam rather than a coincidence. Sibling terms in use: "extension point", "injection point", "hook".
>
> Concrete contrast. Without a seam, refund logic lives inside `Cancel` and every new policy is a new branch:
>
> ```go
> func (s *HotelBookingSystem) Cancel(id string) error {
>     // ...
>     daysToCheckIn := b.Dates.CheckIn.Sub(s.now()).Hours() / 24
>     var refund int
>     switch {
>     case daysToCheckIn > 7:
>         refund = b.TotalAmount        // full
>     case daysToCheckIn > 2:
>         refund = b.TotalAmount / 2    // half
>     default:
>         refund = 0                    // none
>     }
>     // ...
> }
> ```
>
> With a seam, `RefundPolicy` is an interface, injected once in the constructor. `Cancel` never changes; swapping `FullRefundPolicy` for `TieredRefundPolicy` changes the outcome. Tests inject a stub.
>
> ```go
> type RefundPolicy interface {
>     RefundAmount(b *Booking, now time.Time) int
> }
>
> func (s *HotelBookingSystem) Cancel(id string) error {
>     // ...
>     refund := s.refundPolicy.RefundAmount(b, s.now())
>     // ...
> }
> ```

*"Can a booking be modified (change dates, extend by one night), or only cancel-and-rebook?"*
Surface purpose: feature scope. Real purpose: state graph size. Modification is a partial re-run of `Hold` for the new dates, with rollback to the original if the new dates fail. Legal but adds branching. I'll ship cancel-and-rebook and flag modification as an extension.

*"What's explicitly out of scope?"*
Don't wait to be told, offer it: *"Multi-hotel/chain aggregation, user account management, loyalty points, physical key issuance, housekeeping workflow, real-time channel sync with Booking.com/Expedia, dynamic overbooking based on no-show statistics, and the payment gateway internals are all out. I'll model payment as an injected `PaymentGateway` interface. Does that scope work?"*

**Final scope:**

| In | Out |
|---|---|
| Search rooms by type and date range | Multi-hotel aggregation / chain search |
| Two-phase booking: Hold (with TTL) then Confirm | Loyalty programs, user accounts, auth |
| Overlap-safe reservation for a specific room | Type-level pooling with deferred room assignment |
| Pricing via a strategy (flat, weekend-surcharge) | Dynamic pricing based on live occupancy |
| Cancel from Held or Confirmed (full refund policy) | Tiered / non-tiered refund policies |
| Check-in and check-out lifecycle | Housekeeping, key issuance, minibar billing |
| Injected payment gateway (charge + refund, idempotent) | Payment gateway internals, chargeback handling |
| In-memory availability index | Channel-manager sync (Booking.com, Expedia) |

---

## Core Entities

*~5 minutes. Sketch structure before writing code. Communicates intent and catches modeling mistakes early.*

What I'd put on the whiteboard:

```
Concrete types              Interfaces (behaviour that varies)
─────────────────           ────────────────────────────────
HotelBookingSystem          PricingStrategy    ← price(room, dates) → amount
Room                        PaymentGateway     ← Charge, Refund (external boundary)
Guest
Booking
DateRange
AvailabilityIndex
```

Ownership and dependencies:

```
HotelBookingSystem  owns   AvailabilityIndex, bookings map, rooms map, guests map
HotelBookingSystem  uses   PricingStrategy, PaymentGateway         (injected)
AvailabilityIndex   owns   per-room sorted list of bookingRef      (the concurrency-critical bit)
Booking             is-a   value with a status field               (see below on State pattern)
```

**The key insight to say out loud:**

*"There are two data structures for reservations, not one, and they carry different concerns. The `bookings` map is a lookup by ID: given a booking number, fetch the record. It's read-heavy and only mutated on lifecycle transitions. The `AvailabilityIndex` is the concurrency-critical structure: it holds a sorted list of active reservations per room, and it's the sole owner of the overlap-check invariant. Every check for 'is this room free' and every reserve happens under its lock, and nothing else does. Collapsing the two into a single 'bookings by room' map would work but would tie every read of booking metadata (which happens for cancellations, check-in, refunds) to the availability lock, and the whole system would serialise on that mutex."*

**Why I'm *not* using the State pattern for Booking status**, even though there are five statuses (Held, Confirmed, CheckedIn, CheckedOut, Cancelled): the transitions here are driven by *external* triggers (payment completes, staff scans a passport, guest walks out), and the code that runs on each transition needs to talk to the `AvailabilityIndex` and the `PaymentGateway`, both of which live on the service. Putting that logic inside per-status structs would drag the whole service surface into the booking package. In the vending machine and ATM cases, states had genuinely local behaviour and self-transitioned based on inputs the state itself could process; that's the trigger for the pattern. Here, an enum-plus-guarded-transitions in the service methods is more honest about the actual code shape. I'll name this trade-off aloud rather than reach for State by reflex.

**Why I'm *not* using Command either**: the ATM had four transaction types (withdraw, deposit, transfer, balance) with the same `Execute(ctx)` shape and different combinations of bank calls. Here the operations (`Hold`, `Confirm`, `Cancel`, `CheckIn`, `CheckOut`) have genuinely different signatures and different callers; wrapping them in a Command would add a type with no siblings. Pattern-dropping.

**What actually earns a pattern here**:
- **Strategy** for `PricingStrategy` (flat, weekend, seasonal, dynamic occupancy all fit the same signature)
- The **two-phase commit** shape of Hold-then-Confirm, which is not a GoF pattern but is the standard idiom for "reserve now, complete later with external I/O in between"

---

## Class Design

*~10-15 minutes. Go entity by entity, orchestrator first. State first, then behaviour. Explain decisions as you write them.*

*Trivial data holders (`Guest`, `Room`) get stubs only. `AvailabilityIndex` and `HotelBookingSystem` carry the design weight.*

### DateRange

```go
type DateRange struct {
    CheckIn  time.Time // inclusive
    CheckOut time.Time // exclusive: hotel-industry half-open convention
}

func (d DateRange) Nights() int { return int(d.CheckOut.Sub(d.CheckIn).Hours() / 24) }

func (d DateRange) Overlaps(o DateRange) bool {
    return d.CheckIn.Before(o.CheckOut) && o.CheckIn.Before(d.CheckOut)
}
```

**On the half-open convention:** *"The hotel industry treats check-out day as the boundary, not a night. A guest booked `Jan 1 to Jan 3` has slept two nights (Jan 1 and Jan 2) and vacates the morning of Jan 3. Another guest can book `Jan 3 to Jan 5` starting that same afternoon, and the two must not conflict. The overlap predicate `a.CheckIn.Before(b.CheckOut) && b.CheckIn.Before(a.CheckOut)` is exactly this: strict `Before`, not `!After`, so equal boundaries don't count as overlap. Getting this predicate right is the single most important line in the file: a `<=` where a `<` belongs will silently reject every adjacent booking, and the tests will look correct until someone tries to book back-to-back."*

### Room, Guest, Booking

```go
type Room struct {
    ID       string
    Number   string
    Type     RoomType
    BaseRate int // per night, minor units (paise/cents)
}

type Guest struct { ID, Name, Email string }

type Booking struct {
    ID          string
    RoomID      string
    GuestID     string
    Dates       DateRange
    TotalAmount int
    Status      BookingStatus
    HoldExpiry  time.Time // meaningful only while Status == HELD
    CreatedAt   time.Time
    PaymentID   string    // populated on Confirm
}
```

**On `BaseRate` and `TotalAmount` being ints in minor units:** same rule as any code that touches money. Never floats. Integers in the smallest unit (paise, cents) sidestep the entire class of rounding bugs; the presentation layer divides by 100 at the very last moment.

**On `PaymentID` living on the Booking:** it's the audit trail for the refund. Without it, cancelling a confirmed booking would need a separate lookup in the gateway to find "the charge for this booking," which is exactly the kind of cross-system join that fails during an outage. Storing it locally makes cancel a self-contained operation.

> [!NOTE]
> **Audit trail, cross-system join, and the general lesson.**
>
> *Audit trail.* From Latin *audire*, "to hear": Roman ledgers were verified by reading them aloud to an *auditor* (a hearer). The activity became "auditing" long after nobody was actually listening. A *trail* is the sequence of breadcrumbs you leave as you move, so a follower can reconstruct where you went. Put together: **the trail an auditor follows.** In this system, `PaymentID` on the booking is one such breadcrumb: "the confirmation of bk-4711 was caused by charge pay-abc." Combined with timestamps, status transitions, and idempotency keys, the sequence lets anyone (finance, support, a regulator, a debugger at 2 AM) walk backward through what happened. The design-time question the framing forces: *"if an auditor showed up tomorrow and asked what happened to booking X, could I answer using only what I recorded?"* If the answer is "I'd have to call the gateway and hope they still have the record," the trail has a gap.
>
> *Cross-system join.* A "join" in this sense means assembling a complete picture by pulling pieces from two systems and stitching them together. For cancel-without-PaymentID, the pieces are our booking record (our DB) and the gateway's charge record (their DB); cancel needs both. Which means cancel now depends on the gateway being reachable *and* responsive for a lookup call, before we even attempt the refund. Two failure modes chained: the lookup can fail (we don't even know what to refund) or the refund can fail (clearer error). Storing `PaymentID` locally collapses the two into one: given our own data, one external call carries out the operation. That is what "self-contained" means here.
>
> *The general lesson.* **When a value crosses a system boundary in your favour, capture it.** Every time our code calls an external system and gets back an identifier, store that identifier on the local record that caused the call. Same instinct as storing `MessageID` after sending an email, `TransactionID` after a bank transfer, `OrderID` from a supplier, `TrackingID` from a courier. The local record grows a foreign key into every external system it has touched, and every future question about "did this happen?", "what happened?", "undo it" becomes a fast local lookup instead of a fuzzy multi-system reconciliation. Throwing the identifier away and looking it up later is choosing to pay for information you already had for free.

---

### AvailabilityIndex: The Concurrency-Critical Type

```go
type bookingRef struct {
    bookingID string
    dates     DateRange
    status    BookingStatus // HELD or CONFIRMED (Cancelled/CheckedOut are removed, not tracked)
    holdExp   time.Time
}

type roomBookings struct {
    entries []bookingRef // sorted by CheckIn ascending
}

type AvailabilityIndex struct {
    mu    sync.Mutex
    rooms map[string]*roomBookings
    now   func() time.Time
}
```

**While writing the mutex:** *"This is the single most contended mutex in the system. Every search, every hold, every confirm, every cancel touches it. The critical section is deliberately tiny: overlap check plus (maybe) an insert into a small sorted slice. No I/O, no external calls, no expensive computation. Everything slow (pricing, payment) happens outside this lock. If profiling later showed this mutex was a bottleneck, the next step is per-room mutexes (shard the map, one lock per bucket), which is a change to this struct only. I want the interviewer to see that the choice was single-mutex-for-simplicity, not single-mutex-because-I-didn't-think-about-it."*

**On why only `HELD` and `CONFIRMED` entries live in the index:** *"Cancelled bookings free up the room, so leaving them in would force every overlap check to filter them out. Checked-out bookings are in the past, so they can't overlap any future search. Both are removed on transition, keeping the per-room list short in steady state. The `bookings` map on the service still keeps the full history for audit and refund lookups. This is the two-structures split earning its place."*

**On the `now func() time.Time`:** injectable clock. Real-code convention that pays for itself the first time a test needs to simulate a hold expiring. Costs nothing at runtime.

Methods (behaviour narrated in Implementation):

```go
func (a *AvailabilityIndex) Reserve(roomID string, d DateRange, id string, holdExp time.Time) error
func (a *AvailabilityIndex) Confirm(roomID, bookingID string) error
func (a *AvailabilityIndex) Release(roomID, bookingID string) error
func (a *AvailabilityIndex) Available(rooms []Room, t RoomType, d DateRange) []Room
```

**On the O(k) overlap scan being fine here:** *"k is the number of *active* reservations for one specific room, which for a real hotel is typically single-digit at any given moment: the room is either free or booked once. Even a stress case (long-term stay overlapping many day bookings) stays small. If profiling ever showed this scan mattered, the swap is to an interval tree per room, same interface. The current shape is a sorted slice because it's the code an interviewer expects and it's fast in the L1 cache."*

**On pruning expired holds inline in `Reserve`:** *"Lazy eviction. A hold whose TTL has passed is functionally cancelled but the index doesn't know it yet. I could run a background sweeper, but every `Reserve` and `Available` already scans the list, so I prune during that scan. Zero extra data structures, one extra `time.Now()` per call. If the scan somehow became expensive under scale, a per-room next-expiry timestamp lets us skip the prune when nothing's due. Named the alternative aloud."*

> [!NOTE]
> **Piggybacking as a design instinct.** The lazy-eviction choice above is a specific case of a general rule worth naming: **prefer piggybacking on work you're already doing over adding new machinery.** A background sweeper is a whole apparatus: a goroutine to spawn and shut down, a ticker interval to tune, a lock acquisition every N seconds even when nothing needs cleaning, panic recovery, tests for the scheduler itself. Pruning during the existing scan is one line that runs at exactly the moment its output is needed. Same correctness guarantee (dead entries never affect an overlap check, because they're removed just before the check happens), fraction of the code, no configuration.
>
> The pattern shows up widely once you notice the shape: expiring cache entries on the next lookup rather than a cleanup thread; garbage-collecting deleted rows during the next table scan rather than `VACUUM` on a timer; removing stale sessions on the next request rather than on the clock; recomputing derived state on read rather than on every write. Whenever *"we need to eventually clean this up"* is the requirement and *"we already visit these entries for another reason"* is the reality, piggybacking wins on every axis except one: eager cleanup is proactive (dead state is gone before anyone looks), lazy cleanup is reactive (dead state lingers when no one is looking, which is exactly when it's harmless). Pick eager only when the dead state has a cost of its own (memory pressure, log volume, replica lag). For an availability index where dead entries are bytes in a slice nobody reads, lazy wins.
>
> The senior signal is stating the trade-off, not defaulting to the sweeper because it "feels more thorough." Extra machinery is a real cost paid every day for a benefit (proactive cleanup) that often isn't needed.

---

### PricingStrategy

```go
type PricingStrategy interface {
    Price(room Room, dates DateRange) int
}

type FlatPricing struct{}
func (FlatPricing) Price(r Room, d DateRange) int { return r.BaseRate * d.Nights() }

type WeekendPricing struct {
    WeekendMultiplierPct int // e.g. 150 → weekend nights cost 1.5×
}
func (w WeekendPricing) Price(r Room, d DateRange) int { /* per-night sum with Fri/Sat surcharge */ }
```

**Why this is Strategy and not `if room.PremiumSeason`:** *"Pricing is the concern that changes most often in a real system. Marketing wants a Diwali promotion; revenue management wants dynamic pricing tied to occupancy; a bulk-corporate deal wants a per-guest override. Every one of these has the same signature `Price(room, dates) → amount` and none of them belong inside the booking flow. Strategy makes each policy a separate type that plugs in via the constructor. The `FlatPricing` version is the interview demo; the interface is the extension point."*

---

### PaymentGateway: External Boundary

```go
type PaymentGateway interface {
    Charge(guestID string, amount int, idempotencyKey string) (paymentID string, err error)
    Refund(paymentID string, amount int, idempotencyKey string) error
}
```

**On the idempotency key being a required parameter, not optional:** *"Payment is a network call, which means retries, which means the same charge can arrive at the gateway twice. Without an idempotency key, retry-on-timeout will double-charge the guest. Baking it into the interface signature makes it impossible to forget: the compiler enforces it. Every caller passes something like `confirm-{bookingID}` so a replay of the same intent is a no-op on the gateway side. This mirrors the same rule I applied on the ATM's `BankService`."*

**On why the gateway is an interface, not a concrete Stripe/Razorpay client:** *"Testability first. Tests spin up a fake gateway that records calls and can be told to fail on demand. Portability second: any real deployment will swap gateways at least once. The interface is the seam."*

---

### HotelBookingSystem: The Orchestrator

```go
type HotelBookingSystem struct {
    mu       sync.Mutex
    rooms    map[string]Room
    guests   map[string]Guest
    bookings map[string]*Booking
    avail    *AvailabilityIndex
    pricing  PricingStrategy
    payments PaymentGateway
    holdTTL  time.Duration
    now      func() time.Time
    seq      int64
}

func NewHotelBookingSystem(pricing PricingStrategy, gw PaymentGateway) *HotelBookingSystem

// setup
func (s *HotelBookingSystem) AddRoom(r Room)
func (s *HotelBookingSystem) AddGuest(g Guest)

// booking lifecycle
func (s *HotelBookingSystem) Search(t RoomType, dates DateRange) []Room
func (s *HotelBookingSystem) Hold(guestID, roomID string, dates DateRange) (*Booking, error)
func (s *HotelBookingSystem) Confirm(bookingID string) (*Booking, error)
func (s *HotelBookingSystem) Cancel(bookingID string) error
func (s *HotelBookingSystem) CheckIn(bookingID string) error
func (s *HotelBookingSystem) CheckOut(bookingID string) error

// query
func (s *HotelBookingSystem) GetBooking(id string) (*Booking, bool)
```

**On the two-step flow the signatures encode:** `Search` takes a `RoomType` and returns the specific rooms of that type that are free for the requested dates. `Hold` takes a specific `roomID` chosen from that returned set. The type filter is a search-time concern; the reservation itself is always against one specific room, and the availability index tracks bookings per room. This is a deliberate middle ground: guests browse by type ("show me Deluxe Doubles"), but the system commits to a specific room at booking time (simpler overlap check, cleaner data model). Full type-level pooling with deferred room assignment stays as an extension.

**On the two locks (service `mu` and availability `mu`) never being held together:** *"This is a lock-ordering discipline. The service lock guards the metadata maps (`bookings`, `rooms`, `guests`, the ID sequence); the availability lock guards the per-room overlap check. In every method, I acquire one, do work, release it, then acquire the other if needed. Nested acquisition would create a deadlock risk if any future code path ever took them in the reverse order. The Confirm method is the clearest example: read the booking under the service lock, release, call payment (external), then call `avail.Confirm` under its own lock, then re-acquire the service lock to update status. Never both at once."*

The state machine, as an enum with transition guards (not as State pattern):

```
[Held]       --Confirm(payment ok)---> [Confirmed]      (avail: HELD → CONFIRMED)
[Held]       --Confirm(payment fail)-> [Cancelled]      (avail: entry released)
[Held]       --Cancel()--------------> [Cancelled]      (avail: entry released; no refund)
[Held]       --TTL expiry-----------> [Cancelled]       (lazy, on next Reserve/Available scan)

[Confirmed]  --CheckIn()------------> [CheckedIn]      (avail: entry unchanged)
[Confirmed]  --Cancel()-------------> [Cancelled]      (avail: released; refund via gateway)

[CheckedIn]  --CheckOut()----------> [CheckedOut]     (avail: entry released, past dates)
```

**On the two failure scenarios of Confirm being explicit:** payment can fail (gateway declines, timeout after retries exhausted) or the hold can have expired between the caller reading the booking and Confirm being called. Both terminate in `Cancelled` but by different paths, and the tests exercise them separately.

---

## Implementation

*~10-15 minutes. Implement the most interesting method. Start with happy path, then name the edge cases.*

### Confirm: The Two-Phase Commit In Miniature

This is where every design decision above pays off: reading the booking under one lock, calling an external system without any lock, then promoting the availability entry under the other lock, with clear rollback on either failure. If the interviewer asks me to code one method, it's this one.

```go
func (s *HotelBookingSystem) Confirm(bookingID string) (*Booking, error) {
    s.mu.Lock()
    b, ok := s.bookings[bookingID]
    if !ok {
        s.mu.Unlock()
        return nil, errBookingNotFound
    }
    if b.Status != StatusHeld {
        s.mu.Unlock()
        return nil, errIllegalTransition
    }
    if !b.HoldExpiry.After(s.now()) {
        b.Status = StatusCancelled
        s.mu.Unlock()
        _ = s.avail.Release(b.RoomID, b.ID)
        return nil, errHoldExpired
    }
    snapshot := *b // copy fields needed for the payment call outside the lock
    s.mu.Unlock()

    // Payment happens with NO lock held. Gateway call may take seconds.
    payID, err := s.payments.Charge(snapshot.GuestID, snapshot.TotalAmount, "confirm-"+snapshot.ID)
    if err != nil {
        _ = s.avail.Release(snapshot.RoomID, snapshot.ID)
        s.mu.Lock()
        b.Status = StatusCancelled
        s.mu.Unlock()
        return nil, fmt.Errorf("payment: %w", err)
    }

    if err := s.avail.Confirm(snapshot.RoomID, snapshot.ID); err != nil {
        // Should not happen: the entry was HELD when we started, and holds are only
        // removed via Release or promoted via Confirm. But if it does, refund and fail.
        _ = s.payments.Refund(payID, snapshot.TotalAmount, "confirm-refund-"+snapshot.ID)
        return nil, err
    }

    s.mu.Lock()
    b.Status = StatusConfirmed
    b.PaymentID = payID
    b.HoldExpiry = time.Time{}
    s.mu.Unlock()
    return b, nil
}
```

**Four things to narrate while writing this:**

*"The lock is released before the payment call. This is the whole point of the two-phase design. If I held `s.mu` across the gateway call, every booking on the entire system would queue behind every payment. The payment gateway sees whatever throughput it can handle; the booking system stays responsive."*

*"The snapshot copy (`snapshot := *b`) is critical. Between releasing the lock and re-acquiring it, another goroutine could theoretically mutate `b` (in practice only Cancel would, and Cancel already checks status). Working from a value copy for the external call means the gateway sees consistent input, and the post-payment step re-reads from the pointer under the lock before writing status."*

*"The rollback on payment failure calls `avail.Release` before updating the booking status. Order matters: releasing the availability entry first makes the room bookable again for the next caller as quickly as possible. Updating the status second is bookkeeping; the availability release is the user-visible effect."*

*"The 'shouldn't happen' branch (payment succeeded but `avail.Confirm` failed) still gets rollback code. In this specific design the branch is genuinely unreachable, but writing the refund path anyway is cheap insurance: if a future refactor removes the guarantee, the code still fails safely instead of silently double-booking a charged customer. This is the senior instinct: assume every check-then-act boundary can be violated by future code you can't see."*

### Reserve: Where The Race Condition Doesn't Happen

`AvailabilityIndex.Reserve` is short but carries the entire correctness argument of the design:

```go
func (a *AvailabilityIndex) Reserve(roomID string, d DateRange, id string, holdExp time.Time) error {
    a.mu.Lock()
    defer a.mu.Unlock()
    rb := a.rooms[roomID]
    if rb == nil {
        rb = &roomBookings{}
        a.rooms[roomID] = rb
    }
    a.pruneExpired(rb)
    for _, e := range rb.entries {
        if e.dates.Overlaps(d) {
            return errRoomUnavailable
        }
    }
    // Insert in sorted position
    idx := sort.Search(len(rb.entries), func(i int) bool {
        return rb.entries[i].dates.CheckIn.After(d.CheckIn)
    })
    rb.entries = append(rb.entries, bookingRef{})
    copy(rb.entries[idx+1:], rb.entries[idx:])
    rb.entries[idx] = bookingRef{bookingID: id, dates: d, status: StatusHeld, holdExp: holdExp}
    return nil
}
```

**The critical property:** *"The overlap check and the insert happen under the same lock, in the same call, with no `Unlock`/`Lock` between them. That is the entire concurrency argument. If check and insert were separate methods, two goroutines could both call `check`, both see 'available', and both call `insert`. Merging them into one atomic method is what prevents the double-book. The test that stress-runs 50 concurrent `Hold` calls on the same room+dates confirms exactly one wins."*

**On not returning the promoted entry position:** the caller doesn't need it. Everything else operates by `bookingID`, and the position of an entry in the slice is an internal implementation detail. Exposing it would leak the sorted-slice representation, blocking a future swap to an interval tree.

---

## Verification Walkthrough

*~2-3 minutes. Trace one non-trivial scenario out loud to prove the design holds together.*

**Scenario:** Two guests race to book the same room (R1) for overlapping dates. Guest A wins the hold, payment goes through. Meanwhile Guest B tries and is rejected, then tries adjacent dates and succeeds.

```
t0  bookings={}  avail.rooms={}

t1  A: Hold("G-A", "R1", [Sep 1, Sep 4))                     [3 nights]
    ├─ service.mu.Lock: room exists, guest exists, dates valid, seq=1, id="bk-1"
    ├─ service.mu.Unlock
    ├─ pricing.Price → 15,00,000 (₹5000 × 3)
    ├─ avail.Reserve("R1", [Sep 1, Sep 4), "bk-1", exp=t1+10min)
    │    └─ avail.mu.Lock; no entries; insert HELD; avail.mu.Unlock ✓
    └─ bookings["bk-1"] = Booking{Status: HELD, HoldExpiry: t1+10min, Total: 15,00,000}
    returns bk-1

t2  B: Hold("G-B", "R1", [Sep 3, Sep 6))     [overlaps A's booking at Sep 3]
    ├─ … validation passes, id="bk-2"
    ├─ pricing.Price → 15,00,000
    └─ avail.Reserve("R1", [Sep 3, Sep 6), "bk-2", exp=t2+10min)
         └─ avail.mu.Lock
            ├─ entries = [{bk-1, [Sep 1, Sep 4)}]
            ├─ overlap check: [Sep 1, Sep 4) overlaps [Sep 3, Sep 6) → YES
            └─ return errRoomUnavailable; avail.mu.Unlock ✗
    B receives error; no booking created

t3  A: Confirm("bk-1")
    ├─ service.mu.Lock: b.Status == HELD ✓, HoldExpiry > now ✓
    ├─ snapshot = *b; service.mu.Unlock
    ├─ payments.Charge("G-A", 15,00,000, "confirm-bk-1") → "pay-abc" ✓  [no lock held]
    ├─ avail.Confirm("R1", "bk-1")
    │    └─ avail.mu.Lock; find entry; status: HELD → CONFIRMED; avail.mu.Unlock ✓
    └─ service.mu.Lock: b.Status = CONFIRMED, b.PaymentID = "pay-abc"; service.mu.Unlock
    returns bk-1 (confirmed)

t4  B: Hold("G-B", "R1", [Sep 4, Sep 6))     [adjacent to A, no overlap]
    ├─ … validation, id="bk-3"
    └─ avail.Reserve("R1", [Sep 4, Sep 6), "bk-3", exp=t4+10min)
         └─ avail.mu.Lock
            ├─ entries = [{bk-1, [Sep 1, Sep 4), CONFIRMED}]
            ├─ overlap: Sep 4.Before(Sep 4)? NO → no overlap
            └─ insert HELD at index 1; avail.mu.Unlock ✓
    returns bk-3
```

What each tick confirms:

- `t1`: Two locks used in sequence, never together. The service lock protects the metadata write; the availability lock protects the overlap-and-insert. Both critical sections are microsecond-scale.
- `t2`: The overlap predicate is the whole correctness argument. `[Sep 1, Sep 4).Overlaps([Sep 3, Sep 6))` returns `true` because `Sep 1 < Sep 6` and `Sep 3 < Sep 4`. Guest B is rejected before any state changes.
- `t3`: Payment happens entirely outside any lock. This is the design's payoff: if Guest A's payment took 30 seconds, no other operation on any other room would be blocked. The re-acquisition of the service lock after payment is only long enough to write two fields.
- `t4`: The half-open convention earns its place. Guest B checking in on the exact day Guest A checks out is legal. The predicate uses strict `Before`, so `Sep 4.Before(Sep 4)` is `false` and the check passes.

---

## Trade-Offs

*Most trade-offs are named inline at the point of decision: two-phase booking, single mutex on the availability index, enum-plus-guards instead of State pattern, lazy eviction of expired holds, half-open date intervals. What's collected here is scope-level: things I've explicitly chosen not to model.*

- **No overbooking.** Real hotels intentionally sell more rooms than they own, using no-show statistics to stay profitable. Modeling this requires a per-room-type "sold count vs physical count" separation, tolerance for `Reserve` returning success beyond capacity, and a walk-in-recovery workflow. Out of scope; the availability index would need a `capacity` concept per room type, and the overlap check would count overlapping bookings against it.
- **Full refund on cancel, always.** No tiered policy (100% before 48h, 50% after, none within 24h). The seam is a `RefundPolicy` interface passed to `Cancel`; the demo bypasses it.
- **No modification.** A guest wanting to change dates has to cancel and rebook, which risks losing the room to another guest between the two calls. Real systems handle modification atomically: reserve new dates, if success then release old and refund the difference. Not modeled to keep the state machine small.
- **In-memory only.** Nothing survives a restart. The `bookings` map and the availability index are RAM. Real systems persist bookings to a durable store on every mutation; the interfaces don't change, only the backing implementation.
- **Single-node.** All state lives in one process. Multi-node deployments need either a shared database with row-level locking (Postgres `SELECT FOR UPDATE`, MySQL InnoDB locking) or a distributed coordinator (Redis with `SETNX`, Zookeeper). Named as the extension boundary; not implemented.
- **No overbooking recovery, no walk-in flow, no waitlist.** The current design's failure mode when a room is unavailable is a plain error to the caller.

---

## Extensions

*~5 minutes if time allows. For each follow-up, say where it hooks in and what changes; no need to write full code.*

**"How would you support type-level search where any Deluxe Double will do?"**
Add a `RoomTypeIndex` next to `AvailabilityIndex` that tracks per-type per-day inventory counts (`available[type][date] = count`). Search consults it first for "is any room of this type free," then falls back to per-room `Reserve` at hold time (assigning a specific room from the free set). Housekeeping optimisations then decide *which* free room the guest actually stays in, deferred to check-in.

**"How would you prevent double-booking across multiple servers?"**
The in-memory mutex only serialises callers on one process. In a multi-node deployment, move the availability check into the database and use `SELECT FOR UPDATE` on the room-day rows inside a transaction; or use a distributed lock (Redis `SETNX` with expiry keyed by room ID) for the duration of the reserve. The service code doesn't change shape, only the `AvailabilityIndex` implementation does. This is exactly why the interface stayed small.

**"How would you handle expired holds that never get cleaned up?"**
Currently lazy: the next `Reserve` or `Available` call on that room prunes expired entries. Fine for hot rooms; problematic for cold ones (a hold on a rarely-searched room sits forever). Add a background sweeper that walks the map every N seconds and calls `Release` on expired holds. Rate is a knob: too fast wastes CPU, too slow lets holds linger. Alerting on "unusually high hold-expiry rate" catches gateway degradation.

**"How would you implement dynamic pricing based on live occupancy?"**
New `PricingStrategy` implementation that reads the availability index: `if occupancyPct(dates) > 80, multiply base rate by 1.4`. The strategy interface already takes `(room, dates)`; the concrete type gets a reference to `AvailabilityIndex` via its constructor. No changes to the booking flow, no changes to any other pricing implementation.

**"How would you add a waitlist for a fully-booked room?"**
On `Reserve` returning `errRoomUnavailable`, offer the caller a waitlist option that creates a `WaitlistEntry` per room+dates. When any active booking on that room is `Cancel`ed or expires, iterate the waitlist for overlapping entries and notify the first one via `PaymentGateway`-adjacent messaging. Order matters (FIFO by entry time). The cancellation path becomes the trigger; the waitlist itself is a separate service.

**"How would you handle a payment that succeeds but the client never receives the response?"**
The idempotency key already makes retrying `Charge` safe. The gateway sees the same key, returns the previous `paymentID`, no double-charge. The trickier case is `avail.Confirm` failing after payment success: currently the code refunds. In a real system with async retries, the pattern is an outbox table: writing "payment succeeded, confirm pending" to durable storage inside the same transaction as the payment log, then a background worker completes the confirm with retries. The single-node design has no equivalent since there's no separate transaction; naming this as the production upgrade path is worth flagging.

**"How would you support group bookings (5 rooms for one event)?"**
Introduce a `GroupBooking` that holds N child bookings. `Hold` becomes atomic across all N: reserve every room, and if any one fails, release everything reserved so far. This is a distributed-transaction-in-miniature; the rollback logic is the interesting bit. Payment is one charge for the group total; refund on partial cancellation gets nuanced (per-room or all-or-nothing?), which is exactly the kind of business question that would come back to the interviewer.

---

## References

- `system-design/lld/patterns/behavioral/strategy.md`: Strategy pattern, applied here to `PricingStrategy`
- `system-design/lld/design-atm.md`: Sibling case study; the atomic-with-rollback pattern in `Confirm` mirrors the debit-before-dispense structure, and the `PaymentGateway` interface follows the same "external boundary with mandatory idempotency key" shape as `BankService`
- `system-design/lld/design-vending-machine.md`: Sibling case study; the reasoning for *not* using State pattern here is the mirror of the reasoning for *using* it there. States with locally-driven transitions earn the pattern; states driven by external events don't