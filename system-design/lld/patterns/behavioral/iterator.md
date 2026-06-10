# Iterator Pattern

```
Status:       🌳 Evergreen
Created:      2026-06-10
Last Updated: 2026-06-10
```

> GoF Classification: **Behavioural**
> Also known as: **Cursor**


## Table of Contents

- [What the Pattern Is](#what-the-pattern-is)
- [Why It Exists](#why-it-exists)
- [GoF Participants](#gof-participants)
- [Structure](#structure)
- [Sequence Diagram](#sequence-diagram)
- [Go Implementation](#go-implementation)
- [Go Standard Library Examples](#go-standard-library-examples)
- [External vs Internal Iterators](#external-vs-internal-iterators)
- [What Problems It Brings](#what-problems-it-brings)
- [How to Remember This](#how-to-remember-this)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)


## What the Pattern Is

Iterator provides a standard way to **sequentially access elements of a collection without exposing the collection's internal representation**.

The pattern extracts traversal state into a separate object — the iterator — so that:

- The collection does not need to know how it is being traversed
- Multiple traversal strategies (forward, reverse, filtered, depth-first) can coexist without modifying the collection
- Client code works against a uniform interface regardless of whether it is iterating a slice, a tree, or a database cursor

**Why "iterator"?** From the Latin *iterare* — to repeat. The iterator "repeats the process" of advancing through a sequence one element at a time. The same root gives us "iterate" and "reiterate."

**Also called "cursor"** in database contexts — `sql.Rows` in Go, database cursors in SQL — because the object holds a position (cursor) that advances through a result set.


## Why It Exists

Without the Iterator pattern, every client that wants to traverse a collection must reach into its internals:

```
// Without Iterator — client must know the collection is a slice
for i := 0; i < shelf.Len(); i++ {
    book := shelf.books[i]   // direct field access — tight coupling
}
```

This means:

- Changing the collection's internal structure (from `[]Book` to a linked list or BST) breaks all clients
- Adding a new traversal strategy (e.g., reverse, filtered) requires modifying the collection
- Two collections with different implementations cannot be traversed with shared client code

The Iterator pattern isolates this: the client only ever holds an `Iterator[T]` interface and calls `HasNext()` / `Next()`. The collection can be refactored freely as long as it still returns a conforming iterator.


## GoF Participants

```
PARTICIPANT          ROLE
───────────          ──────────────────────────────────────────────────
Iterator[T]          Defines the traversal contract: HasNext(), Next()
ConcreteIterator     Holds traversal state (current position, filters).
                     Implements Iterator[T].
Iterable[T]          Collection declares it can be iterated via
                     CreateIterator() — returns an Iterator[T].
ConcreteCollection   Implements Iterable[T]. On CreateIterator(), it
                     constructs and returns a ConcreteIterator.
Client               Uses Iterator[T] only — never the concrete types.
```


## Structure

```
        Client
          |
          | uses (only the Iterator interface)
          v

+---------------------+            +---------------------------+
| «interface»         |            | «interface»               |
| Iterator[T]         |            | Iterable[T]               |
+---------------------+            +---------------------------+
| + HasNext() bool    |            | + CreateIterator()        |
| + Next() T          |            |     Iterator[T]           |
+---------------------+            +---------------------------+
          ^                                    ^
          | implements                         | implements
          |                                    |
+---------------------+   creates   +---------------------------+
| ConcreteIterator    |<────────────| ConcreteCollection        |
+---------------------+             +---------------------------+
| - items []T         |             | - items []T               |
| - pos   int         |             | + Add(item T)             |
| + HasNext() bool    |             | + CreateIterator()        |
| + Next() T          |             |     Iterator[T]           |
+---------------------+             +---------------------------+
```

The `creates` arrow from ConcreteCollection to ConcreteIterator reflects that `CreateIterator()` instantiates and returns the concrete iterator — but the caller receives only the `Iterator[T]` interface.


## Sequence Diagram

```
Client          ConcreteCollection     ConcreteIterator
  |                    |                      |
  |-- CreateIterator-->|                      |
  |                    |-- new(iterator) ----->|
  |<---- Iterator[T] --|                      |
  |                                           |
  |-- HasNext() --------------------------------->|
  |<---- true -----------------------------------|
  |                                           |
  |-- Next() ---------------------------------->|
  |<---- item T --------------------------------|
  |                                           |
  |   (loop until HasNext() returns false)    |
  |                                           |
  |-- HasNext() --------------------------------->|
  |<---- false ----------------------------------|
```


## Go Implementation

### Basic External Iterator

```go
package main

import "fmt"

// Iterator is the traversal contract.
// The type parameter T makes this generic over any element type.
type Iterator[T any] interface {
    HasNext() bool
    Next() T
}

// Iterable is implemented by any collection that can produce an iterator.
type Iterable[T any] interface {
    CreateIterator() Iterator[T]
}

// ---------- Concrete types ----------

type Book struct {
    Title  string
    Author string
}

// BookShelf is the concrete collection.
type BookShelf struct {
    books []Book
}

func (bs *BookShelf) Add(b Book) {
    bs.books = append(bs.books, b)
}

// CreateIterator returns an iterator over the current snapshot of books.
// The slice is copied so modifications to the shelf after this call
// do not affect the iterator (stale iterator prevention).
func (bs *BookShelf) CreateIterator() Iterator[Book] {
    snapshot := make([]Book, len(bs.books))
    copy(snapshot, bs.books)
    return &BookIterator{books: snapshot, pos: 0}
}

// BookIterator is the concrete iterator — it holds all traversal state.
type BookIterator struct {
    books []Book
    pos   int
}

func (it *BookIterator) HasNext() bool {
    return it.pos < len(it.books)
}

func (it *BookIterator) Next() Book {
    b := it.books[it.pos]
    it.pos++
    return b
}

// ---------- Usage ----------

func printAll(it Iterator[Book]) {
    for it.HasNext() {
        book := it.Next()
        fmt.Printf("  • %s by %s\n", book.Title, book.Author)
    }
}

func main() {
    shelf := &BookShelf{}
    shelf.Add(Book{"The Go Programming Language", "Donovan & Kernighan"})
    shelf.Add(Book{"Designing Data-Intensive Applications", "Kleppmann"})
    shelf.Add(Book{"Clean Code", "Martin"})

    it := shelf.CreateIterator()
    printAll(it)
}
```

Output:
```
  • The Go Programming Language by Donovan & Kernighan
  • Designing Data-Intensive Applications by Kleppmann
  • Clean Code by Martin
```

`printAll` accepts `Iterator[Book]` — it works with any concrete collection, not just `BookShelf`.


### Filtered Iterator Variant

You can add traversal strategies without touching `BookShelf`:

```go
// FilteredIterator wraps any Iterator[Book] and skips non-matching elements.
type FilteredIterator struct {
    inner     Iterator[Book]
    predicate func(Book) bool
    next      *Book
}

func NewFilteredIterator(it Iterator[Book], predicate func(Book) bool) *FilteredIterator {
    fi := &FilteredIterator{inner: it, predicate: predicate}
    fi.advance()
    return fi
}

func (fi *FilteredIterator) advance() {
    fi.next = nil
    for fi.inner.HasNext() {
        b := fi.inner.Next()
        if fi.predicate(b) {
            fi.next = &b
            return
        }
    }
}

func (fi *FilteredIterator) HasNext() bool { return fi.next != nil }

func (fi *FilteredIterator) Next() Book {
    val := *fi.next
    fi.advance()
    return val
}

// Usage: iterate only books by Kleppmann
it := shelf.CreateIterator()
filtered := NewFilteredIterator(it, func(b Book) bool {
    return b.Author == "Kleppmann"
})
printAll(filtered) // FilteredIterator satisfies Iterator[Book]
```


### Channel-Based Generator (Go-Idiomatic Lazy Iteration)

For sequences that are expensive to compute or infinite, a goroutine + channel is idiomatic Go:

```go
// BookStream returns a receive-only channel that emits books one at a time.
// The goroutine runs lazily — it blocks until the consumer reads.
func (bs *BookShelf) BookStream() <-chan Book {
    ch := make(chan Book)
    go func() {
        defer close(ch)
        for _, book := range bs.books {
            ch <- book
        }
    }()
    return ch
}

// Usage — clean range loop, no HasNext/Next boilerplate
for book := range shelf.BookStream() {
    fmt.Println(book.Title)
}
```

The channel acts as a buffered boundary: the producer (goroutine) and consumer (range loop) run concurrently. When the consumer `break`s early, the goroutine blocks on `ch <-` and is leaked unless the channel is drained or the goroutine uses a `context` for cancellation.


### Go 1.23+ — `iter.Seq` (Standard Library Native)

Go 1.23 introduced the `iter` package and support for ranging over functions. This is now the idiomatic way to expose custom iterators:

```go
import "iter"

// All returns an iter.Seq[Book] that the caller can use in a range loop.
func (bs *BookShelf) All() iter.Seq[Book] {
    return func(yield func(Book) bool) {
        for _, b := range bs.books {
            if !yield(b) {
                return // caller broke out of the loop; stop producing
            }
        }
    }
}

// Usage — works with built-in range directly
for book := range shelf.All() {
    fmt.Println(book.Title)
}
```

The `yield` function is called for each element. If the caller breaks out of the range, `yield` returns `false` — the iterator must respect this and stop immediately.


## Go Standard Library Examples

The Iterator pattern appears throughout the Go standard library — you have used it without necessarily naming it:

**`bufio.Scanner`** — iterate over lines of text:

```go
scanner := bufio.NewScanner(file)
for scanner.Scan() {           // HasNext()
    line := scanner.Text()    // current element
    fmt.Println(line)
}
if err := scanner.Err(); err != nil {
    log.Fatal(err)
}
```

**`database/sql.Rows`** — iterate over query results:

```go
rows, err := db.Query("SELECT id, name FROM users WHERE active = true")
if err != nil { log.Fatal(err) }
defer rows.Close()

for rows.Next() {              // HasNext()
    var id int
    var name string
    rows.Scan(&id, &name)     // extract from current position
    fmt.Printf("%d: %s\n", id, name)
}
```

Both examples follow the same contract: call the advance method (`Scan` / `Next`), then read the current value. The underlying implementation — whether it is reading from a file, a network socket, or a database cursor — is invisible to the caller.


## External vs Internal Iterators

### External Iterator

The **client** controls the loop. The client explicitly calls `HasNext()` and `Next()`.

```go
it := shelf.CreateIterator()
for it.HasNext() {
    process(it.Next())
}
```

- Client can pause, resume, or skip elements
- Client can pass the iterator to another function
- More flexible, but more error-prone — calling `Next()` without `HasNext()` panics

### Internal Iterator

The **collection** drives the loop. The client supplies a callback.

```go
shelf.ForEach(func(b Book) {
    process(b)
})
```

- Cleaner API for simple traversal
- Harder to break early (unless the callback returns a bool)
- Go's `range` keyword is essentially a built-in internal iterator for slices, maps, and channels


## What Problems It Brings

**Stale iterator**
If the collection is mutated after `CreateIterator()` is called, the iterator's position may skip elements, repeat elements, or panic on an out-of-bounds index. Mitigation: copy the slice at iterator creation (snapshot semantics) or version-stamp the collection and check on each `Next()`.

**Leaked goroutine in channel-based iterators**
If the consumer breaks early from a `range` over a channel, the producer goroutine blocks forever on `ch <-`. Fix: pass a `context.Context` and select on `<-ctx.Done()` in the producer.

**Misuse of external iterators**
Calling `Next()` without checking `HasNext()` first will panic or return a zero value. Design the API so this is hard to do — e.g., return `(T, bool)` from `Next()`.

**Concurrency**
The iterator holds a reference to (or snapshot of) the collection and is not goroutine-safe by default. If multiple goroutines share an iterator, a mutex is required.

**Performance on hot paths**
Creating an iterator struct per traversal is negligible for most use cases, but for tight loops over millions of elements, direct slice indexing is measurably faster. Profile before abstracting.


## How to Remember This

**Mental image: a TV remote control**

The remote is the iterator. You press `NEXT` (→ `Next()`) to advance through channels and `GUIDE` (→ `HasNext()`) to check whether there are more. You do not know how channels are stored inside the TV — the remote hides that completely.

**The three-part memory hook:**

```
1. SEPARATE  — traversal state lives in the iterator, not the collection
2. STANDARD  — all iterators look the same to the client (HasNext + Next)
3. SWAP      — you can swap the collection's internals without changing client code
```

**Go specific — three ways to iterate:**

```
struct-based   ->  HasNext() / Next()                (explicit, flexible)
channel-based  ->  for x := range ch {}              (lazy, goroutine)
iter.Seq       ->  for x := range coll.All() {}      (Go 1.23+, idiomatic)
```


## Interview Cheat Sheet

### Signal Phrases

- "Iterator decouples traversal strategy from the collection — you can add reverse, filtered, or paginated iteration without touching the collection itself."
- "Go's `range` keyword is a built-in internal iterator; `bufio.Scanner` and `database/sql.Rows` are external iterators from the standard library."
- "External iterators give the client control and the ability to pause mid-traversal. Internal iterators are simpler but require the client to commit to processing every element."
- "In Go 1.23+, `iter.Seq` makes custom iterators first-class so they work with `range` — that is the idiomatic direction."
- "The main footgun with channel-based iterators is goroutine leaks — if the consumer breaks early, the producer blocks forever unless you signal it with a context."

### Red Flags to Avoid

- "I would just use a for loop with an index" — this misses the abstraction: it couples the client to the collection's internal representation and prevents swapping implementations
- Confusing Iterator with the Composite pattern — Composite is about tree structures; Iterator is about traversal of any collection
- Forgetting to handle early termination in `iter.Seq` (not checking the return value of `yield`)
- Building a channel-based iterator without a cancellation path

### Common Interview Probes

- "When would you use Iterator over just returning a `[]T` slice?"
  - When the sequence is lazy (too large to materialise), when it streams from an I/O source (file, network, DB cursor), or when multiple traversal strategies need to be swapped
- "How would you make your iterator thread-safe?"
  - Snapshot the collection at iterator creation, or wrap `HasNext`/`Next` in a `sync.Mutex`, or use a channel-based iterator (channel operations are goroutine-safe)
- "What is the difference between external and internal iterators?"
  - External: client calls `HasNext`/`Next`, controls the loop. Internal: collection calls a user-supplied callback for each element.
- "How does `bufio.Scanner` implement this pattern?"
  - `Scan()` advances and returns whether there is more (`HasNext`). `Text()` returns the current line (`current`). State is hidden in the `Scanner` struct.
- "How would you implement a paginated database iterator?"
  - Iterator holds `offset` and `pageSize`; `HasNext` fetches the next page if the current page is exhausted and the last page was full; `Next` returns elements from the buffered page


## References

- Gamma et al., *Design Patterns: Elements of Reusable Object-Oriented Software* — Iterator chapter
- Refactoring.Guru — [Iterator Pattern](https://refactoring.guru/design-patterns/iterator)
- Go standard library — `bufio.Scanner`, `database/sql.Rows`, `iter` package (Go 1.23+)
- Go blog — "Range Over Function Types" (Go 1.22/1.23 proposal and rationale)