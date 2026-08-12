---
Status: 🌳 Evergreen
Created: 2026-08-13
Last Updated: 2026-08-13
---

# Errors

## Table of Contents

- [What Errors Are In Go](#what-errors-are-in-go)
- [Why Go Chose Values Over Exceptions](#why-go-chose-values-over-exceptions)
- [Why It Is Called "error"](#why-it-is-called-error)
- [The Three Error Design Styles](#the-three-error-design-styles)
- [Wrapping With `%w`, `Unwrap`, `errors.Is`, `errors.As`](#wrapping-with-w-unwrap-errorsis-errorsas)
- [`errors.Join` And Multi-Errors](#errorsjoin-and-multi-errors)
- [Panic And Recover](#panic-and-recover)
- [Where Errors Live In Memory](#where-errors-live-in-memory)
- [Common Pitfalls](#common-pitfalls)
- [Error Design For gRPC Services](#error-design-for-grpc-services)
- [Production Practice](#production-practice)
- [How To Remember](#how-to-remember)
- [Problems The Model Brings](#problems-the-model-brings)
- [Self Test](#self-test)
- [References](#references)
- [Related Topics](#related-topics)

## What Errors Are In Go

In Go, an error is just a **value that satisfies the `error` interface**:

```go
type error interface {
    Error() string
}
```

There is no `try`/`catch`, no exception object, no unwinding stack magic. A function that can fail returns an extra return value of type `error`, and the caller decides — right there, in normal control flow — what to do with it. `nil` means success; anything else means failure.

That is the whole model. Every construct discussed below (sentinels, error types, wrapping, `errors.Is`, `errors.Join`, even `panic`/`recover`) is a convention or a helper layered on top of "an error is a value implementing one method".

## Why Go Chose Values Over Exceptions

Exceptions are invisible in the type signature. `int parseAge(String s)` in Java can throw — you don't know from the signature which failures, or whether they'll be caught two frames up or twenty. In production Go code you can grep for `if err != nil` and see every branch a failure can take. That verbosity is the point: **error handling gets the same attention as the happy path**, because it takes the same number of lines.

Three practical consequences:

1. **No hidden control flow.** A `return` is a `return`. The scheduler, the stack layout, and the CPU pipeline all know that.
2. **Errors are ordinary data.** You can compare them, wrap them, log them, marshal them, put them in a channel, or attach them to a span. An exception object can do most of that, but only after you've caught it — in Go the value never leaves the calling code's hands.
3. **The compiler can help.** An unused error return is a common lint (`errcheck`), and staticcheck can flag ignored wraps. Exception-based languages have no equivalent because the exception isn't in the signature.

## Why It Is Called "error"

Chosen for boringness. Rob Pike and Robert Griesemer explicitly wanted a name that carried no baggage from other languages. Not `Exception` (implies unwinding), not `Failure` (implies terminality), not `Result` (implies a sum type like Rust's). Just **`error`** — lowercase, one word, describing "something went wrong" with no commitment to a mechanism.

The lowercase matters too: it makes `error` a *builtin* identifier (like `int`, `string`, `bool`), signalling that it is part of the base vocabulary, not a special construct.

## The Three Error Design Styles

Every Go error you'll design fits into one of three categories. Knowing which style you're using — and why — is the single most useful thing to internalise.

### 1. Sentinel Errors

A package-level `var` of type `error`, exported so callers can compare against it.

```go
var ErrNotFound = errors.New("user: not found")

func findUser(id int) (*User, error) {
    // ...
    return nil, ErrNotFound
}

// caller
u, err := findUser(id)
if errors.Is(err, ErrNotFound) {
    // handle
}
```

**When to use:** the caller needs to branch on the *specific* failure and there's no extra data to carry. Canonical examples: `io.EOF`, `sql.ErrNoRows`, `context.Canceled`.

**Cost:** sentinels form an API contract. Once exported, changing them is a breaking change. Callers can (and will) compare against them, so you can never remove them without a major version bump.

### 2. Error Types

A struct that implements `error`, carrying structured fields.

```go
type ValidationError struct {
    Field string
    Msg   string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation: field %q: %s", e.Field, e.Msg)
}
```

**When to use:** the caller needs *data* from the failure — which field, which HTTP status, which retry-after value. Canonical examples: `*os.PathError`, `*net.OpError`, `*json.SyntaxError`.

**Cost:** larger API surface than a sentinel. Every exported field becomes part of the contract. Prefer pointer receivers for these so that `errors.As` extraction returns a pointer callers can mutate or inspect.

### 3. Opaque Errors

Just return `error`. Don't expose a type, don't expose a sentinel. The caller can only log, propagate, or return.

```go
func loadConfig(path string) error {
    if _, err := os.Open(path); err != nil {
        return fmt.Errorf("loadConfig(%s): %w", path, err)
    }
    return nil
}
```

**When to use:** the default. Most errors are diagnostics — the caller just needs to know it failed, log something useful, and return. Dave Cheney's rule of thumb: "assert errors for behaviour, not type" — meaning most calling code should not care what the error is, only that it happened.

**Cost:** callers can't programmatically distinguish. If a caller later *does* need to distinguish, you have to upgrade the error into a sentinel or a type — a real breaking change if the caller had been matching on string.

**Rule of thumb ordering:** start opaque. Upgrade to a sentinel when a caller needs to branch. Upgrade to a type when a caller needs data.

## Wrapping With `%w`, `Unwrap`, `errors.Is`, `errors.As`

Introduced in Go 1.13. Before this, everyone used `github.com/pkg/errors`. The stdlib version is deliberately simpler.

### Wrapping With `%w`

```go
if err := doThing(); err != nil {
    return fmt.Errorf("outer context: %w", err)
}
```

`fmt.Errorf` with `%w` produces a new error that **wraps** the original. Two things happen:

1. The formatted string includes the original's `Error()` output.
2. The returned error's `Unwrap() error` method returns the original.

That second point is the whole trick. It builds a linked list of errors.

```
outer error  --Unwrap()-->  middle error  --Unwrap()-->  original error
```

### `errors.Is` — Sentinel Match

```go
if errors.Is(err, sql.ErrNoRows) { ... }
```

Walks the `Unwrap` chain and returns `true` if any link equals the target (by `==`, or by a custom `Is(target error) bool` method on the error).

### `errors.As` — Type Match

```go
var pe *fs.PathError
if errors.As(err, &pe) {
    log.Printf("path %s failed: %v", pe.Path, pe.Err)
}
```

Walks the chain looking for an error whose dynamic type matches the target pointer's element type. On success, it assigns and returns `true`.

**Mnemonic:** *`Is` compares, `As` extracts.* `Is` gives a `bool`, `As` gives you the typed value.

### A Custom `Is` Method

You can override equivalence for value error types (useful when you want "any HTTP 404" to match regardless of message):

```go
type HTTPError struct {
    Code int
    Msg  string
}

func (e HTTPError) Error() string { return e.Msg }
func (e HTTPError) Is(target error) bool {
    t, ok := target.(HTTPError)
    if !ok {
        return false
    }
    return e.Code == t.Code
}
```

Now `errors.Is(err, HTTPError{Code: 404})` matches any 404.

### Working Example

```go
package main

import (
    "errors"
    "fmt"
    "io/fs"
    "os"
)

func loadConfig(path string) error {
    if _, err := os.Open(path); err != nil {
        return fmt.Errorf("loadConfig(%s): %w", path, err)
    }
    return nil
}

func main() {
    err := loadConfig("/no/such/file")
    fmt.Println(errors.Is(err, fs.ErrNotExist)) // true

    var pe *fs.PathError
    fmt.Println(errors.As(err, &pe), pe.Path)   // true /no/such/file
}
```

The wrap chain here is `loadConfig error → *fs.PathError → syscall.Errno`. `errors.Is` walks it, `errors.As` extracts the `*fs.PathError` in the middle.

## `errors.Join` And Multi-Errors

Added in Go 1.20. Sometimes an operation has *several* independent failures — parallel goroutines, batch validation, cleanup after an error where the cleanup itself fails.

```go
err := errors.Join(err1, err2, err3)
```

Produces a single `error` whose `Error()` prints each on its own line. The magic is that `errors.Is` and `errors.As` **descend into all branches**:

```go
var ErrAuth = errors.New("auth failed")

func login() error {
    return errors.Join(ErrAuth, errors.New("token expired"))
}

// caller
if errors.Is(login(), ErrAuth) { /* still matches */ }
```

Before 1.20 this required `hashicorp/go-multierror` or `uber-go/multierr`. The stdlib version is minimal — no formatting hooks, no per-error prefixes. If you need those, use one of the libraries.

## Panic And Recover

`panic` is not an error. It is a runtime alarm that unwinds the stack, running deferred functions along the way. `recover` — only meaningful inside a `defer` — stops the unwind and hands you the panic value.

```go
func safeCall(fn func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered panic: %v", r)
        }
    }()
    fn()
    return nil
}
```

### When To `panic`

Not for expected failures. `panic` is for **programmer errors** and **truly unrecoverable states**:

- Index out of range, nil map write, division by zero — the runtime panics you.
- `must*` helpers at init time: `regexp.MustCompile`, `template.Must` — you know the input is a constant, and if it doesn't parse the program is broken.
- Broken invariants: an "impossible" default case in a `switch` over an exhaustive enum.

### When To `recover`

Only at process/goroutine boundaries. Typical places:

- **HTTP middleware** — one bad handler shouldn't crash the whole server. Recover, log, return 500.
- **Worker goroutines** in a pool — one bad job shouldn't take out the pool. Recover, mark the job failed, keep the goroutine alive.
- **gRPC interceptors** — same reasoning as HTTP middleware.

**A `recover` in one goroutine cannot catch a panic in another goroutine.** Each goroutine has its own stack; each unwind is local. If you spawn a goroutine, you must set up its own `defer recover()` or the whole process dies when it panics.

```
goroutine A                    goroutine B
   |                              |
   defer recover()                panic("boom")
   go B()                            |
   ...                            [unwinds B's stack;
                                    A's defer never runs;
                                    process crashes]
```

## Where Errors Live In Memory

Standard four-segment framing:

- **Stack** — the `error` interface value (two words: type pointer + data pointer) lives on the caller's stack when returned from a function, unless it escapes.
- **Heap** — the concrete error struct behind the interface typically escapes to the heap. `errors.New("foo")` heap-allocates a `*errorString`. `fmt.Errorf` allocates a new struct wrapping the format arguments. This is why hot paths that allocate errors on every call show up in pprof heap profiles.
- **Data/BSS Segment** — package-level sentinel `var`s like `ErrNotFound = errors.New(...)` — the `*errorString` is heap-allocated at package init, and the `error` interface value that holds the pointer lives in the data segment for the lifetime of the program.
- **Text/Code Segment** — the method table for `*errorString.Error() string` and every other error type's methods.

**Practical consequence:** using a sentinel is cheap (one allocation at init, then just an interface copy). Using `fmt.Errorf` on a hot path is not — it allocates. If a hot loop needs to return an error, pre-allocate a sentinel or use `errors.New` once at package level.

## Common Pitfalls

### The Typed-Nil Trap

An `error` interface holding a *nil pointer of a concrete type* is **not** `nil`.

```go
type MyError struct{ Msg string }
func (e *MyError) Error() string { return e.Msg }

func doWork() error {
    var e *MyError  // nil pointer
    return e        // but the interface has (type=*MyError, data=nil)
}

func main() {
    if err := doWork(); err != nil {
        fmt.Println("caught:", err) // prints — the interface is NON-nil
    }
}
```

Fix: return literal `nil`, not a typed nil variable, when there's no error.

```go
if e != nil {
    return e
}
return nil
```

### Wrapping A `nil`

`fmt.Errorf("...: %w", nil)` produces a non-nil error whose `Unwrap()` returns nil. Almost always a bug — the caller wraps blindly instead of checking `if err != nil` first. Always guard the wrap.

### Shadowing `err` In A Nested Scope

```go
if val, err := doA(); err == nil {
    val, err := doB()  // <-- new err inside the if
    _ = val
    _ = err
}
// the outer err is unchanged
```

Rare but nasty. The lint `govet -shadow` catches it.

### Comparing Wrapped Errors With `==`

```go
if err == io.EOF { ... } // fragile
```

Fine only if you know `err` is not wrapped. In library code, always use `errors.Is`.

### `panic` Across Goroutines

Covered above. Every spawned goroutine needs its own defer/recover if it can panic.

## Error Design For gRPC Services

gRPC has a first-class error model: every RPC returns a `status.Status` with a `Code` (an enum from the `codes` package) and a message, plus optional structured details.

```go
import (
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

// server side
return nil, status.Error(codes.NotFound, "user not found")

// client side
if s, ok := status.FromError(err); ok {
    switch s.Code() {
    case codes.NotFound:
        // ...
    case codes.DeadlineExceeded:
        // retry?
    }
}
```

### The Bridge Pattern

Inside your service you use idiomatic Go errors — sentinels, types, wrapped. At the gRPC boundary, you translate:

```
domain layer:  ErrUserNotFound (sentinel)
     |
     |  service layer: fmt.Errorf("GetUser: %w", err)
     v
gRPC handler: errors.Is(err, ErrUserNotFound) --> status.Error(codes.NotFound, ...)
```

Don't leak internal wrap context to the client — it's noise and a potential info-disclosure risk. Translate to a `codes.*` value + a stable message. Log the full wrapped error server-side.

### Code Selection Cheat Sheet

| Domain condition | gRPC code |
|---|---|
| Not found | `NotFound` |
| Bad input | `InvalidArgument` |
| Authn missing | `Unauthenticated` |
| Authz denied | `PermissionDenied` |
| Deadline hit | `DeadlineExceeded` |
| Client cancelled | `Canceled` |
| Duplicate / conflict | `AlreadyExists` or `FailedPrecondition` |
| Rate limited / overloaded | `ResourceExhausted` |
| Unexpected bug | `Internal` |

**`codes.Unknown` is a smell.** It usually means someone returned a plain Go error from a gRPC handler; the framework wrapped it as `Unknown`. Fix the handler to return `status.Error(...)` explicitly.

## Production Practice

What "shipped Go code at senior-review quality" actually looks like — the conventions that separate "compiles" from "would pass code review at a place like Google, Uber, or GitLab." Anything a reviewer would flag as "not how we do it here" lives here.

### Wrap Once Per Layer, With Caller-Useful Context

The single most common mistake in Go error handling is either wrapping at every function (producing `outer: middle: inner: dial: connect: no such host` chains ten deep) or wrapping at none (losing all context). The convention:

> Wrap when you cross a **layer boundary** — repository → service, service → handler, handler → transport. Not on every function call inside a layer.

```go
var ErrUserNotFound = errors.New("user not found")

func repoGet(id int) error {
    if id == 0 { return ErrUserNotFound }
    return nil
}

func serviceGet(id int) error {
    if err := repoGet(id); err != nil {
        return fmt.Errorf("serviceGet: %w", err)   // wrap once at layer boundary
    }
    return nil
}

func handlerGet(id int) error {
    if err := serviceGet(id); err != nil {
        return fmt.Errorf("handlerGet: %w", err)   // wrap once at layer boundary
    }
    return nil
}

// Result: "handlerGet: serviceGet: user not found"
// errors.Is(err, ErrUserNotFound) still returns true.
```

Each wrap adds *what this layer was doing*, not *what the caller passed in*. `"handlerGet"` tells you the boundary; the wrapped chain tells you the path. `errors.Is` still reaches the sentinel.

### Guard `%w` Against `nil`

`fmt.Errorf("%w", nil)` produces a non-nil error whose `Unwrap()` returns nil. A caller writing `if err != nil { ... }` will enter the branch when there was no failure. Always guard:

```go
if err := doThing(); err != nil {
    return fmt.Errorf("layer: %w", err)
}
return nil
```

Never `return fmt.Errorf("layer: %w", doThing())` — the wrap runs even on success.

### Sentinels Only For Cross-Package Contracts

A sentinel is an API commitment. Once `var ErrX = errors.New(...)` is exported, callers write `errors.Is(err, pkg.ErrX)` and you cannot remove it. In production code:

- **Do** export sentinels for real branching contracts: `io.EOF`, `sql.ErrNoRows`, `context.Canceled`, your own `ErrUserNotFound` / `ErrDuplicateKey` at the repository layer.
- **Do not** export a sentinel just because "the caller might want to check it someday." Keep it unexported (`errNotReady`) until a real caller needs to branch.
- **Do not** create sentinels for one-off conditions inside a single package. Just return an opaque `fmt.Errorf("...")`.

### `errgroup` For Parallel Failure, Not Hand-Rolled `errors.Join`

`errors.Join` composes errors *you already have*. It does not orchestrate goroutines. For "run N things in parallel, return the first error and cancel the rest," the canonical answer is `golang.org/x/sync/errgroup`:

```go
import "golang.org/x/sync/errgroup"

func fanOut(ctx context.Context, ids []int) error {
    g, ctx := errgroup.WithContext(ctx)
    for _, id := range ids {
        id := id
        g.Go(func() error {
            return process(ctx, id)
        })
    }
    return g.Wait()
}
```

`errgroup.WithContext` gives every worker a context that gets cancelled the instant any worker returns an error — no leaked goroutines, no need to hand-roll a done channel. Hand-rolling this with `sync.WaitGroup` + a shared `error` + a mutex is a classic reviewer flag. Use `errgroup`.

Use `errors.Join` for the different case: "I ran N cleanup steps sequentially, collected all their failures, and want to report them together."

### Log With Structured Attributes, Not Formatted Strings

The idiomatic way to log an error in modern Go (`log/slog`, stdlib since 1.21) is as an **attribute**, not folded into the message:

```go
// Wrong — the error is buried inside the message string.
logger.Error(fmt.Sprintf("get user %d failed: %v", id, err))

// Right — the error is its own structured field.
logger.LogAttrs(ctx, slog.LevelError, "get user failed",
    slog.Int("user_id", id),
    slog.Any("err", err),
)
```

Downstream log processors (Datadog, Loki, ELK) can then index on `err`, alert on specific error patterns, and correlate with traces without regex-scraping the message field. The same discipline applies to `zap` and `zerolog` — pass the error as a field, never `fmt.Sprintf` it into the message.

### Never Log-And-Return

If you log an error at every layer *and* return it, the same failure appears three or four times in the logs by the time it hits the top. Discipline:

> **Log at exactly one place: the boundary that decides to stop propagating.** Everywhere below that, return the wrapped error and stay silent.

For an HTTP or gRPC service, that's the handler (or the interceptor). Inside services and repositories, just wrap and return.

### `codes.Unknown` At A gRPC Boundary Is A Bug

A gRPC handler that lets a plain Go `error` escape produces a client-visible `Unknown` code, because `grpc-go` doesn't know how to translate it. Any handler you write should end with an explicit translation:

```go
func (s *Server) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    u, err := s.svc.GetUser(ctx, req.Id)
    if err != nil {
        switch {
        case errors.Is(err, ErrUserNotFound):
            return nil, status.Error(codes.NotFound, "user not found")
        case errors.Is(err, context.DeadlineExceeded):
            return nil, status.Error(codes.DeadlineExceeded, "timeout")
        default:
            s.log.LogAttrs(ctx, slog.LevelError, "GetUser internal", slog.Any("err", err))
            return nil, status.Error(codes.Internal, "internal error")
        }
    }
    return toProto(u), nil
}
```

Note: the `default` branch logs the wrapped chain server-side (all the context you need to debug) and returns a bland `Internal` message to the client (no information disclosure, no wrap chain leaked). This is the standard shape.

### Recover In A Middleware, Not Everywhere

Sprinkling `defer func() { recover() }()` inside business code is a smell — it hides bugs. The production pattern is **one recover, at the request/goroutine boundary**:

- HTTP: a recovery middleware (`chi/middleware.Recoverer`, gin's default, or hand-rolled) wrapped around all handlers.
- gRPC: a `grpc_recovery` unary and stream interceptor from `go-grpc-middleware`.
- Worker pools: a wrapper around the task function that recovers and marks the task failed.

Everywhere else: let panics propagate. They almost always indicate a bug that should be caught in test, not smothered in prod.

### Don't Compare Error Strings

Ever:

```go
if err.Error() == "connection refused" { ... } // NO
```

Error messages are diagnostic, not API. They change between Go versions, between OSes, between library releases. Use `errors.Is` against a sentinel or `errors.As` against a type. If neither works because the library doesn't expose them, file an issue upstream — string-comparing is a last-resort hack that will break at the worst possible time.

### Errors Are Cheap, But Not Free

Every `errors.New` and `fmt.Errorf` allocates. On a hot path (millions of calls per second — a serialization inner loop, a per-packet check in a proxy) this shows up in pprof. Two mitigations:

- **Pre-allocate sentinels** at package init — they cost one allocation, forever.
- **Return a pre-built sentinel** from hot paths instead of `fmt.Errorf(...)` with dynamic context. Add context at the outer layer, not the inner one.

For 99% of code, don't premature-optimize. But when you see `runtime.newobject` from `errors.New` in a hot profile, you know why.

## How To Remember

**"Errors are values. Values are cheap. Cheap means you handle them."**

For the wrap trio:

- **`%w`** — *wrap* (adds a link).
- **`Is`** — *is it this sentinel?* (bool).
- **`As`** — *as this type, give me the data* (extract).

For panic/recover: **"defer catches the fall, only on the same stack."** One goroutine, one recover; if you spawn, you catch.

## Problems The Model Brings

- **Verbosity.** `if err != nil { return err }` everywhere. Real cost, no idiomatic fix — a Go 2 draft (`try` / `check`) was proposed and rejected.
- **Wrap fatigue.** Deep call stacks produce error chains like `handler: service: repo: db: dial tcp: ...`. Readable but noisy. Discipline: wrap once per layer, with a caller-useful context.
- **Sentinel API lock-in.** Every exported sentinel is a permanent contract. Callers write `errors.Is(err, ErrX)` and you can't remove `ErrX` without breaking them.
- **Lossy `==`.** Old code that predates 1.13 compares errors directly. `errors.Is` is the fix but requires a rewrite.
- **Stack traces are not built in.** The stdlib error does not capture a stack. If you need one, use `pkg/errors` or attach it yourself in your logger. This is a genuine ergonomics gap versus most exception languages.

## Self Test

<details>
<summary>What is an error in Go, and how does it differ from exceptions?</summary>

An `error` is any value implementing `Error() string`. It's returned as an ordinary value alongside successful returns, so failure is visible in the type signature and handled in normal control flow — no stack unwinding, no invisible propagation. See [What Errors Are In Go](#what-errors-are-in-go).
</details>

<details>
<summary>Sentinel error, error type, opaque error — when do you use each?</summary>

Sentinel for "the caller must branch on this specific failure" (small, permanent API contract). Error type when the caller needs structured data (fields, codes). Opaque for the default case — just return `error` and let the caller propagate. Start opaque, upgrade only when a caller demands it. See [The Three Error Design Styles](#the-three-error-design-styles).
</details>

<details>
<summary>How does <code>%w</code> work, and what's the difference between <code>errors.Is</code> and <code>errors.As</code>?</summary>

`%w` in `fmt.Errorf` produces an error with an `Unwrap()` method, forming a linked list. `errors.Is` walks the chain looking for equality with a sentinel (bool answer). `errors.As` walks the chain looking for a type match, extracting the typed value into the pointer you pass. Mnemonic: `Is` compares, `As` extracts. See [Wrapping With `%w`, `Unwrap`, `errors.Is`, `errors.As`](#wrapping-with-w-unwrap-errorsis-errorsas).
</details>

<details>
<summary>What is the typed-nil pitfall?</summary>

Returning a `nil` pointer of a concrete error type as `error` produces a *non-nil interface value* (type is set, data is nil), so `err != nil` is true even though "there's no error." Always return literal `nil` when you mean no error. See [Common Pitfalls](#common-pitfalls).
</details>

<details>
<summary>When would you <code>panic</code>? When would you <code>recover</code>?</summary>

`panic` for programmer errors and impossible states — nil map write, `MustCompile`, unreachable `default` cases. `recover` only at process or goroutine boundaries (HTTP middleware, worker pools, gRPC interceptors) so one bad request doesn't take down the process. A recover in one goroutine cannot catch a panic in another. See [Panic And Recover](#panic-and-recover).
</details>

<details>
<summary>What does <code>errors.Join</code> do, and when do you use it?</summary>

Combines multiple errors into one whose `Error()` prints each on its own line, and whose `Is`/`As` descend into every branch. Use it for parallel goroutine results, batch validation, or cleanup errors alongside the original failure. Go 1.20+. See [`errors.Join` And Multi-Errors](#errorsjoin-and-multi-errors).
</details>

<details>
<summary>How do you design errors at a gRPC boundary?</summary>

Use idiomatic Go errors internally; translate at the handler using `status.Error(codes.X, msg)`. Match domain sentinels with `errors.Is` and map to gRPC codes (`NotFound`, `InvalidArgument`, `DeadlineExceeded`, etc.). Don't leak wrap context to clients — log it server-side. `codes.Unknown` at the boundary is a smell. See [Error Design For gRPC Services](#error-design-for-grpc-services).
</details>

<details>
<summary>What are the costs of the "errors are values" model?</summary>

Verbosity (`if err != nil` everywhere), wrap fatigue in deep stacks, sentinel API lock-in, no built-in stack traces, and lossy pre-1.13 `==` comparisons in old code. Discipline (wrap once per layer, prefer opaque, guard the wrap) mitigates most of it. See [Problems The Model Brings](#problems-the-model-brings).
</details>

## References

- [Go Blog — Error Handling And Go](https://go.dev/blog/error-handling-and-go) — the original 2011 post by Andrew Gerrand explaining the values model. Foundational.
- [Go Blog — Working With Errors In Go 1.13](https://go.dev/blog/go1.13-errors) — introduces `%w`, `errors.Is`, `errors.As`, and the `Unwrap` interface. The canonical reference for the wrap trio.
- [`errors` package — pkg.go.dev](https://pkg.go.dev/errors) — up-to-date API including `Join`. Read the doc comments for `Is` and `As`; they contain corner cases most blog posts skip.
- [Dave Cheney — Don't Just Check Errors, Handle Them Gracefully](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully) — the "assert errors for behaviour, not type" principle. Explains why opaque should be the default.
- [Dave Cheney — Constant Errors](https://dave.cheney.net/2016/04/07/constant-errors) — clever pattern for building errors that cannot be modified by callers. Worth reading once.
- [Rob Pike — Errors Are Values](https://go.dev/blog/errors-are-values) — short essay showing how "boilerplate" error handling is often refactorable when you treat errors as data.
- [Go FAQ — Why Does Go Not Have Exceptions?](https://go.dev/doc/faq#exceptions) — the language authors' own explanation. One paragraph, high signal.
- [Go FAQ — Why Does My Nil Error Value Not Equal Nil?](https://go.dev/doc/faq#nil_error) — the typed-nil pitfall, straight from the source.
- [Effective Go — Errors, Panic, Recover](https://go.dev/doc/effective_go#errors) — the "when to panic" guidance you should be able to paraphrase in an interview.
- [Go Blog — Defer, Panic, And Recover](https://go.dev/blog/defer-panic-and-recover) — how the unwind mechanism actually works.
- [go/src/errors — source](https://go.googlesource.com/go/+/refs/heads/master/src/errors/) — `errors.go`, `wrap.go`, `join.go`. The whole package is under 300 lines. Reading it once demystifies everything.
- [google.golang.org/grpc/status](https://pkg.go.dev/google.golang.org/grpc/status) and [google.golang.org/grpc/codes](https://pkg.go.dev/google.golang.org/grpc/codes) — the gRPC error model. Skim `codes.Code` — it's short and interviewers ask about it.

## Related Topics

- [`../hexagonal-architecture.md`](../hexagonal-architecture.md) — sentinel errors are the standard way to expose "expected failure" from a core/domain layer without leaking infrastructure detail; the translation to HTTP or gRPC codes happens in the primary adapter.
- `../concurrency/context.md` (planned) — `context.Canceled` and `context.DeadlineExceeded` are the two sentinels every Go service handles; error design and context propagation are joined at the hip.
- `../runtime/goroutines-and-scheduler.md` (planned) — panic/recover boundaries are per-goroutine; understanding scheduling explains why.
- `../stdlib/grpc-and-protobuf.md` (planned) — deeper coverage of the `status`/`codes` bridge and interceptor-based panic recovery.