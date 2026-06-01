# Composite Pattern

## Table of Contents

- [What It Is and Why It Exists](#what-it-is-and-why-it-exists)
- [The Core Problem](#the-core-problem)
- [GoF Definition](#gof-definition)
- [Participants](#participants)
- [Safe vs. Transparent Design](#safe-vs-transparent-design)
- [Go Implementation](#go-implementation)
- [UML Class Diagram](#uml-class-diagram)
- [Sequence Diagram](#sequence-diagram)
- [Real-World Analogies](#real-world-analogies)
- [When to Use](#when-to-use)
- [Trade-offs](#trade-offs)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

---

## What It Is and Why It Exists

The Composite pattern lets you **treat individual objects and groups of objects through the same interface**. It models part-whole tree hierarchies — where a node can be either a leaf (no children) or a composite (contains children that are themselves nodes).

**Why it exists:** Without this pattern, client code must distinguish between leaves and composites manually. Every traversal becomes an explicit type-check loop, repeated everywhere the tree is walked. The pattern pushes traversal logic inside the objects themselves, giving the client a uniform interface.

**Why it is called "Composite":** The name refers to the `Composite` participant — the node that *composes* (aggregates) other nodes. The pattern is named after its structurally interesting participant, not the leaf.

**Memory anchor:** Think of a file system. `File.Size()` returns the file's own size. `Directory.Size()` returns the sum of its children's sizes — and each child may itself be a `Directory`. The caller just writes `node.Size()` and the tree handles itself recursively. That's Composite.

---

## The Core Problem

Consider a file system without the pattern:

```go
// Caller must inspect and branch manually, everywhere
func totalSize(node Node) int {
    if node.IsFile() {
        return node.FileSize()
    }
    total := 0
    for _, child := range node.Children() {
        total += totalSize(child) // manual recursive dispatch
    }
    return total
}
```

Problems:
- The branching logic is the caller's responsibility.
- Adding a new node type (e.g., a `Symlink`) requires changing every traversal in the codebase.
- Violates the Open/Closed Principle — open to extension, closed to modification.

With Composite, `totalSize` becomes `node.Size()` — one call, no branching.

---

## GoF Definition

> "Compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions of objects uniformly."

— Design Patterns: Elements of Reusable Object-Oriented Software (GoF)

**Key phrase:** "part-whole hierarchy" — a structure where the whole is made of parts, and those parts can themselves be wholes.

---

## Participants

| Participant | Role |
|---|---|
| **Component** | The interface (or abstract type) shared by Leaf and Composite. Defines the operation clients call. |
| **Leaf** | A node with no children. Implements `Operation()` with actual work. |
| **Composite** | A node that holds children. Implements `Operation()` by delegating to each child. Also manages children (Add/Remove). |
| **Client** | Uses the Component interface only. Unaware of whether it is talking to a Leaf or Composite. |

---

## Safe vs. Transparent Design

This is the most important design decision in Composite, and interviewers ask about it directly.

**Transparent design:** `Add` and `Remove` are declared on the `Component` interface itself.
- Advantage: client code is fully uniform — every node is a `Component`, no downcasting.
- Disadvantage: calling `Add` on a `Leaf` is a logical error at runtime (Leaf has no children). Slightly violates LSP.

**Safe design:** `Add` and `Remove` are only on the `Composite` type.
- Advantage: compile-time safety — you cannot call `Add` on a `Leaf` because `Leaf` doesn't have that method.
- Disadvantage: client code must downcast to `Composite` to add children — less uniform.

**In Go, safe design is idiomatic.** Go interfaces should be minimal. Putting `Add`/`Remove` on the `Component` interface forces every `Leaf` to implement methods it cannot meaningfully use. Go prefers small interfaces and type assertion when needed.

```
Transparent (not idiomatic Go):          Safe (idiomatic Go):

Component interface:                     Component interface:
  Operation()                              Operation()
  Add(Component)     <-- on Leaf too
  Remove(Component)  <-- on Leaf too
                                          Composite struct:
                                            Operation()
                                            Add(Component)
                                            Remove(Component)
```

---

## Go Implementation

### Component Interface

```go
// Component is the uniform interface for both leaves and composites.
// Kept minimal — only the operation clients care about.
type Component interface {
    Name() string
    Size() int
    Display(indent string)
}
```

### Leaf — File

```go
// File is a leaf node. It has no children.
type File struct {
    name string
    size int
}

func NewFile(name string, size int) *File {
    return &File{name: name, size: size}
}

func (f *File) Name() string { return f.name }
func (f *File) Size() int    { return f.size }

func (f *File) Display(indent string) {
    fmt.Printf("%s- %s (%d bytes)\n", indent, f.name, f.size)
}
```

### Composite — Directory

```go
// Directory is a composite node. It aggregates Component children.
// Add/Remove live here (safe design), not on the Component interface.
type Directory struct {
    name     string
    children []Component
}

func NewDirectory(name string) *Directory {
    return &Directory{name: name}
}

func (d *Directory) Name() string { return d.name }

// Size delegates to children — the recursive step.
func (d *Directory) Size() int {
    total := 0
    for _, child := range d.children {
        total += child.Size()
    }
    return total
}

func (d *Directory) Display(indent string) {
    fmt.Printf("%s+ %s/\n", indent, d.name)
    for _, child := range d.children {
        child.Display(indent + "  ")
    }
}

// Add and Remove are only on Directory — safe design.
func (d *Directory) Add(c Component) {
    d.children = append(d.children, c)
}

func (d *Directory) Remove(name string) {
    filtered := d.children[:0]
    for _, c := range d.children {
        if c.Name() != name {
            filtered = append(filtered, c)
        }
    }
    d.children = filtered
}
```

### Client Code

```go
func main() {
    // Build the tree
    root := NewDirectory("root")

    src := NewDirectory("src")
    src.Add(NewFile("main.go", 1200))
    src.Add(NewFile("handler.go", 800))

    docs := NewDirectory("docs")
    docs.Add(NewFile("README.md", 400))
    docs.Add(NewFile("design.md", 600))

    root.Add(src)
    root.Add(docs)
    root.Add(NewFile(".gitignore", 50))

    // Client uses the Component interface uniformly
    root.Display("")
    fmt.Printf("\nTotal size: %d bytes\n", root.Size())

    // To add children, client must have a *Directory (safe design)
    // If the client only has a Component, it type-asserts:
    var c Component = root
    if dir, ok := c.(*Directory); ok {
        dir.Add(NewFile("Makefile", 200))
    }
}
```

**Output:**
```
+ root/
  + src/
    - main.go (1200 bytes)
    - handler.go (800 bytes)
  + docs/
    - README.md (400 bytes)
    - design.md (600 bytes)
  - .gitignore (50 bytes)

Total size: 3050 bytes
```

### Why This Works

The `Size()` and `Display()` calls on `root` recursively propagate through the tree. Each `Directory` iterates its children and calls the same methods on them — it does not know or care whether any child is a `File` or another `Directory`. The polymorphism handles it. The client wrote `root.Size()` and got the correct answer without branching.

---

## UML Class Diagram

```
+------------------+
|   <<interface>>  |
|    Component     |
+------------------+
| + Name() string  |
| + Size() int     |
| + Display()      |
+------------------+
        ^
        |
   _____|_____
  |           |
+--------+  +---------------------+
|  File  |  |      Directory      |
+--------+  +---------------------+
| -name  |  | -name               |
| -size  |  | -children []Component|
+--------+  +---------------------+
| Name() |  | Name()              |
| Size() |  | Size()  <-- loops   |
|Display()|  | Display()           |
+--------+  | Add(Component)      |
            | Remove(name string) |
            +---------------------+
                      |
                      | children (0..*)
                      v
                  Component
                (File or Directory)
```

Note: The `children` field holds `[]Component`, meaning each child can be either a `File` or a `Directory`. This self-referential relationship is the heart of the pattern.

---

## Sequence Diagram

Scenario: client calls `root.Size()` on a tree with one `Directory` containing one `File`.

```
Client       root:Directory   src:Directory    main.go:File
  |               |                |                |
  | Size()        |                |                |
  |-------------> |                |                |
  |               | Size()         |                |
  |               |--------------> |                |
  |               |                | Size()         |
  |               |                |--------------> |
  |               |                |    1200        |
  |               |                | <------------- |
  |               |      1200      |                |
  |               | <------------- |                |
  |      1200     |                |                |
  | <------------ |                |                |
```

Each `Directory.Size()` delegates to children. The recursion bottoms out at `File.Size()`, which returns a concrete value.

---

## Real-World Analogies

- **File system:** Files are leaves; Directories are composites. `du -sh` walks the tree and sums sizes.
- **UI component tree:** A `Button` is a leaf. A `Panel` contains `Button`s, `Label`s, and other `Panel`s. `Render()` propagates through the tree.
- **Organisation chart:** An employee is a leaf. A manager is a composite whose `Salary()` sums the team's salaries.
- **HTML/DOM:** Text nodes are leaves. Element nodes (`<div>`, `<ul>`) are composites containing other nodes.
- **Go's `io.MultiWriter`:** Writes to multiple writers as if they were one — structurally analogous to Composite for the `Write` operation.

---

## When to Use

Use Composite when:
- You have a **tree structure** where leaves and composites need to be treated uniformly by the client.
- Client code should not need to distinguish between individual objects and collections of objects.
- You want to add new node types without modifying existing traversal code (Open/Closed Principle).

Do not use Composite when:
- The structure is not genuinely tree-shaped. Forcing Composite onto a flat list or graph is over-engineering.
- You need different behaviour for leaves vs. composites that cannot be unified behind a single interface.

---

## Trade-offs

**Advantages:**
- Client code is simple — one interface, no type-checking branches.
- Adding new leaf or composite types does not require changing existing code.
- Recursive operations (sum, search, render) are naturally expressed.

**Disadvantages:**
- The `Component` interface must be broad enough for both leaves and composites. If they differ significantly, the interface becomes a compromise.
- Safe design requires downcasting when the client needs to call `Add`/`Remove` — this is not always elegant.
- Deep trees can cause deep call stacks; very large trees may have performance implications for recursive operations.

---

## Interview Cheat Sheet

**Common Questions**

- *"What problem does Composite solve?"*
  It allows clients to treat individual objects and groups of objects through the same interface, without branching on type. Eliminates manual recursive dispatch from client code.

- *"What is the difference between safe and transparent Composite design?"*
  Transparent puts `Add`/`Remove` on the `Component` interface — uniform but unsafe (Leaf must implement them, runtime error if called). Safe puts `Add`/`Remove` only on `Composite` — type-safe but requires downcasting. Go favours the safe approach.

- *"Where does Composite differ from Decorator?"*
  Both wrap objects behind a shared interface. Composite focuses on **tree structure** — one composite, many children, recursive operations. Decorator focuses on **single-object enhancement** — one wrapper around one object, adding behaviour.

- *"How does Go's interface model interact with Composite?"*
  Go's implicit interface satisfaction makes the Component interface natural. The main consideration is keeping the interface minimal (safe design) so Leaves aren't burdened with methods they can't implement meaningfully.

**Strong Signal Phrases**
- "I'd keep the Component interface minimal and put child management methods only on the Composite — that's the safe variant and it's more idiomatic in Go."
- "The recursive delegation in `Directory.Size()` is the core of Composite — each node calls the same method on its children, and the tree traversal is owned by the objects, not the client."
- "This is the same structure as the DOM, a UI widget tree, or a file system — any part-whole hierarchy where you want to treat leaves and branches uniformly."
- "I'd consider the Composite pattern any time I'm writing recursive type-switch logic in client code — that's usually a signal the pattern is missing."

---

## References

- [GoF — Design Patterns: Elements of Reusable Object-Oriented Software](https://www.oreilly.com/library/view/design-patterns-elements/0201633612/) — Chapter 4, Structural Patterns, Composite
- [Refactoring Guru — Composite Pattern](https://refactoring.guru/design-patterns/composite)
- [Refactoring Guru — Composite in Go](https://refactoring.guru/design-patterns/composite/go/example)
- [Go Blog — Interfaces](https://go.dev/blog/laws-of-reflection) — background on Go's interface model