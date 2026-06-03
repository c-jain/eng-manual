# Flyweight Pattern

## Table of Contents

- [Overview](#overview)
- [Core Decomposition — Intrinsic vs Extrinsic State](#core-decomposition--intrinsic-vs-extrinsic-state)
- [Structure](#structure)
- [Go Implementation — The Forest Example](#go-implementation--the-forest-example)
- [Thread Safety in Go](#thread-safety-in-go)
- [Flyweight vs sync.Pool](#flyweight-vs-syncpool)
- [When to Use and When Not To](#when-to-use-and-when-not-to)
- [Real-World Examples](#real-world-examples)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

---

## Overview

**What it is:** Flyweight is a structural design pattern that reduces memory usage when a large number of objects share significant amounts of identical state. Instead of storing that shared state in every object, it is extracted into a single shared object — the flyweight — and referenced from each context object.

**Why it exists:** The naive approach to modelling a large number of fine-grained objects (characters in a document, trees in a forest, particles in a physics simulation) allocates one fully populated object per instance. When most of the data is identical across instances — font name, sprite image, material properties — this wastes memory proportional to the number of instances. Flyweight restructures object ownership so that shared data is stored exactly once.

**Why it is called what it is:** In boxing, a **flyweight** is the lightest weight class — fighters with the minimum possible body mass. The pattern is named for the lightweight objects it produces: objects stripped of everything that can be shared, carrying only the minimum per-instance state.

**The problem it brings:** Flyweight introduces complexity. The caller must supply the unique (extrinsic) state at call time rather than letting the object own it. The flyweight itself must be immutable (or its mutation must be carefully synchronised), because it is shared. The factory adds a layer of indirection. These costs are worth paying only when the memory savings are substantial.

**Memory aid:** Think of a typeface in traditional printing. A physical type block is a flyweight — one cast metal letter `a` is shared for every `a` in the document. The position of each `a` on the page is extrinsic state supplied by the compositor at print time, not stored in the type block itself.

---

## Core Decomposition — Intrinsic vs Extrinsic State

The entire pattern rests on splitting an object's state into two categories:

**Intrinsic state** — data that is the same across many instances; context-independent; stored inside the flyweight; must be immutable (so sharing is safe).

- Example: the species name, colour, and texture image of a tree species
- Example: the glyph bitmap and font metrics of a character
- Example: the material properties and mesh of a game sprite

**Extrinsic state** — data that is unique to each usage; context-dependent; supplied by the caller at call time; not stored in the flyweight.

- Example: the x, y coordinates and age of a specific tree on the map
- Example: the position of a character in the document
- Example: the position, rotation, and health of a specific enemy instance

**Memory aid — Intrinsic = Inside the flyweight. Extrinsic = External, passed in.**

The test: if you removed this field from the flyweight and passed it as a parameter to every method call, would the flyweight still be correct? If yes, the field is extrinsic. If no, it is intrinsic.

The flyweight object is effectively a **value object** — immutable, equality-defined by its fields, safe to share.

---

## Structure

```
                ┌─────────────────────────┐
                │    TreeTypeFactory      │  (flyweight factory)
                │  GetTreeType(key)       │
                │  cache map[string]*     │
                │         TreeType        │
                └──────────┬──────────────┘
                           │ creates / returns cached
                           ▼
                ┌─────────────────────────┐
                │      TreeType           │  (flyweight)
                │  name    string         │  ← intrinsic
                │  color   string         │  ← intrinsic
                │  texture string         │  ← intrinsic
                │  (shared · immutable)   │
                └─────────────────────────┘
                           ▲
               ┌───────────┴───────────┐
               │                       │
  ┌────────────────────┐  ┌────────────────────┐
  │   Tree  (context)  │  │   Tree  (context)  │  ...n instances
  │  x, y, age  int    │  │  x, y, age  int    │  ← extrinsic
  │  *TreeType         │  │  *TreeType         │  ← shared ref
  └────────────────────┘  └────────────────────┘
```

**Components:**

- **Flyweight** (`TreeType`) — holds intrinsic state; immutable; intended to be shared
- **Flyweight factory** (`TreeTypeFactory`) — creates flyweights on first request; caches them by key; returns the existing flyweight on subsequent requests for the same key
- **Context** (`Tree`) — holds extrinsic state; references a flyweight; does not duplicate the flyweight's data
- **Client** (`Forest`) — creates context objects by asking the factory for the appropriate flyweight

---

## Go Implementation — The Forest Example

```go
package flyweight

import (
    "fmt"
    "sync"
)

// TreeType is the Flyweight. It holds only intrinsic (shared) state.
// It is intentionally unexported — callers access it through the factory.
// Immutability is enforced by making all fields unexported and exposing
// no setters.
type TreeType struct {
    name    string
    color   string
    texture string // would be an image reference in a real renderer
}

// Draw renders a single tree using its intrinsic state combined with
// the extrinsic state (position, age) supplied by the caller.
// Note: extrinsic state is NOT stored here — it is passed in.
func (t *TreeType) Draw(x, y, age int) {
    fmt.Printf("drawing %s (color=%s) at (%d, %d), age=%d years\n",
        t.name, t.color, x, y, age)
}

// TreeTypeFactory is the Flyweight Factory. It caches TreeType instances
// keyed by their intrinsic properties. On cache hit it returns the existing
// shared instance; on cache miss it creates and stores a new one.
type TreeTypeFactory struct {
    mu    sync.Mutex
    cache map[string]*TreeType
}

func NewTreeTypeFactory() *TreeTypeFactory {
    return &TreeTypeFactory{
        cache: make(map[string]*TreeType),
    }
}

// GetTreeType returns the cached flyweight for the given key, creating
// it if necessary. The key encodes all intrinsic fields so that two calls
// with identical intrinsic properties return the same pointer.
func (f *TreeTypeFactory) GetTreeType(name, color, texture string) *TreeType {
    key := name + "|" + color + "|" + texture

    f.mu.Lock()
    defer f.mu.Unlock()

    if tt, ok := f.cache[key]; ok {
        return tt // return the shared flyweight
    }

    tt := &TreeType{name: name, color: color, texture: texture}
    f.cache[key] = tt
    return tt
}

func (f *TreeTypeFactory) CacheSize() int {
    f.mu.Lock()
    defer f.mu.Unlock()
    return len(f.cache)
}

// Tree is the Context object. It holds extrinsic (unique) state
// plus a reference to a shared flyweight.
type Tree struct {
    x, y     int        // extrinsic — unique per tree
    age      int        // extrinsic — unique per tree
    treeType *TreeType  // intrinsic — shared flyweight
}

func (t *Tree) Draw() {
    // extrinsic state is passed to the flyweight at call time
    t.treeType.Draw(t.x, t.y, t.age)
}

// Forest is the Client. It manages context objects and delegates
// flyweight acquisition to the factory.
type Forest struct {
    trees   []*Tree
    factory *TreeTypeFactory
}

func NewForest() *Forest {
    return &Forest{factory: NewTreeTypeFactory()}
}

// PlantTree creates a new Tree context. The flyweight for the given
// species/color/texture is retrieved (or created) by the factory.
// Many trees of the same species share a single *TreeType in memory.
func (f *Forest) PlantTree(x, y, age int, name, color, texture string) {
    tt := f.factory.GetTreeType(name, color, texture)
    f.trees = append(f.trees, &Tree{x: x, y: y, age: age, treeType: tt})
}

func (f *Forest) Draw() {
    for _, t := range f.trees {
        t.Draw()
    }
}
```

**Usage:**

```go
func main() {
    forest := NewForest()

    // Plant 1,000,000 trees: only 3 species → only 3 TreeType objects in memory.
    // All other allocations are the lightweight Tree context structs.
    for i := 0; i < 400_000; i++ {
        forest.PlantTree(i*3, i*2, 10+i%50, "Oak", "dark green", "oak_bark.png")
    }
    for i := 0; i < 350_000; i++ {
        forest.PlantTree(i*4, i*3, 5+i%30, "Pine", "green", "pine_bark.png")
    }
    for i := 0; i < 250_000; i++ {
        forest.PlantTree(i*2, i*5, 20+i%80, "Maple", "orange", "maple_bark.png")
    }

    // factory.CacheSize() == 3, regardless of how many trees were planted
    fmt.Printf("flyweight cache size: %d (one per species)\n", forest.factory.CacheSize())

    // Memory usage before flyweight (rough):
    //   1,000,000 trees × (name 8B + color 12B + texture 16B + x 8B + y 8B + age 8B)
    //   ≈ 60 MB for intrinsic fields alone
    //
    // Memory usage after flyweight:
    //   3 TreeType objects × 36B ≈ negligible
    //   1,000,000 Tree contexts × (x 8B + y 8B + age 8B + pointer 8B) ≈ 32 MB
    //   Savings: ~28 MB (plus reduced GC pressure)
}
```

---

## Thread Safety in Go

The factory uses `sync.Mutex` to protect its cache. An alternative is `sync.Map`, which is optimised for read-heavy workloads where entries are written once and read many times — exactly the Flyweight access pattern.

```go
// Alternative factory using sync.Map for lock-free reads after initial write
type TreeTypeFactory struct {
    cache sync.Map // key: string → value: *TreeType
}

func (f *TreeTypeFactory) GetTreeType(name, color, texture string) *TreeType {
    key := name + "|" + color + "|" + texture

    if val, ok := f.cache.Load(key); ok {
        return val.(*TreeType)
    }

    // Use LoadOrStore to avoid duplicate creation under concurrent first-access
    tt := &TreeType{name: name, color: color, texture: texture}
    actual, _ := f.cache.LoadOrStore(key, tt)
    return actual.(*TreeType)
}
```

`LoadOrStore` handles the race between two goroutines simultaneously creating the same flyweight for the first time: only one wins, and both callers get back the same pointer. The loser's allocation is simply garbage collected.

Note: with `LoadOrStore` there can be one redundant allocation per key on first concurrent access. This is acceptable because flyweights are created rarely (once per unique intrinsic combination) and the allocation is small.

---

## Flyweight vs sync.Pool

`sync.Pool` reuses objects to reduce GC pressure, which sounds similar to Flyweight. They solve different problems:

| Aspect | Flyweight | sync.Pool |
|---|---|---|
| Purpose | Share immutable intrinsic state | Reuse mutable objects to reduce GC allocation |
| Object state | Immutable; shared concurrently | Mutable; used exclusively, then returned |
| Object identity | Same pointer = same logical object | Any pooled object is interchangeable |
| Lifecycle | Lives as long as needed | Can be evicted by the GC at any time |
| Typical use | Shared metadata, sprites, font glyphs | Temporary buffers, `bytes.Buffer`, JSON decoders |

The idiomatic Go use of Flyweight is for long-lived, truly shared read-only data. `sync.Pool` is for short-lived temporary allocations where you want to amortise allocation cost.

---

## When to Use and When Not To

**Use Flyweight when:**

- You have a large number of objects — hundreds of thousands or more
- Most of the per-object state is identical across instances (good signal: most fields have the same value for the same "type" of object)
- Memory consumption or GC pressure is a measurable problem, not a theoretical one
- You can cleanly separate intrinsic from extrinsic state — if the separation is forced or unclear, the pattern adds complexity without benefit

**Do not use Flyweight when:**

- The number of objects is small — the factory overhead and added indirection are not worth it
- Objects are frequently mutated — a shared mutable flyweight requires careful synchronisation and erodes the simplicity of the pattern
- The state does not decompose cleanly — if "shared" and "unique" state are entangled, the refactoring cost exceeds the memory benefit
- A simpler approach (enum, constant, config struct) achieves the same sharing without a factory

The pattern is commonly over-applied. Profile first. If memory or GC is not a demonstrated problem, a plain struct per object is simpler and easier to reason about.

---

## Real-World Examples

**Text rendering** — A `Glyph` or `Font` flyweight holds the rasterised bitmap and metrics for each character. The document layout engine holds per-character context objects (position, baseline) that reference the shared glyph. This is the canonical GoF example.

**Tile maps in game engines** — A tile type (grass, stone, water) is a flyweight holding the sprite, walkability, and render layer. The map holds a grid of tile-context objects (coordinate, elevation) that reference flyweights.

**Go's string interning** — Go's compiler interns string literals with identical content at compile time; identical literals share backing storage. This is Flyweight applied at the language runtime level.

**Database row type descriptors** — Some database drivers keep one type descriptor per result-set column type, shared across all rows of the same schema. Per-row context objects hold column values but not type metadata.

**`net/http` header canonicalisation** — `textproto.CanonicalMIMEHeaderKey` returns canonical header name strings. For common header names ("Content-Type", "Authorization"), the same backing string is reused across many request objects rather than reallocated per request.

---

## Interview Cheat Sheet

**"What is the Flyweight pattern?"**

Flyweight reduces memory usage by sharing the immutable, context-independent portion of object state (intrinsic state) across many objects. Each object holds only its unique state (extrinsic state) and a reference to the shared flyweight.

**"What is the difference between intrinsic and extrinsic state?"**

Intrinsic state is identical across many instances and stored in the flyweight — it is independent of context. Extrinsic state is unique per usage, context-dependent, and passed in by the caller at call time rather than stored in the flyweight.

**"When would you actually use this in production?"**

Game engines (tiles, sprites), document renderers (glyph caches), geographic visualisation (millions of map objects), and any domain where you have large numbers of objects with a small number of distinct "types." Always profile first — the pattern adds real complexity.

**"How does the factory work?"**

The factory holds a cache keyed by the flyweight's intrinsic state. On first request for a given key it creates and stores a new flyweight. On subsequent requests it returns the cached instance. The cache is the only place flyweights are created; callers never call `new(TreeType)` directly.

**"How do you handle thread safety in Go?"**

Use `sync.Mutex` around the factory cache, or `sync.Map` with `LoadOrStore` for a lock-free hot path after first-write. The flyweight itself needs no synchronisation because it is immutable after construction.

**"What is the relationship between Flyweight and caching?"**

Flyweight IS a form of object-level caching. The factory's cache stores objects keyed by intrinsic state and reuses them on hit. The difference from a general cache: Flyweight cache entries are never evicted (they're long-lived shared objects), and the reuse is for identity rather than temporal locality.

**Signal phrases to use:** "separate intrinsic from extrinsic state," "factory caches flyweights by key," "immutable shared object," "pass extrinsic state at call time."

**Red flags to avoid:**

- Storing extrinsic state inside the flyweight — this breaks sharing and is the most common mistake
- Forgetting thread safety in the factory (read-write race on the cache map)
- Using Flyweight without a demonstrated memory problem — profile before applying
- Mutating a shared flyweight — always treat flyweights as immutable

---

## References

- [Refactoring Guru — Flyweight Pattern](https://refactoring.guru/design-patterns/flyweight)
- [GoF — Design Patterns: Elements of Reusable Object-Oriented Software, pp. 195–206](https://www.oreilly.com/library/view/design-patterns-elements/0201633612/)
- [Dave Cheney — Practical Go (memory and allocation considerations)](https://dave.cheney.net/practical-go/presentations/gophercon-singapore-2019.html)
- [Go standard library — sync.Map](https://pkg.go.dev/sync#Map)
- [Go standard library — sync.Pool (contrast with Flyweight)](https://pkg.go.dev/sync#Pool)
- [Go spec — string interning behaviour](https://go.dev/ref/spec#String_types)