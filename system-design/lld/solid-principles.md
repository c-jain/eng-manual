# SOLID principles

Each principle is presented with a motivation, a violation, a corrected design, and the interviewer angle.

## Table of Contents

- [Overview](#Overview)
- [S — single responsibility principle](#s--single-responsibility-principle)
- [O — open/closed principle](#o--openclosed-principle)
- [L — liskov substitution principle](#l--liskov-substitution-principle)
- [I — interface segregation principle](#i--interface-segregation-principle)
- [D — dependency inversion principle](#d--dependency-inversion-principle)
- [How they interact](#How-they-interact)
- [Interview cheat sheet](#Interview-cheat-sheet)
- [References](#References)

---

## Overview

SOLID is a set of five object-oriented design principles introduced by Robert Martin (Uncle Bob). They describe characteristics of code that is maintainable, extensible, and testable.

They are not independent rules. Violating one tends to force violations of the others. Conversely, applying DIP (interfaces everywhere) makes SRP easier to achieve (you can split types cleanly), and ISP (small interfaces) makes DIP more useful (consumers depend only on what they use).

In Go interviews, these surface as:

- Direct: "Explain the SOLID principles."
- Indirect: "Why is this code hard to test?" / "How would you refactor this service?" / "What's wrong with this interface?"

The trap is reciting definitions. Interviewers want you to identify violations in real code and articulate the trade-off — not just name the principle.

---

## S — single responsibility principle

> A type should have only one reason to change.

"Reason to change" is the key phrase. A **reason** means an actor (a business role, a team, a system) that cares about that code. If marketing owns email logic and the backend team owns DB schema, those are two different reasons — they belong in different types.

### violation

```go
// UserService does too much.
// Marketing, backend, security, and ops all have reasons to change this one type.
type UserService struct {
    db     *sql.DB
    mailer *SMTPClient
    logger *AuditLogger
}

func (s *UserService) Register(email, password string) error {
    // Validation (owned by: product/backend)
    if !isValidEmail(email) {
        return errors.New("invalid email")
    }
    // Hashing (owned by: security)
    hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    if err != nil {
        return err
    }
    // DB write (owned by: backend/data)
    _, err = s.db.Exec("INSERT INTO users (email, hash) VALUES (?, ?)", email, string(hash))
    if err != nil {
        return err
    }
    // Email (owned by: marketing)
    s.mailer.Send(email, "Welcome to the platform!")
    // Audit log (owned by: ops/compliance)
    s.logger.Log("user_registered", email)
    return nil
}
```

When marketing changes the welcome email template, you're modifying the same struct that contains DB logic. A bug introduced there can break registration.

### corrected design

```go
// Each type has exactly one owner and one reason to change.

type UserValidator struct{}

func (v *UserValidator) Validate(email, password string) error {
    if !isValidEmail(email) {
        return errors.New("invalid email")
    }
    if len(password) < 8 {
        return errors.New("password too short")
    }
    return nil
}

type PasswordHasher struct{}

func (h *PasswordHasher) Hash(password string) (string, error) {
    b, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(b), err
}

type UserRepository struct{ db *sql.DB }

func (r *UserRepository) Save(ctx context.Context, u User) error {
    _, err := r.db.ExecContext(ctx, "INSERT INTO users (email, hash) VALUES (?, ?)", u.Email, u.PasswordHash)
    return err
}

// UserRegistrar orchestrates — it delegates, it does not implement.
type UserRegistrar struct {
    validator *UserValidator
    hasher    *PasswordHasher
    repo      *UserRepository
    notifier  Notifier // interface
    auditor   Auditor  // interface
}

func (r *UserRegistrar) Register(ctx context.Context, email, password string) error {
    if err := r.validator.Validate(email, password); err != nil {
        return fmt.Errorf("validation: %w", err)
    }
    hash, err := r.hasher.Hash(password)
    if err != nil {
        return fmt.Errorf("hashing: %w", err)
    }
    u := User{Email: email, PasswordHash: hash}
    if err := r.repo.Save(ctx, u); err != nil {
        return fmt.Errorf("save: %w", err)
    }
    r.notifier.Notify(u)
    r.auditor.Log("user_registered", email)
    return nil
}
```

Now each type can change independently. Tests for `PasswordHasher` need no DB. Tests for `UserRegistrar` mock the repository.

### interviewer angle

Interviewers test whether you understand that "one responsibility" is not "one method." A struct with 10 methods all serving the same owner (e.g., all HTTP handler logic for one resource) is fine. A struct with 3 methods serving three different owners is a violation.

---

## O — open/closed principle

> Open for extension, closed for modification.

Add new behaviour by writing new code, not by changing existing, tested code. The standard mechanism is an interface (or function type) as an extension point.

### violation

```go
// Every new payment provider requires modifying ProcessPayment.
// Each change risks breaking existing providers.
type OrderService struct{}

func (s *OrderService) ProcessPayment(method string, amount int64) error {
    switch method {
    case "stripe":
        // hardcoded Stripe SDK call
        return stripe.Charge(amount)
    case "razorpay":
        // hardcoded Razorpay SDK call
        return razorpay.CreateOrder(amount)
    default:
        return fmt.Errorf("unknown payment method: %s", method)
    }
}
```

Adding PhonePe means touching `ProcessPayment` — which already works and is tested. You risk introducing a regression.

### corrected design

```go
// Extension point: the PaymentProvider interface.
// New providers are new types — OrderService is never touched.

type PaymentProvider interface {
    Charge(ctx context.Context, amount int64, currency string) (txnID string, err error)
}

type StripeProvider struct {
    client *stripe.Client
}

func (p *StripeProvider) Charge(ctx context.Context, amount int64, currency string) (string, error) {
    params := &stripe.ChargeParams{Amount: stripe.Int64(amount), Currency: stripe.String(currency)}
    ch, err := p.client.Charges.New(params)
    if err != nil {
        return "", err
    }
    return ch.ID, nil
}

type RazorpayProvider struct {
    client *razorpay.Client
}

func (p *RazorpayProvider) Charge(ctx context.Context, amount int64, currency string) (string, error) {
    // Razorpay-specific implementation
    order, err := p.client.Order.Create(map[string]interface{}{"amount": amount, "currency": currency}, nil)
    if err != nil {
        return "", err
    }
    return order["id"].(string), nil
}

type OrderService struct {
    payment PaymentProvider
}

func (s *OrderService) ProcessPayment(ctx context.Context, amount int64, currency string) (string, error) {
    return s.payment.Charge(ctx, amount, currency)
}

// Adding PhonePe: implement the interface. Zero changes to OrderService.
type PhonePeProvider struct{ ... }
func (p *PhonePeProvider) Charge(ctx context.Context, amount int64, currency string) (string, error) { ... }
```

### trade-off

OCP is not about zero modification ever. It is about anticipating the right extension points — places where variation is likely or frequent. Over-applying OCP creates unnecessary abstraction and indirection where nothing will ever change. Apply it where you genuinely expect variation: payment providers, notification channels, serialisation formats, storage backends, feature flags.

### interviewer angle

"How would you add a new payment provider to this system?" If the answer is "I'd open the switch statement in ProcessPayment," that's an OCP violation. The correct answer: "I'd implement the PaymentProvider interface in a new type. The OrderService doesn't need to change."

---

## L — liskov substitution principle

> If S is a subtype of T, objects of type T may be replaced with objects of type S without altering correctness.

In Go, there is no classical inheritance — LSP applies to interface satisfaction. Any type that claims to implement an interface must honour its **behavioural contract**, not just its method signatures. Signatures are syntactic. Contracts are semantic.

### violation

```go
type ReadOnlyStore interface {
    Get(key string) (string, error)
}

// InMemoryReadStore — correct implementation
type InMemoryReadStore struct {
    data map[string]string
}

func (s *InMemoryReadStore) Get(key string) (string, error) {
    v, ok := s.data[key]
    if !ok {
        return "", errors.New("not found")
    }
    return v, nil
}

// CachingReadStore — violates LSP.
// A caller expecting ReadOnlyStore receives a type that secretly mutates state.
// This breaks the caller's expectations even though the signature compiles.
type CachingReadStore struct {
    inner ReadOnlyStore
    cache map[string]string // hidden mutable state
}

func (s *CachingReadStore) Get(key string) (string, error) {
    if v, ok := s.cache[key]; ok {
        return v, nil
    }
    v, err := s.inner.Get(key)
    if err == nil {
        s.cache[key] = v // side effect: mutates state on a "read" operation
    }
    return v, err
}
// Issue: the caller passed a ReadOnlyStore and received one that silently writes.
// If Get is called concurrently, the map write is a data race.
// The contract was: read-only. The implementation is not.
```

### corrected design

Make the contract explicit. Either acknowledge that caching requires state and expose that:

```go
// Approach 1: expose mutation in the type signature (not ReadOnlyStore)
type CachingStore struct {
    inner ReadOnlyStore
    mu    sync.RWMutex
    cache map[string]string
}

func (s *CachingStore) Get(key string) (string, error) {
    s.mu.RLock()
    if v, ok := s.cache[key]; ok {
        s.mu.RUnlock()
        return v, nil
    }
    s.mu.RUnlock()

    v, err := s.inner.Get(key)
    if err != nil {
        return "", err
    }

    s.mu.Lock()
    s.cache[key] = v
    s.mu.Unlock()

    return v, nil
}

// CachingStore does NOT claim ReadOnlyStore — it's a distinct type.
// Callers that need caching accept *CachingStore explicitly.

// Approach 2: define a separate interface that acknowledges the caching contract
type CacheableStore interface {
    Get(key string) (string, error)
    Invalidate(key string)
    Flush()
}
```

### interviewer angle

LSP violations are often subtle. The canonical real-world case is a read-only store that panics on write, or a sorted collection wrapper that violates the sort invariant. In Go, the question becomes: "Does this type honour the full behavioural contract of the interface, or just the signatures?" Race conditions on implicit state are a common LSP-adjacent issue.

---

## I — interface segregation principle

> No client should be forced to depend on methods it does not use.

Fat interfaces couple consumers to methods they don't need. This makes mocks large, dependencies opaque, and refactoring harder.

### violation

```go
// A single fat interface mixes concerns.
// A password reset service has to depend on SMS and push methods it will never call.
type Notifier interface {
    SendEmail(to, subject, body string) error
    SendSMS(to, body string) error
    SendPush(deviceToken, body string) error
    LogAuditEvent(event, actor string) error // logging is not notification
}

type PasswordResetService struct {
    notifier Notifier // forced to depend on SMS, Push, and audit methods
}

// Test mock must implement four methods even though PasswordResetService uses one.
type mockNotifier struct{}
func (m *mockNotifier) SendEmail(...) error  { return nil }
func (m *mockNotifier) SendSMS(...) error    { return nil } // unused boilerplate
func (m *mockNotifier) SendPush(...) error   { return nil } // unused boilerplate
func (m *mockNotifier) LogAuditEvent(...) error { return nil } // wrong layer
```

### corrected design

```go
// Narrow, focused interfaces. Each is independently implementable and mockable.

type Emailer interface {
    SendEmail(to, subject, body string) error
}

type SMSSender interface {
    SendSMS(to, body string) error
}

type PushNotifier interface {
    SendPush(deviceToken, body string) error
}

type Auditor interface {
    LogAuditEvent(event, actor string) error
}

// PasswordResetService only declares what it actually uses.
type PasswordResetService struct {
    email  Emailer
    audit  Auditor
}

func (s *PasswordResetService) Reset(ctx context.Context, userEmail string) error {
    token := generateResetToken()
    if err := s.email.SendEmail(userEmail, "Password reset", resetBody(token)); err != nil {
        return err
    }
    s.audit.LogAuditEvent("password_reset_requested", userEmail)
    return nil
}

// Test mock is minimal.
type mockEmailer struct{ sent []string }
func (m *mockEmailer) SendEmail(to, subject, body string) error {
    m.sent = append(m.sent, to)
    return nil
}

// An SMS notification service composes a different set:
type OrderConfirmationService struct {
    email  Emailer
    sms    SMSSender
    audit  Auditor
}
```

### Go idiom

Go is a natural fit for ISP because interfaces are implicitly satisfied — you define them at the point of use, not at the point of implementation. The standard library is a model: `io.Reader`, `io.Writer`, `io.Closer`, `io.Seeker` are each one method. `io.ReadWriter`, `io.ReadWriteCloser`, etc. are composites. Define small, single-method interfaces at the consumer, and let implementations satisfy multiple interfaces without modification.

### interviewer angle

"What's wrong with accepting an interface with 12 methods as a parameter?" It couples the consumer to methods it doesn't use, makes testing expensive, and makes it harder to substitute implementations. Narrow the interface to exactly what the consumer needs.

---

## D — dependency inversion principle

> High-level modules should not depend on low-level modules. Both should depend on abstractions.

If `OrderService` directly instantiates `MySQLOrderRepository`, you cannot test `OrderService` without a live MySQL connection. Concretions make code inflexible. Abstractions make code testable and swappable.

### violation

```go
// OrderService is tightly coupled to MySQL. Untestable without a DB.
type MySQLOrderRepository struct {
    db *sql.DB
}

func (r *MySQLOrderRepository) Save(o Order) error {
    _, err := r.db.Exec("INSERT INTO orders ...")
    return err
}

type OrderService struct {
    repo *MySQLOrderRepository // concrete dependency — cannot be swapped
}

func NewOrderService(db *sql.DB) *OrderService {
    return &OrderService{
        repo: &MySQLOrderRepository{db: db}, // hardcoded construction
    }
}

func (s *OrderService) PlaceOrder(ctx context.Context, o Order) error {
    return s.repo.Save(o) // tightly coupled to MySQL
}
```

### corrected design

```go
// Both depend on the interface. Neither depends on the other.

// The abstraction (interface) — lives near the consumer (high-level module).
type OrderRepository interface {
    Save(ctx context.Context, o Order) error
    FindByID(ctx context.Context, id string) (Order, error)
    ListByUser(ctx context.Context, userID string) ([]Order, error)
}

// Low-level: MySQL implementation of the interface.
type MySQLOrderRepository struct {
    db *sql.DB
}

func (r *MySQLOrderRepository) Save(ctx context.Context, o Order) error {
    _, err := r.db.ExecContext(ctx, "INSERT INTO orders (id, user_id, total) VALUES (?, ?, ?)",
        o.ID, o.UserID, o.Total)
    return err
}

func (r *MySQLOrderRepository) FindByID(ctx context.Context, id string) (Order, error) {
    var o Order
    err := r.db.QueryRowContext(ctx, "SELECT id, user_id, total FROM orders WHERE id = ?", id).
        Scan(&o.ID, &o.UserID, &o.Total)
    return o, err
}

func (r *MySQLOrderRepository) ListByUser(ctx context.Context, userID string) ([]Order, error) {
    rows, err := r.db.QueryContext(ctx, "SELECT id, user_id, total FROM orders WHERE user_id = ?", userID)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    var orders []Order
    for rows.Next() {
        var o Order
        if err := rows.Scan(&o.ID, &o.UserID, &o.Total); err != nil {
            return nil, err
        }
        orders = append(orders, o)
    }
    return orders, rows.Err()
}

// High-level: OrderService depends only on interfaces.
type OrderService struct {
    repo    OrderRepository
    payment PaymentProvider
    events  EventPublisher
}

// Constructor injection — dependencies passed from outside.
func NewOrderService(repo OrderRepository, payment PaymentProvider, events EventPublisher) *OrderService {
    return &OrderService{repo: repo, payment: payment, events: events}
}

func (s *OrderService) PlaceOrder(ctx context.Context, o Order) error {
    txnID, err := s.payment.Charge(ctx, o.Total, "INR")
    if err != nil {
        return fmt.Errorf("payment: %w", err)
    }
    o.TransactionID = txnID
    if err := s.repo.Save(ctx, o); err != nil {
        return fmt.Errorf("save: %w", err)
    }
    s.events.Publish(ctx, OrderPlacedEvent{Order: o})
    return nil
}

// In tests: use a fast in-memory repository. Zero DB setup, runs in microseconds.
type InMemoryOrderRepository struct {
    mu     sync.Mutex
    orders map[string]Order
}

func NewInMemoryOrderRepository() *InMemoryOrderRepository {
    return &InMemoryOrderRepository{orders: make(map[string]Order)}
}

func (r *InMemoryOrderRepository) Save(_ context.Context, o Order) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.orders[o.ID] = o
    return nil
}

func (r *InMemoryOrderRepository) FindByID(_ context.Context, id string) (Order, error) {
    r.mu.Lock()
    defer r.mu.Unlock()
    o, ok := r.orders[id]
    if !ok {
        return Order{}, errors.New("not found")
    }
    return o, nil
}

func (r *InMemoryOrderRepository) ListByUser(_ context.Context, userID string) ([]Order, error) {
    r.mu.Lock()
    defer r.mu.Unlock()
    var result []Order
    for _, o := range r.orders {
        if o.UserID == userID {
            result = append(result, o)
        }
    }
    return result, nil
}

// Test:
func TestPlaceOrder(t *testing.T) {
    repo := NewInMemoryOrderRepository()
    payment := &mockPaymentProvider{txnID: "txn_abc"}
    events := &mockEventPublisher{}
    svc := NewOrderService(repo, payment, events)

    o := Order{ID: "ord_1", UserID: "usr_1", Total: 1000}
    if err := svc.PlaceOrder(context.Background(), o); err != nil {
        t.Fatal(err)
    }

    saved, err := repo.FindByID(context.Background(), "ord_1")
    if err != nil {
        t.Fatal(err)
    }
    if saved.TransactionID != "txn_abc" {
        t.Errorf("expected txn_abc, got %s", saved.TransactionID)
    }
}
```

### DIP and constructor injection

DIP is what makes **dependency injection** possible. Go rarely needs a DI framework — pass dependencies as interface-typed constructor parameters. Wire them up in `main.go` (or a `cmd/` package), which is the only place that knows about concrete types.

```
main.go / cmd/
  │── constructs MySQLOrderRepository, StripeProvider, KafkaPublisher
  │── passes them as interfaces to NewOrderService
  └── registers handler

OrderService
  │── knows only OrderRepository, PaymentProvider, EventPublisher (interfaces)
  └── no import of mysql, stripe, or kafka packages
```

This is the dependency rule: source code dependencies point inward toward abstractions. Concrete implementations are outermost.

### interviewer angle

"How would you test this service without a database?" If the answer requires a test database or mocking at the SQL level (sqlmock), DIP has been violated. The correct architecture passes `OrderRepository` as an interface — tests swap in an in-memory implementation. DIP is the single biggest enabler of fast, reliable unit tests.

---

## How they interact

```
SRP  — tells you what to split.
       One actor, one type. Split when two different owners care about the same code.

OCP  — tells you where to put extension points.
       Identify what varies, extract it as an interface. New variants are new types.

LSP  — keeps extension points honest.
       Every implementation of an interface must honour the full behavioural contract,
       not just the method signatures.

ISP  — keeps interfaces small so consumers aren't over-coupled.
       Define interfaces at the point of use. Only declare what you actually call.

DIP  — makes everything testable.
       High-level business logic depends on abstractions. Concretions are wired
       at the outermost layer (main, cmd/).
```

A codebase that violates DIP (concretions everywhere) typically violates SRP too (you can't cleanly split concerns without interfaces to inject between them). A codebase with fat interfaces (ISP violation) makes DIP less useful because consumers are still coupled to more than they need.

### common combined patterns in Go

**Repository pattern** — DIP + SRP: `UserRepository` interface (abstraction, SRP: one data concern), `MySQLUserRepository` (implementation, DIP: concrete at the boundary).

**Strategy pattern** — OCP + DIP: `SortStrategy` interface, injected into a `Sorter`. New sort algorithms are new types.

**Decorator pattern** — OCP + LSP: wrap an implementation with caching, logging, or circuit-breaking. The wrapper implements the same interface as the wrapped type.

```go
// LoggingOrderRepository wraps any OrderRepository and adds structured logging.
type LoggingOrderRepository struct {
    inner  OrderRepository
    logger *slog.Logger
}

func (r *LoggingOrderRepository) Save(ctx context.Context, o Order) error {
    err := r.inner.Save(ctx, o)
    r.logger.InfoContext(ctx, "order.save", "id", o.ID, "err", err)
    return err
}
```

---

## Interview cheat sheet

**"Explain SOLID"** — Avoid definitions-only. Give one real example per principle. Focus on what problem each solves.

**"Why is this code hard to test?"** — Look for: concrete dependencies (`new(X)` in constructor), fat interfaces, multiple reasons to change in one struct. Map to DIP, ISP, SRP.

**"How would you add a new feature without touching existing code?"** — Open/closed. Extract an interface, implement a new type.

**"What's wrong with a 15-method interface?"** — ISP: consumers depend on methods they don't use; mocks are huge; implementations are constrained. Define narrow interfaces at the point of use.

**"What's forward dependency vs backward dependency?"** — High-level modules depending on low-level = bad (DIP violation). Both depending on an interface = correct. Source code import direction should point toward abstractions.

**"Is Go OOP?"** — No classical inheritance, but Go is fully compatible with SOLID via interfaces. The Go standard library (`io.Reader`, `http.Handler`, `sort.Interface`) is a masterclass in ISP + DIP.

---

## References

- [Robert Martin — The Principles of OOD](http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod)
- [Robert Martin — Clean Architecture](https://www.goodreads.com/book/show/18043011-clean-architecture) (chapters 7–11 cover SOLID in depth)
- [Go io package — ISP in practice](https://pkg.go.dev/io)
- [Dave Cheney — Practical Go: Interface Composition](https://dave.cheney.net/2016/08/20/solid-go-design)
- [Uber Go Style Guide — Interfaces](https://github.com/uber-go/guide/blob/master/style.md#interfaces)