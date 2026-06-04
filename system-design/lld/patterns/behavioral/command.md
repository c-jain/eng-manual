# Command Pattern

## Table of Contents

- [What Is the Command Pattern](#what-is-the-command-pattern)
- [Why It Exists](#why-it-exists)
- [The Four Roles](#the-four-roles)
- [Class Diagram](#class-diagram)
- [Go Implementation](#go-implementation)
- [Macro Commands](#macro-commands)
- [Go-Idiomatic Shortcut](#go-idiomatic-shortcut)
- [When to Use and When Not To](#when-to-use-and-when-not-to)
- [Real-World Appearances](#real-world-appearances)
- [Command vs. Strategy](#command-vs-strategy)
- [Trade-offs](#trade-offs)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [How to Remember It](#how-to-remember-it)
- [References](#references)

---

## What Is the Command Pattern

The Command pattern encapsulates a request — its receiver, the method to call, and any arguments — as a standalone object. Instead of a caller invoking a method directly on a receiver, it creates a command object and hands it to an invoker.

**Why it is called that:** A "command" in natural language is a discrete, executable instruction. The pattern promotes method calls to first-class objects — they can be stored, passed around, queued, logged, and executed later.

---

## Why It Exists

Direct method calls have three limitations:

- **Coupling** — the caller must know the receiver and its interface at the point of invocation
- **No history** — once called, the invocation is gone; you cannot undo it or replay it
- **No mobility** — you cannot schedule it, queue it, serialize it, or retry it

Command solves all three by turning the invocation into a value that can be passed to whoever will actually trigger it, whenever that is appropriate.

**Problems the pattern itself brings:**

- Adds classes — one ConcreteCommand per operation
- Simple operations that never need undo or queuing end up with unnecessary indirection
- Undo requires the command to capture pre-execution state, which means careful design of what to snapshot inside `Execute()`

---

## The Four Roles

- **Command** — interface declaring `Execute()` and optionally `Undo()`
- **ConcreteCommand** — implements Command; holds a reference to the Receiver and captures all arguments and pre-execute state needed for undo
- **Receiver** — the object that knows how to perform the actual work; it is called by ConcreteCommand
- **Invoker** — holds a Command reference and calls `Execute()` at the right time; it has no knowledge of what the command does or who the receiver is
- **Client** — creates ConcreteCommands, wires them to a Receiver, and hands them to an Invoker

---

## Class Diagram

```
Client
  |
  | creates and wires
  v
ConcreteCommand ----implements----> Command (interface)
  |                                   Execute()
  | holds reference                   Undo()
  v
Receiver
  DoWork()

Invoker
  | holds Command
  | calls Execute() / Undo()
  v
Command (interface)
```

The Invoker never knows about ConcreteCommand or Receiver — it only knows the Command interface.

---

## Go Implementation

A text editor with insert and delete operations, an undo stack as the Invoker.

```go
package command

import "fmt"

// Receiver — knows how to actually do the work

type TextEditor struct {
    content string
}

func (e *TextEditor) InsertText(text string) {
    e.content += text
    fmt.Printf("inserted %q  →  content: %q\n", text, e.content)
}

func (e *TextEditor) DeleteText(n int) {
    if n > len(e.content) {
        n = len(e.content)
    }
    e.content = e.content[:len(e.content)-n]
    fmt.Printf("deleted %d chars  →  content: %q\n", n, e.content)
}

func (e *TextEditor) Content() string { return e.content }

// Command interface

type Command interface {
    Execute()
    Undo()
}

// ConcreteCommand: insert

type InsertCommand struct {
    editor *TextEditor
    text   string
}

func NewInsertCommand(editor *TextEditor, text string) *InsertCommand {
    return &InsertCommand{editor: editor, text: text}
}

func (c *InsertCommand) Execute() { c.editor.InsertText(c.text) }
func (c *InsertCommand) Undo()    { c.editor.DeleteText(len(c.text)) }

// ConcreteCommand: delete
// Must capture the deleted text in Execute() so Undo() can restore it.

type DeleteCommand struct {
    editor  *TextEditor
    n       int
    deleted string // captured during Execute for Undo
}

func NewDeleteCommand(editor *TextEditor, n int) *DeleteCommand {
    return &DeleteCommand{editor: editor, n: n}
}

func (c *DeleteCommand) Execute() {
    n := c.n
    if n > len(c.editor.content) {
        n = len(c.editor.content)
    }
    // Capture state before modifying
    c.deleted = c.editor.content[len(c.editor.content)-n:]
    c.editor.DeleteText(c.n)
}

func (c *DeleteCommand) Undo() { c.editor.InsertText(c.deleted) }

// Invoker: history stack

type CommandHistory struct {
    history []Command
}

func (h *CommandHistory) Execute(cmd Command) {
    cmd.Execute()
    h.history = append(h.history, cmd)
}

func (h *CommandHistory) Undo() {
    if len(h.history) == 0 {
        fmt.Println("nothing to undo")
        return
    }
    last := h.history[len(h.history)-1]
    h.history = h.history[:len(h.history)-1]
    last.Undo()
}
```

**Client wiring:**

```go
func main() {
    editor := &TextEditor{}
    history := &CommandHistory{}

    history.Execute(NewInsertCommand(editor, "Hello"))
    history.Execute(NewInsertCommand(editor, ", world"))
    // content: "Hello, world"

    history.Undo()
    // content: "Hello"

    history.Undo()
    // content: ""
}
```

---

## Macro Commands

A MacroCommand is a Command that holds a slice of Commands and executes them all in sequence. Undo runs them in reverse.

```go
type MacroCommand struct {
    commands []Command
}

func (m *MacroCommand) Add(cmd Command) {
    m.commands = append(m.commands, cmd)
}

func (m *MacroCommand) Execute() {
    for _, cmd := range m.commands {
        cmd.Execute()
    }
}

func (m *MacroCommand) Undo() {
    for i := len(m.commands) - 1; i >= 0; i-- {
        m.commands[i].Undo()
    }
}
```

This is the Composite pattern applied to commands. Every Command, whether simple or macro, looks identical to the Invoker.

---

## Go-Idiomatic Shortcut

For operations that will never need undo or queuing, a plain function value is a lightweight command:

```go
type Task func()

queue := []Task{
    func() { fmt.Println("send email") },
    func() { fmt.Println("update database") },
}

for _, t := range queue {
    t()
}
```

Reach for the full struct-based pattern when:
- Undo state must be captured inside the command
- The command must be serialisable (e.g., stored in a database or sent over the wire)
- Commands need to be composed into macros

---

## When to Use and When Not To

**Use it when:**

- Undo/redo is a feature requirement (editors, IDEs, drawing tools)
- Operations need to be queued, scheduled, or retried asynchronously
- You need an audit log — each command object can log itself before and after execution
- You need transactional rollback — execute a sequence, and on failure call Undo in reverse
- Operations need to be bundled into macros

**Do not use it when:**

- The operation is simple and fire-and-forget — a direct method call is clearer
- No undo, queuing, or logging requirement exists — you are adding indirection for no benefit
- A function value (closure) already captures everything needed — the struct is overhead

---

## Real-World Appearances

- **Database transactions** — each SQL statement is a command; ROLLBACK is the undo stack
- **Job queues** (Asynq, Sidekiq, Celery) — jobs are serialised commands dispatched to worker pools
- **CLI frameworks** (Cobra) — each subcommand is a Command object with a Run function
- **GUI applications** — every user action (draw, resize, move) is pushed to a command history
- **HTTP retry middleware** — the HTTP request is a command that can be re-executed on failure
- **Infrastructure-as-code** (Terraform) — a plan is a command; apply executes it; destroy is undo

---

## Command vs. Strategy

These two are often confused because both wrap behaviour in an object.

| | Command | Strategy |
|---|---|---|
| Encapsulates | A request — *what to do and when* | An algorithm — *how to do something* |
| Intent | Decouple invocation from execution; enable undo/queue | Swap interchangeable behaviours at runtime |
| Statefulness | Commands often capture pre-execute state for undo | Strategies are typically stateless |
| Invoker | An explicit Invoker role exists | No Invoker — the context calls the strategy directly |

**Rule of thumb:** Strategy answers *how*; Command answers *what and when*.

---

## Trade-offs

- **Undo accuracy** depends on faithfully snapshotting pre-execute state inside `Execute()` — if you miss something, undo is broken
- **Command proliferation** — one class per operation; in a large system this can be dozens of types. Mitigated by using function values for simple cases.
- **Serialisation complexity** — serialising commands (for persistent job queues or distributed undo) requires every command to be serialisable, which constrains what can be captured as state
- **Ordering matters for undo** — undo must run in strict reverse order; any deviation breaks correctness

---

## Interview Cheat Sheet

**Strong signal phrases:**

- "Command turns a method call into a first-class object — that's what enables undo, queuing, and audit logging without modifying the receiver"
- "The Invoker is deliberately blind to what the command does — it only knows Execute() and Undo()"
- "Undo requires capturing pre-execute state inside Execute() itself — you cannot reconstruct it after the fact"
- "In Go I'd use a closure for simple fire-and-forget cases; the full struct-based pattern when I need to store undo state inside the command"
- "MacroCommand composes commands using the Composite pattern — the Invoker can't tell the difference"

**Red flags:**

- Confusing Command with Strategy — Strategy swaps *how*; Command encapsulates *what and when*
- Not mentioning the Invoker role — reducing the pattern to "just wrap a function in a struct"
- Forgetting that `Undo()` depends on state captured during `Execute()` — attempting undo with no snapshot
- Proposing the full pattern for a simple one-shot operation with no undo or queuing requirement

---

## How to Remember It

**Analogy: A Restaurant Order Slip**

- You (client) write an order (ConcreteCommand) specifying what you want (Receiver = kitchen)
- You hand it to the waiter (Invoker)
- The waiter doesn't cook — they carry and execute the slip
- A mistake? The slip (command) knows what was ordered, so it can be corrected (undo)

**The four roles in one sentence:**
_The Client creates a Command that knows its Receiver; the Invoker fires it without knowing either._

**When to use, one word each:**
- Undo → Command
- Swap algorithm → Strategy
- Queue work → Command
- Notify observers → Observer

---

## References

- [Refactoring.Guru — Command Pattern](https://refactoring.guru/design-patterns/command)
- GoF, *Design Patterns*, pp. 233–242 — Command
- [Go Patterns — Command](https://github.com/tmrts/go-patterns/blob/master/behavioral/command.md)