# OOP Pillars in Go

## Table of Contents

1. [Go's take on OOP](#1-gos-take-on-oop)
2. [Encapsulation](#2-encapsulation)
3. [Abstraction](#3-abstraction)
4. [Inheritance — Go uses composition instead](#4-inheritance--go-uses-composition-instead)
5. [Polymorphism](#5-polymorphism)
6. [Pillar interactions and SOLID connection](#6-pillar-interactions-and-solid-connection)
7. [OOP → HLD mapping](#7-oop--hld-mapping)
8. [Common pitfalls](#8-common-pitfalls)
9. [Quick reference cheat sheet](#9-quick-reference-cheat-sheet)
10. [References](#10-references)

---

## 1. Go's take on OOP

Go is not a classical OOP language — it has no classes, no `extends`, no method overloading, and no access modifiers like `private` / `public` on fields within a type. Yet it implements all four OOP concepts through different, often cleaner mechanisms.

| Classical OOP | Go equivalent |
|---------------|--------------|
| Class | `struct` + methods |
| `private` field | Unexported identifier (lowercase) |
| `public` field/method | Exported identifier (Uppercase) |
| Interface + `implements` | Interface with *implicit satisfaction* |
| Inheritance (`extends`) | Struct embedding (promotion) |
| Method overloading | Not supported — use generics or variadic params |
| Runtime polymorphism | Interface dispatch (itab lookup) |
| Compile-time polymorphism | Generics (Go 1.18+) |

**Interview talking point:** "Go deliberately excludes inheritance. The language enforces what Effective Java recommends: prefer composition over inheritance. The better default is the only option."

---

## 2. Encapsulation

### Definition

Bundle data and behaviour into a single unit, and restrict direct access to internal state. The object enforces its own invariants through its methods.

### How Go does it

Go uses **package-level visibility**, not class-level access modifiers.

- **Lowercase identifier** → unexported (visible only within the package)
- **Uppercase identifier** → exported (visible to any package)

The unit of encapsulation in Go is the **package**, not the struct.

### Example

```go
package bank

import "fmt"

// BankAccount — balance is unexported; only methods can touch it
type BankAccount struct {
    balance   float64 // unexported — no direct access outside this package
    accountID string
}

// Constructor enforces the creation invariant
func NewBankAccount(id string, initial float64) (*BankAccount, error) {
    if initial < 0 {
        return nil, fmt.Errorf("initial balance cannot be negative")
    }
    return &BankAccount{accountID: id, balance: initial}, nil
}

func (a *BankAccount) Deposit(amount float64) error {
    if amount <= 0 {
        return fmt.Errorf("deposit amount must be positive")
    }
    a.balance += amount
    return nil
}

func (a *BankAccount) Withdraw(amount float64) error {
    if amount <= 0 {
        return fmt.Errorf("withdraw amount must be positive")
    }
    if amount > a.balance {
        return fmt.Errorf("insufficient funds: have %.2f, need %.2f", a.balance, amount)
    }
    a.balance -= amount
    return nil
}

// Balance is a read-only accessor — caller cannot set it directly
func (a *BankAccount) Balance() float64 { return a.balance }
```

Outside the `bank` package, `account.balance = -999` does not compile. The invariant "balance ≥ 0" is enforced by the struct's methods.

### Pointer vs value receiver

```go
// Value receiver — works on a copy; cannot modify the original
func (a BankAccount) Balance() float64 { return a.balance }

// Pointer receiver — modifies the original struct
func (a *BankAccount) Withdraw(amount float64) error { ... }
```

**Rule:** If any method on a type needs a pointer receiver, use pointer receivers consistently across all methods of that type.

### Pitfall — returning mutable internal collections

```go
type UserProfile struct {
    tags []string // unexported
}

// Bad — caller holds a reference to the internal slice; can mutate it
func (u *UserProfile) Tags() []string { return u.tags }

// Good — return a copy; caller cannot affect internal state
func (u *UserProfile) Tags() []string {
    cp := make([]string, len(u.tags))
    copy(cp, u.tags)
    return cp
}
```

### Interview insight

"In Go, encapsulation is enforced at the package boundary. A well-designed package exposes a minimal API — unexported types, constructors instead of struct literals, and pointer receivers on mutating methods. If a type has 40 exported methods, it's poorly encapsulated even if every field is lowercase."

---

## 3. Abstraction

### Definition

Expose only what a caller needs; hide implementation details. Define *what*, not *how*.

### How Go does it

Go uses **interfaces with implicit satisfaction** — no `implements` keyword. If your type has the required methods, it satisfies the interface. This is called *structural typing*.

A key Go convention: **define interfaces in the consuming package, not the producing package.** This keeps dependencies pointing in the right direction.

### Example

```go
// In the notification service package (consumer defines the contract)
type Sender interface {
    Send(userID string, message string) error
}

// In the email package — no reference to the Sender interface
type EmailSender struct {
    smtpHost string
    port     int
}

func (e *EmailSender) Send(userID, message string) error {
    // SMTP implementation — callers never see this
    fmt.Printf("[email] to=%s msg=%s\n", userID, message)
    return nil
}

// In the SMS package
type SMSSender struct {
    apiKey string
}

func (s *SMSSender) Send(userID, message string) error {
    fmt.Printf("[sms] to=%s msg=%s\n", userID, message)
    return nil
}

// NotificationService depends only on the Sender interface — never a concrete type
type NotificationService struct {
    sender Sender
}

func NewNotificationService(s Sender) *NotificationService {
    return &NotificationService{sender: s}
}

func (n *NotificationService) Notify(userID, msg string) error {
    return n.sender.Send(userID, msg)
}

// Swap senders at the call site — NotificationService never changes
svc := NewNotificationService(&EmailSender{smtpHost: "smtp.example.com", port: 587})
// or:
svc := NewNotificationService(&SMSSender{apiKey: "sk_live_..."})
```

### Interface size — prefer small, composable interfaces

Go's standard library uses tiny interfaces: `io.Reader` has one method, `io.Writer` has one method. This is the **Interface Segregation Principle**:

```go
// Bad — forces all implementors to provide everything
type Storage interface {
    Read(key string) ([]byte, error)
    Write(key string, val []byte) error
    Delete(key string) error
    List(prefix string) ([]string, error)
    Stats() StorageStats // not all callers need this
}

// Good — split by use case; compose when needed
type Reader  interface { Read(key string) ([]byte, error) }
type Writer  interface { Write(key string, val []byte) error }
type Deleter interface { Delete(key string) error }

type ReadWriter interface {
    Reader
    Writer
}
```

### Abstract class vs interface (Go context)

Go has no abstract classes. The closest equivalent is an interface (for pure contracts) or a struct with an embedded struct (to provide partial default implementations):

```go
// Provide a default "no-op" implementation — embed to get defaults
type BaseHandler struct{}
func (b BaseHandler) OnError(err error) { log.Println("error:", err) }

// Custom handler gets OnError for free; overrides what it needs
type MyHandler struct {
    BaseHandler
}
func (h MyHandler) Handle(req Request) Response { ... }
```

### Encapsulation vs Abstraction — the classic confusion

These are distinct concepts that work together:

- **Encapsulation** — a *mechanism*: bundle + hide. Enforces invariants. Prevents external mutation.
- **Abstraction** — a *design goal*: simplify the interface. Manage complexity. Hide the "how".

Encapsulation is one way to achieve abstraction, but abstraction also operates at larger scales: a service hiding its storage technology, a module hiding its algorithm, an API hiding its infrastructure.

---

## 4. Inheritance — Go uses composition instead

### Definition (classical OOP)

A subclass reuses fields and methods from a parent, establishing an IS-A relationship.

### Go's position

**Go has no inheritance by design.** There is no `extends`. The Go team's rationale: inheritance creates tight coupling between parent and child, and the fragile base class problem is a real source of production bugs.

Go's answer:
- **Struct embedding** for code reuse (field/method promotion)
- **Interface satisfaction** for IS-A / substitutability

### Struct embedding — promotion, not inheritance

```go
type Animal struct {
    Name string
}

func (a Animal) Breathe() {
    fmt.Printf("%s is breathing\n", a.Name)
}

// Dog embeds Animal — this is composition (HAS-A), not inheritance (IS-A)
// Dog promotes Animal's fields and methods to its own method set
type Dog struct {
    Animal        // embedded — no field name, type name is used
    Breed  string
}

func (d Dog) Speak() {
    // Name is promoted from Animal — no d.Animal.Name needed
    fmt.Printf("%s says Woof\n", d.Name)
}

func main() {
    d := Dog{Animal: Animal{Name: "Rex"}, Breed: "Labrador"}
    d.Breathe() // promoted from Animal — no redeclaration needed
    d.Speak()   // Dog's own method
    fmt.Println(d.Name)  // promoted field
}
```

### Embedding ≠ IS-A — you still need an interface for substitutability

```go
// Dog embeds Animal, but Dog is NOT substitutable for Animal
// This won't compile:
var a Animal = Dog{...}  // type mismatch

// For substitutability, define an interface:
type Speaker interface { Speak() }

type Dog struct{ Animal; Breed string }
func (d Dog) Speak() { fmt.Printf("%s says Woof\n", d.Name) }

type Cat struct{ Animal }
func (c Cat) Speak() { fmt.Printf("%s says Meow\n", c.Animal.Name) }

// Polymorphic function — works for any Speaker
func MakeNoise(s Speaker) { s.Speak() }

MakeNoise(Dog{Animal: Animal{Name: "Rex"}, Breed: "Labrador"})
MakeNoise(Cat{Animal: Animal{Name: "Whiskers"}})
```

### Method shadowing — Go's equivalent of overriding

```go
type Base struct{}
func (b Base) Describe() string { return "I am Base" }

type Child struct{ Base }
// "Override" by defining a method with the same name on Child
func (c Child) Describe() string { return "I am Child" }

c := Child{}
fmt.Println(c.Describe())       // "I am Child" — shadows Base.Describe
fmt.Println(c.Base.Describe())  // "I am Base"  — explicit access to embedded method
```

### IS-A vs HAS-A

| Relationship | Use | Go mechanism |
|---|---|---|
| IS-A: Dog is an Animal | Substitutability, polymorphism | Interface satisfaction |
| HAS-A: Car has an Engine | Code reuse, delegation | Struct embedding or field |

### Why composition beats inheritance

```go
// Classical OOP inheritance problem:
// If you subclass ArrayList to count adds, you're coupled to ArrayList's internals.
// When ArrayList changes how addAll calls add, your count breaks silently.

// Go composition — delegate without coupling:
type CountingStore struct {
    delegate Store // interface — not tied to any concrete type
    addCount int
}

func (c *CountingStore) Set(key string, val []byte) error {
    c.addCount++
    return c.delegate.Set(key, val)
}

func (c *CountingStore) Get(key string) ([]byte, error) {
    return c.delegate.Get(key)
}
```

---

## 5. Polymorphism

### Definition

The ability of objects of different types to respond to the same message (method call) in type-specific ways.

### Two forms in Go

#### Runtime polymorphism — interface dispatch

```go
type PaymentGateway interface {
    Charge(token string, amount float64) (string, error) // returns txID
    Refund(txID string) error
}

type StripeGateway struct{ apiKey string }

func (s *StripeGateway) Charge(token string, amount float64) (string, error) {
    // Stripe API call — implementation hidden
    return "stripe_tx_123", nil
}

func (s *StripeGateway) Refund(txID string) error {
    return nil
}

type PayPalGateway struct{ clientID, secret string }

func (p *PayPalGateway) Charge(token string, amount float64) (string, error) {
    return "paypal_tx_456", nil
}

func (p *PayPalGateway) Refund(txID string) error { return nil }

// CheckoutService is closed to modification, open to extension
type CheckoutService struct {
    gateway PaymentGateway // interface — never a concrete type
}

func NewCheckoutService(gw PaymentGateway) *CheckoutService {
    return &CheckoutService{gateway: gw}
}

func (c *CheckoutService) ProcessOrder(token string, amount float64) error {
    txID, err := c.gateway.Charge(token, amount) // dispatched at runtime
    if err != nil {
        return fmt.Errorf("charge failed: %w", err)
    }
    fmt.Printf("Order charged: txID=%s\n", txID)
    return nil
}

// Swap gateways at the call site — CheckoutService never changes
svc := NewCheckoutService(&StripeGateway{apiKey: "sk_live_..."})
// or:
svc := NewCheckoutService(&PayPalGateway{clientID: "...", secret: "..."})
```

#### How interface dispatch works internally

An interface value in Go is a pair of pointers:

```
interface value layout:
┌───────────────────────────────────────────────────┐
│  *itab  ──► [ *type_info | method_ptr_1 | ... ]   │
│  *data  ──► [ actual struct value ]               │
└───────────────────────────────────────────────────┘
```

When you call a method on an interface, Go uses `*itab` to find the concrete method pointer at runtime. This is analogous to a vtable in C++ or Java.

#### Compile-time polymorphism — generics (Go 1.18+)

Go has no method overloading. Generics fill this role for type-parametric code:

```go
import "golang.org/x/exp/constraints"

// Works for any ordered type — resolved at compile time, no runtime overhead
func Min[T constraints.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}

Min(3, 5)       // T inferred as int
Min(3.14, 2.71) // T inferred as float64
Min("a", "b")   // T inferred as string

// Generic data structure
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(v T) {
    s.items = append(s.items, v)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    n := len(s.items) - 1
    v := s.items[n]
    s.items = s.items[:n]
    return v, true
}

// Usage
s := &Stack[int]{}
s.Push(1); s.Push(2)
v, ok := s.Pop() // v=2, ok=true
```

### Strategy pattern — polymorphism in system design

The most practical application of polymorphism in HLD:

```go
// Without polymorphism — brittle switch on type
func notify(notifType, userID, msg string) {
    switch notifType {
    case "email": sendEmail(userID, msg)
    case "sms":   sendSMS(userID, msg)
    // Adding Slack means modifying this function — violates Open/Closed Principle
    }
}

// With interface-based polymorphism — Open/Closed
type Sender interface {
    Send(userID, msg string) error
    Type() string
}

type EmailSender struct{}
func (e *EmailSender) Send(u, m string) error { fmt.Printf("[email] %s: %s\n", u, m); return nil }
func (e *EmailSender) Type() string           { return "email" }

type SMSSender struct{}
func (s *SMSSender) Send(u, m string) error { fmt.Printf("[sms] %s: %s\n", u, m); return nil }
func (s *SMSSender) Type() string           { return "sms" }

type SlackSender struct{}
func (s *SlackSender) Send(u, m string) error { fmt.Printf("[slack] %s: %s\n", u, m); return nil }
func (s *SlackSender) Type() string           { return "slack" }  // ← added without touching existing code

// Registry — maps type string to implementation
type SenderRegistry struct {
    senders map[string]Sender
}

func NewSenderRegistry(senders ...Sender) *SenderRegistry {
    r := &SenderRegistry{senders: make(map[string]Sender)}
    for _, s := range senders {
        r.senders[s.Type()] = s
    }
    return r
}

func (r *SenderRegistry) Notify(notifType, userID, msg string) error {
    s, ok := r.senders[notifType]
    if !ok {
        return fmt.Errorf("unknown notification type: %s", notifType)
    }
    return s.Send(userID, msg) // polymorphic dispatch
}

// Wiring — only this changes when you add a new sender
registry := NewSenderRegistry(
    &EmailSender{},
    &SMSSender{},
    &SlackSender{},
)
registry.Notify("email", "user_123", "Your order shipped")
```

### No method overloading — idiomatic alternatives

```go
// Option 1: Functional options pattern (preferred for structs)
type ServerConfig struct {
    host    string
    port    int
    timeout time.Duration
}

type Option func(*ServerConfig)

func WithHost(h string) Option       { return func(c *ServerConfig) { c.host = h } }
func WithPort(p int) Option          { return func(c *ServerConfig) { c.port = p } }
func WithTimeout(t time.Duration) Option { return func(c *ServerConfig) { c.timeout = t } }

func NewServer(opts ...Option) *Server {
    cfg := &ServerConfig{host: "localhost", port: 8080, timeout: 30 * time.Second} // defaults
    for _, o := range opts {
        o(cfg)
    }
    return &Server{cfg: cfg}
}

// Option 2: Variadic parameter
func Sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}
Sum(1, 2)
Sum(1, 2, 3, 4, 5)
```

---

## 6. Pillar interactions and SOLID connection

```
Encapsulation ──► S  Single Responsibility — each type owns one responsibility
Abstraction   ──► D  Dependency Inversion  — depend on interfaces, not concrete types
                  I  Interface Segregation — small, focused interfaces (io.Reader, io.Writer)
Composition   ──► L  Liskov Substitution   — any implementor of an interface is substitutable
Polymorphism  ──► O  Open/Closed           — add a new Sender without touching existing code
```

### Liskov Substitution in Go

Any type that satisfies an interface must honour the full contract — not just the method signatures, but the behavioural expectations.

```go
type Bird interface {
    Move() string
}

// Penguin satisfies Bird.Move() — but is it substitutable everywhere Bird is used?
// If callers expect Move() to return "flying", a Penguin returning "swimming" breaks assumptions.
// Solution: define narrower interfaces that accurately model what each type can do.

type FlyingBird interface {
    Fly() string
}

type SwimmingBird interface {
    Swim() string
}

type Duck struct{}
func (d Duck) Fly() string  { return "duck flying" }
func (d Duck) Swim() string { return "duck swimming" }

type Penguin struct{}
func (p Penguin) Swim() string { return "penguin swimming" }
// Penguin doesn't implement FlyingBird — that's correct, not a violation
```

---

## 7. OOP → HLD mapping

| HLD concept | OOP pillar | Go implementation |
|---|---|---|
| Service contract (REST / gRPC API) | Abstraction | Interface defined in consumer package |
| Private DB — only accessed via service | Encapsulation | Unexported repo struct; exported methods |
| Strategy pattern (payment, notifications) | Polymorphism | Interface dispatch; registry pattern |
| Shared middleware / base handler | Composition (not inheritance) | Struct embedding |
| Dependency injection | Abstraction + Polymorphism | Accept interfaces in constructors |
| Repository pattern | Abstraction + Encapsulation | `Repository` interface; unexported SQL/NoSQL impl |
| Plugin / extension architecture | Polymorphism | Register implementations against an interface |

---

## 8. Common pitfalls

| Mistake | Fix |
|---------|-----|
| Returning internal slice/map directly | Return a copy; caller cannot mutate internal state |
| Embedding a concrete type for "inheritance" | Embed for promotion, but use interface for substitutability |
| Large interfaces (> 3–4 methods) | Split by use case; compose where needed |
| Storing concrete types in structs | Store interfaces; enables testing, swapping, mocking |
| Calling a method on a nil interface | Check for nil before calling; panic at runtime otherwise |
| Using `interface{}` (or `any`) everywhere | Use generics for type-safe containers (Go 1.18+) |
| Defining interfaces in the producer package | Define in the consumer package — keeps dependency direction correct |
| Forgetting that value receivers copy the struct | Use pointer receivers on mutating methods; be consistent within a type |

---

## 9. Quick reference cheat sheet

```
Encapsulation:   lowercase fields + Uppercase methods
                 package is the unit of encapsulation, not the struct
                 use constructors (NewX) — not struct literals — to enforce invariants

Abstraction:     define small interfaces in the consuming package
                 implicit satisfaction — no implements keyword
                 io.Reader, io.Writer are the gold standard: one method each

Inheritance:     Go has none — use struct embedding for promotion (HAS-A)
                 use interface for IS-A / substitutability
                 method shadowing = override equivalent

Polymorphism:    runtime: interface dispatch via itab
                 compile-time: generics [T constraints.Ordered]
                 no method overloading — use functional options or variadic params

Pattern → OOP:
  if/switch on type string → replace with interface dispatch (Strategy)
  "I need a default impl" → embed struct + shadow methods
  "I want to swap DB/cache/gateway" → accept an interface, inject the concrete impl
```

---

## 10. References

- [Effective Go — Interfaces and methods](https://go.dev/doc/effective_go#interfaces_and_types)
- [The Go Programming Language — Donovan & Kernighan](https://www.gopl.io/) — Chapter 6 (Methods), Chapter 7 (Interfaces)
- [Go by Example — Interfaces](https://gobyexample.com/interfaces)
- [Go standard library — io package](https://pkg.go.dev/io) — gold standard for small interface design
- [Composition over inheritance — Go blog](https://go.dev/blog/laws-of-reflection)
- [Generics in Go 1.18 — Go blog](https://go.dev/blog/intro-generics)
- [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md) — practical interface and struct conventions
- [Effective Java (3rd ed.) — Bloch](https://www.oreilly.com/library/view/effective-java-3rd/9780134686097/) — Items 18–20: Composition over inheritance (language-agnostic principles)