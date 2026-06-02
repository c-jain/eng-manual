# Bridge Pattern

## Table of Contents

- [The Problem: Cartesian Product Explosion](#the-problem-cartesian-product-explosion)
- [What Bridge Is](#what-bridge-is)
- [Why It Is Called Bridge](#why-it-is-called-bridge)
- [Structure](#structure)
- [Go Implementation](#go-implementation)
- [How to Remember It](#how-to-remember-it)
- [Bridge vs Similar Patterns](#bridge-vs-similar-patterns)
- [Real-World Analogies](#real-world-analogies)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

---

## The Problem: Cartesian Product Explosion

Suppose you're building a drawing library. You have shapes (Circle, Square) and rendering engines (Vector, Raster). A naive hierarchy adds one class per combination:

```
Shape
├── VectorCircle
├── RasterCircle
├── VectorSquare
└── RasterSquare
```

Add one new shape → 2 new classes.
Add one new renderer → 2 new classes.
With M shapes and N renderers → M × N classes.

This is the **Cartesian product explosion**: two independent axes of variation are baked into one inheritance hierarchy, so they cannot grow independently.

The root cause: using inheritance to express what should be composition. The shape "is a" VectorCircle couples two concerns — what the shape is and how it is drawn — into a single type.

---

## What Bridge Is

Bridge decouples an abstraction from its implementation so that both can vary independently.

**Abstraction** — the high-level control layer. In the example, Shape. It defines the interface the client uses. It does NOT implement rendering itself; it delegates to the implementor.

**Implementor** — the low-level implementation interface. In the example, Renderer. Its concrete types do the actual work (VectorRenderer, RasterRenderer).

The abstraction holds a **reference** (not inheritance) to the implementor. This reference is the bridge. The two hierarchies are now independent: add a new shape without touching renderers, add a new renderer without touching shapes.

```
Abstraction side          bridge          Implementor side
─────────────────         ──────          ────────────────
Shape                   ─────────►        Renderer (interface)
├── Circle              composition       ├── VectorRenderer
└── Square                                └── RasterRenderer
```

---

## Why It Is Called Bridge

The reference from the abstraction to the implementor literally **bridges** two otherwise separate class hierarchies. Without it, the hierarchies would either merge (causing the M × N explosion) or remain completely disconnected (unable to cooperate). The bridge is the composition link that lets them work together while staying separate.

---

## Structure

GoF roles and their Go equivalents:

**Implementor** — an interface defining the operations the abstraction delegates to. In Go this is a plain interface defined in the abstraction's package (consumer-defined interface idiom).

**Concrete Implementor** — structs that satisfy the implementor interface. They contain the actual logic.

**Abstraction** — a struct that holds a field of the implementor interface type. Defines the high-level operations the client calls.

**Refined Abstraction** — structs that embed the Abstraction struct and add shape-specific data (radius, side length, etc.). Their methods call through to the embedded implementor reference.

```
Abstraction (Shape)
  - renderer Renderer   ← bridge field
  + [no Draw() here; refined abstractions define Draw()]

Refined Abstraction (Circle)
  embeds Shape
  + radius float64
  + Draw()  →  calls c.renderer.RenderCircle(c.radius)

Refined Abstraction (Square)
  embeds Shape
  + side float64
  + Draw()  →  calls s.renderer.RenderSquare(s.side)

Implementor (Renderer interface)
  + RenderCircle(radius float64)
  + RenderSquare(side float64)

Concrete Implementors
  VectorRenderer  -- implements Renderer
  RasterRenderer  -- implements Renderer
```

---

## Go Implementation

```go
package bridge

import "fmt"

// Implementor — defined in the abstraction's package (consumer-defined interface)
type Renderer interface {
	RenderCircle(radius float64)
	RenderSquare(side float64)
}

// Concrete Implementor A
type VectorRenderer struct{}

func (v *VectorRenderer) RenderCircle(radius float64) {
	fmt.Printf("Drawing vector circle, radius=%.2f\n", radius)
}

func (v *VectorRenderer) RenderSquare(side float64) {
	fmt.Printf("Drawing vector square, side=%.2f\n", side)
}

// Concrete Implementor B
type RasterRenderer struct{}

func (r *RasterRenderer) RenderCircle(radius float64) {
	fmt.Printf("Rasterising circle, radius=%.2f\n", radius)
}

func (r *RasterRenderer) RenderSquare(side float64) {
	fmt.Printf("Rasterising square, side=%.2f\n", side)
}

// Abstraction — holds the bridge reference
type shape struct {
	renderer Renderer
}

// Refined Abstraction: Circle
type Circle struct {
	shape
	radius float64
}

func NewCircle(r Renderer, radius float64) *Circle {
	return &Circle{shape: shape{renderer: r}, radius: radius}
}

func (c *Circle) Draw() {
	c.renderer.RenderCircle(c.radius)
}

func (c *Circle) Resize(factor float64) {
	c.radius *= factor
}

// Refined Abstraction: Square
type Square struct {
	shape
	side float64
}

func NewSquare(r Renderer, side float64) *Square {
	return &Square{shape: shape{renderer: r}, side: side}
}

func (s *Square) Draw() {
	s.renderer.RenderSquare(s.side)
}

// Compile-time check: both renderers satisfy the interface
var _ Renderer = (*VectorRenderer)(nil)
var _ Renderer = (*RasterRenderer)(nil)
```

```go
package main

import "bridge"

func main() {
	vector := &bridge.VectorRenderer{}
	raster := &bridge.RasterRenderer{}

	c1 := bridge.NewCircle(vector, 5.0)
	c1.Draw()        // Drawing vector circle, radius=5.00

	c2 := bridge.NewCircle(raster, 5.0)
	c2.Draw()        // Rasterising circle, radius=5.00

	sq := bridge.NewSquare(vector, 3.0)
	sq.Draw()        // Drawing vector square, side=3.00

	// Switch renderer at runtime
	c1.shape = bridge.shape{renderer: raster} // if exported — or inject via setter
	c1.Draw()        // Rasterising circle, radius=5.00
}
```

The implementor can be swapped at runtime because it is held as an interface field, not a concrete type. This is one advantage over inheritance-based approaches.

---

## How to Remember It

**The name is the mnemonic.** Two separate hierarchies — abstraction on one side, implementation on the other — cannot cooperate without collapsing into one. The bridge (a composition reference) connects them while keeping them separate. Draw two towers. Draw a bridge between them. The bridge is the `renderer Renderer` field.

**The problem is the mnemonic.** Any time you see a class named `<Adjective><Noun>` where the adjective and noun vary independently — `VectorCircle`, `RasterSquare`, `DarkModeButton`, `WindowsScrollbar` — think Bridge. The pattern exists to prevent exactly that naming explosion.

---

## Bridge vs Similar Patterns

**Bridge vs Adapter**

Adapter is a retrofit: you have two existing, incompatible interfaces and you write an adapter to make one look like the other. Bridge is designed upfront: you plan two hierarchies from the start and connect them through a bridge to prevent coupling.

Intent distinguishes them. Bridge intent: "prevent M×N class explosion by separating two axes of variation." Adapter intent: "make an existing interface work where a different interface is expected."

**Bridge vs Strategy**

Strategy swaps algorithms (behaviours) at runtime; the strategy interface represents a behaviour the context delegates to. Bridge separates abstraction from implementation; the implementor interface represents a platform or technology the abstraction delegates to.

Structurally they look similar — both use a field holding an interface. The difference is in intent and scope. Strategy is about choosing an algorithm. Bridge is about separating two class hierarchies.

**Bridge vs Decorator**

Decorator wraps an object to add behaviour, stacking wrappers arbitrarily deep. Bridge holds exactly one implementor reference to delegate platform-specific work. Decorators form a chain; Bridge forms a pair.

---

## Real-World Analogies

**Device + Remote Control** — a TV (implementor) and a remote control (abstraction) are separate hierarchies. Any remote can control any TV through a standard interface. Add a new remote model without changing the TV, and vice versa. This is the canonical GoF example.

**UI Toolkit + Platform Renderer** — a Button widget (abstraction) delegates rendering to a platform-specific renderer (Windows GDI, macOS Core Graphics, Linux Cairo). The widget hierarchy grows independently of the renderer hierarchy. This is how cross-platform toolkits (Qt, Flutter) work internally.

**Database ORM + Driver** — an ORM query (abstraction) delegates execution to a database driver (MySQL, Postgres, SQLite). The query API is the same; the driver is the implementor. Adding a new database requires only a new driver, not changes to the ORM.

**Go `database/sql`** — `*sql.DB` is the abstraction. The `database/sql/driver.Driver` interface is the implementor. `lib/pq` (Postgres) and `go-sqlite3` are concrete implementors. This is Bridge in the standard library.

---

## Interview Cheat Sheet

**One-line definition:**

> Bridge decouples an abstraction from its implementation so both hierarchies can grow independently — connected by composition, not inheritance.

**Common probes and strong answers:**

- *"When would you use Bridge?"* — When you anticipate two independent axes of variation. If the number of classes you'd need grows as M × N where M and N are separate concerns, Bridge applies.
- *"How is Bridge different from Adapter?"* — Bridge is designed upfront to prevent coupling. Adapter is a retrofit to reconcile two existing incompatible interfaces.
- *"How is Bridge different from Strategy?"* — Strategy is about swapping algorithms for the same abstraction. Bridge is about separating two full class hierarchies. Structurally similar; different intent and scale.
- *"Can you give a real-world example in Go?"* — `database/sql`. `*sql.DB` is the abstraction, `driver.Driver` is the implementor interface, `lib/pq` is a concrete implementor.
- *"What principle does Bridge demonstrate?"* — Composition over inheritance. Prefer a has-a reference to vary behaviour independently rather than an is-a inheritance chain that bakes two concerns together.

**Red flags to avoid:**

- Saying Bridge and Adapter are the same — intent is the key distinction.
- Implementing Bridge with inheritance in Go — Go has no classical inheritance; the pattern uses struct embedding + interface field (composition), which is idiomatic and cleaner.
- Forgetting to mention that the implementor can be swapped at runtime — this is a concrete advantage over static inheritance.

---

## References

- [Refactoring Guru: Bridge](https://refactoring.guru/design-patterns/bridge)
- [GoF: Design Patterns — Bridge, pp. 151–161]
- [Go `database/sql` package](https://pkg.go.dev/database/sql) — Bridge in the standard library
- [Dave Cheney: Composition over Inheritance in Go](https://dave.cheney.net/2015/05/22/struct-composition)