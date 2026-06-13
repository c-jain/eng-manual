---
Status: 🌳 Evergreen
Created: 2026-06-13
Last Updated: 2026-06-13
---

# Iterator Pattern (Rust)

> GoF Classification: **Behavioural**
> Also known as: **Cursor** (in database contexts)
>
> In Rust, this "pattern" is largely built into the language itself via the
> `Iterator` trait — `for` loops, `collect`, `sum`, and most of `std` all sit
> on top of it. This file teaches it as Rust's native model for sequential
> access, from first principles.

## Table of Contents

- [What It Is](#what-it-is)
- [Why Rust Bakes Iterator Into The Language](#why-rust-bakes-iterator-into-the-language)
- [The Iterator Trait](#the-iterator-trait)
- [IntoIterator And The For Loop](#intoiterator-and-the-for-loop)
- [Three Forms — iter, iter_mut, into_iter](#three-forms--iter-iter_mut-into_iter)
- [Adapters vs Consumers — Laziness](#adapters-vs-consumers--laziness)
- [Custom Iterator Implementation](#custom-iterator-implementation)
- [What Problems It Brings](#what-problems-it-brings)
- [How To Remember This](#how-to-remember-this)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

## What It Is

The GoF Iterator pattern's intent — sequential access to elements without
exposing the collection's internal representation — is, in Rust, realised as
a single trait in `std::iter`:

```rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;

    // ~70 more methods (map, filter, fold, collect, ...) — all provided
    // by default, implemented purely in terms of next().
}
```

That's the whole contract. Implement `next()` once, and the type
automatically gains every adapter and consumer method in the standard
library. There is no separate `Iterable` interface to implement (that role
is played by `IntoIterator`, covered below).

**Naming** — from the Latin *iterare*, "to repeat." The name describes
exactly what the trait does: it repeats the act of producing the next value,
until there's nothing left to produce.

## Why Rust Bakes Iterator Into The Language

Three forces shaped this design:

- **Zero-cost abstraction** — an iterator chain like
  `v.iter().map(f).filter(g).sum()` compiles, via monomorphization and
  inlining, to the same machine code as a hand-written `for` loop with an
  `if`. You get the readability of functional chains without a runtime
  penalty.
- **Laziness by default** — adapters (`map`, `filter`, `zip`, ...) build a
  chain of wrapper structs but do no work until a *consumer* (`collect`,
  `sum`, `for_each`, ...) pulls values through `next()`. This avoids
  allocating an intermediate collection at every step of a chain — the whole
  pipeline runs in a single pass, element by element.
- **Ownership encodes traversal mode** — Rust has three iteration entry
  points (`iter`, `iter_mut`, `into_iter`) precisely because "can I read
  this?", "can I mutate this in place?", and "can I take this away?" are
  different operations with different safety guarantees, and the type
  system should know which one you're doing.

## The Iterator Trait

```
Iterator Trait
+--------------------------------------------+
| type Item                                  |
| fn next(&mut self) -> Option<Item>         |  <- required (you write this)
|                                             |
| fn map(self, f) -> Map<Self, F>            |  <- provided (default impl)
| fn filter(self, p) -> Filter<Self, P>      |  <- provided (default impl)
| fn fold(self, init, f) -> B                |  <- provided (default impl)
| fn collect(self) -> C                      |  <- provided (default impl)
| ... ~70 more provided methods              |
+--------------------------------------------+

Implement next() once -> inherit the entire adapter ecosystem for free.
```

`Option<Self::Item>` bundles two questions into one answer:

- `Some(item)` = "there is a next element, and here it is"
- `None` = "there is nothing left"

This is what makes `next()` safe to call repeatedly without any separate
bookkeeping: pattern matching on the `Option` handles "is there more?" and
"give it to me" as a single atomic step. There's no extra check to remember
— the answer to "is there a value?" and the value itself arrive together,
or not at all.

## IntoIterator And The For Loop

`IntoIterator` is the trait that plays the role of GoF's `Iterable`:

```rust
pub trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item = Self::Item>;

    fn into_iter(self) -> Self::IntoIter;
}
```

Every `for` loop in Rust desugars to a call to `IntoIterator::into_iter`
followed by repeated `next()` calls:

```rust
// what you write
for x in collection {
    println!("{x}");
}

// what the compiler generates (roughly)
let mut iter = IntoIterator::into_iter(collection);
loop {
    match iter.next() {
        Some(x) => println!("{x}"),
        None => break,
    }
}
```

This is what makes `for` loops uniform across the entire language: `Vec<T>`,
`HashMap<K, V>`, `String` (via `.chars()`), and your own custom types all
work the same way in a `for` loop, governed by one trait rather than being
special-cased per type.

## Three Forms — iter, iter_mut, into_iter

```
Vec<Book>  (an owned collection)
   |
   +-- .iter()      -> Iterator<Item = &Book>     (borrow — shelf still usable)
   +-- .iter_mut()  -> Iterator<Item = &mut Book> (mut borrow — shelf still usable)
   +-- .into_iter() -> Iterator<Item = Book>      (move — shelf consumed)
```

```rust
struct Book {
    title: String,
    author: String,
}

let mut shelf: Vec<Book> = vec![
    Book { title: "The Rust Programming Language".into(), author: "Klabnik & Nichols".into() },
    Book { title: "Designing Data-Intensive Applications".into(), author: "Kleppmann".into() },
];

// iter() — read-only borrow, Item = &Book
for book in shelf.iter() {
    println!("{}", book.title);
}
// shelf is still usable here

// iter_mut() — mutable borrow, Item = &mut Book
for book in shelf.iter_mut() {
    book.title.push_str(" (read)");
}
// shelf is still usable here

// into_iter() — takes ownership, Item = Book
for book in shelf.into_iter() {
    println!("{}", book.title);
}
// shelf is MOVED — using `shelf` after this line is a compile error
```

A useful rule of thumb: the **number of underscores correlates with how much
access you're taking**. `iter` (look), `iter_mut` (look and touch),
`into_iter` (take home).

## Adapters vs Consumers — Laziness

```
shelf.iter()  ->  .filter(...)  ->  .map(...)  ->  .count()
    |                 |                |              |
 Iterator          Iterator         Iterator        usize
 (lazy)             (lazy)           (lazy)      (eager — runs the chain)

Nothing executes until .count() is called.
At that point, next() is pulled through filter -> map -> iter,
one element at a time, with no intermediate Vec allocated.
```

```rust
let kleppmann_count = shelf.iter()
    .filter(|b| b.author == "Kleppmann")  // adapter — lazy, returns Filter<...>
    .map(|b| &b.title)                     // adapter — lazy, returns Map<...>
    .count();                              // consumer — eager, drives the whole chain
```

**Adapters (lazy):** `map`, `filter`, `take`, `skip`, `enumerate`, `zip`,
`rev`, `chain`, `scan`, `flat_map`

**Consumers (eager):** `collect`, `sum`, `fold`, `for_each`, `count`, `max`,
`min`, `any`, `all`

If you never call a consumer, the chain never runs — the compiler will warn
about an unused `impl Iterator` value, but no panic, no error. It's simply
inert data describing a computation.

## Custom Iterator Implementation

Let's build a custom iterator from scratch — a `BookShelf` that yields
`Book`s one at a time — to see how all the pieces fit together:

```rust
struct Book {
    title: String,
    author: String,
}

struct BookShelf {
    books: Vec<Book>,
}

// The concrete iterator. The lifetime 'a ties the iterator's validity
// to the borrow of `books` it holds.
struct BookShelfIter<'a> {
    books: &'a [Book],
    pos: usize,
}

impl<'a> Iterator for BookShelfIter<'a> {
    type Item = &'a Book;

    fn next(&mut self) -> Option<Self::Item> {
        if self.pos < self.books.len() {
            let book = &self.books[self.pos];
            self.pos += 1;
            Some(book)
        } else {
            None
        }
    }
}

impl BookShelf {
    fn iter(&self) -> BookShelfIter<'_> {
        BookShelfIter { books: &self.books, pos: 0 }
    }
}

fn main() {
    let shelf = BookShelf {
        books: vec![
            Book { title: "The Rust Programming Language".into(), author: "Klabnik & Nichols".into() },
            Book { title: "Designing Data-Intensive Applications".into(), author: "Kleppmann".into() },
        ],
    };

    // Basic traversal
    for book in shelf.iter() {
        println!("• {} by {}", book.title, book.author);
    }

    // Because BookShelfIter implements Iterator, it gets map/filter/collect
    // for free — no extra code needed.
    let kleppmann_titles: Vec<&String> = shelf.iter()
        .filter(|b| b.author == "Kleppmann")
        .map(|b| &b.title)
        .collect();

    println!("{:?}", kleppmann_titles);
}
```

That `filter`/`map`/`collect` chain is the entire payoff of implementing
`Iterator`: one `next()` method, and you get filtering, mapping, counting,
summing, and ~65 other operations for free — no extra code needed for any
of them.

### Making The Collection Itself Iterable

To support `for book in shelf` directly (consuming the shelf), implement
`IntoIterator`:

```rust
impl IntoIterator for BookShelf {
    type Item = Book;
    type IntoIter = std::vec::IntoIter<Book>;

    fn into_iter(self) -> Self::IntoIter {
        self.books.into_iter()
    }
}

// Usage — this moves `shelf`
for book in shelf {
    println!("{}", book.title);
}
```

## What Problems It Brings

**Borrow checker friction (a feature, not really a bug)**
The classic "modify a collection while iterating over it" bug is a compile
error in Rust, not a runtime surprise:

```rust
let mut v = vec![1, 2, 3];

for x in v.iter() {
    v.push(*x); // ERROR: cannot borrow `v` as mutable
                 // because it is also borrowed as immutable
}
```

Catching this at compile time, rather than as a confusing runtime bug, is a
genuine safety win — but it's the single biggest source of "fighting the
compiler" for newcomers, especially because the error message is correct
without immediately feeling intuitive.

**Lifetime annotations on custom iterators**
Any iterator struct that holds a reference into a collection needs a
lifetime parameter (`BookShelfIter<'a>` above). This is the first place most
people encounter explicit lifetimes, and it can feel like ceremony before
the payoff (free adapters) becomes clear.

**Infinite iterators**
Ranges like `0..` are infinite. Calling an eager consumer on one without
`.take(n)` first will hang or OOM:

```rust
let fib = (0..).scan((0u64, 1u64), |state, _| {
    let next = state.0;
    *state = (state.1, state.0 + state.1);
    Some(next)
});

let first_ten: Vec<u64> = fib.take(10).collect(); // .take() before .collect() — required
```

**Not every iterator is double-ended**
`.rev()` requires the type to implement `DoubleEndedIterator`. Most
standard collection iterators do, but custom iterators don't get this for
free — `next()` alone doesn't imply `next_back()`.

**Adapter chain readability**
Long chains (`.iter().filter(...).map(...).flat_map(...).fold(...)`) are
expressive but can become hard to debug — a `dbg!()` or intermediate
`.collect::<Vec<_>>()` is often needed to inspect a middle step.

## How To Remember This

**The "library book" mnemonic** for the three access forms:

```
.iter()       -> "reading it at the library"   (borrow,     book stays on shelf)
.iter_mut()   -> "reading with a pencil,
                  scribbling in the margins"   (mut borrow, book stays on shelf)
.into_iter()  -> "checking it out"             (move,       book leaves the shelf)
```

**Option fuses "is there more?" and "what is it?"**
`Some(x)` is "yes, and here it is." `None` is "no." One return value, two
answers — there's nothing extra to check before calling `next()` again.

**One method in, seventy methods out**
If you remember nothing else: implement `next()`, and `map`/`filter`/
`fold`/`collect`/`sum`/`zip`/... all arrive for free. This single fact
explains most of what makes iterating in Rust feel different from a plain
hand-written loop.

**Lazy until consumed**
Adapters are recipes. Consumers are the oven. Nothing bakes until you turn
the oven on (`collect`, `sum`, `for_each`, ...).

## Interview Cheat Sheet

### Signal Phrases

- "Rust's `Iterator` trait has one required method, `next() -> Option<Item>`
  — everything else is a default method implemented in terms of `next`, so
  a custom type gets the full adapter ecosystem by implementing one
  function."
- "Iterator adapters are lazy — `map`/`filter`/`zip` build a chain of
  wrapper types but do no work until a consumer like `collect` or `sum`
  drives the chain."
- "`iter`, `iter_mut`, and `into_iter` aren't three names for the same
  thing — they correspond to borrow, mutable borrow, and move, and the
  compiler enforces which one is valid in a given context."
- "The classic 'mutate while iterating' bug becomes a compile-time borrow
  error in Rust rather than a runtime panic or silent corruption."
- "`for` loops desugar to `IntoIterator::into_iter` plus repeated `next()`
  calls — that's why any type implementing `IntoIterator` works in a `for`
  loop, with no special-casing."

### Red Flags To Avoid

- Saying `.iter()` "consumes" the collection — it borrows; `.into_iter()`
  consumes
- Calling `.collect()` on an unbounded range (`0..`) without `.take()`
  first
- Forgetting the lifetime parameter on a custom iterator struct that holds
  `&'a [T]`
- Assuming every iterator supports `.rev()` — that requires
  `DoubleEndedIterator`
- Treating adapter chains as executing top-to-bottom immediately — they
  don't, until a consumer is reached

### Common Interview Probes

- "Implement a custom iterator for a linked list / binary tree node."
  - One `next()` implementation; mention whether it's depth-first or
    breadth-first and what auxiliary stack/queue state the iterator struct
    needs to hold
- "What's the difference between `iter()`, `iter_mut()`, and
  `into_iter()`?"
  - Borrow vs mutable borrow vs move — and the resulting `Item` type
    (`&T`, `&mut T`, `T`)
- "Why does `for x in vec { ... }` move `vec`, but
  `for x in &vec { ... }` doesn't?"
  - `for x in vec` calls `vec.into_iter()` (via `IntoIterator for Vec<T>`,
    which yields owned `T`); `for x in &vec` calls
    `(&vec).into_iter()` (yielding `&T`) via the blanket impl for
    `&Vec<T>`
- "How would you write an iterator over Fibonacci numbers?"
  - State held in the iterator struct (or via `.scan()`), `next()` advances
    and returns `Some(value)` — note it's naturally infinite, so callers
    must `.take(n)`
- "Why is chaining `.filter().map().collect()` not less efficient than a
  hand-written loop?"
  - Monomorphization + inlining: the adapter types are zero-sized wrappers
    around closures, and LLVM collapses the whole chain into a single loop

## References

- *The Rust Programming Language* (The Book) — Chapter 13, "Functional
  Language Features: Iterators and Closures"
- `std::iter` module documentation — [doc.rust-lang.org/std/iter](https://doc.rust-lang.org/std/iter/index.html)
- `std::iter::Iterator` trait documentation (full list of provided methods)
- Gamma et al., *Design Patterns: Elements of Reusable Object-Oriented
  Software* — Iterator chapter (general background on the pattern's intent)