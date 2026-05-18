# Prototype Pattern

## Table of Contents

- [What Is the Prototype Pattern?](#what-is-the-prototype-pattern)
- [The Core Concept: Shallow vs Deep Clone](#the-core-concept-shallow-vs-deep-clone)
- [GoF Formulation](#gof-formulation)
- [Go Implementation: Document Cloning](#go-implementation-document-cloning)
- [Prototype Registry](#prototype-registry)
- [When to Use vs When Not to Use](#when-to-use-vs-when-not-to-use)
- [Prototype vs Other Creational Patterns](#prototype-vs-other-creational-patterns)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)


## What Is the Prototype Pattern?

The **Prototype pattern** is a creational design pattern where you create new objects by **cloning an existing object** (the *prototype*) rather than constructing a new one from scratch via `new` or a factory.

**Why it exists:** Sometimes, creating an object is expensive — it may require a DB query, a network call, complex initialisation, or heavy computation. If you need many similar objects that differ only slightly, it's far cheaper to clone an already-set-up object than to construct each one fresh.

**Why it's called "Prototype":** From product design — a *prototype* is the reference specimen that others are modelled after. In this pattern, one configured object serves as the specimen; all others are clones of it.

**The core problem it solves:**
- Avoid expensive repeated construction
- Decouple the client from the concrete class — the client just says "give me a copy of this" without needing to know how to build one
- Produce object variants without subclassing

**What problems it introduces:**
- **Shallow clone pitfalls:** naive cloning shares mutable reference-type fields between the original and the clone — mutations in one unexpectedly affect the other
- **Non-clonable resources:** objects holding open file handles, DB connections, mutexes, or OS resources cannot be meaningfully cloned
- **Circular references:** if object A holds a reference to object B and B holds back to A, a deep clone recurses infinitely without cycle detection


## The Core Concept: Shallow vs Deep Clone

This is the most important concept in the pattern — and the #1 interview probe point.

**Shallow clone:** copies the object's fields directly. For value types (int, float, bool, small structs), this is a complete, independent copy. For reference types (pointers, slices, maps), the clone shares the same underlying memory as the original.

**Deep clone:** recursively copies all referenced data. The clone is fully independent — mutations in the clone do not affect the original.

```
Original:  ┌──────────────────────────────┐
           │ name: "Alice"                │
           │ tags: ptr ─────────────────────────► ["admin", "user"]
           └──────────────────────────────┘              ▲

Shallow:   ┌──────────────────────────────┐              │
           │ name: "Alice"                │              │ (shared backing array!)
           │ tags: ptr ───────────────────────────────────┘
           └──────────────────────────────┘
              Appending to clone's tags affects the original.

Deep:      ┌──────────────────────────────┐
           │ name: "Alice"                │
           │ tags: ptr ─────────────────────────► ["admin", "user"]  (new copy)
           └──────────────────────────────┘
              Fully independent — no shared memory.
```

**In Go specifically:**

A struct assignment (`cloned := *original`) copies all fields by value — but a slice field copies only the slice *header* (a pointer, a length, and a capacity), not the backing array. Two struct values then share the same backing array. Any append that doesn't grow the slice writes to shared memory silently.

```go
type Doc struct {
    Tags []string
}

original := Doc{Tags: []string{"a", "b"}}
shallow := original          // struct copy — Tags header is copied, backing array is shared

shallow.Tags[0] = "CHANGED"  // modifies original.Tags[0] too!

// For a deep copy:
deep := Doc{Tags: make([]string, len(original.Tags))}
copy(deep.Tags, original.Tags)
deep.Tags[0] = "CHANGED"     // original is unaffected
```


## GoF Formulation

The classical GoF structure:

```
«interface»
Prototype
  + Clone() Prototype
       ▲
       │ implements
       │
ConcretePrototype
  - field1
  - field2
  + Clone() Prototype    ← returns a deep copy of itself
```

- `Prototype` — the interface declaring `Clone()`
- `ConcretePrototype` — implements `Clone()` with the correct deep-copy logic
- `Client` — holds a reference to a `Prototype`; calls `Clone()` to get new instances without knowing the concrete type

In Go, this maps naturally to an interface:

```go
// Prototype is the interface any cloneable type implements.
type Prototype interface {
    Clone() Prototype
}
```


## Go Implementation: Document Cloning

A document editor where creating a document from a template is expensive (pre-fetched metadata, styles, permissions from a DB). Instead of re-fetching everything for each new document, clone the template.

```go
package main

import "fmt"

// Document is the concrete prototype.
// It mixes value fields (cheap to copy) and reference fields (need deep copy).
type Document struct {
    Title    string            // value type — copied correctly by struct assignment
    Content  string            // value type
    Tags     []string          // reference type — must deep copy the backing array
    Metadata map[string]string // reference type — must deep copy the map
}

// Clone returns a fully independent deep copy of the Document.
// This is the heart of the Prototype pattern.
func (d *Document) Clone() *Document {
    // Value fields are correctly copied via struct literal assignment.
    cloned := &Document{
        Title:   d.Title,
        Content: d.Content,
    }

    // Deep copy the slice — allocate a new backing array and copy elements.
    cloned.Tags = make([]string, len(d.Tags))
    copy(cloned.Tags, d.Tags)

    // Deep copy the map — allocate a new map and copy each key-value pair.
    cloned.Metadata = make(map[string]string, len(d.Metadata))
    for k, v := range d.Metadata {
        cloned.Metadata[k] = v
    }

    return cloned
}

func (d *Document) String() string {
    return fmt.Sprintf("Document{Title: %q, Tags: %v, Metadata: %v}",
        d.Title, d.Tags, d.Metadata)
}

func main() {
    // The prototype — imagine this came from an expensive DB query.
    template := &Document{
        Title:   "Q4 Report Template",
        Content: "## Executive Summary\n...",
        Tags:    []string{"finance", "quarterly"},
        Metadata: map[string]string{
            "author":  "finance-team",
            "version": "1",
        },
    }

    // Clone the prototype — cheap; no DB query needed.
    doc1 := template.Clone()
    doc1.Title = "Q4 2025 Report — APAC"
    doc1.Tags = append(doc1.Tags, "apac")   // does NOT affect template.Tags
    doc1.Metadata["version"] = "2"          // does NOT affect template.Metadata

    doc2 := template.Clone()
    doc2.Title = "Q4 2025 Report — EMEA"
    doc2.Tags = append(doc2.Tags, "emea")

    fmt.Println("Template:", template)
    // Template: Document{Title: "Q4 Report Template", Tags: [finance quarterly], ...}

    fmt.Println("Doc1:", doc1)
    // Doc1: Document{Title: "Q4 2025 Report — APAC", Tags: [finance quarterly apac], ...}

    fmt.Println("Doc2:", doc2)
    // Doc2: Document{Title: "Q4 2025 Report — EMEA", Tags: [finance quarterly emea], ...}
}
```

**Key observation:** `doc1` and `doc2` are fully independent from `template` and from each other. Mutations to their `Tags` and `Metadata` do not bleed back. This is only guaranteed because `Clone()` explicitly deep-copies the reference-type fields.


## Prototype Registry

A common extension — the **Prototype Registry** (also called a Prototype Manager) stores named prototypes and hands out clones on demand. Clients don't need to hold prototype references themselves; they request clones by name.

```
┌───────────────────────────────────────┐
│           DocumentRegistry            │
│                                       │
│  "quarterly-report" → Document (proto)│
│  "incident-report"  → Document (proto)│
│                                       │
│  + Register(name, doc)                │
│  + Get(name) → *Document (clone)      │
└───────────────────────────────────────┘
              │
              │ Get("quarterly-report") returns Clone()
              ▼
         New Document (independent copy)
```

```go
// DocumentRegistry stores named prototypes and returns clones on request.
// Clients are decoupled from both the construction logic and the concrete type.
type DocumentRegistry struct {
    prototypes map[string]*Document
}

func NewDocumentRegistry() *DocumentRegistry {
    return &DocumentRegistry{prototypes: make(map[string]*Document)}
}

// Register stores the given document as a named prototype.
// The registry takes ownership of the reference; Clone() is called on retrieval.
func (r *DocumentRegistry) Register(name string, doc *Document) {
    r.prototypes[name] = doc
}

// Get returns a deep clone of the named prototype.
// Returns nil if the name is not registered.
func (r *DocumentRegistry) Get(name string) *Document {
    if proto, ok := r.prototypes[name]; ok {
        return proto.Clone()
    }
    return nil
}

func main() {
    registry := NewDocumentRegistry()

    registry.Register("quarterly-report", &Document{
        Title:    "Quarterly Report Template",
        Tags:     []string{"finance"},
        Metadata: map[string]string{"version": "1"},
    })

    registry.Register("incident-report", &Document{
        Title:    "Incident Report Template",
        Tags:     []string{"ops", "incident"},
        Metadata: map[string]string{"severity": "unknown"},
    })

    // Client requests clones by name — no Document construction knowledge needed.
    apacReport := registry.Get("quarterly-report")
    apacReport.Title = "Q1 2026 APAC Report"

    incidentA := registry.Get("incident-report")
    incidentA.Metadata["severity"] = "P1"

    fmt.Println(apacReport)  // Title: Q1 2026 APAC Report, Tags: [finance]
    fmt.Println(incidentA)   // Metadata: {severity: P1, ...}
}
```


## When to Use vs When Not to Use

**Use Prototype when:**
- Object creation is expensive (involves I/O, network calls, heavy computation)
- You need many variants of a configured object that differ only slightly
- You want to decouple the client from the concrete type — it only needs the `Clone()` interface
- You want point-in-time snapshots of mutable objects (clone captures state at that moment)

**Do not use Prototype when:**
- Objects are cheap to construct — cloning adds complexity with no payoff
- The object holds resources that cannot be meaningfully cloned: open file handles, DB connections, mutexes, goroutine channels
- Objects have circular references — deep cloning recurses infinitely without explicit cycle detection
- The number of required object variants is better handled by a Builder (step-by-step configuration) or Factory (one type per variant)


## Prototype vs Other Creational Patterns

| | Factory Method | Abstract Factory | Builder | Prototype |
|---|---|---|---|---|
| Creates via | Subclass / function dispatch | Family of coordinated factories | Step-by-step director + builder | Cloning an existing instance |
| Knows the concrete type? | Yes | Yes | Yes | No — only needs `Clone()` |
| Cost per creation | Full construction each time | Full construction each time | Full construction each time | Cheap if the prototype is already built |
| Primary strength | Single product type variants | Coordinated product families | Complex object construction | Many similar objects; expensive construction |
| Primary weakness | Requires subclassing for each variant | High structural overhead | Verbose for simple objects | Shallow-copy bugs; non-clonable resources |


## Interview Cheat Sheet

**The one-sentence definition:**
> Prototype creates new objects by cloning an existing instance rather than constructing from scratch — useful when construction is expensive and you need many similar variants.

**Shallow vs deep clone — what to say:**
> In Go, a struct assignment copies value fields correctly but only copies the slice header for slice fields — pointer, length, capacity — not the backing array. Two struct values then share the same backing array, so mutations in one affect the other silently. A deep clone must allocate new memory for every reference-type field.

**When to prefer Prototype over Factory:**
> When you already have a valid, fully-configured object and construction would require repeating expensive work (a DB query, a network call). Factory builds from scratch every time; Prototype pays the construction cost once.

**Common follow-up questions:**
- "What's the difference between shallow and deep copy?" — above; mention the slice header issue in Go specifically
- "How do you clone an object with nested structs?" — recurse: for each nested struct that contains reference types, call a dedicated Clone method on it; build a deep-clone chain
- "What's a Prototype Registry?" — named store of prototypes; clients get independent clones by name; decouples client from construction AND from the type
- "What are the risks of Prototype?" — accidental shallow copies (silent shared state), non-clonable resources (connections, locks), circular references causing infinite recursion
- "How does Prototype relate to the copy constructor in C++?" — the same idea; Go has no built-in copy constructor, so `Clone()` on an interface is the idiomatic equivalent


## References

- [GoF — Design Patterns: Elements of Reusable Object-Oriented Software](https://www.goodreads.com/book/show/85009.Design_Patterns) — Gamma et al., Creational Patterns chapter
- [Refactoring Guru — Prototype](https://refactoring.guru/design-patterns/prototype)
- [Go Blog — The Laws of Reflection](https://go.dev/blog/laws-of-reflection) (relevant for generic deep-copy via reflection)
- [Go specification — Slice types and backing arrays](https://go.dev/ref/spec#Slice_types)