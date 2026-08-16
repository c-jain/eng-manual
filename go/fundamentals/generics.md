---
Status: 🌳 Evergreen
Created: 2026-08-13
Last Updated: 2026-08-13
---

# Generics

## Table of Contents

- [What Generics Are](#what-generics-are)
- [Why Generics Exist](#why-generics-exist)
- [Why It Is Called "Generics"](#why-it-is-called-generics)
- [Type Parameters And Constraints](#type-parameters-and-constraints)
- [`any` Vs `comparable`](#any-vs-comparable)
- [The `~` Approximation Element](#the--approximation-element)
- [Type Inference And Its Limits](#type-inference-and-its-limits)
- [Why No Generic Methods](#why-no-generic-methods)
- [How The Compiler Implements Generics](#how-the-compiler-implements-generics)
- [Generics Vs Interfaces](#generics-vs-interfaces)
- [Where Generic Code Lives In Memory](#where-generic-code-lives-in-memory)
- [Common Pitfalls](#common-pitfalls)
- [Production Practice](#production-practice)
- [How To Remember](#how-to-remember)
- [Problems The Feature Brings](#problems-the-feature-brings)
- [Self Test](#self-test)
- [References](#references)
- [Related Topics](#related-topics)

## What Generics Are

Generics let you write a function or type that works with **any type that satisfies a stated contract**, without duplicating code and without giving up compile-time type safety. In Go, that contract is called a **type constraint**, and the placeholder for "some type" is a **type parameter**.

```go
func Min[T cmp.Ordered](a, b T) T {
    if a < b { return a }
    return b
}
```

- `[T cmp.Ordered]` — the type parameter list. `T` is the name; `cmp.Ordered` is the constraint.
- Inside the body, `T` behaves like a normal type — but the compiler only allows operations the constraint guarantees (here: `<`, `>`, `==`).
- At call sites the caller either passes types explicitly (`Min[int](1, 2)`) or lets **type inference** fill them in (`Min(1, 2)`).

Generics landed in Go 1.18 (March 2022). They are the single largest language addition since Go 1.0.

## Why Generics Exist

Before 1.18, if you needed a "container that works for any type" or "reduce for any slice," you had exactly two options and both were bad:

1. **Copy the code per type.** `IntMin`, `Float64Min`, `StringMin`. Real production code — even the stdlib's `math` package — did this.
2. **Use `interface{}` (now `any`).** Loses type safety, requires callers to type-assert every returned value, and forces boxing of value types onto the heap.

```go
// Old-world "generic" min. Every caller pays a type assertion tax.
func MinAny(xs []any) any { /* ... */ }

n := MinAny([]any{3, 1, 4}).(int) // runtime panic if you get the type wrong
```

Generics eliminate both. `Min[T cmp.Ordered]` is one implementation, the compiler checks types at build time, and no boxing happens for value types.

The other, less-advertised motivation: **algorithms in libraries**. Sorting, searching, LRU caches, priority queues, `slices.Clone`, `maps.Keys` — every one of these existed as a hand-rolled hack per type before generics. `slices` and `maps` in the stdlib (added 1.21) are the payoff.

## Why It Is Called "Generics"

The term is 1980s CS lineage. Ada had *generic packages*, Modula-3 had *generic modules*, C++ had *templates* (a different mechanism, same idea), and Java added *generic types* in 1.5 (2004). All express the same thing: **code that is generic over a type — not tied to any single one**.

Go's own docs and the design doc call them "type parameters" more often than "generics." That's the precise name (a *parameter* that happens to hold a type, just like a value parameter holds a value). "Generics" is the umbrella term the industry uses.

## Type Parameters And Constraints

A constraint is just an **interface** — Go didn't invent a new construct for it. Any interface can be used as a constraint; the reverse is also true (with limits, see below).

```go
// A constraint written as a normal interface.
type Number interface {
    int | int64 | float32 | float64
}

func Sum[T Number](xs []T) T {
    var s T
    for _, x := range xs { s += x }
    return s
}
```

Two things generics added to the interface syntax:

- **Type union** (`A | B | C`) — this interface is satisfied by any of `A`, `B`, `C`.
- **Approximation** (`~T`) — this interface is satisfied by any type whose *underlying type* is `T`. See the section below.

`any` (an alias for `interface{}`, added as a keyword in 1.18) is the trivial constraint — "any type at all." `comparable` is a built-in constraint — "any type that supports `==` and `!=`." Everything else, you write yourself or import from `cmp`/`constraints`.

## `any` Vs `comparable`

They look similar but the difference is enforced at the constraint level and matters constantly.

| Constraint | Meaning | Common Use |
|---|---|---|
| `any` | Any type, including slices, maps, functions | Containers that don't need equality (`Stack[T]`, `List[T]`) |
| `comparable` | Only types usable as a map key (`==` and `!=` work) | Sets, map keys, `Contains`, deduplication |

```go
type Stack[T any]         struct { /* push, pop — no equality needed */ }
type Set[T comparable]    struct { /* uses map[T]struct{} internally */ }
```

Try to write `Set[[]int]` and you get a **compile error** — slices aren't comparable. That's the point: the constraint pushes the failure to build time, not to a runtime panic buried inside a hash function.

Gotcha since Go 1.20: interface types are `comparable`, even though comparing them can panic at runtime (if the dynamic type is not comparable — e.g., a slice). This was a deliberate relaxation to let generic code work with interfaces as map keys. Don't be surprised.

## The `~` Approximation Element

`~int` means "any type whose *underlying type* is `int`" — so both `int` itself and named types like `type Age int` satisfy it. Without `~`, only `int` exactly satisfies `int`.

```go
type Signed interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64
}

func Abs[T Signed](x T) T {
    if x < 0 { return -x }
    return x
}

type Celsius int
Abs(Celsius(-42)) // works, because Celsius's underlying type is int
```

Without the `~`, `Abs(Celsius(-42))` would be a compile error, and you'd force the caller to write `Abs(int(c))` — losing the domain type at the boundary. In practice, **almost every numeric constraint you write should use `~`**. The stdlib `cmp.Ordered` uses it internally for exactly this reason.

Read `~T` as "**T-ish**" — like it, or based on it.

## Type Inference And Its Limits

The compiler tries to infer type parameters from function arguments so callers don't have to write them.

```go
Min(3, 7)              // inferred T = int
Map([]int{1,2,3}, f)   // inferred In = int, Out = ...
```

Two limits worth memorising:

1. **No arguments, no inference.** `func Zero[T any]() T` must be called as `Zero[int]()`. There's nothing for the compiler to look at.
2. **Return-type-only inference is limited.** The compiler infers from *argument* types, not from the type of the assignment target. `var x int = MakeSomething()` will not infer `T = int` from `x`.

Go 1.21 improved inference significantly (partial inference, better untyped constant handling), but the two rules above still bite.

```go
func Zero[T any]() T { var z T; return z }

// z := Zero()      // compile error: cannot infer T
z := Zero[int]()    // works
```

## Why No Generic Methods

You cannot declare a method with its own type parameter list:

```go
type Box[T any] struct{ v T }

// COMPILE ERROR: method must have no type parameters
func (b Box[T]) MapTo[U any](f func(T) U) Box[U] { /* ... */ }
```

The type parameters on the *receiver* (`Box[T]`) are fine — they were fixed when the type was instantiated. What Go forbids is introducing a **new** type parameter on the method itself.

Why? Go's method sets underpin interface satisfaction. If `MapTo[U]` were allowed, an interface referring to `MapTo` would need to be satisfied only after picking a `U` — but interfaces are checked structurally at every use site, and you'd need infinitely many instantiations of the interface (one per possible `U`) to check satisfaction. The proposal authors explicitly excluded this to keep interface satisfaction decidable and cheap.

The workaround is a **generic top-level function** that takes the receiver as its first argument:

```go
func MapBox[T, U any](b Box[T], f func(T) U) Box[U] {
    return Box[U]{v: f(b.v)}
}
```

This is a common shape in `golang.org/x/exp/slices` and `golang.org/x/exp/maps` — none of the stdlib generic helpers are methods; they're all package-level functions.

## How The Compiler Implements Generics

Two textbook implementation strategies exist:

- **Monomorphization** (C++, Rust) — the compiler generates a fresh copy of the code for every concrete type combination used. Zero runtime cost, but code bloat and slow builds.
- **Boxing / dictionary passing** (Java, older OCaml) — one copy of the code that works via a pointer to type information at runtime. Small binaries, but every value gets boxed and every operation goes through an indirect call.

Go picked a **hybrid**, called **GC shape stenciling**:

- The compiler generates one copy of the function per distinct **GC shape** (roughly: "types that look the same to the garbage collector — pointers, non-pointers, size, alignment").
- All `int`, `int32`, `float32` share a shape. All pointer types share a shape. So `Sum[int32]`, `Sum[float32]`, and `Sum[uint32]` end up sharing the *same generated function body*.
- Inside that shared body, operations that depend on the concrete type (method calls, `==`, arithmetic on differently-sized types) go through a per-instantiation **dictionary** — a small runtime table passed in as an implicit argument.

The upshot:

- **Binaries stay small** — no full monomorphization explosion.
- **There is a runtime cost** — dictionary lookups are indirect calls, and the compiler's inliner cannot see through them as easily as a monomorphized copy.
- **In benchmarks**, generic code is typically **as fast as** a hand-written type-specific version for pointer types and small ints, and **noticeably slower** (10-30%) than a hand-rolled non-generic version for hot inner loops where the dictionary indirection matters. It is almost always faster than the `interface{}` alternative.

The interview-quality summary: *"Go's generics use GC shape stenciling — one function body per GC shape, plus a per-instantiation dictionary for type-specific operations. Not full monomorphization, not full boxing."*

## Generics Vs Interfaces

Both let you write code that works for multiple types. They solve different problems.

| Question | Use An Interface | Use Generics |
|---|---|---|
| Different types with different **behaviour** you want to dispatch on? | ✅ (`io.Reader`, `Stringer`) | ❌ |
| Same algorithm parameterised over the **type of data**? | ❌ | ✅ (`Min`, `Map`, `Contains`) |
| Need to hold **heterogeneous** values in one container? | ✅ (`[]any`, `[]io.Reader`) | ❌ (`Stack[T]` is one type) |
| Need to preserve the concrete type across a boundary? | ❌ (interfaces lose the type) | ✅ (generics keep `T`) |
| Care about **allocation** on a value type? | ❌ (interface boxes) | ✅ (no box) |

Rule of thumb: **interfaces express polymorphism of behaviour; generics express polymorphism of data**. If two implementations of the same interface would have identical method bodies with only the type changing, you want generics. If they'd have genuinely different bodies (`File.Read` vs `bytes.Buffer.Read`), you want an interface.

Sometimes you use both. `slices.SortFunc[S ~[]E, E any](s S, cmp func(a, b E) int)` — generic over the element type, but takes an ordinary function for the behaviour.

## Where Generic Code Lives In Memory

Per the standing memory-layout rule — walk the four segments:

- **Text/Code segment** — the *generated function bodies* live here. There is one body per distinct GC shape used in the program. `Sum[int32]` and `Sum[uint32]` typically share a body; `Sum[int]` and `Sum[*User]` do not (different shapes).
- **Data/BSS segment** — the per-instantiation **dictionaries** live here. A dictionary is a small static table (method pointers, size info, type descriptor pointers) generated once at compile time per concrete instantiation and passed to the shared function body as a hidden argument.
- **Stack** — type parameters that resolve to value types stay on the stack like any other value. `Min[int](3, 7)` runs entirely on the stack; no allocation, no indirection.
- **Heap** — a `Stack[T]`'s internal slice, or any type parameter that resolves to a pointer type, follows normal escape analysis. Generics do not change the escape rules; the shared body is compiled once with escape analysis based on how the parameter is used, not on the concrete type.

The interview-relevant fact: **there is no per-call heap allocation caused by generics** the way there is for `interface{}` boxing. That's the whole point of the design.

## Common Pitfalls

### Instantiating An Overly Narrow Constraint

Writing `[T int | float64]` when what you meant was "any ordered type" makes your function useless the moment someone has an `int32`. Prefer stdlib constraints (`cmp.Ordered`, `comparable`) or well-named unions with `~`.

### Forgetting `~` On Numeric Constraints

```go
type MyInt int
func F[T int](x T) {}   // MyInt does not satisfy T
func G[T ~int](x T) {}  // MyInt does
```

Almost every constraint of the form "a numeric type" wants `~`. Forgetting it turns the constraint into "the exact literal type" and callers with wrapper types (`type UserID int64`) can't use it.

### Using `any` Where `comparable` Would Do

`func Contains[T any](xs []T, target T) bool` — how do you compare `target` to `xs[i]`? You can't with `any`. You need `comparable`. Reaching for `any` because it "just works" and then getting stuck is a classic novice mistake. Pick the tightest constraint that lets you write the body.

### Assuming `comparable` Never Panics

A `comparable` type can still panic at runtime if it's an interface whose dynamic type isn't comparable. `map[any]int` with a slice key at runtime → panic. Constraint check at build time, dynamic check at insertion time.

### Trying To Write Generic Methods

Covered above — you can't. Move it to a top-level function.

### Type Assertions On A Generic `T`

You cannot type-assert a value of a type parameter:

```go
func Show[T any](v T) {
    if s, ok := v.(string); ok { /* COMPILE ERROR */ }
}
```

The workaround if you really need it: assert against `any(v)`:

```go
if s, ok := any(v).(string); ok { /* works */ }
```

But this is a smell. If you're pattern-matching on the type, you probably wanted an interface or a type switch on `any`, not generics.

## Production Practice

- **Use `slices` and `maps` from the stdlib** (both 1.21+) rather than writing your own generic helpers. `slices.Contains`, `slices.SortFunc`, `slices.Clone`, `maps.Keys`, `maps.Values`. Rolling your own is a reviewer flag now.
- **Prefer `cmp.Ordered` over `constraints.Ordered`.** `golang.org/x/exp/constraints` still works but the stdlib moved `Ordered` to `cmp` in 1.21, and the `x/exp` version is now considered "old world." New code uses `cmp.Ordered`.
- **Don't reach for generics first.** A code review culture of "add a type parameter because we can" produces `Stack[T]` for a use case that only ever stores `*User`. If there is exactly one caller with one type, write the non-generic version.
- **Small, well-named constraint interfaces.** A constraint is documentation. `type Number interface { ~int | ~int64 | ~float64 }` at the top of a package tells a reader what's expected. A five-line inline union at each function signature does not.
- **Never generalise a receiver you can't inline.** `Stack[T]` where the hot path is `Push` and `Pop` on `int` is fine (shared body works well). Generic wrappers around a per-element method call (e.g., `Map[T, U]` that calls `f(x)` millions of times) are where dictionary indirection shows up in pprof. Benchmark first if you're on a hot path.
- **Constraints go in a `constraints.go` file at the package root** if they're used across the package. Scattering `type Number interface { ... }` at the top of each file that uses it fragments them and invites drift.
- **`any` is fine as a constraint, but not as a return type of a generic function.** If you write `func F[T any](t T) any`, you've thrown the parameterisation away — the caller has to cast the result. Return `T`.

## How To Remember

**"One shape, one body — one type, one dictionary."**

That single sentence covers Go's implementation strategy: bodies are shared per GC shape, dictionaries specialise per concrete type.

For syntax:

- **`[T C]`** — square brackets carry the *type* parameters (parens carry the *value* parameters). Think: `[]` = types, `()` = values.
- **`~T`** — "T-ish" — T, or anything with T underneath.
- **`any`** — no requirements. **`comparable`** — must support `==`.

For the design choice: **"Generics generalise data. Interfaces generalise behaviour."**

## Problems The Feature Brings

- **Slower compile times.** Every generic call site widens the compiler's work. Not catastrophic, but measurable on large codebases.
- **Dictionary indirection.** On very hot paths the generic version is slower than a hand-written specialisation. Benchmark before committing.
- **Debuggability.** A stack frame in a shared body doesn't tell you which instantiation you're in. Delve and pprof handle it, but symbol names get noisier (`main.Stack[int].Push` etc.).
- **Read-time cost.** A signature with three type parameters and three constraints is genuinely harder to read than a plain function. Adds cognitive load for maintainers who don't use generics daily.
- **API design pressure.** Once you export `func F[T comparable](...)`, the constraint is a public API commitment. Widening it (`comparable` → `any`) is a breaking change if the body relies on `==`. Tightening it always breaks callers.
- **No generic methods.** Forces some designs (fluent builders, monadic chains) to become top-level function calls, which reads awkwardly to people used to method-chaining languages.
- **No sum types.** Union constraints look like sum types but aren't — you can constrain to `int | string` but you cannot pattern-match on which branch a value is inside a generic function. Genuine sum types are a separate, still-open Go proposal.

## Self Test

<details>
<summary>What is a type parameter, and what is a type constraint?</summary>

A type parameter is a placeholder for a type, declared in square brackets on a function or type (`func F[T C](...)`). A constraint is an interface stating what operations `T` must support — either a method set, or a union of types (possibly with `~` for underlying-type matches). Constraints are checked at compile time. See [Type Parameters And Constraints](#type-parameters-and-constraints).
</details>

<details>
<summary>What's the difference between <code>any</code> and <code>comparable</code>?</summary>

`any` (alias for `interface{}`) imposes no requirement — works for slices, maps, functions, anything. `comparable` requires the type to support `==` and `!=` — necessary for map keys, sets, and `Contains`-style lookups. Since 1.20, interface types satisfy `comparable` even though comparing them can panic at runtime if the dynamic type isn't comparable. See [`any` Vs `comparable`](#any-vs-comparable).
</details>

<details>
<summary>What does <code>~int</code> mean, and why does it matter?</summary>

"Any type whose underlying type is `int`" — so both `int` itself and named types like `type Age int` satisfy it. Without `~`, only `int` exactly satisfies. Almost every numeric constraint you write should use `~`; forgetting it means callers with wrapper types (`type UserID int64`) cannot use your function. `cmp.Ordered` uses `~` internally. See [The `~` Approximation Element](#the--approximation-element).
</details>

<details>
<summary>How do you constrain to numeric or ordered types?</summary>

Use `cmp.Ordered` from the stdlib (since 1.21). Pre-1.21, `golang.org/x/exp/constraints.Ordered` was the answer — it still works but new code uses `cmp.Ordered`. For custom constraints, write a union with `~`: `type Number interface { ~int | ~int64 | ~float64 }`. See [Production Practice](#production-practice).
</details>

<details>
<summary>Why can't you write generic methods (methods with their own type parameters)?</summary>

Because it would make interface satisfaction undecidable — an interface referencing a generic method would need infinitely many instantiations (one per possible type argument) to check whether a type satisfies it. The workaround is a top-level generic function that takes the receiver as its first argument, which is the pattern used throughout `slices` and `maps`. See [Why No Generic Methods](#why-no-generic-methods).
</details>

<details>
<summary>When does a generic function get instantiated? What's the runtime cost?</summary>

At compile time, once per distinct **GC shape** used at any call site — not per concrete type. `Sum[int32]`, `Sum[uint32]`, and `Sum[float32]` typically share one body; `Sum[int]` and `Sum[*User]` get separate bodies. Runtime cost is the dictionary indirection for type-specific operations: negligible for simple bodies, 10-30% overhead in hot inner loops versus hand-written specialisations. Nearly always faster than `interface{}` boxing. See [How The Compiler Implements Generics](#how-the-compiler-implements-generics).
</details>

<details>
<summary>Monomorphization vs. GC shape stenciling — how does Go implement generics?</summary>

Neither extreme. C++/Rust do full monomorphization (one body per concrete type, zero runtime cost, code bloat). Java does boxing/dictionary passing (one body, boxed values, indirect calls everywhere). Go does **GC shape stenciling**: one body per GC shape plus a per-instantiation dictionary of type-specific operations passed as a hidden argument. Small binaries, no boxing for value types, but a small indirection cost on hot paths. See [How The Compiler Implements Generics](#how-the-compiler-implements-generics).
</details>

<details>
<summary>When would you use generics vs. an interface?</summary>

Interfaces for **polymorphism of behaviour** — different implementations of the same operation (`io.Reader`, `Stringer`). Generics for **polymorphism of data** — the same algorithm across many types (`Min`, `Map`, `Contains`). Sometimes both together: `slices.SortFunc[S ~[]E, E any](s S, cmp func(a, b E) int)` is generic over element type and takes a function for the behaviour. See [Generics Vs Interfaces](#generics-vs-interfaces).
</details>

<details>
<summary>What are the limits of type inference in generic function calls?</summary>

Two you'll hit in practice: (1) no arguments means no inference — `func Zero[T any]() T` must be called as `Zero[int]()`; (2) inference works from argument types, not from the assignment target — `var x int = MakeSomething()` will not infer `T = int` from `x`. Go 1.21 improved partial inference significantly, but these two rules still bite. See [Type Inference And Its Limits](#type-inference-and-its-limits).
</details>

<details>
<summary>Why did Go add generics? What did they *not* solve?</summary>

Solved: hand-copied per-type versions of algorithms (`IntMin`, `StringMin`), and the `interface{}`-with-type-assertions tax that lost type safety and forced boxing of value types. The payoff is visible in the stdlib's `slices` and `maps` packages. Did not solve: sum types, generic methods, covariance, algebraic data types, higher-kinded types. See [Why Generics Exist](#why-generics-exist) and [Problems The Feature Brings](#problems-the-feature-brings).
</details>

## References

- [Go 1.18 Release Notes — Generics](https://go.dev/doc/go1.18#generics) — the shipping-day summary. Terse, authoritative.
- [Go Blog — An Introduction To Generics](https://go.dev/blog/intro-generics) — the official primer. Read this first if you've never touched them.
- [Go Blog — When To Use Generics](https://go.dev/blog/when-generics) — Ian Lance Taylor's guidance on when to reach for them and when to stick with interfaces. Directly answers Q28.
- [Type Parameters Proposal (design doc)](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md) — the canonical design document. Long, but the "Why not X?" sections are gold for interviews.
- [Go Language Specification — Type Parameters](https://go.dev/ref/spec#Type_parameters) — precise grammar and rules. Skim once to know it exists.
- [`cmp` package — pkg.go.dev](https://pkg.go.dev/cmp) — `cmp.Ordered`, `cmp.Compare`, `cmp.Less`. New home for the numeric/ordered constraints since 1.21.
- [`slices` package — pkg.go.dev](https://pkg.go.dev/slices) and [`maps` package](https://pkg.go.dev/maps) — the payoff of generics. Read the signatures; they teach idiomatic constraint design.
- [Go Blog — Generics Facilitators In The Standard Library](https://go.dev/blog/generic-slice-functions) — walks through the `slices` API and the reasoning.
- [PlanetScale Blog — Generics Can Make Your Go Code Slower](https://planetscale.com/blog/generics-can-make-your-go-code-slower) — benchmarks showing dictionary-indirection overhead. Careful reading — the takeaway is "measure on your hot paths," not "avoid generics."
- [GopherCon 2021 — Generics! By Robert Griesemer & Ian Lance Taylor](https://www.youtube.com/watch?v=Pa_e9EeCdy8) — the language designers walking through the proposal. Watch once.
- [go/src/cmd/compile — GC shape stenciling design doc](https://github.com/golang/proposal/blob/master/design/generics-implementation-gcshape.md) — the implementation strategy in the compiler's own words. Interview-decisive if you can paraphrase it.

## Related Topics

- [`./errors.md`](./errors.md) — generics are often used to wrap "result" types or Option-like helpers, but Go's error model doesn't need them (errors are already values). Worth knowing why the two don't overlap the way `Result<T, E>` and generics do in Rust.
- `./type-system.md` (planned) — interface representation (itab + data pointer), value vs pointer receivers, and the typed-nil trap all sit next to generics; deciding between an interface and a type parameter presupposes understanding both.
- `../concurrency/channels.md` (planned) — generic channel helpers (`Fan-in[T]`, `Merge[T]`) are a natural fit; the dictionary cost is negligible next to channel synchronisation.
- `../runtime/memory-model.md` (planned) — GC shape stenciling shares a body across types with the same GC layout; understanding pointer vs non-pointer classification explains why `[]*User` and `[]*Order` share a body but `[]int` doesn't.