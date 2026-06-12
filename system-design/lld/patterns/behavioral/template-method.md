---
Status: 🌳 Evergreen
Created: 2026-06-12
Last Updated: 2026-06-12
---

# Template Method Pattern

## Table of Contents

- [What It Is](#what-it-is)
- [Why It Exists](#why-it-exists)
- [Why It Is Called Template Method](#why-it-is-called-template-method)
- [Structure](#structure)
- [Participants](#participants)
- [Hooks vs Abstract Steps](#hooks-vs-abstract-steps)
- [Go Implementation](#go-implementation)
- [Template Method vs Strategy](#template-method-vs-strategy)
- [Real-World Examples](#real-world-examples)
- [Problems It Brings](#problems-it-brings)
- [How to Remember This](#how-to-remember-this)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

---

## What It Is

**Classification**: Behavioral (GoF)

A base class defines the *skeleton* of an algorithm in a single method — the template method —
and delegates specific steps to subclasses. The overall algorithm structure is fixed; only the
individual steps vary.

The template method calls a sequence of steps in a defined order. Some steps are abstract
(subclasses must implement). Some are "hooks" — methods with a default no-op implementation
that subclasses may optionally override.

---

## Why It Exists

You have a multi-step process where every variant follows the same sequence but implements
individual steps differently. Without this pattern, you end up in one of two bad places:

**Copy-paste inheritance**: each subclass duplicates the orchestration logic and tweaks a few
steps. When the sequence changes, you update N files.

**Bloated conditionals in the base**: the base class checks `if type == "A"` for each step
variant. Every new type touches the base class.

Template Method says: "I own the sequence. You own the steps." The orchestration logic lives
in exactly one place — the template method.

---

## Why It Is Called Template Method

"Template" as in a form with blanks — a fixed structure where specific spots are left open to
be filled in (think: a legal document template or a tax form). "Method" because a single method
on the base class *is* the template — it orchestrates everything.

The method IS the template. The template IS the method.

---

## Structure

```
Template Method — Class Diagram

┌──────────────────────────────────────────────┐
│              AbstractClass                   │
├──────────────────────────────────────────────┤
│ + TemplateMethod()                           │  ← fixed; defines algorithm order
│   calls: Step1 → Step2 → Hook → Step3        │
├──────────────────────────────────────────────┤
│ # Step1()  [abstract]                        │  ← must override
│ # Step2()  [abstract]                        │  ← must override
│ # Hook()   [default: no-op]                  │  ← may override
│ # Step3()  [concrete, shared]                │  ← rarely overridden
└───────────────────────┬──────────────────────┘
                        △  (inherits)
             ┌──────────┴──────────┐
             │                     │
  ┌──────────┴─────┐    ┌──────────┴─────┐
  │  ConcreteA     │    │  ConcreteB     │
  ├────────────────┤    ├────────────────┤
  │ Step1()        │    │ Step1()        │
  │ Step2()        │    │ Step2()        │
  │ Hook()         │    │  (no override) │
  └────────────────┘    └────────────────┘
```

---

## Participants

**AbstractClass** — defines `TemplateMethod()`, which orchestrates the algorithm by calling
abstract steps, hooks, and shared concrete steps. This is where the invariant logic lives.

**ConcreteClass** — implements the abstract steps. May optionally override hooks. Does NOT
change the algorithm's structure or the order of calls.

**Template Method** — the single orchestrating method. In classic GoF, it is `final`
(not overridable). In Go this is enforced by convention — you put the template method on a
standalone function or on a type the caller does not extend.

**Hook** — an optional extension point. The base class provides a default (usually empty)
implementation. Subclasses can override if they need to act at that point in the algorithm.

---

## Hooks vs Abstract Steps

```
Scenario A — Mandatory step (abstract)

  AbstractClass.TemplateMethod()
    calls Step1()
    Step1 has no default → subclass MUST implement it
    (missing implementation = compile error)

Scenario B — Optional step (hook)

  AbstractClass.TemplateMethod()
    calls Hook()
    Hook has a no-op default → subclass MAY override
    (not overriding = default behavior, no error)
```

Use abstract steps for things that fundamentally differ per subclass and have no sensible
default. Use hooks for optional behaviors like logging, notifications, or pre/post actions.

---

## Go Implementation

Go has no abstract classes. Template Method maps to two idiomatic patterns.

### Approach 1 — Interface as the Abstract Class

The interface declares the variable steps. A standalone function is the template method.

```go
package pipeline

import (
    "fmt"
    "strings"
)

// DataPipeline declares the variable steps.
// The interface IS the abstract class.
type DataPipeline interface {
    Fetch() []byte
    Transform(data []byte) []byte
    Store(data []byte) error
}

// Run is the template method — fixed algorithm order, injected steps.
func Run(p DataPipeline) error {
    data := p.Fetch()
    transformed := p.Transform(data)
    if err := p.Store(transformed); err != nil {
        return fmt.Errorf("store failed: %w", err)
    }
    return nil
}

// CSVPipeline — concrete implementation A.
type CSVPipeline struct{}

func (c *CSVPipeline) Fetch() []byte {
    return []byte("name,age\nAlice,30")
}

func (c *CSVPipeline) Transform(data []byte) []byte {
    return []byte(strings.ToUpper(string(data)))
}

func (c *CSVPipeline) Store(data []byte) error {
    fmt.Printf("Saving CSV: %s\n", data)
    return nil
}

// JSONPipeline — concrete implementation B.
type JSONPipeline struct{}

func (j *JSONPipeline) Fetch() []byte {
    return []byte(`{"name":"Bob","age":25}`)
}

func (j *JSONPipeline) Transform(data []byte) []byte {
    return data // no transformation needed for this source
}

func (j *JSONPipeline) Store(data []byte) error {
    fmt.Printf("Saving JSON: %s\n", data)
    return nil
}
```

### Approach 2 — Embedding for Hooks with Defaults

When you need hooks (optional overridable steps with a sensible default), embed a base struct
to provide those defaults. Concrete types embed the base struct and override only what they need.

```go
package report

import "fmt"

// BaseReport provides default implementations for hook methods.
// Embed this to get free no-op hooks without implementing them yourself.
type BaseReport struct{}

func (b *BaseReport) OnBefore() {}                        // hook: default is no-op
func (b *BaseReport) OnAfter()  {}                        // hook: default is no-op
func (b *BaseReport) Footer() string {                    // shared concrete step
    return "--- END OF REPORT ---"
}

// ReportRenderer declares both mandatory steps and hooks in one interface.
type ReportRenderer interface {
    OnBefore()
    Title() string
    Body() string
    Footer() string
    OnAfter()
}

// Render is the template method.
func Render(r ReportRenderer) string {
    r.OnBefore()
    out := r.Title() + "\n" + r.Body() + "\n" + r.Footer()
    r.OnAfter()
    return out
}

// SalesReport — implements mandatory steps; inherits hook defaults from BaseReport.
type SalesReport struct {
    BaseReport // gets OnBefore, OnAfter, Footer for free
}

func (s *SalesReport) Title() string { return "=== SALES REPORT ===" }
func (s *SalesReport) Body() string  { return "Q3 Revenue: $1,200,000" }
// Footer, OnBefore, OnAfter → inherited from BaseReport (defaults)

// AuditReport — overrides both a hook and a concrete step.
type AuditReport struct {
    BaseReport
}

func (a *AuditReport) Title() string  { return "=== AUDIT REPORT ===" }
func (a *AuditReport) Body() string   { return "All ledgers balanced." }
func (a *AuditReport) Footer() string { return "--- CONFIDENTIAL ---" } // override concrete step
func (a *AuditReport) OnAfter()       { fmt.Println("Audit log written.") } // override hook

// Usage
func main() {
    sales := &SalesReport{}
    audit := &AuditReport{}
    fmt.Println(Render(sales))
    fmt.Println(Render(audit))
}
```

---

## Template Method vs Strategy

Both solve "varying an algorithm." The difference is mechanism and granularity.

```
Template Method — inheritance / embedding

  AbstractClass
    TemplateMethod() → calls Step1, Step2 in fixed order
    Step1 (abstract)
    Step2 (abstract)

  ConcreteA embeds AbstractClass → provides Step1, Step2
  ConcreteB embeds AbstractClass → provides Step1, Step2

  Algorithm STRUCTURE is fixed. Individual steps vary.
  Variation decided at class definition time (compile time).


Strategy — composition / interface injection

  Context
    Execute(s Strategy) → delegates the whole algorithm to s

  StrategyA implements Strategy
  StrategyB implements Strategy

  The ENTIRE algorithm varies.
  Variation decided at runtime (inject whichever strategy you want).
```

| Dimension | Template Method | Strategy |
|-----------|-----------------|----------|
| Mechanism | Inheritance / embedding | Composition / injection |
| What varies | Individual steps | Entire algorithm |
| When decided | Compile time (type) | Runtime (injection) |
| Go idiom | Embedding + interface | Interface field / function arg |
| Coupling | Tighter (base ↔ subclass) | Looser (context ↔ strategy) |

**Rule of thumb**: vary *specific steps* of a fixed workflow → Template Method. Swap out the
*entire* algorithm at runtime → Strategy. In Go, Strategy is more natural because Go
discourages deep inheritance — prefer composition.

---

## Real-World Examples

- **Data processing pipelines** — fetch → validate → transform → sink; the sequence is fixed,
  concrete source/sink types vary the steps
- **Report generation** — header → body → footer; structure fixed, content differs by report type
- **Game turn loops** — initialise → player input → update state → render → check win; the loop
  is fixed, what each step does varies per game
- **`sort.Interface` in Go** — `sort.Sort()` is the template method; `Len()`, `Less()`, `Swap()`
  are the abstract steps you implement for your type
- **HTTP middleware chains** — the overall request lifecycle is fixed (parse → authenticate →
  route → respond); individual handlers vary specific steps

---

## Problems It Brings

**Fragile base class**: changes to the algorithm structure in the base class silently change
behavior in all subclasses. They still compile but the semantics shift. This is the primary
reason Template Method is less favored than Strategy in modern Go code.

**Tight coupling**: subclasses depend on internal details of the base — the order of calls,
which steps exist, which state is shared. Adding a new step to the base forces an audit of
every subclass.

**Inverted control is confusing**: the base class calls the subclass ("Hollywood Principle:
don't call us, we'll call you"). Understanding when your code runs requires reading the base.

**Hard to test steps in isolation**: you cannot unit-test `Step2` without running `Step1`
first unless you extract them — which partially defeats the encapsulation.

**Liskov Substitution risk**: a subclass that overrides steps to do something semantically
incompatible with the base class's expectations breaks LSP silently.

---

## How to Remember This

**"The boss writes the schedule; you fill in the tasks."**

The base class (boss) writes the template method (the schedule — what happens in what order).
Subclasses (you) fill in the actual implementation of each step.

For the naming: "template" = a fixed form with blanks. "Method" = one method on the base class
that owns the whole algorithm. The method IS the template.

For hooks vs abstract steps: **"abstract = must do, hook = may do."**

For Template Method vs Strategy: **"Template fixes the sequence. Strategy replaces the whole thing."**

---

## Interview Cheat Sheet

**Signal Phrases**

- "Template Method defines the algorithm skeleton in the base class and defers specific steps
  to subclasses — the overall sequence is fixed, not the individual steps."
- "Hooks give subclasses an optional extension point with a default no-op. They don't have to
  override it, but they can."
- "In Go there are no abstract classes, so Template Method maps to a standalone function
  accepting an interface — the function is the template method, the interface is the abstract
  class."
- "The primary risk is the fragile base class problem — changing the template method silently
  changes behavior in every subclass."
- "Template Method is inheritance-based and compile-time. If I need runtime flexibility,
  Strategy is the right choice."

**Red Flags**

- Confusing Template Method with Strategy — similar problem, opposite mechanisms
- Saying the template method should be overridable — in GoF it is explicitly `final`; the
  whole point is that the skeleton is fixed
- Treating hooks and abstract steps as the same thing — hooks are optional with defaults,
  abstract steps are mandatory
- Not mentioning the fragile base class problem when asked about drawbacks

**Common Interview Probes**

- *"Implement a data migration tool where CSV, JSON, and XML sources all follow fetch →
  transform → store"* → Template Method; interface as abstract class, standalone `Run()`
  as template method
- *"What is the difference between Template Method and Strategy?"* → mechanism (inheritance vs
  composition), granularity (steps vs whole algorithm), compile-time vs runtime
- *"How do you implement Template Method in a language without abstract classes?"* → interface
  + standalone function (Go), duck typing (Python / JS)
- *"When would you prefer Strategy over Template Method?"* → when the entire algorithm varies
  at runtime, or when you want looser coupling between the caller and the implementation

---

## References

- Design Patterns: Elements of Reusable Object-Oriented Software — GoF, p. 325
- [Refactoring.Guru — Template Method](https://refactoring.guru/design-patterns/template-method)
- [Go by Example — Interfaces](https://gobyexample.com/interfaces)