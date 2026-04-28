# Abstract Factory

## Table of Contents

- [What It Is](#what-it-is)
- [Why It Exists](#why-it-exists)
- [Why It Is Called "Abstract Factory"](#why-it-is-called-abstract-factory)
- [Core Structure](#core-structure)
- [Go Implementation — Cross-Platform UI](#go-implementation--cross-platform-ui)
- [Abstract Factory vs Factory Method](#abstract-factory-vs-factory-method)
- [Problems It Introduces](#problems-it-introduces)
- [Go-Specific Notes](#go-specific-notes)
- [Real-World Analogies in Go Codebases](#real-world-analogies-in-go-codebases)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

---

## What It Is

Abstract Factory is a **creational** design pattern that provides an interface for creating **families of related objects** without specifying their concrete types.

A family is a set of objects that are designed to work together. The pattern guarantees that a client always gets a consistent family — it will never accidentally mix a Windows button with a macOS checkbox, or a MySQL repository with a MongoDB session.

---

## Why It Exists

Consider a GUI toolkit that must run on both Windows and macOS. Both platforms have buttons, checkboxes, and dialogs — but each platform's version looks and behaves differently. The naive approach:

```go
// Client scattered with platform conditionals — the anti-pattern
if platform == "windows" {
    btn = NewWindowsButton()
    chk = NewWindowsCheckbox()
} else {
    btn = NewMacButton()
    chk = NewMacCheckbox()
}
```

Problems with this:

- The client knows about every concrete type — any new platform change requires touching every callsite
- Nothing enforces consistency — a developer could accidentally call `NewWindowsButton()` and `NewMacCheckbox()` together
- Adding a new platform (Linux) requires hunting down every `if platform` block in the codebase

Abstract Factory solves this by:

- Hiding all concrete types behind a factory interface
- Grouping the construction of a whole family into one object (the factory)
- Letting the client be completely unaware of which family it is using

---

## Why It Is Called "Abstract Factory"

- **Factory** — it creates objects. Like a factory method, it centralises construction
- **Abstract** — the factory itself is defined as an interface (or abstract type). The client holds a reference to the abstract factory, not any concrete implementation. It never calls `NewWindowsFactory()` — it receives a `GUIFactory` and calls methods on that

The "abstract" distinguishes it from a simple factory (a concrete struct with factory methods) or a factory method (a single-product creation hook). Here, the factory contract is what gets abstracted, and it covers an entire family.

---

## Core Structure

```
         «interface»                    «interface»
         GUIFactory                       Button
    ┌──────────────────┐             ┌────────────┐
    │ CreateButton()   │             │  Render()  │
    │ CreateCheckbox() │             └────────────┘
    └────────┬─────────┘
             │ implemented by
    ┌────────┴────────┐
    │                 │
    ▼                 ▼
WindowsFactory     MacFactory           «interface»
                                         Checkbox
WindowsFactory ──► WindowsButton    ┌─────────────┐
               ──► WindowsCheckbox  │   Render()  │
                                    └─────────────┘
MacFactory     ──► MacButton
               ──► MacCheckbox

Client holds: GUIFactory (abstract)
Client uses:  Button, Checkbox (abstract)
Client never sees: Windows*, Mac* concrete types
```

Participants:

- **AbstractFactory** (`GUIFactory`) — declares creation methods, one per product type in the family
- **ConcreteFactory** (`WindowsFactory`, `MacFactory`) — implements the factory interface; each is responsible for one family
- **AbstractProduct** (`Button`, `Checkbox`) — the interface each product type must satisfy
- **ConcreteProduct** (`WindowsButton`, `MacButton`, etc.) — the actual implementations; the client never references these directly
- **Client** — receives an `AbstractFactory`; uses only the abstract factory and abstract product interfaces

---

## Go Implementation — Cross-Platform UI

```go
package main

import "fmt"

// ── Abstract Products ────────────────────────────────────────────────────────

type Button interface {
	Render()
	OnClick(handler func())
}

type Checkbox interface {
	Render()
	SetChecked(v bool)
}

// ── Abstract Factory ─────────────────────────────────────────────────────────

// GUIFactory is the abstract factory interface.
// It is defined here, in the consumer package, following Go's idiom of
// small, consumer-owned interfaces.
type GUIFactory interface {
	CreateButton() Button
	CreateCheckbox() Checkbox
}

// ── Windows Family ───────────────────────────────────────────────────────────

type windowsButton struct {
	label string
}

func (b *windowsButton) Render() {
	fmt.Printf("[Windows] Button: %q\n", b.label)
}

func (b *windowsButton) OnClick(handler func()) {
	fmt.Printf("[Windows] Registering click handler for %q\n", b.label)
	handler()
}

type windowsCheckbox struct {
	checked bool
}

func (c *windowsCheckbox) Render() {
	mark := " "
	if c.checked {
		mark = "x"
	}
	fmt.Printf("[Windows] Checkbox: [%s]\n", mark)
}

func (c *windowsCheckbox) SetChecked(v bool) { c.checked = v }

// WindowsFactory creates Windows-family UI components.
type WindowsFactory struct{}

func (f *WindowsFactory) CreateButton() Button {
	return &windowsButton{label: "OK"}
}

func (f *WindowsFactory) CreateCheckbox() Checkbox {
	return &windowsCheckbox{}
}

// ── Mac Family ───────────────────────────────────────────────────────────────

type macButton struct {
	label string
}

func (b *macButton) Render() {
	fmt.Printf("[macOS]   Button: (%s)\n", b.label)
}

func (b *macButton) OnClick(handler func()) {
	fmt.Printf("[macOS]   Registering click handler for (%s)\n", b.label)
	handler()
}

type macCheckbox struct {
	checked bool
}

func (c *macCheckbox) Render() {
	mark := "○"
	if c.checked {
		mark = "●"
	}
	fmt.Printf("[macOS]   Checkbox: %s\n", mark)
}

func (c *macCheckbox) SetChecked(v bool) { c.checked = v }

// MacFactory creates macOS-family UI components.
type MacFactory struct{}

func (f *MacFactory) CreateButton() Button {
	return &macButton{label: "OK"}
}

func (f *MacFactory) CreateCheckbox() Checkbox {
	return &macCheckbox{}
}

// ── Client Code ──────────────────────────────────────────────────────────────

// Application is the client. It knows nothing about Windows or Mac.
// It receives a GUIFactory and works entirely through interfaces.
type Application struct {
	factory  GUIFactory
	button   Button
	checkbox Checkbox
}

func NewApplication(factory GUIFactory) *Application {
	return &Application{factory: factory}
}

func (a *Application) BuildUI() {
	a.button = a.factory.CreateButton()
	a.checkbox = a.factory.CreateCheckbox()
}

func (a *Application) Render() {
	a.button.Render()
	a.checkbox.Render()
}

// ── Entry Point ──────────────────────────────────────────────────────────────

func getFactory(os string) GUIFactory {
	switch os {
	case "windows":
		return &WindowsFactory{}
	case "mac":
		return &MacFactory{}
	default:
		panic("unknown OS: " + os)
	}
}

func main() {
	for _, os := range []string{"windows", "mac"} {
		fmt.Printf("\n── %s ──\n", os)
		factory := getFactory(os)
		app := NewApplication(factory)
		app.BuildUI()
		app.Render()
	}
}
```

Expected output:

```
── windows ──
[Windows] Button: "OK"
[Windows] Checkbox: [ ]

── mac ──
[macOS]   Button: (OK)
[macOS]   Checkbox: ○
```

Notice:

- `Application.BuildUI` and `Application.Render` contain zero `if platform` conditionals
- Adding a Linux family means writing `LinuxFactory`, `linuxButton`, `linuxCheckbox` — and changing exactly one line in `getFactory`. No client code changes
- The concrete types (`windowsButton`, `macButton`, etc.) are unexported. The client can never reference them directly, which is enforced by the compiler

---

## Abstract Factory vs Factory Method

These two are frequently confused. The distinction is in scope.

```
Factory Method                       Abstract Factory
──────────────────────────────────   ────────────────────────────────────────
Creates ONE type of product          Creates a FAMILY of related products

Subclasses override a single         A concrete factory implements multiple
creation method                      creation methods (one per product type)

Solves: "which concrete type         Solves: "which concrete family should
should I instantiate?"               I use, and keep consistent?"

Analogy: A bakery that lets          Analogy: A bakery chain where each
each branch decide what bread        branch makes all its products
to bake                              (bread, cake, coffee) in its own style
```

In Go terms:

- Factory Method → a function or interface method that returns one interface type
- Abstract Factory → an interface with multiple methods, each returning a different product interface

---

## Problems It Introduces

Abstract Factory is not free. Know the costs:

**Adding a new product type is expensive**

If you decide the family needs a third component (e.g., `Dialog`), you must:
1. Add `CreateDialog() Dialog` to the `GUIFactory` interface
2. Implement `CreateDialog` on every existing concrete factory (`WindowsFactory`, `MacFactory`, ...)
3. Define the `Dialog` interface and all concrete implementations

This violates the Open/Closed Principle for the factory interface itself — extending the family is a breaking change to all implementors.

**Combinatorial explosion at scale**

With N product types and M families, you have N × M concrete products plus M concrete factories. In large systems this gets unwieldy.

**Indirection can obscure what is being built**

When debugging, the call to `factory.CreateButton()` tells you little about which concrete type is running. You need to trace back to where the factory was constructed.

**Can be overkill for simple cases**

If you only have one product type per family, Factory Method is simpler. Abstract Factory earns its complexity when there are multiple product types that must stay consistent.

---

## Go-Specific Notes

**Interfaces are implicit**

In Go, concrete types satisfy interfaces silently — no `implements` keyword. This means `windowsButton` satisfies `Button` as long as it has the right method set. The compiler enforces it; no registration needed. This is the normal way Abstract Factory works in Go.

**Verify interface satisfaction at compile time**

A common Go idiom to catch missing method implementations early:

```go
// These lines compile to nothing, but cause a compile error if the
// concrete type no longer satisfies the interface.
var _ GUIFactory = (*WindowsFactory)(nil)
var _ GUIFactory = (*MacFactory)(nil)
var _ Button = (*windowsButton)(nil)
```

Place these near the concrete type definitions.

**Constructor injection over global factory**

Pass the factory into types that need it via their constructor (`NewApplication(factory GUIFactory)`). Avoid package-level variables holding a factory — they make tests harder and introduce hidden coupling.

**Testing with the pattern**

Abstract Factory makes testing clean. Inject a mock factory that returns deterministic stubs:

```go
type stubButton struct{ rendered bool }
func (b *stubButton) Render()            { b.rendered = true }
func (b *stubButton) OnClick(func())     {}

type StubFactory struct{ Btn *stubButton }
func (f *StubFactory) CreateButton() Button   { return f.Btn }
func (f *StubFactory) CreateCheckbox() Checkbox { return &stubCheckbox{} }

func TestApplicationRendersButton(t *testing.T) {
    stub := &stubButton{}
    app := NewApplication(&StubFactory{Btn: stub})
    app.BuildUI()
    app.Render()
    if !stub.rendered {
        t.Fatal("expected button to be rendered")
    }
}
```

No Windows or Mac code runs in tests. The factory interface is the seam.

---

## Real-World Analogies in Go Codebases

**Database repositories**

A `RepositoryFactory` interface creates a consistent set of repositories that all target the same datastore:

```go
type RepositoryFactory interface {
    UserRepo()    UserRepository
    OrderRepo()   OrderRepository
    ProductRepo() ProductRepository
}

// PostgresFactory returns repos all sharing the same *sql.DB
// MongoFactory returns repos all sharing the same *mongo.Client
```

Swapping the factory in tests gives you in-memory fakes for the whole set at once.

**Notification providers**

A `NotificationFactory` that creates a consistent family of `Sender`, `Logger`, and `Tracker` pointing at the same third-party provider (e.g., Twilio vs SendGrid). Mixing providers for different components of the same notification would cause billing and tracking inconsistencies — Abstract Factory prevents it.

**Cloud provider SDK abstraction**

A `CloudFactory` that creates matching `ObjectStore`, `Queue`, and `SecretManager` clients all targeting the same cloud (AWS vs GCP). Each concrete factory wraps the respective SDK. In tests, a `LocalFactory` returns in-process fakes.

---

## Interview Cheat Sheet

**What it solves**

- Enforces family consistency: ensures related objects always come from the same provider
- Decouples client from concrete types completely
- Makes swapping entire families easy (change one line where the factory is constructed)

**When to use it**

- You have multiple families of related objects
- Objects within a family must be used together
- You want the client to be oblivious to which family it uses
- You want to swap implementations for testing without conditional logic in the client

**When not to use it**

- Only one product type per family → use Factory Method instead
- Families rarely change → the interface overhead is not worth it
- Adding new product types frequently → the interface change cost is too high

**Key trade-off to mention**

- Easy to add new families (just a new ConcreteFactory + products)
- Hard to add new product types (requires changing the factory interface, breaking all implementors)

**Distinguish from related patterns**

- Factory Method: one product, creation delegated to a subclass/function
- Builder: constructs one complex object step by step
- Abstract Factory: whole family of objects, selection of family is the variable

**Go-specific signal phrases**

- "I'd define the factory as a small, consumer-owned interface"
- "Concrete types would be unexported; the interface is the contract"
- "Constructor injection: the client receives the factory, it doesn't look it up globally"
- "I'd add compile-time interface checks with `var _ GUIFactory = (*WindowsFactory)(nil)`"

---

## References

- [Refactoring.Guru — Abstract Factory](https://refactoring.guru/design-patterns/abstract-factory)
- [GoF — Design Patterns: Elements of Reusable Object-Oriented Software (Creational Patterns, Ch. 3)](https://en.wikipedia.org/wiki/Design_Patterns)
- [Go Blog — Laws of Reflection (interface internals)](https://go.dev/blog/laws-of-reflection)
- [Effective Go — Interfaces](https://go.dev/doc/effective_go#interfaces)
- [Dave Cheney — Practical Go: Interface design](https://dave.cheney.net/practical-go/presentations/gophercon-israel.html)