---
Status: 🌳 Evergreen
Created: 2026-08-03
Last Updated: 2026-08-03
---

# Hexagonal Architecture (Ports and Adapters)

## Table of Contents

- [What Is Hexagonal Architecture](#what-is-hexagonal-architecture)
- [Why It Exists](#why-it-exists)
- [Why It Is Called "Hexagonal"](#why-it-is-called-hexagonal)
- [Ports and Adapters](#ports-and-adapters)
- [How It Looks](#how-it-looks)
- [Worked Example in Go](#worked-example-in-go)
- [Go Gotcha: Where Adapters Live in Memory](#go-gotcha-where-adapters-live-in-memory)
- [Related Architectures](#related-architectures)
- [Problems and Trade-Offs](#problems-and-trade-offs)
- [Edge Cases and Extensions](#edge-cases-and-extensions)
- [How to Remember](#how-to-remember)
- [Interview Questions and Answers](#interview-questions-and-answers)
- [References](#references)

## What Is Hexagonal Architecture

Hexagonal Architecture — also called **Ports and Adapters**, its more literal and arguably more accurate name — is a way of structuring an application so the business logic (the "core") has zero dependency on how it's triggered or what it's stored in.

The core defines a small set of interfaces it needs — called **ports**. Anything outside the core — an HTTP server, a CLI, a Postgres database, a Kafka producer — plugs into those ports through a small translation layer called an **adapter**. The core never imports `net/http`, `database/sql`, or any framework package. It only knows about its own ports.

The payoff: you can run and test your business logic with no web server and no database running at all, and you can swap the web framework or the database technology without touching a single line of business logic.

## Why It Exists

Picture a typical layered ("N-tier") service: a handler layer calls a service layer, which calls a database layer. Drawn as boxes, the dependency arrow points straight down — Presentation → Business → Data. That arrow is the problem: it means the business layer's source code literally imports the database package. You cannot compile, let alone run, the business logic without the data layer also compiling.

In practice this causes three concrete pains:

1. **You can't unit-test business rules without a database.** Every "unit" test secretly becomes an integration test, because the business package won't build without `database/sql` or an ORM present.
2. **Business logic quietly absorbs infrastructure concerns.** An `if err == sql.ErrNoRows` check, or an HTTP status code decision, creeps into what's supposed to be a pure business rule, because the layers aren't actually isolated — they're just named differently in the same dependency chain.
3. **Swapping technology means touching the core.** Moving from Postgres to DynamoDB, or from REST to gRPC, forces edits inside the business layer, because the business layer is the one holding the concrete import.

Hexagonal Architecture fixes this by inverting the arrow. The core defines an interface (a port) for what it needs — `type OrderRepository interface { Save(...) error }` — and the database package *implements* that interface. Now the dependency arrow points from infrastructure **into** the core, not the other way around. This is the Dependency Inversion Principle (the "D" in SOLID) applied one level up — not to two classes, but to two whole layers of the application.

## Why It Is Called "Hexagonal"

This is a genuinely common point of confusion, worth getting right: **the number six means nothing.** Alistair Cockburn, who introduced the pattern in 2005, didn't intend "six sides" to map to "six specific kinds of ports." He simply needed a shape with more than the traditional top-and-bottom of a layered diagram, so that the picture didn't visually suggest "there are only two kinds of outside world — UI on top, database on bottom." A hexagon has room to draw a port on any side, for any actor — a UI, a test harness, a message queue, a second service — without implying a ranking or a fixed count between them. You could draw the same architecture as a triangle or a nonagon; the shape is a drawing convenience, not a specification.

The name Cockburn actually preferred, and the one that describes the mechanism rather than the picture, is **Ports and Adapters**.

## Ports and Adapters

Two vocabulary pairs cover the whole pattern, and interview answers that mix them up are an instant tell that the concept isn't fully internalized.

**Port** — an interface. Always defined by (or for) the core, never by the adapter.

- **Primary port** (also "driving port"): what the core *offers*. Something calls **into** the application through this — e.g. `OrderService.PlaceOrder(...)`.
- **Secondary port** (also "driven port"): what the core *needs*. The core calls **out** through this — e.g. `OrderRepository.Save(...)`.

**Adapter** — a concrete implementation that translates between a specific technology and a port.

- **Primary / driving adapter**: sits on the side that *drives* the app. An HTTP handler, a CLI command, a gRPC server, a scheduled job, even a test — anything that calls the primary port.
- **Secondary / driven adapter**: sits on the side the app *drives*. A Postgres repository, an in-memory repository, an SMTP client, a Kafka producer — anything that implements the secondary port.

A simple test to keep these straight: if a human or another system triggers your code, that's the primary/driving side. If your code reaches out and triggers something else, that's the secondary/driven side.

## How It Looks

```
        HTTP Handler                    (primary / driving adapter)
              |
              |  calls primary port: OrderService
              v
   +----------------------------+
   |  Core: domain + use cases  |
   |  (knows nothing about      |
   |   HTTP or SQL)             |
   +----------------------------+
              |
              |  calls secondary port: OrderRepository
              v
        In-Memory Repo                   (secondary / driven adapter)
```

Swap the HTTP Handler for a CLI command or a gRPC server and nothing inside the Core box changes. Swap the In-Memory Repo for a Postgres repo and, again, nothing inside the Core box changes. Both swaps only ever touch an adapter plugged into a port — the ports themselves stay fixed, which is exactly the point.

## Worked Example in Go

A minimal but complete order-placement service. Every file below compiles, `go vet`s cleanly, and the test file passes under `go test -race`.

### Project Layout

```
hexdemo/
├── domain/           entity + business rules — imports nothing infrastructural
├── ports/            interfaces: OrderService (primary), OrderRepository (secondary)
├── core/             implements OrderService; depends only on ports.OrderRepository
├── adapters/
│   ├── http/           primary adapter: HTTP handler
│   └── memory/          secondary adapter: in-memory repository
└── main.go           composition root — wires everything together
```

(A larger production repo would typically nest this under `cmd/` and `internal/`, Go's conventions for the entry point and private packages. That's a packaging choice — it doesn't change the port/adapter boundary itself.)

### The Domain

```go
package domain

import (
	"errors"
	"time"
)

// Order is the core business entity. Note what it does NOT import:
// no net/http, no database/sql, no encoding/json. The domain package
// knows nothing about how an Order arrives or where it's stored.
type Order struct {
	ID         string
	CustomerID string
	Amount     float64
	PlacedAt   time.Time
}

var ErrInvalidAmount = errors.New("order amount must be greater than zero")

// NewOrder validates and constructs an Order. The rule "amount must
// be positive" lives here, not in an HTTP handler, so it's enforced
// no matter which adapter creates an Order — HTTP, CLI, gRPC, or a test.
func NewOrder(id, customerID string, amount float64) (Order, error) {
	if amount <= 0 {
		return Order{}, ErrInvalidAmount
	}
	return Order{
		ID:         id,
		CustomerID: customerID,
		Amount:     amount,
		PlacedAt:   time.Now(),
	}, nil
}
```

### The Ports

```go
package ports

import (
	"context"

	"hexdemo/domain"
)

// OrderService is the PRIMARY (driving) port. It's the boundary the
// outside world calls INTO. Driving adapters — an HTTP handler, a
// CLI command, a gRPC server, a test — all talk to the core only
// through this interface. They never import the core package's
// concrete struct directly.
type OrderService interface {
	PlaceOrder(ctx context.Context, customerID string, amount float64) (domain.Order, error)
	GetOrder(ctx context.Context, id string) (domain.Order, error)
}

// OrderRepository is a SECONDARY (driven) port. It's the boundary
// the core calls OUT to. It's phrased in terms the core understands
// (save an Order, find one by ID) — not in terms of SQL, a specific
// driver, or a table schema.
type OrderRepository interface {
	Save(ctx context.Context, o domain.Order) error
	FindByID(ctx context.Context, id string) (domain.Order, error)
}
```

### The Core Service

```go
package core

import (
	"context"
	"fmt"
	"sync/atomic"

	"hexdemo/domain"
	"hexdemo/ports"
)

// orderService is the core use-case implementation. It depends on
// ports.OrderRepository — an INTERFACE — never on a concrete
// database package. This is dependency inversion applied at the
// architecture level: infrastructure depends on the core, the core
// does not depend on infrastructure.
type orderService struct {
	repo ports.OrderRepository
}

// NewOrderService wires a concrete secondary adapter into the core
// through the port. Any type satisfying ports.OrderRepository works
// here — in-memory, Postgres, a hand-written test fake.
func NewOrderService(repo ports.OrderRepository) ports.OrderService {
	return &orderService{repo: repo}
}

func (s *orderService) PlaceOrder(ctx context.Context, customerID string, amount float64) (domain.Order, error) {
	order, err := domain.NewOrder(nextID(), customerID, amount)
	if err != nil {
		return domain.Order{}, fmt.Errorf("place order: %w", err)
	}
	if err := s.repo.Save(ctx, order); err != nil {
		return domain.Order{}, fmt.Errorf("place order: %w", err)
	}
	return order, nil
}

func (s *orderService) GetOrder(ctx context.Context, id string) (domain.Order, error) {
	order, err := s.repo.FindByID(ctx, id)
	if err != nil {
		return domain.Order{}, fmt.Errorf("get order: %w", err)
	}
	return order, nil
}

// idCounter is a package-level atomic counter standing in for a real
// ID generator (e.g. a UUID library), kept dependency-free for this
// example. atomic.Int64 avoids a data race if PlaceOrder is ever
// called concurrently from multiple goroutines.
var idCounter atomic.Int64

func nextID() string {
	return fmt.Sprintf("order-%d", idCounter.Add(1))
}
```

### The Secondary Adapter

```go
// Package memory is a SECONDARY ADAPTER. It implements
// ports.OrderRepository using a plain map instead of a real
// database. The core has no idea this isn't Postgres — it only
// knows the port.
package memory

import (
	"context"
	"errors"
	"fmt"
	"sync"

	"hexdemo/domain"
)

var ErrNotFound = errors.New("order not found")

type OrderRepository struct {
	mu     sync.RWMutex
	orders map[string]domain.Order
}

func NewOrderRepository() *OrderRepository {
	return &OrderRepository{orders: make(map[string]domain.Order)}
}

func (r *OrderRepository) Save(_ context.Context, o domain.Order) error {
	r.mu.Lock()
	defer r.mu.Unlock()
	r.orders[o.ID] = o
	return nil
}

func (r *OrderRepository) FindByID(_ context.Context, id string) (domain.Order, error) {
	r.mu.RLock()
	defer r.mu.RUnlock()
	o, ok := r.orders[id]
	if !ok {
		return domain.Order{}, fmt.Errorf("id %s: %w", id, ErrNotFound)
	}
	return o, nil
}
```

### The Primary Adapter

```go
// Package httpadapter is a PRIMARY ADAPTER. It translates HTTP
// requests into calls on the primary port (ports.OrderService) and
// translates the result back into an HTTP response. It contains
// zero business logic — validation and persistence decisions live
// in the core, not here.
package httpadapter

import (
	"encoding/json"
	"net/http"

	"hexdemo/ports"
)

type Handler struct {
	service ports.OrderService
}

func NewHandler(service ports.OrderService) *Handler {
	return &Handler{service: service}
}

type placeOrderRequest struct {
	CustomerID string  `json:"customer_id"`
	Amount     float64 `json:"amount"`
}

func (h *Handler) PlaceOrder(w http.ResponseWriter, r *http.Request) {
	var req placeOrderRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "invalid request body", http.StatusBadRequest)
		return
	}

	order, err := h.service.PlaceOrder(r.Context(), req.CustomerID, req.Amount)
	if err != nil {
		http.Error(w, err.Error(), http.StatusUnprocessableEntity)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	json.NewEncoder(w).Encode(order) //nolint:errcheck
}
```

Notice that `placeOrderRequest` — the JSON shape — lives here, in the adapter, not in `domain`. If a CLI adapter or a gRPC adapter is added later, each defines its own translation struct. The domain's `Order` type never grows a `json:"..."` tag just to satisfy one particular adapter.

### The Composition Root

```go
// main is the composition root: the ONE place in the codebase that
// knows about every concrete type. Every other package (core,
// handler, repository) only knows about ports. Wiring flows:
// secondary adapter -> core -> primary adapter.
package main

import (
	"log"
	"net/http"

	httpadapter "hexdemo/adapters/http"
	"hexdemo/adapters/memory"
	"hexdemo/core"
)

func main() {
	repo := memory.NewOrderRepository()
	service := core.NewOrderService(repo)
	handler := httpadapter.NewHandler(service)

	http.HandleFunc("/orders", handler.PlaceOrder)
	log.Println("listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Why This Is Testable

This is the actual payoff, not just a talking point. `core/service_test.go` tests the entire use case — validation, ID generation, persistence, error propagation — with **no HTTP server and no database**, by handing the core a hand-written fake that satisfies `ports.OrderRepository`:

```go
package core

import (
	"context"
	"errors"
	"testing"

	"hexdemo/domain"
)

// fakeRepo is a test-only SECONDARY ADAPTER. It satisfies the same
// ports.OrderRepository interface as the real in-memory and
// Postgres adapters, so the core can be tested with zero HTTP
// server and zero database spun up. This is the actual payoff of
// depending on a port instead of a concrete package.
type fakeRepo struct {
	saveErr error
	saved   []domain.Order
}

func (f *fakeRepo) Save(_ context.Context, o domain.Order) error {
	if f.saveErr != nil {
		return f.saveErr
	}
	f.saved = append(f.saved, o)
	return nil
}

func (f *fakeRepo) FindByID(_ context.Context, id string) (domain.Order, error) {
	for _, o := range f.saved {
		if o.ID == id {
			return o, nil
		}
	}
	return domain.Order{}, errors.New("not found")
}

func TestPlaceOrder_Success(t *testing.T) {
	repo := &fakeRepo{}
	svc := NewOrderService(repo)

	order, err := svc.PlaceOrder(context.Background(), "cust-1", 49.99)
	if err != nil {
		t.Fatalf("PlaceOrder() unexpected error: %v", err)
	}
	if order.CustomerID != "cust-1" {
		t.Errorf("CustomerID = %q, want %q", order.CustomerID, "cust-1")
	}
	if len(repo.saved) != 1 {
		t.Fatalf("expected 1 saved order, got %d", len(repo.saved))
	}
}

func TestPlaceOrder_InvalidAmount(t *testing.T) {
	repo := &fakeRepo{}
	svc := NewOrderService(repo)

	_, err := svc.PlaceOrder(context.Background(), "cust-1", -5)
	if !errors.Is(err, domain.ErrInvalidAmount) {
		t.Errorf("expected ErrInvalidAmount, got %v", err)
	}
}

func TestPlaceOrder_RepoFailure(t *testing.T) {
	repo := &fakeRepo{saveErr: errors.New("db down")}
	svc := NewOrderService(repo)

	_, err := svc.PlaceOrder(context.Background(), "cust-1", 10)
	if err == nil {
		t.Fatal("expected error when repo fails, got nil")
	}
}

func TestGetOrder_NotFound(t *testing.T) {
	repo := &fakeRepo{}
	svc := NewOrderService(repo)

	_, err := svc.GetOrder(context.Background(), "missing")
	if err == nil {
		t.Fatal("expected error for missing order, got nil")
	}
}
```

`fakeRepo` is not a mocking-library object — it's a plain struct, four lines to satisfy the interface. That's normal for a two-method port; the section on trade-offs below covers when hand-written fakes stop being the right call.

## Go Gotcha: Where Adapters Live in Memory

Following the memory-layout habit — always framed across all four segments, not just stack vs. heap:

- **Stack**: If a concrete adapter value were created and used only within a single function, with no reference escaping that function (no interface assignment, no return, no goroutine capture), the Go compiler could keep it stack-allocated.
- **Heap**: That's not what happens in `main()` above. `memory.NewOrderRepository()` returns a `*OrderRepository`, which is immediately stored inside the `orderService.repo` field — a field of type `ports.OrderRepository` (an interface). An interface value is a two-word pair: a pointer to type metadata and a pointer to the underlying data. Because that pair is held by a struct (`service`) that outlives the `main()` call stack — it's used for the lifetime of the HTTP server — escape analysis forces the `OrderRepository` to the heap. This is the general rule for composition-root-created adapters: anything wired into a long-lived interface field almost always escapes to the heap, regardless of how it was allocated.
- **Data/BSS Segment**: `idCounter` (declared `var idCounter atomic.Int64` at package scope in `core`) lives here — it's a package-level variable with static storage duration, mutated at runtime via `atomic.Int64.Add`.
- **Text/Code Segment**: Not applicable to any of the data above, but this is where the compiled machine code for `PlaceOrder`, `Save`, and `FindByID` lives — including the small indirect jump the runtime performs when a call goes through an interface's method table instead of being a plain direct call.

## Related Architectures

Hexagonal Architecture isn't alone in this family — Clean Architecture (Robert C. Martin, 2012) and Onion Architecture (Jeffrey Palermo, 2008) express the same core idea: dependencies point inward, toward business logic, never outward toward infrastructure. Clean and Onion typically draw this as concentric rings (entities in the center, use cases around them, infrastructure on the outside) rather than a hexagon with ports on its sides, and Clean Architecture adds more formal internal layering (entities vs. use cases vs. interface adapters). In practice, Hexagonal is often the simplest entry point into this whole family for a Go codebase — the ring subdivisions matter more once a codebase is large enough that "the core" itself needs internal structure. A dedicated compare-and-contrast note is a good future addition to this repo rather than a full derivation here.

## Problems and Trade-Offs

- **Indirection has a cost.** Every dependency the core needs turns into an interface plus at least one concrete implementation. For a two-method repository this is trivial; for a service with a dozen collaborators, it's a lot of files to read to understand one flow.
- **Nothing in the language enforces the boundary.** The compiler will happily let someone `import "net/http"` inside the `domain` package under deadline pressure. Keeping the core pure is a code-review and architecture-test discipline (e.g. a lint rule or a `go list` check in CI that fails if `domain` imports anything outside the standard library's non-I/O packages), not something Go itself prevents.
- **It's overkill for a small CRUD service.** If the whole app is "validate a request, write a row, return the row," the ports-and-adapters ceremony adds files without adding much protection — there's rarely a second adapter coming.
- **The vocabulary itself is a hurdle.** Primary vs. secondary, driving vs. driven — this is consistently the first thing people get backwards, which is exactly why it shows up in interview questions.

## Edge Cases and Extensions

- **DTO / JSON mapping between the HTTP layer and the domain?** Already shown above: `placeOrderRequest` lives in `httpadapter`, not `domain`. Keep translation structs local to each adapter so the domain type never has to serve two masters (e.g. JSON tags for HTTP and a different shape for gRPC).
- **A use case that needs more than one secondary port** (e.g. save an order *and* publish an event)? No change to the pattern — add a second field: `type orderService struct { repo ports.OrderRepository; publisher ports.EventPublisher }`. One field per dependency, same as any other constructor injection.
- **Transactions spanning multiple repositories?** Introduce a dedicated secondary port for it rather than leaking a `*sql.Tx` into the core:
  ```go
  type UnitOfWork interface {
      WithinTransaction(ctx context.Context, fn func(ctx context.Context) error) error
  }
  ```
  A Postgres adapter implements this by opening a `sql.Tx`, running `fn`, and committing or rolling back; the core just calls `uow.WithinTransaction(ctx, func(ctx) error { ... })` without knowing what "transaction" means underneath.
- **Hand-written fakes (like `fakeRepo` above) vs. a mocking library (gomock, testify/mock)?** For a port with one or two methods, a hand-written fake is faster to write and easier to read than generated mock code — there's no code-generation step to keep in sync. Reach for a mocking library once a port grows large, or once many test files need the same mock and hand-writing it repeatedly becomes the actual maintenance burden.
- **Listing or searching orders** (a `FindAll()` on the in-memory repository)? Iterating a Go map has no guaranteed order, so a naive `for range r.orders` would return orders in a different order on every call. Collect the keys into a slice and sort them (or maintain a separate ordered slice alongside the map) before returning — the same map-iteration gotcha that shows up anywhere a Go map backs anything user-visible.

## How to Remember

- **The hexagon has no top or bottom — every side is a port, and any number of adapters can plug into it.** The number six is a drawing choice, not a spec. If an answer starts with "there are six ports," that's the tell it's memorized wrong.
- **Primary pushes in, secondary gets pulled out.** Primary/driving = something outside calls *into* the core. Secondary/driven = the core calls *out* to something outside.
- **Console-and-controller anchor:** think of the core as a game console, ports as controller sockets, and adapters as the controllers themselves. The console doesn't care whether an Xbox pad or a keyboard adapter is plugged in, as long as it fits the socket — and you can swap the controller without opening the console.

## Interview Questions and Answers

**Q: What is Hexagonal Architecture, and what's its other name?**
A: A pattern that isolates business logic from infrastructure using interfaces ("ports") that the core defines, and concrete implementations ("adapters") that plug into them. Its original, more descriptive name is Ports and Adapters.

**Q: How is this actually different from a traditional layered (N-tier) architecture?**
A: The dependency direction. In a layered architecture, the business layer imports the data layer directly — the arrow points down and the business code can't compile without the data layer present. In Hexagonal, the core defines the interface it needs, and the data layer implements it — the arrow points from infrastructure into the core.

**Q: What do "primary" and "secondary" (or "driving" and "driven") mean?**
A: Primary/driving ports and adapters are on the side that calls into the application (HTTP, CLI, gRPC, tests). Secondary/driven ports and adapters are on the side the application calls out to (databases, message queues, external APIs).

**Q: Does the hexagon shape mean there are exactly six ports?**
A: No. Cockburn used a hexagon simply because it has more than two sides, so the diagram wouldn't imply "only a top (UI) and a bottom (database)" the way a layered box diagram does. Any number of ports can exist.

**Q: Concretely, how does this make unit testing easier?**
A: The core depends only on interfaces, so a test can hand it a small hand-written fake instead of a real database or HTTP server — no network calls, no test containers, tests run in milliseconds. `fakeRepo` in the worked example above is exactly that.

**Q: What's a real trade-off — when would you not reach for this?**
A: For a small CRUD service with one obvious adapter on each side and no near-term plan to swap technology, the extra interfaces and files are ceremony without much payoff. It earns its cost once there's real complexity to isolate, multiple adapters per port, or a genuine testability requirement.

**Q: How does this relate to Clean Architecture?**
A: Same underlying principle — dependencies point inward toward business logic — expressed with a different diagram (concentric rings instead of hexagon sides) and more formal internal layering.

## References

- Alistair Cockburn, ["Hexagonal Architecture"](https://alistair.cockburn.us/hexagonal-architecture/) — the original 2005 article defining Ports and Adapters.
- [Hexagonal architecture (software) — Wikipedia](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software)) — history and renaming to "Ports and Adapters."
- Robert C. Martin, ["The Clean Architecture"](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) — the closest well-known relative of this pattern.