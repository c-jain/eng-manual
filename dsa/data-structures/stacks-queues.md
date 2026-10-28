---
Status: 🌳 Evergreen
Created: 2026-10-28
Last Updated: 2026-10-28
---

# Stacks and Queues

## Table of Contents

1. [What They Are and Why They Exist](#what-they-are-and-why-they-exist)
2. [Why "Stack" and Why "Queue"](#why-stack-and-why-queue)
3. [Stack Internals](#stack-internals)
4. [Stacks in Go](#stacks-in-go)
5. [Queue Internals](#queue-internals)
6. [Queues in Go](#queues-in-go)
7. [Circular Queue](#circular-queue)
8. [Deque](#deque)
9. [LRU Cache Structure](#lru-cache-structure)
10. [Go Template Code](#go-template-code)
11. [LeetCode Problems](#leetcode-problems)
12. [References](#references)

## What They Are and Why They Exist

A **stack** is a sequence with one restriction: you can only add to and remove from the same end. That end is called the **top**. The last thing pushed is the first thing popped — **LIFO** (Last-In, First-Out).

A **queue** is the mirror restriction: you add at one end (the **back**) and remove from the other end (the **front**). The first thing enqueued is the first thing dequeued — **FIFO** (First-In, First-Out).

Both exist because access order is not a decoration — it is often the entire point of the algorithm. Two questions clarify why:

**Why not just use an array or list?** You can. Every stack and queue is implemented on top of arrays or linked nodes. The abstraction exists because *hiding all other operations* is what makes an algorithm correct. If a function calls itself, the pending calls form a stack whether you think of it that way or not: the most recent unfinished call must resume first. If you let a caller peek at the middle, you introduce bugs and destroy the O(1) guarantees.

**Why do we need two different orderings?** Because the problems they solve are fundamentally different:

- Stacks match **nested / recursive** structure. Parentheses, function calls, undo history, DFS: each new item logically lives *inside* the previous, so the innermost must finish first. LIFO is the correct closing order for anything nested.
- Queues match **temporal fairness**. Print jobs, request handlers, BFS level exploration: whoever arrived first is served first. FIFO is the correct order for anything time-ordered.

### The Problems They Introduce

- **No random access.** By design, you cannot ask for element `i`. If your problem needs indexing, a stack or queue is the wrong tool.
- **Naive queue on an array is O(n) per dequeue.** Removing from the front shifts every remaining element left one slot. This is why real queues use a circular buffer or a linked list — covered below.
- **Bounded capacity trade-off.** A fixed-size ring buffer avoids reallocation cost but forces the caller to handle "full". A growable queue never rejects, but has amortised cost and can hold on to memory long after peak load.
- **Slice-backed stack retention.** A Go stack that pops by `s = s[:len(s)-1]` still keeps the popped element reachable through the backing array's capacity. If the element is a pointer or struct with pointers, the garbage collector cannot reclaim it. See [Stacks in Go](#stacks-in-go).

## Why "Stack" and Why "Queue"

**Stack** is a direct physical analogy: a stack of plates, books, or trays. You place a new one on top and take one off the top. The word entered CS through Alan Turing's 1946 report on the ACE computer, where he described a "stack of instructions" for subroutine calls. Dijkstra later formalised it for expression evaluation. The name reflects the geometry — vertical, one access point.

**Queue** comes from French *queue* meaning "tail" (originally Latin *coda*). A queue at a shop is a line where people join at the tail and leave from the head — exactly the data structure's behaviour. First used in a computing sense by Alan Turing and D. Michie in the late 1940s to describe waiting lines of jobs.

**Deque** ("deck") is short for **d**ouble-**e**nded **que**ue. Coined by Donald Knuth in *The Art of Computer Programming*. Both ends behave as a stack and a queue simultaneously.

## Stack Internals

A stack needs three operations, all O(1):

- `push(x)` — add `x` at the top
- `pop()` — remove and return the top element
- `peek()` — return the top element without removing

Two standard implementations exist. Both give O(1) amortised per operation; the choice is about memory behaviour, not asymptotics.

### Array-Backed (Dynamic Array)

The top of the stack is the end of the array. `push` is `append`; `pop` is truncate-by-one. Elements sit in contiguous memory, so pushes and pops are cache-friendly. The only cost is occasional reallocation when the backing array fills up — amortised O(1) with a 2× growth factor (see `dsa/data-structures/arrays.md`).

```
After push(A):     [ A ]
                     ↑ top

After push(B):     [ A | B ]
                         ↑ top

After push(C):     [ A | B | C ]
                             ↑ top

pop() returns C (the top); the array becomes [ A | B ] with top at B.
```

This is the default choice in Go. `[]T` with `append` and slice-shrink is idiomatic.

### Linked-List-Backed

Each element is a node with a value and a `next` pointer. The top is the head; `push` prepends, `pop` removes the head. No amortised analysis needed — every operation is genuinely O(1) with no reallocation. The cost is one allocation per push and one pointer dereference per access, both of which are cache-unfriendly.

```
Stack after push(A), push(B), push(C):

top → [C|●]──▶[B|●]──▶[A|●]──▶ nil
```

Use this only when you cannot afford the occasional O(n) copy of the array-backed version — for example, in real-time systems with hard latency bounds — or when nodes must be shared across structures.

## Stacks in Go

Go has no built-in stack type. The idiomatic implementation is a slice.

```go
var s []int
s = append(s, 1)          // push
s = append(s, 2)
top := s[len(s)-1]         // peek
s = s[:len(s)-1]           // pop (discard)
```

That is it. In most code this is enough. Wrap it in a struct when the type warrants a name or when you want to make the API explicit.

### The GC Retention Gotcha

`s = s[:len(s)-1]` moves the length back but leaves the popped value sitting in the backing array's capacity region. For `[]int`, this is harmless — it is a plain integer. But for `[]*User` or `[]struct{ payload []byte }`, that stale reference keeps the pointed-to memory alive.

```go
// Wrong — keeps popped element reachable through backing array
s = s[:len(s)-1]

// Right — zero the slot first, then shrink
var zero T
s[len(s)-1] = zero
s = s[:len(s)-1]
```

A senior reviewer will flag this on any stack of pointers or heap-heavy structs. For pure `int` / `bool` / small value types it does not matter.

## Queue Internals

A queue needs:

- `enqueue(x)` — add `x` at the back
- `dequeue()` — remove and return the front element
- `peek()` — return the front element without removing

The naive slice implementation is a trap.

### Why a Slice Alone Fails

```go
// The trap
q = append(q, x)   // enqueue — O(1) amortised, fine
head := q[0]        // peek — O(1), fine
q = q[1:]           // dequeue — looks O(1), is O(n) over time
```

`q = q[1:]` looks constant-time — and the individual call is. The problem is the backing array. Every `q[1:]` advances the slice pointer but never gives the memory back. After N enqueues and N dequeues, you have used O(N) memory even though the queue is empty. Worse, the backing array only compacts when a full copy happens on a growth event, so long-running programs leak.

There are two real fixes: a **circular buffer** or a **linked list**.

### Circular Buffer (Ring Buffer)

Allocate a fixed-size array and maintain two indices: `head` (front) and `tail` (back). Enqueue writes at `tail` and advances `tail`. Dequeue reads at `head` and advances `head`. Both indices are taken modulo the array length, so they wrap around.

```
Capacity 6 array. Enqueue A, B, C, D, then dequeue twice, then enqueue E, F:

Step 1 — enqueue A B C D:
  [A][B][C][D][ ][ ]
   ↑           ↑
  head        tail        size = 4

Step 2 — dequeue twice:
  [.][.][C][D][ ][ ]
         ↑     ↑
        head  tail        size = 2

Step 3 — enqueue E, F:
  [.][.][C][D][E][F]
         ↑           ↑ (tail wrapped to index 0 after F)
        head       tail  size = 4

Step 4 — enqueue G:
  [G][.][C][D][E][F]
         ↑  ↑
        head tail(=1)     size = 5
```

Space stays bounded to the capacity. Every operation is genuinely O(1), no amortised trickery. When the buffer fills, grow it: allocate double the capacity, copy elements in logical order (head to tail), reset `head = 0` and `tail = size`.

This is what `container/list`-free production code uses. It is also how ring buffers in kernel drivers and message queues work.

### Linked List

Head and tail pointers into a singly linked list. Enqueue appends at tail; dequeue removes from head.

```
head → [A|●]──▶[B|●]──▶[C|●]──▶ nil ← tail
```

Every operation is O(1) with no amortisation. The overhead is one allocation per enqueue and pointer chasing on dequeue. Use when the queue size is highly unpredictable and you never want a reallocation pause, or when you also need to splice queues together (concatenate two linked lists is O(1)).

## Queues in Go

There is no built-in queue. Options in order of preference:

1. **Roll your own circular buffer.** ~40 lines of Go. Fastest and most memory-efficient. See the template code.
2. **`container/list`.** Doubly linked list. Use when you also need arbitrary removal. Values are boxed as `any`, which costs an allocation per push for small types.
3. **Channels as queues.** A buffered channel `make(chan T, capacity)` is a bounded thread-safe queue. Use this only when you actually want the concurrency semantics — a channel is far more expensive than a slice for single-goroutine work.

```go
// container/list example — clean but boxes into any
import "container/list"

q := list.New()
q.PushBack(1)               // enqueue
v := q.Front().Value.(int)   // peek + unbox
q.Remove(q.Front())          // dequeue
```

## Circular Queue

The ring buffer from the previous section, but with a fixed capacity that never grows. Enqueue on full returns false — the caller decides to block, drop, or overwrite. This matches LeetCode #622 and any bounded producer-consumer setting.

### Full vs Empty Ambiguity

Both `head == tail` states — completely empty and completely full — look identical if you only track the two indices. Two standard resolutions:

- **Track size explicitly.** Keep a `size int`. `size == 0` is empty, `size == cap` is full. One extra field, easiest to reason about; this is what the template and LC #622 solution use.
- **Leave one slot always empty.** Full is defined as `(tail + 1) % cap == head`. Saves the size field at the cost of one unusable slot. Historical (register-tight embedded code) — skip it unless you have a reason.

## Deque

A **deque** (double-ended queue) allows push and pop at both ends, all in O(1). It generalises the stack (top = one end) and the queue (front and back).

```
                 PushFront ─┐         ┌─ PushBack
                            ▼         ▼
                         ┌───┬───┬───┬───┐
                    ...  │ A │ B │ C │ D │  ...
                         └───┴───┴───┴───┘
                            ▲         ▲
                 PopFront ──┘         └── PopBack
```

Two implementations, same trade-off as the queue: doubly linked list (true O(1) at both ends, one allocation per element) or circular buffer (cache-friendly, no per-element allocation, occasional grow). Go's `container/list` is the doubly-linked version and is the default when you do not want to write your own.

## LRU Cache Structure

A least-recently-used (LRU) cache remembers the last N distinct keys accessed and evicts the one that has been idle longest. It shows up in every operating system page cache, database buffer pool, and CPU cache-line replacement policy.

The interview question is not "what is an LRU cache" — it is "give me `Get` and `Put` in O(1)". Neither a hash map nor a doubly linked list alone can do this. Together they can.

### The Two-Structure Design

- **Doubly linked list** ordered by recency: front = most recently used, back = least recently used. Moving a node to the front and removing the back are both O(1) *if you have a pointer to the node already*.
- **Hash map** from key to that node's pointer. Gives O(1) lookup of any node without walking the list.

```
            MRU end                             LRU end
               │                                   │
               ▼                                   ▼
   head ──▶ [K3|V3] ──▶ [K1|V1] ──▶ [K5|V5] ──▶ [K2|V2] ──▶ tail
               ▲           ▲           ▲           ▲
               │           │           │           │
            map[K3]     map[K1]     map[K5]     map[K2]
```

`Get(K5)`: look up in the map (O(1)), unlink the node from its current position, insert at head. `Put(K, V)`:

- If K exists: update value, move node to head.
- If K is new and size < capacity: create a node at head, add to the map.
- If K is new and size == capacity: remove the node at tail (LRU), delete its key from the map, then add the new node at head.

Every step is O(1). The trick is that the map stores *pointers into* the list, not values — that is what makes list surgery constant-time.

### Sentinel Head and Tail

The idiomatic implementation uses two dummy nodes: a `head` sentinel and a `tail` sentinel. Real nodes live between them. This kills every "if node is at the boundary" special case — every real node always has a non-nil `prev` and `next`. The template code below uses this pattern.

## Go Template Code

Generic stack, ring-buffer queue, and a deque wrapper around `container/list`. All verified.

```go
package ds

import "container/list"

// -------------------- Stack --------------------

type Stack[T any] struct {
    data []T
}

func (s *Stack[T]) Push(v T)    { s.data = append(s.data, v) }
func (s *Stack[T]) Len() int    { return len(s.data) }
func (s *Stack[T]) Empty() bool { return len(s.data) == 0 }

func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    n := len(s.data)
    if n == 0 {
        return zero, false
    }
    v := s.data[n-1]
    s.data[n-1] = zero // GC hygiene for pointer element types
    s.data = s.data[:n-1]
    return v, true
}

func (s *Stack[T]) Peek() (T, bool) {
    var zero T
    if len(s.data) == 0 {
        return zero, false
    }
    return s.data[len(s.data)-1], true
}

// -------------------- Queue (growable ring buffer) --------------------

type Queue[T any] struct {
    buf        []T
    head, tail int
    size       int
}

func NewQueue[T any](initCap int) *Queue[T] {
    if initCap < 1 {
        initCap = 1
    }
    return &Queue[T]{buf: make([]T, initCap)}
}

func (q *Queue[T]) Len() int    { return q.size }
func (q *Queue[T]) Empty() bool { return q.size == 0 }

func (q *Queue[T]) Enqueue(v T) {
    if q.size == len(q.buf) {
        q.grow()
    }
    q.buf[q.tail] = v
    q.tail = (q.tail + 1) % len(q.buf)
    q.size++
}

func (q *Queue[T]) Dequeue() (T, bool) {
    var zero T
    if q.size == 0 {
        return zero, false
    }
    v := q.buf[q.head]
    q.buf[q.head] = zero
    q.head = (q.head + 1) % len(q.buf)
    q.size--
    return v, true
}

func (q *Queue[T]) grow() {
    newBuf := make([]T, len(q.buf)*2)
    for i := 0; i < q.size; i++ {
        newBuf[i] = q.buf[(q.head+i)%len(q.buf)]
    }
    q.buf = newBuf
    q.head = 0
    q.tail = q.size
}

// -------------------- Deque (wraps container/list) --------------------

type Deque[T any] struct {
    l *list.List
}

func NewDeque[T any]() *Deque[T] { return &Deque[T]{l: list.New()} }

func (d *Deque[T]) PushFront(v T) { d.l.PushFront(v) }
func (d *Deque[T]) PushBack(v T)  { d.l.PushBack(v) }
func (d *Deque[T]) Len() int      { return d.l.Len() }

func (d *Deque[T]) PopFront() (T, bool) {
    var zero T
    e := d.l.Front()
    if e == nil {
        return zero, false
    }
    d.l.Remove(e)
    return e.Value.(T), true
}

func (d *Deque[T]) PopBack() (T, bool) {
    var zero T
    e := d.l.Back()
    if e == nil {
        return zero, false
    }
    d.l.Remove(e)
    return e.Value.(T), true
}
```

## LeetCode Problems

### 1. Valid Parentheses — [#20 (Easy)](https://leetcode.com/problems/valid-parentheses/)

Given a string of `()[]{}`, decide whether every opener has a matching closer in the correct order.

<details>
<summary>Brute Force</summary>

Repeatedly scan the string looking for an adjacent matched pair (`()`, `[]`, or `{}`) and splice it out. Keep going until a full pass finds no pair. If the string is now empty, the input was valid; otherwise invalid.

Time: O(n²) worst case — up to n/2 removal passes, each a linear scan. Space: O(n) for the mutating string.

The waste: re-scanning everything after each removal re-does work. A single left-to-right stack pass identifies the "innermost unclosed opener" at every closer in one shot.
</details>

<details>
<summary>Hint 1</summary>
Every closing bracket must match the most recently opened, still-unclosed bracket. Which access order gives you the most recently added item first?
</details>

<details>
<summary>Hint 2</summary>
Push openers as you see them. On a closer, pop and verify the popped opener matches. At the end, the stack must be empty.
</details>

<details>
<summary>Solution (Go)</summary>

```go
func isValid(s string) bool {
    pairs := map[rune]rune{')': '(', ']': '[', '}': '{'}
    stack := make([]rune, 0, len(s))
    for _, c := range s {
        switch c {
        case '(', '[', '{':
            stack = append(stack, c)
        case ')', ']', '}':
            if len(stack) == 0 || stack[len(stack)-1] != pairs[c] {
                return false
            }
            stack = stack[:len(stack)-1]
        }
    }
    return len(stack) == 0
}
```

**Why this works:** LIFO matches the "innermost bracket must close first" rule exactly. O(n) time, O(n) space in the worst case (all openers).
</details>

### 2. Min Stack — [#155 (Medium)](https://leetcode.com/problems/min-stack/)

Design a stack that also returns its current minimum in O(1).

<details>
<summary>Brute Force</summary>

Store only the values in the stack. For `GetMin`, scan the entire stack and return the smallest.

Time: Push/Pop/Top O(1), GetMin O(n). Space: O(n).

The waste: `GetMin` repeatedly rediscovers information that was already known at push time. The problem demands all four operations in O(1), so this approach fails the stated contract — not just a constant-factor loss.
</details>

<details>
<summary>Hint 1</summary>
After every push, the minimum can either stay the same or drop to the new value. After every pop, the minimum returns to what it was before the last push.
</details>

<details>
<summary>Hint 2</summary>
Keep a second stack, parallel to the first, storing the running minimum after each push. Popping both together restores the previous minimum for free.
</details>

<details>
<summary>Solution (Go)</summary>

```go
type MinStack struct {
    data []int
    mins []int
}

func Constructor() MinStack { return MinStack{} }

func (s *MinStack) Push(x int) {
    s.data = append(s.data, x)
    if len(s.mins) == 0 || x <= s.mins[len(s.mins)-1] {
        s.mins = append(s.mins, x)
    } else {
        s.mins = append(s.mins, s.mins[len(s.mins)-1])
    }
}

func (s *MinStack) Pop() {
    n := len(s.data)
    s.data = s.data[:n-1]
    s.mins = s.mins[:n-1]
}

func (s *MinStack) Top() int    { return s.data[len(s.data)-1] }
func (s *MinStack) GetMin() int { return s.mins[len(s.mins)-1] }
```

**Why this works:** The parallel `mins` stack encodes the entire history of "what was the minimum when this element was pushed". Popping restores state automatically. All operations O(1).

**Space optimisation follow-up:** an interviewer may ask to avoid duplicating minimums. Store `(value, count)` pairs in `mins` and increment the count when the same minimum is pushed again — worth mentioning but not defaulting to, since the code is fiddlier.
</details>

### 3. Implement Queue using Stacks — [#232 (Easy)](https://leetcode.com/problems/implement-queue-using-stacks/)

Build a FIFO queue using only two LIFO stacks.

<details>
<summary>Brute Force</summary>

Keep a single input stack. On `Push`, append. On `Pop`/`Peek`, pop every element into a temporary stack, read or remove the top of the temporary stack (which is the queue's front), then push everything back onto the input stack.

Time: Push O(1), Pop/Peek O(n) per call. Space: O(n).

The waste: every `Pop` shuffles the entire data set back and forth, even though only one element at the bottom was needed. The optimised version keeps the already-reversed elements in a second stack instead of shoving them back.
</details>

<details>
<summary>Hint 1</summary>
Reversing a stack gives you the opposite order — which is exactly what a queue needs. One reversal per element is too expensive, so think about *when* to reverse.
</details>

<details>
<summary>Hint 2</summary>
Use an `in` stack for pushes and an `out` stack for pops. Only transfer from `in` to `out` when `out` is empty. Each element gets moved between stacks at most once, giving amortised O(1) per operation.
</details>

<details>
<summary>Solution (Go)</summary>

```go
type MyQueue struct {
    in, out []int
}

func Constructor() MyQueue { return MyQueue{} }

func (q *MyQueue) Push(x int) { q.in = append(q.in, x) }

func (q *MyQueue) shift() {
    if len(q.out) == 0 {
        for len(q.in) > 0 {
            n := len(q.in) - 1
            q.out = append(q.out, q.in[n])
            q.in = q.in[:n]
        }
    }
}

func (q *MyQueue) Pop() int {
    q.shift()
    n := len(q.out) - 1
    v := q.out[n]
    q.out = q.out[:n]
    return v
}

func (q *MyQueue) Peek() int {
    q.shift()
    return q.out[len(q.out)-1]
}

func (q *MyQueue) Empty() bool { return len(q.in) == 0 && len(q.out) == 0 }
```

**Why this works:** Transferring the `in` stack into `out` reverses its order — the oldest element now sits on top of `out`, ready to pop. Because we only shift when `out` empties, each element is moved twice total (in, then out), giving amortised O(1). Worst-case single-op is O(n) — worth calling out to the interviewer.
</details>

<details>
<summary>Follow-Up</summary>

Official LeetCode follow-up: *Can you implement the queue such that each operation is amortized O(1) time complexity?*

The two-stack lazy-transfer solution above already satisfies this. The amortisation argument: assign each element two "credits" when it is pushed onto `in`. Over its lifetime, that element pays for exactly two moves — once from `in` to `out` during a `shift()`, and once off `out` during its own `Pop`. No element ever moves a third time. So n operations do at most 2n stack moves, giving O(1) amortised per op, even though a single `Pop` that triggers a shift can cost O(n).
</details>

### 4. Implement Stack using Queues — [#225 (Easy)](https://leetcode.com/problems/implement-stack-using-queues/)

The reverse: build a LIFO stack from FIFO queues. The point of the problem is to show you understand the trade-off in the other direction.

<details>
<summary>Brute Force</summary>

Use two queues. Push: enqueue into `q1`. Pop: move all but the last element from `q1` into `q2`, dequeue and return the remaining element in `q1`, then swap `q1` and `q2`.

Time: Push O(1), Pop/Top O(n). Space: O(n) across both queues.

The waste: the whole second queue exists only to hold the elements we are about to put back. Doing the rotation inside a single queue on push removes the need for `q2` entirely — which is also what the follow-up asks for.
</details>

<details>
<summary>Hint 1</summary>
Every push needs to leave the most recent element at the front of the queue so a dequeue returns it. Can you rearrange the queue on push instead of pop?
</details>

<details>
<summary>Hint 2</summary>
On push: enqueue the new element, then rotate the queue by dequeuing and re-enqueuing every earlier element. The new element is now at the front. Push becomes O(n), pop stays O(1). A single queue is enough.
</details>

<details>
<summary>Solution (Go)</summary>

```go
type MyStack struct {
    q []int
}

func Constructor() MyStack { return MyStack{} }

func (s *MyStack) Push(x int) {
    s.q = append(s.q, x)
    // Rotate: move every element in front of x to the back.
    for i := 0; i < len(s.q)-1; i++ {
        s.q = append(s.q, s.q[0])
        s.q = s.q[1:]
    }
}

func (s *MyStack) Pop() int {
    v := s.q[0]
    s.q = s.q[1:]
    return v
}

func (s *MyStack) Top() int   { return s.q[0] }
func (s *MyStack) Empty() bool { return len(s.q) == 0 }
```

**Why this works:** Push does the rearrangement work up front so that the queue's natural FIFO order already matches LIFO for the outside caller. Push is O(n), pop / top are O(1). Compare with #232 — same problem class, opposite cost distribution.
</details>

<details>
<summary>Follow-Up</summary>

Official LeetCode follow-up: *Can you implement the stack using only one queue?*

The solution above already uses one queue. The rotation inside `Push` — a sequence of `len(q)-1` dequeue-then-enqueue pairs — is the only way a pure queue can bring the just-added element to the front, since a queue has no random access or back-removal. No peeking at arbitrary indices, no secondary structure.
</details>

### 5. Evaluate Reverse Polish Notation — [#150 (Medium)](https://leetcode.com/problems/evaluate-reverse-polish-notation/)

Evaluate an expression in postfix (RPN) form, e.g. `["2", "1", "+", "3", "*"]` → 9.

<details>
<summary>Brute Force</summary>

Scan the token list for the first operator. Apply it to the two tokens immediately before it, replace those three tokens with the computed result, and repeat the scan from the beginning. Stop when only one token remains.

Time: O(n²) — each reduction removes two tokens but re-scans from the start. Space: O(n).

The waste: the repeated restart. Postfix was designed so operators appear in the right order to apply greedily left-to-right once; a stack of pending numbers captures that without any back-tracking.
</details>

<details>
<summary>Hint 1</summary>
In postfix, every operator applies to the two most recently produced values. What data structure gives you "the two most recent items" in O(1)?
</details>

<details>
<summary>Hint 2</summary>
Scan left to right. Push numbers. On an operator, pop the top two (right operand first, then left), compute, push the result. The final stack has exactly one value — the answer.
</details>

<details>
<summary>Solution (Go)</summary>

```go
import "strconv"

func evalRPN(tokens []string) int {
    stack := make([]int, 0, len(tokens))
    for _, t := range tokens {
        switch t {
        case "+", "-", "*", "/":
            b := stack[len(stack)-1]
            a := stack[len(stack)-2]
            stack = stack[:len(stack)-2]
            var r int
            switch t {
            case "+":
                r = a + b
            case "-":
                r = a - b
            case "*":
                r = a * b
            case "/":
                r = a / b
            }
            stack = append(stack, r)
        default:
            n, _ := strconv.Atoi(t)
            stack = append(stack, n)
        }
    }
    return stack[0]
}
```

**Why this works:** Postfix notation was invented precisely to remove parentheses by using stack-friendly evaluation order. Each token is visited once, so O(n) time and O(n) space. Watch out for operand order on non-commutative operators (`-` and `/`): the top of the stack is the *right* operand.
</details>

### 6. Design Circular Queue — [#622 (Medium)](https://leetcode.com/problems/design-circular-queue/)

Implement a fixed-capacity ring buffer with `EnQueue`, `DeQueue`, `Front`, `Rear`, `IsEmpty`, `IsFull`.

<details>
<summary>Brute Force</summary>

Use a fixed-size array of length `k`. `EnQueue` appends at the next free index (track a size counter). `DeQueue` removes the element at index 0 by shifting every other element one position left.

Time: EnQueue O(1), DeQueue O(k), Front/Rear O(1). Space: O(k).

The waste: the leftward shift re-touches every live element on every dequeue, purely to keep the front at index 0. Letting the "front" position itself move (an index that wraps around) removes the shift entirely and gets DeQueue down to O(1).
</details>

<details>
<summary>Hint 1</summary>
Track two indices — front and back — and wrap them modulo the capacity so they reuse array slots once dequeued.
</details>

<details>
<summary>Hint 2</summary>
`head == tail` is ambiguous between empty and full unless you also track the current size or waste one slot. Store the size explicitly and the full / empty checks reduce to `size == 0` and `size == cap`.
</details>

<details>
<summary>Solution (Go)</summary>

```go
type MyCircularQueue struct {
    buf        []int
    head, tail int
    size, cap  int
}

func Constructor(k int) MyCircularQueue {
    return MyCircularQueue{buf: make([]int, k), cap: k}
}

func (c *MyCircularQueue) EnQueue(v int) bool {
    if c.size == c.cap {
        return false
    }
    c.buf[c.tail] = v
    c.tail = (c.tail + 1) % c.cap
    c.size++
    return true
}

func (c *MyCircularQueue) DeQueue() bool {
    if c.size == 0 {
        return false
    }
    c.head = (c.head + 1) % c.cap
    c.size--
    return true
}

func (c *MyCircularQueue) Front() int {
    if c.size == 0 {
        return -1
    }
    return c.buf[c.head]
}

func (c *MyCircularQueue) Rear() int {
    if c.size == 0 {
        return -1
    }
    return c.buf[(c.tail-1+c.cap)%c.cap]
}

func (c *MyCircularQueue) IsEmpty() bool { return c.size == 0 }
func (c *MyCircularQueue) IsFull() bool  { return c.size == c.cap }
```

**Why this works:** `Rear()` uses `(tail - 1 + cap) % cap` so the subtraction stays non-negative when `tail == 0`. Every operation is O(1) with fixed O(k) space. This is the same skeleton as the growable queue in the template code, minus the `grow` call.
</details>

### 7. Decode String — [#394 (Medium)](https://leetcode.com/problems/decode-string/)

Given an encoded string like `3[a2[c]]`, return its decoded form `accaccacc`. The pattern is `k[encoded]` where `k` is a positive integer and `encoded` may itself contain further `k[...]` segments nested to any depth.

<details>
<summary>Brute Force</summary>

Scan left to right. When you hit a digit followed by `[`, find the matching `]` by counting bracket depth, recursively decode the substring between them, repeat it `k` times, splice back in. Repeat until no brackets remain.

Time: O(n · d) where `d` is the maximum nesting depth — each outer repetition triggers a fresh scan over the already-decoded content on each pass. Space: O(n) for recursion plus the growing output. Correct, but the repeated scans and string splicing are wasteful.
</details>

<details>
<summary>Hint 1</summary>
Every time you enter a new `[`, you need to remember two things: how many times the inner result will repeat, and what string you had built up before the bracket. On `]`, you combine them.
</details>

<details>
<summary>Hint 2</summary>
Two parallel stacks — one for repeat counts, one for the string accumulated so far — let you suspend and resume the outer context around each nested segment. One linear pass, no recursion needed.
</details>

<details>
<summary>Solution (Go)</summary>

```go
import "strings"

func decodeString(s string) string {
    countStack := []int{}
    strStack := []string{}
    cur := strings.Builder{}
    k := 0

    for _, c := range s {
        switch {
        case c >= '0' && c <= '9':
            k = k*10 + int(c-'0') // multi-digit counts like "12[a]"
        case c == '[':
            countStack = append(countStack, k)
            strStack = append(strStack, cur.String())
            k = 0
            cur.Reset()
        case c == ']':
            reps := countStack[len(countStack)-1]
            prev := strStack[len(strStack)-1]
            countStack = countStack[:len(countStack)-1]
            strStack = strStack[:len(strStack)-1]
            repeated := strings.Repeat(cur.String(), reps)
            cur.Reset()
            cur.WriteString(prev)
            cur.WriteString(repeated)
        default:
            cur.WriteRune(c)
        }
    }
    return cur.String()
}
```

**Why this works:** Opening `[` is "save my state and start fresh for the inside"; closing `]` is "I'm done with the inside, multiply it and stitch it back". The two stacks hold exactly that saved state. Each character is processed once; the only non-constant work per `]` is the string repeat, which is unavoidable because it is part of the output. Time: O(n · maxReps) where the product bounds output size. Space: O(d) for the stacks, where `d` is nesting depth.

The multi-digit parsing (`k = k*10 + ...`) matters — missing it silently breaks `12[a]`.
</details>

## References

- [Turing's ACE report (1946)](https://www.alanturing.net/turing_archive/archive/p/p01/p01.php) — original mention of a "stack of instructions" for subroutines
- Knuth, *The Art of Computer Programming*, Vol. 1, §2.2 — the definitive treatment of stacks, queues, and deques
- [Go `container/list` docs](https://pkg.go.dev/container/list) — Go's built-in doubly linked list, useful as a deque
- [Go `container/ring` docs](https://pkg.go.dev/container/ring) — circular list; different from a ring buffer, rarely used
- [LeetCode Explore: Queue & Stack](https://leetcode.com/explore/learn/card/queue-stack/) — LeetCode's introductory card
- `dsa/data-structures/arrays.md` — slice internals, growth policy, amortised append
- `dsa/data-structures/linked-lists.md` — node-based implementations (planned)
- `dsa/patterns/monotonic-stack.md` — next greater element, histogram area, daily temperatures (planned)
- `dsa/patterns/prefix-sum.md` — monotonic deque case study (LC 862)
- `dsa/fundamentals/complexity-analysis.md` — amortised analysis derivation for growable buffers