# Adapter Pattern

## Table of Contents

- [What Is the Adapter Pattern](#what-is-the-adapter-pattern)
- [Why It Exists](#why-it-exists)
- [The Three Roles](#the-three-roles)
- [Object Adapter vs. Class Adapter](#object-adapter-vs-class-adapter)
- [Structure Diagram](#structure-diagram)
- [Go Implementation](#go-implementation)
- [Standard Library Examples](#standard-library-examples)
- [Problems It Introduces](#problems-it-introduces)
- [Adapter vs. Similar Patterns](#adapter-vs-similar-patterns)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)


## What Is the Adapter Pattern

The Adapter is a **structural** design pattern that allows two incompatible interfaces to work together. It does this by wrapping the existing component (which cannot or should not be changed) with a new type that implements the interface the client expects.

The name comes from the physical analogy of a power plug adapter: your laptop (the client) expects a specific socket shape (the target interface). The foreign wall outlet (the adaptee) has a different shape. The adapter sits between them — it does not change what either side does, it only translates the interface.

Category in GoF: **Structural** (alongside Bridge, Composite, Decorator, Facade, Flyweight, Proxy).


## Why It Exists

Software regularly encounters components that cannot be modified:

- Third-party libraries or vendor SDKs
- Legacy internal services
- Standard library types with fixed signatures
- External APIs with their own calling conventions

When such a component does what you need but speaks a different "language" than your system expects, the Adapter pattern lets you integrate it without modifying either the client code or the foreign component.

**Without an adapter:** the client must be rewritten to understand the foreign interface, or the foreign component must be forked and modified — both create tight coupling and maintenance burden.

**With an adapter:** a thin translation layer sits between them; both sides remain unchanged and independently evolvable.


## The Three Roles

- **Target** — the interface your client code depends on and expects. This is the contract the rest of your system is written against.
- **Adaptee** — the existing component with an incompatible interface. You cannot or should not modify this. It may be a third-party library, a legacy type, or a standard library type.
- **Adapter** — implements the Target interface; internally holds a reference to the Adaptee and delegates calls, translating between the two signatures as needed.


## Object Adapter vs. Class Adapter

### Object Adapter (Composition)

The adapter **holds a reference** to the adaptee as a field. This is the only viable form in Go because Go has no class inheritance.

```
Adapter struct {
    adaptee *Adaptee  // composition
}
```

Advantages: the adapter can work with any subtype of the adaptee (in languages that have subtypes); the adaptee does not need to be known at compile time.

### Class Adapter (Inheritance)

The adapter **inherits** from both the target and the adaptee simultaneously (via multiple inheritance). This is a Java/C++ concept. In Java, it is achieved by extending the adaptee and implementing the target interface.

Go does not support this pattern — there is no multiple inheritance or class extension. In Go, always use Object Adapter.


## Structure Diagram

```
  Client
    |
    | depends on
    v
+-------------------+
|   <<interface>>   |
|     Target        |
|  + Method(...)    |
+-------------------+
         ^
         | implements
+-------------------+        holds reference
|     Adapter       |--------------------------> +-------------------+
|  + Method(...)    |                            |     Adaptee       |
|    (translates)   |                            |  + OtherMethod()  |
+-------------------+                            +-------------------+
```

The client only ever sees Target. The Adaptee is invisible to the client — it is an implementation detail of the Adapter.


## Go Implementation

### Scenario

Your application code logs via a `Logger` interface. You want to use a third-party analytics library that has its own logging type with a different method signature. You cannot modify the library.

```go
// target.go
// Logger is the interface your application code depends on.
// All call sites in the application are written against this contract.
package adapter

type Logger interface {
    Log(level, message string)
}
```

```go
// adaptee.go
// ThirdPartyLogger is from an external package you cannot modify.
// It exposes separate methods per log level rather than a unified method.
package adapter

type ThirdPartyLogger struct{}

func (t *ThirdPartyLogger) Info(msg string) {
    println("[VENDOR INFO]", msg)
}

func (t *ThirdPartyLogger) Error(msg string) {
    println("[VENDOR ERROR]", msg)
}
```

```go
// adapter.go
// ThirdPartyLoggerAdapter wraps ThirdPartyLogger and implements Logger.
// The client sees only the Logger interface; the translation is invisible.
package adapter

type ThirdPartyLoggerAdapter struct {
    adaptee *ThirdPartyLogger // object adapter: composition, not inheritance
}

func NewThirdPartyLoggerAdapter(a *ThirdPartyLogger) *ThirdPartyLoggerAdapter {
    return &ThirdPartyLoggerAdapter{adaptee: a}
}

// Log satisfies the Logger interface.
// It translates the unified (level, message) call into the adaptee's
// level-specific methods — this translation is the adapter's only job.
func (a *ThirdPartyLoggerAdapter) Log(level, message string) {
    switch level {
    case "error":
        a.adaptee.Error(message)
    default:
        a.adaptee.Info(message)
    }
}
```

```go
// client.go
// RunService depends only on Logger — it has no knowledge of ThirdPartyLogger
// or the adapter. This is the key benefit: client code is fully decoupled
// from the concrete implementation.
package adapter

func RunService(logger Logger) {
    logger.Log("info", "service started")
    logger.Log("error", "connection refused")
}

func Example() {
    vendor := &ThirdPartyLogger{}
    adapted := NewThirdPartyLoggerAdapter(vendor) // adapted satisfies Logger
    RunService(adapted)
}
```

**Output:**
```
[VENDOR INFO] service started
[VENDOR ERROR] connection refused
```

### What Just Happened

- `RunService` was not changed
- `ThirdPartyLogger` was not changed
- `ThirdPartyLoggerAdapter` is the only new code, and it is a thin translation layer


## Standard Library Examples

Go's `io` package is a practical demonstration of the Adapter pattern in production:

- **`strings.NewReader(s string) *strings.Reader`** — a plain `string` does not implement `io.Reader`. `strings.NewReader` wraps it in a type that does. The string is the adaptee; `io.Reader` is the target.

- **`bytes.NewReader(b []byte) *bytes.Reader`** — same pattern for `[]byte`. A byte slice cannot satisfy `io.Reader` directly; the adapter bridges them.

- **`ioutil.NopCloser(r io.Reader) io.ReadCloser`** — wraps any `io.Reader` with a no-op `Close()` method, making it satisfy `io.ReadCloser`. The reader is the adaptee; `io.ReadCloser` is the target.

Each of these is exactly: "I have X, the function needs Y, here is the adapter that makes X look like Y."


## Problems It Introduces

- **Added indirection** — a call to the target method now traverses the adapter before reaching the adaptee; debugging adds one hop
- **Adapter proliferation** — if many adaptees need adapting, you accumulate many thin wrapper types; this can obscure the real structure of the codebase ("adapter soup")
- **Masks deeper incompatibilities** — if the adaptee's data model is fundamentally different from what the client expects, the adapter can grow into a complex transformation layer that arguably deserves its own abstraction (not just a naming/signature bridge)
- **Does not change behaviour** — the adapter only translates method signatures; if the adaptee is slow, buggy, or has undesirable semantics, the adapter does not fix that


## Adapter vs. Similar Patterns

A frequent interview question is distinguishing Adapter from related structural patterns.

| Pattern | Core Intent | Changes Interface? | Adds Behaviour? | Typical Timing |
|---|---|---|---|---|
| **Adapter** | Make an incompatible interface fit a target | Yes — translates one interface to another | No | Retrofit; post-facto integration |
| **Decorator** | Add behaviour while preserving the interface | No — same interface in and out | Yes | Design time or runtime wrapping |
| **Facade** | Simplify a complex subsystem | Yes — collapses many interfaces into one simpler one | No | Design time |
| **Bridge** | Decouple abstraction from implementation structurally | Structured separation from the start | No | Design time |
| **Proxy** | Control access to an object | No — same interface as the subject | Optionally (caching, auth) | Design time or runtime |

### Key Distinguishers

**Adapter vs. Facade:**
- Facade unifies *many* components into *one simpler interface* (reducing complexity of a subsystem)
- Adapter translates *one* existing interface into *another specific* interface (resolving incompatibility)
- Facade is about simplification; Adapter is about compatibility

**Adapter vs. Decorator:**
- Decorator preserves the interface and adds behaviour
- Adapter changes the interface and preserves behaviour
- `bufio.NewReader` is a borderline case — it wraps `io.Reader` (same interface out), but adds buffering. That makes it a Decorator. `strings.NewReader` changes the type to satisfy `io.Reader` — that is an Adapter.

**Adapter vs. Bridge:**
- Bridge is designed upfront to separate two dimensions of variation (abstraction and implementation)
- Adapter is a retrofit — it resolves an incompatibility that was not anticipated in the original design


## Interview Cheat Sheet

### Likely Questions and What Interviewers Are Probing

**"What is the Adapter pattern and when would you use it?"**
Define the three roles (Target, Adaptee, Adapter) and the core problem it solves (incompatible interfaces). Give a concrete example — third-party library, legacy service, or io.Reader. Show you know when to reach for it: specifically when you cannot modify the adaptee.

**"How would you implement Adapter in Go?"**
Object Adapter via composition. Define the Target interface; the Adapter struct holds the Adaptee; the Adapter implements the Target interface by delegating and translating. Go has no class inheritance so Class Adapter is not applicable.

**"What's the difference between Adapter and Facade?"**
Facade simplifies a complex subsystem (many interfaces → one simpler interface). Adapter resolves incompatibility between two interfaces (one interface → one different interface). Facade is about reducing complexity; Adapter is about bridging a mismatch.

**"What's the difference between Adapter and Decorator?"**
Decorator preserves the interface and adds behaviour. Adapter changes the interface without adding behaviour. Simple litmus: if the input and output interface are the same type, it is a Decorator. If they are different, it is an Adapter.

**"Can you give a real-world example from Go's standard library?"**
`strings.NewReader`, `bytes.NewReader`, `ioutil.NopCloser` — all wrap a type that does not satisfy an interface to produce one that does.

### Signal Phrases That Show Depth

- "I'd use Object Adapter via composition — Go has no inheritance, and composition gives us more flexibility anyway."
- "The adapter's only job is interface translation; if I find myself adding logic in the adapter, I'd reconsider whether it should be a different pattern."
- "This is a retrofit pattern — I'd reach for it when integrating code I don't own, not when designing new components from scratch."
- "Facade reduces complexity; Adapter resolves incompatibility — the distinction matters when an interviewer says 'isn't this just a Facade?'"

### Common Misconceptions

- "Adapter adds new functionality" — it does not; it only translates interface. A component that adds behaviour is a Decorator.
- "Adapter and Facade are the same thing" — they are both wrappers but for different purposes; Facade reduces a subsystem's surface area, Adapter makes one interface fit another
- "You need inheritance to implement Adapter" — in Go, composition is the correct and idiomatic approach; inheritance is the Class Adapter variant and is a Java/C++ concept


## References

- Gamma, Helm, Johnson, Vlissides — *Design Patterns: Elements of Reusable Object-Oriented Software* (GoF), Adapter chapter
- [Refactoring Guru: Adapter](https://refactoring.guru/design-patterns/adapter)
- [Go standard library: io package](https://pkg.go.dev/io) — practical adapter examples
- [strings.NewReader source](https://cs.opensource.google/go/go/+/refs/tags/go1.22.0:src/strings/reader.go) — minimal adapter implementation