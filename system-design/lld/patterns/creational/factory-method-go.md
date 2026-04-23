# Factory Method Pattern

## Table of Contents

- [What It Is and Why It Exists](#what-it-is-and-why-it-exists)
- [Why "Factory Method"](#why-factory-method)
- [The Core Problem It Solves](#the-core-problem-it-solves)
- [Structure](#structure)
- [Go Implementation](#go-implementation)
- [Factory Method vs Simple Factory vs Abstract Factory](#factory-method-vs-simple-factory-vs-abstract-factory)
- [Go-Specific Idioms](#go-specific-idioms)
- [Trade-offs and Pitfalls](#trade-offs-and-pitfalls)
- [Real-World Examples](#real-world-examples)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

---

## What It Is and Why It Exists

**What it is:** A creational design pattern that defines an interface for creating an object, but defers the decision of which concrete type to instantiate to a subclass or implementing function.

**Why it exists:** When code needs to create objects but the exact type to create is only known at runtime — based on config, user input, environment, or feature flags — hard-coding `ConcreteType{}` everywhere creates tight coupling between the creation logic and the usage logic. Every new variant requires changes spread across the codebase. The Factory Method centralises that decision.

**Problems it solves:**
- Decouples the caller from concrete types — callers depend only on the product interface
- Makes adding new variants safe — add a new concrete type and register it in one place, callers change nothing
- Enables dependency injection and testability — swap a real implementation for a mock by changing the factory

**What problems it brings:**
- Adds indirection — harder to trace at a glance what type is actually being created
- Can grow into a large switch/if-else block if not structured carefully (violates Open/Closed Principle)
- More types in the codebase for simple cases where a constructor would suffice

---

## Why "Factory Method"

A factory in manufacturing is a facility that produces goods on demand. You place an order, you receive a product, and you do not care about the assembly line details. A "factory method" is a method that plays exactly this role: you call it, it returns an object implementing an interface, and you — the caller — never need to know the concrete type behind it.

The "method" in the name distinguishes it from the Abstract Factory pattern. Here, a single method (not a whole factory object) is responsible for the creation decision.

---

## The Core Problem It Solves

Without the pattern:

```go
// Caller is tightly coupled to every concrete type
func processNotification(channel string, msg string) {
    if channel == "email" {
        n := &EmailNotifier{smtpHost: "smtp.example.com"}
        n.Send(msg)
    } else if channel == "sms" {
        n := &SMSNotifier{apiKey: "..."}
        n.Send(msg)
    } else if channel == "slack" {
        n := &SlackNotifier{webhookURL: "..."}
        n.Send(msg)
    }
    // Adding a new channel means editing this function
}
```

With the pattern, the caller is insulated:

```go
notifier := NewNotifier(channel)  // factory method — returns Notifier interface
notifier.Send(msg)                // caller never sees the concrete type
```

---

## Structure

The classical GoF (Gang of Four) structure has four roles: Product, ConcreteProduct, Creator, and ConcreteCreator.

**Scenario A — Classical GoF (inheritance-based, e.g. Java)**

```
  <<interface>>                     <<abstract>>
    Product                           Creator
      △                            ─────────────────────
      |                            + someOperation()
      | implements                 # createProduct(): Product  ← abstract
      |                                    △
ConcreteProduct                           | extends
                                          |
                                   ConcreteCreator
                                   ─────────────────────
                                   # createProduct(): Product
                                         |
                                         | returns
                                         ▼
                                   ConcreteProduct
```

- `Creator` declares the abstract factory method `createProduct()` and contains business logic in `someOperation()` that calls it — without knowing the concrete type
- `ConcreteCreator` extends `Creator` and overrides `createProduct()` to return the appropriate `ConcreteProduct`
- The caller only ever references `Creator` and `Product` — never the concrete types

**Scenario B — Go (no inheritance, constructor function)**

```
  Product (interface)
       |
       |── ConcreteProductA (struct, implements Product)
       |── ConcreteProductB (struct, implements Product)
       |── ConcreteProductC (struct, implements Product)

  NewProduct(kind string) Product   ← factory method (standalone function)
```

There is no abstract Creator class. The constructor function `NewProduct` plays the role of ConcreteCreator — it centralises the instantiation decision and returns the `Product` interface. Callers only depend on `Product`; they never see the concrete structs.

The factory method is a standalone constructor function in Go, not a method on a creator class. This is idiomatic Go.

---

## Go Implementation

A complete, worked example: a notification system where the concrete notifier type is chosen at runtime.

```go
package notify

import "fmt"

// Product interface — callers depend only on this
type Notifier interface {
    Send(msg string) error
}

// ── Concrete products ───────────────────────────────────────────────────────

type EmailNotifier struct {
    smtpHost string
    from     string
}

func (e *EmailNotifier) Send(msg string) error {
    fmt.Printf("[Email via %s] %s\n", e.smtpHost, msg)
    return nil
}

type SMSNotifier struct {
    apiKey string
}

func (s *SMSNotifier) Send(msg string) error {
    fmt.Printf("[SMS] %s\n", msg)
    return nil
}

type SlackNotifier struct {
    webhookURL string
}

func (sl *SlackNotifier) Send(msg string) error {
    fmt.Printf("[Slack -> %s] %s\n", sl.webhookURL, msg)
    return nil
}

// ── Factory method ──────────────────────────────────────────────────────────

type NotifierConfig struct {
    SMTPHost   string
    From       string
    SMSKey     string
    WebhookURL string
}

// NewNotifier is the factory method. Callers receive a Notifier — they never
// import or reference EmailNotifier, SMSNotifier, or SlackNotifier directly.
func NewNotifier(channel string, cfg NotifierConfig) (Notifier, error) {
    switch channel {
    case "email":
        return &EmailNotifier{smtpHost: cfg.SMTPHost, from: cfg.From}, nil
    case "sms":
        return &SMSNotifier{apiKey: cfg.SMSKey}, nil
    case "slack":
        return &SlackNotifier{webhookURL: cfg.WebhookURL}, nil
    default:
        return nil, fmt.Errorf("unknown channel: %s", channel)
    }
}
```

Caller:

```go
func main() {
    cfg := notify.NotifierConfig{SMTPHost: "smtp.example.com", From: "no-reply@example.com"}

    n, err := notify.NewNotifier("email", cfg)
    if err != nil {
        log.Fatal(err)
    }
    n.Send("Your order has shipped.")
    // Swap "email" for "slack" — main() changes nothing else
}
```

### Extending Without Breaking Callers

Adding a push notification channel:

```go
type PushNotifier struct{ deviceToken string }

func (p *PushNotifier) Send(msg string) error {
    fmt.Printf("[Push -> %s] %s\n", p.deviceToken, msg)
    return nil
}
```

Add one case to the switch in `NewNotifier`. Callers — and their tests — change nothing.

### Testability

Because the factory returns an interface, tests inject a mock without touching production code:

```go
type MockNotifier struct {
    Sent []string
}

func (m *MockNotifier) Send(msg string) error {
    m.Sent = append(m.Sent, msg)
    return nil
}

func TestOrderService(t *testing.T) {
    mock := &MockNotifier{}
    svc := OrderService{notifier: mock}  // inject mock — same Notifier interface
    svc.PlaceOrder("item-42")
    if len(mock.Sent) != 1 {
        t.Fatalf("expected 1 notification, got %d", len(mock.Sent))
    }
}
```

---

## Factory Method vs Simple Factory vs Abstract Factory

These three are often confused. They are distinct patterns.

**Simple Factory (not a GoF pattern):**
A single function or struct with a method that centralises object creation. No interface for the factory itself. It is what the switch-based `NewNotifier` above is. Simple, practical, not formally a GoF pattern.

```
NewNotifier(channel) ──> returns ConcreteType
```

**Factory Method (GoF pattern):**
The creation logic is part of an interface/abstract class. Different implementations of the creator produce different products. The pattern is about letting subclasses (or implementing types) decide what to create.

```
Creator interface:
  CreateNotifier() Notifier

EmailCreator implements Creator → creates EmailNotifier
SlackCreator  implements Creator → creates SlackNotifier
```

This is useful when the creator itself has complex behaviour beyond just creation, and different creators naturally pair with different products.

**Abstract Factory:**
A factory of factories. Produces families of related objects. You switch the whole factory, not just one product.

```
NotifierFactory interface:
  CreateSender()   Sender
  CreateFormatter() Formatter

ProdFactory   → ProdSender   + ProdFormatter
TestingFactory → StubSender  + StubFormatter
```

In Go, the distinction between Simple Factory and Factory Method is often academic — idiomatic Go uses the constructor function pattern (`NewX`) which blends both.

---

## Go-Specific Idioms

**Go does not have inheritance**, so the classical GoF Factory Method (abstract creator class + subclass override) does not map directly. Instead Go uses:

**1. Constructor functions returning interfaces:**
```go
func NewStorage(kind string) Storage { ... }
```
The function is the factory method. The caller only knows `Storage`.

**2. Functional options + factory:**
When construction has many optional parameters, combine factory method with functional options:
```go
type Option func(*EmailNotifier)

func WithTLS() Option { return func(e *EmailNotifier) { e.tls = true } }

func NewEmailNotifier(opts ...Option) Notifier {
    n := &EmailNotifier{smtpHost: "localhost"}
    for _, o := range opts {
        o(n)
    }
    return n
}
```

**3. Registry pattern (open/closed):**
Avoid the ever-growing switch by using a registry map:
```go
var registry = map[string]func(cfg Config) Notifier{}

func Register(name string, constructor func(cfg Config) Notifier) {
    registry[name] = constructor
}

func NewNotifier(name string, cfg Config) (Notifier, error) {
    fn, ok := registry[name]
    if !ok {
        return nil, fmt.Errorf("unknown notifier: %s", name)
    }
    return fn(cfg), nil
}

// Each package registers itself via init()
func init() {
    notify.Register("email", func(cfg Config) Notifier {
        return &EmailNotifier{smtpHost: cfg.SMTPHost}
    })
}
```
New channels are added in new packages with `init()` registration — the core factory function never changes. This is the Go stdlib `database/sql` driver model.

---

## Trade-offs and Pitfalls

**When to use:**
- Multiple concrete types share a common interface
- The type to create is determined at runtime (config, flags, user input)
- You want callers to be isolated from concrete types for testability or extensibility

**When not to use:**
- Only one concrete type exists and no extension is anticipated — just use a constructor directly
- The type is statically known at compile time — generics or direct construction are cleaner

**Growing switch problem:**
A switch-based factory method violates the Open/Closed Principle as new types are added. Mitigate with the registry pattern (see above).

**Leaking concrete types:**
The factory method is undermined if callers import and type-assert to the concrete type. The interface contract must be the only thing callers depend on.

**Confusion with Abstract Factory:**
Factory Method creates one product. Abstract Factory creates families of products. If you only have one axis of variation, Factory Method is the right choice.

---

## Real-World Examples

**Go standard library — `database/sql`:**
`sql.Open("postgres", dsn)` is a factory method. The caller receives a `*sql.DB`. The `postgres` driver registered itself in `init()`. `database/sql` never imports the driver package — pure factory method + registry pattern.

**Go standard library — `net/http` transport:**
`http.NewRequest` and `http.Client` accept an interface (`http.RoundTripper`), letting callers inject custom transports (mock, retry-wrapped, metrics-emitting) without the HTTP client knowing their concrete types.

**Logger factories:**
`zap.New`, `slog.New`, `logrus.New` all return a logger interface (or concrete struct acting as one), centralising configuration and letting callers be agnostic to the underlying implementation.

---

## Interview Cheat Sheet

**"Explain the Factory Method pattern"**
- Defines an interface for object creation. Defers the concrete type decision to a function or implementing type.
- Decouples callers from concrete types. Callers work only with the product interface.
- In Go: a `NewX(kind string) Interface` function pattern.

**"How is Factory Method different from Abstract Factory?"**
- Factory Method: one factory method, one product type, one axis of variation
- Abstract Factory: a factory interface that creates families of related products, multiple axes of variation
- Use Factory Method when you vary one thing; use Abstract Factory when you need coordinated families

**"How is Factory Method different from Simple Factory?"**
- Simple Factory: a plain function, not a GoF pattern, no interface for the factory itself
- Factory Method: the creation logic is part of an interface contract; different implementations of the creator produce different products
- In practice in Go, the line is blurry — the constructor function pattern is idiomatic and covers both

**"How do you extend a factory without modifying existing code?"**
- Registry pattern: each new type registers itself in `init()`. The factory function just looks up the registry. This is how `database/sql` drivers work in Go.

**"How does this enable testability?"**
- Because the factory returns an interface, tests inject a mock by implementing the same interface. No real DB, no real network, no real file I/O needed in unit tests.

**"When would you not use a factory method?"**
- When only one concrete type exists
- When the type is known at compile time — use generics or direct construction
- When the indirection adds complexity without a real extension need ("YAGNI")

---

## References

- [Refactoring Guru — Factory Method](https://refactoring.guru/design-patterns/factory-method)
- [GoF Design Patterns — Creational: Factory Method](https://en.wikipedia.org/wiki/Factory_method_pattern)
- [Go database/sql driver registration (registry pattern in the wild)](https://pkg.go.dev/database/sql#Register)
- [Effective Go — Interfaces](https://go.dev/doc/effective_go#interfaces)