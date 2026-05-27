# Proxy Pattern

## Table of Contents

- [What It Is and Why It Exists](#what-it-is-and-why-it-exists)
- [Structure](#structure)
- [The Four Classic Subtypes](#the-four-classic-subtypes)
  - [Virtual Proxy](#virtual-proxy)
  - [Protection Proxy](#protection-proxy)
  - [Caching Proxy](#caching-proxy)
  - [Remote Proxy](#remote-proxy)
- [Proxy vs. Decorator](#proxy-vs-decorator)
- [Trade-offs](#trade-offs)
- [How to Remember It](#how-to-remember-it)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

## What It Is and Why It Exists

The Proxy pattern provides a **surrogate or placeholder for another object to control access to it**. It is a structural design pattern from the GoF catalogue.

The word "proxy" means "authority to act on behalf of another." In the pattern, the proxy object stands in for the real object and intercepts every call to it — doing something before, after, or instead of delegating to the real object.

It exists because you often want to add behaviour *around* accessing an object — lazy initialisation, access control, caching, logging — without modifying the real object itself and without the client knowing any of this is happening. The proxy satisfies the same interface as the real object, so the client is completely unaware of the indirection.

## Structure

Three participants:

- **Subject** — the interface that both the proxy and the real object implement. The client codes to this interface exclusively.
- **RealSubject** — the actual implementation that does the real work.
- **Proxy** — implements Subject, holds a reference to RealSubject, and intercepts calls — applying pre- or post-logic, or short-circuiting the call entirely.

```
        «interface»
          Subject
          +Do()
             |
      +------+-------+
      |               |
 RealSubject        Proxy
 +Do()           -real: Subject
                 +Do()
                    |
               [pre-logic]
               real.Do()
               [post-logic]
```

The client only ever holds a `Subject` reference. It calls `Do()` on whatever it receives — Proxy or RealSubject — identically.

```go
// Subject — the shared interface
type Subject interface {
    Do() string
}

// RealSubject — does the actual work
type RealSubject struct{}

func (r *RealSubject) Do() string {
    return "real work done"
}

// Proxy — intercepts access to RealSubject
type Proxy struct {
    real *RealSubject
}

func (p *Proxy) Do() string {
    // pre-logic: logging, auth, lazy init, etc.
    result := p.real.Do()
    // post-logic: caching, metrics, etc.
    return result
}

// Client codes only to Subject — oblivious to Proxy
func Client(s Subject) {
    s.Do()
}
```

## The Four Classic Subtypes

### Virtual Proxy

**Problem:** The RealSubject is expensive to create — a heavy database connection, a large file load, a complex computation. You don't want to pay that cost until the object is actually used.

**Solution:** The Proxy delays creation of the RealSubject until the first method call (lazy initialisation). Before that first call, the RealSubject does not exist.

```go
type ImageLoader interface {
    Display()
}

// RealSubject — expensive to instantiate
type RealImage struct {
    filename string
}

func NewRealImage(filename string) *RealImage {
    // Simulates expensive load from disk
    fmt.Println("Loading image from disk:", filename)
    return &RealImage{filename: filename}
}

func (r *RealImage) Display() {
    fmt.Println("Rendering:", r.filename)
}

// Virtual Proxy — defers RealImage creation until Display() is called
type ImageProxy struct {
    filename string
    real     *RealImage // nil until first use
}

func NewImageProxy(filename string) *ImageProxy {
    return &ImageProxy{filename: filename} // cheap — no disk I/O yet
}

func (p *ImageProxy) Display() {
    if p.real == nil {
        p.real = NewRealImage(p.filename) // expensive, happens once
    }
    p.real.Display()
}
```

Usage:

```go
var img ImageLoader = NewImageProxy("hero.png")
// At this point, the image has NOT been loaded from disk

img.Display() // "Loading image from disk: hero.png" + "Rendering: hero.png"
img.Display() // "Rendering: hero.png" — no reload
```

The client calls `Display()` the same way regardless of whether it's a proxy or a real image.

### Protection Proxy

**Problem:** The RealSubject should only be accessible under certain conditions — role checks, permission flags, rate limits. You do not want auth logic spread across every caller, and you do not want it inside the RealSubject (which should only care about its core job).

**Solution:** The Proxy checks conditions before forwarding. If conditions are not met, it rejects the call without ever touching the RealSubject.

```go
type UserService interface {
    DeleteUser(userID string)
}

// RealSubject — pure business logic, no auth
type UserServiceImpl struct{}

func (u *UserServiceImpl) DeleteUser(userID string) {
    fmt.Println("User deleted:", userID)
}

// Protection Proxy — enforces access control
type ProtectedUserService struct {
    real       *UserServiceImpl
    callerRole string
}

func NewProtectedUserService(role string) *ProtectedUserService {
    return &ProtectedUserService{
        real:       &UserServiceImpl{},
        callerRole: role,
    }
}

func (p *ProtectedUserService) DeleteUser(userID string) {
    if p.callerRole != "admin" {
        fmt.Println("Access denied: admin role required")
        return
    }
    p.real.DeleteUser(userID)
}
```

Usage:

```go
var svc UserService

svc = NewProtectedUserService("viewer")
svc.DeleteUser("u123") // "Access denied: admin role required"

svc = NewProtectedUserService("admin")
svc.DeleteUser("u123") // "User deleted: u123"
```

### Caching Proxy

**Problem:** The RealSubject performs expensive operations (database queries, external API calls, heavy computations) that produce the same result for the same input. Calling through every time is wasteful.

**Solution:** The Proxy caches results keyed by input. On a cache hit, it returns the stored result immediately without calling the RealSubject.

```go
type DataFetcher interface {
    Fetch(query string) string
}

// RealSubject — hits the database
type DBFetcher struct{}

func (d *DBFetcher) Fetch(query string) string {
    fmt.Println("Querying DB:", query)
    return "result:" + query
}

// Caching Proxy
type CachingFetcher struct {
    real  *DBFetcher
    cache map[string]string
}

func NewCachingFetcher() *CachingFetcher {
    return &CachingFetcher{
        real:  &DBFetcher{},
        cache: make(map[string]string),
    }
}

func (c *CachingFetcher) Fetch(query string) string {
    if val, ok := c.cache[query]; ok {
        fmt.Println("Cache hit:", query)
        return val
    }
    result := c.real.Fetch(query)
    c.cache[query] = result
    return result
}
```

Usage:

```go
var f DataFetcher = NewCachingFetcher()
f.Fetch("SELECT * FROM users") // "Querying DB: ..."
f.Fetch("SELECT * FROM users") // "Cache hit: ..."  — DB not touched
```

### Remote Proxy

**Problem:** The RealSubject lives in a different address space — a different process, a different machine. The client should be able to call it as if it were a local object, without caring about serialisation or network transport.

**Solution:** The Proxy sits locally, serialises the call, sends it over the network, receives the response, and deserialises it. The client sees only the interface.

This subtype is more architectural than code-level in Go, but its canonical real-world example is worth knowing cold: a **gRPC generated stub** is a remote proxy. When you call `client.GetUser(ctx, req)` on the generated client struct, you are calling a proxy — it marshals the request to protobuf, sends it over HTTP/2, receives the response, and returns it. The RealSubject is the server-side handler on a different machine.

```
Client machine                   Server machine
+-----------------+              +-----------------+
| App code        |              | gRPC server     |
|   client.Get()  |  ──HTTP/2──> |   GetUser()     |
| [Remote Proxy]  |              | [RealSubject]   |
+-----------------+              +-----------------+
```

## Proxy vs. Decorator

Both patterns implement the same interface as the object they wrap. This is the most common Proxy interview trap.

- **Proxy** controls *access* to the object. It manages whether, when, or how the real subject is reached. It often manages the real subject's lifecycle (creates it, decides when to instantiate it).
- **Decorator** adds *behaviour* to the object. It always delegates to the wrapped object and augments what happens around that delegation. The real object is created by the client and passed in.

```
Proxy:
  Client → Proxy (decides if/when to call) → RealSubject
  Proxy often owns and creates RealSubject

Decorator:
  Client creates RealSubject → passes to Decorator → Decorator always calls through
  Decorator enhances, not gates
```

A concrete way to tell them apart at a glance:
- If the wrapper can decide **not to call** the inner object at all (access denied, cache hit), it is a Proxy.
- If the wrapper **always calls** the inner object and just adds something around it (logging, compression), it is a Decorator.

In Go, both look structurally similar — a struct with a field of the same interface type. The distinction is in intent and behaviour, not in the language-level shape.

## Trade-offs

Advantages:
- Separates cross-cutting concerns (auth, caching, lazy init) from the real object's core logic.
- The client is oblivious to the proxy — no changes needed at call sites.
- Protection and caching proxies can improve security and performance without touching the real subject.
- Follows the Open/Closed principle — new proxy behaviours (logging, metrics) can be added without modifying the real subject or existing proxies.

Disadvantages:
- Adds indirection — the call chain becomes harder to trace in a debugger.
- Lazy initialisation hides a latency spike on first use. The caller cannot distinguish a cheap call from an expensive one just by reading the interface.
- Every method on the Subject interface must be implemented by every Proxy, even if the proxy only cares about one method. In Go, keep Subject interfaces small (interface segregation) to limit this burden.
- Overuse leads to a proliferation of thin wrapper types that obscure the actual object graph.

## How to Remember It

Think of a **hotel concierge (reverse of expectations)**:

The concierge is your proxy to the hotel's services. You ask the concierge for dinner reservations. The concierge checks if the restaurant is open (protection), looks up your saved preferences (caching), and calls the restaurant on your behalf (delegation). You never called the restaurant directly, and the restaurant never saw you directly.

The concierge and the restaurant both speak the same "reservation" language (Subject interface). You, the client, just talk to whoever is in front of you.

Memory hook for subtypes:
- **Virtual** → *lazy* — don't build it until asked
- **Protection** → *bouncer* — check the list before letting through
- **Caching** → *memory* — remember the answer, skip the work
- **Remote** → *ambassador* — speaks on behalf of something far away

## Interview Cheat Sheet

**"What is the Proxy pattern?"**
> "It provides a surrogate for another object to control access to it. The proxy and the real object implement the same interface, so the client is unaware of the indirection. The proxy intercepts calls and can do pre/post logic, short-circuit entirely, or delay the creation of the real object."

**"What are the main subtypes?"**
> "Virtual proxy delays creation of an expensive object until first use. Protection proxy gates access based on permissions. Caching proxy stores results of expensive calls and returns them on repeated identical requests. Remote proxy represents an object in a different address space — a gRPC stub is the canonical real-world example."

**"How is Proxy different from Decorator?"**
> "Both wrap an object implementing the same interface, but their intent differs. Proxy controls access — it manages whether or when the real object is reached, and often manages its lifecycle. Decorator adds behaviour — it always delegates to the inner object and augments what happens around that call. A quick tell: if the wrapper can choose not to call the inner object at all, it's a Proxy."

**"How do you implement Proxy in Go?"**
> "Define a Subject interface. Implement it with RealSubject. Implement it again with Proxy — Proxy holds a field of type Subject or *RealSubject. In the Proxy's method implementations, apply whatever logic is needed and delegate to the real object when appropriate. Go's implicit interface satisfaction means the client can hold a Subject reference and never know whether it's a proxy or the real thing."

**"Where would you use a caching proxy in a real system?"**
> "Any service that repeatedly fetches the same data from a slow upstream — database read replicas, external APIs, configuration stores. The caching proxy sits between the application and the data source. The application calls through the same interface; the proxy decides whether to hit the upstream or return a stored result. This is exactly what HTTP reverse proxy caches like Varnish do at the infrastructure level."

## References

- [GoF — Design Patterns: Elements of Reusable Object-Oriented Software](https://en.wikipedia.org/wiki/Design_Patterns) (Proxy, Chapter 4, pp. 207–217)
- [Refactoring.Guru — Proxy Pattern](https://refactoring.guru/design-patterns/proxy)
- [Go Patterns — Proxy](https://github.com/tmrts/go-patterns)
- [gRPC Go — Generated Client Stubs as Remote Proxies](https://grpc.io/docs/languages/go/basics/)