---
Status: 🌳 Evergreen
Created: 2026-10-26
Last Updated: 2026-10-26
---

# Linked Lists

## Table of Contents

1. [What It Is and Why It Exists](#what-it-is-and-why-it-exists)
2. [Singly Linked List](#singly-linked-list)
3. [Doubly Linked List](#doubly-linked-list)
4. [Circular Linked List](#circular-linked-list)
5. [Dummy Head Node Technique](#dummy-head-node-technique)
6. [Linked Lists in Go](#linked-lists-in-go)
7. [Core Operations](#core-operations)
8. [Go Template Code](#go-template-code)
9. [Where JavaScript Differs](#where-javascript-differs)
10. [LeetCode Problems](#leetcode-problems)
11. [References](#references)

## What It Is and Why It Exists

A linked list is a linear sequence of nodes where each node holds a value and a **pointer to the next node**. Unlike an array, the nodes are not contiguous in memory; the list is held together by the pointers alone.

The reason it exists is the exact weakness of arrays: insertion and deletion in the middle of an array cost O(n) because everything after the target must shift. A linked list decouples logical order from physical order. To insert a new node between two existing nodes, you only rewire two pointers. Nothing shifts, nothing reallocates. That is the trade at the heart of the structure: you give up O(1) random access, and in return you buy O(1) structural edits at any position you already hold a reference to.

The name is literal at both words. "List" is the abstract data type (ordered sequence). "Linked" describes the implementation: elements joined by explicit pointers rather than by adjacency in memory. The idea goes back to Newell, Shaw, and Simon's Information Processing Language (1955–56); Lisp (1958) made it the primitive data structure of the whole language via the `cons` cell.

### The Problems It Introduces

The pointer-per-node design brings four defining costs:

- **No random access.** Reaching index `i` costs O(i) because you must walk the chain. `list[500]` does not exist as an operation.
- **Cache-unfriendly.** Nodes are allocated wherever the heap has room. Traversal is a sequence of pointer dereferences into arbitrary addresses, each risking a cache miss. Iterating a linked list is often 5x to 10x slower than iterating a slice of the same length, at the same asymptotic complexity.
- **Memory overhead.** Each node stores at least one pointer (8 bytes on a 64-bit machine) alongside its value.
- **Pointer bugs.** Reverse, delete, or splice mid-list and you must hold the right references in the right order. Lose a `next` before saving it and the tail of the list becomes unreachable.

## Singly Linked List

The minimum viable form. Each node holds a value and one `Next` pointer. The list is identified by its `head`. The last node's `Next` is `nil`.

```
head
 |
 v
[1|*] -> [2|*] -> [3|*] -> [4|nil]
```

Cheap: prepend to head (O(1)), append if you track a `tail` (O(1)), delete the head (O(1)).
Expensive: access by index (O(n)), delete the tail (O(n), you need the node *before* the tail), delete a node given only that node (O(n) to find its predecessor).

Default choice. Reach for a doubly linked list only when you have a concrete reason to walk backwards or delete in O(1) given a node reference.

## Doubly Linked List

Each node holds a value, a `Next`, and a `Prev`. The list is identified by both `head` and `tail`.

```
head                                         tail
 |                                            |
 v                                            v
[nil|1|*] <-> [*|2|*] <-> [*|3|*] <-> [*|4|nil]
```

The extra pointer buys three things:

- **O(1) deletion given a node reference.** No walk to find the predecessor. This is the reason LRU caches use doubly linked lists: the hash map hands you a direct pointer to the node, and the delete-and-reinsert must be O(1).
- **Backward traversal.**
- **O(1) tail deletion** via `tail.Prev`.

The cost is one extra pointer per node and one extra rewire on every insert or delete.

### Insertion Rewiring Sketch

Insert new node `X` between `A` and `B`:

```
Before:  ... [A] <-> [B] ...

Steps (order matters):
  1. X.Prev = A
  2. X.Next = B
  3. A.Next = X
  4. B.Prev = X

After:   ... [A] <-> [X] <-> [B] ...
```

Do 1–2 before 3–4. Overwrite `A.Next` first and you lose `B` unless you had already saved it.

## Circular Linked List

A linked list where the last node's `Next` points back to the head instead of `nil`. Can be singly or doubly linked.

```
    +---------------------------+
    v                           |
   [1|*] -> [2|*] -> [3|*] -> [4|*]
    ^                           |
    +---------------------------+
```

Use cases where it earns itself:

- **Round-robin scheduling.** A ready queue of processes where "next" always exists; the scheduler never checks for the end of the list.
- **Josephus-style problems.** Any problem where elements are eliminated in a circular order.

The defining hazard is that naive traversal (`for curr != nil`) becomes an infinite loop. You must terminate on `curr == head` or on a counter.

## Dummy Head Node Technique

A pattern, not a data structure. Before any algorithm that may modify the head, allocate a **dummy node** whose `Next` points to the real head, operate on the dummy's chain, and return `dummy.Next`.

```
Without dummy:
  head -> [1] -> [2] -> [3]
  Deleting [1] forces a special case: "is the node being deleted the head?"

With dummy:
  dummy -> [1] -> [2] -> [3]
  Deleting [1] is just: prev.Next = prev.Next.Next, where prev = dummy.
```

The dummy erases one branch: "am I operating on the head?" That branch is where most linked-list bugs live.

When to use it:

- Any problem that may delete or replace the first node (`Remove Nth From End`, `Delete Duplicates`, `Remove Elements`).
- Any problem that builds a new list by appending (`Merge Two Sorted Lists`, `Add Two Numbers`, `Partition List`). The dummy gives you a `tail` to append to before the first real node exists.

When not to use it:

- Read-only traversal.
- In-place reversal; the standard three-pointer template handles the head change without one.

The dummy is sometimes called a *sentinel node*. Same idea.

## Linked Lists in Go

Go has no built-in singly linked list. `container/list` provides a doubly linked list with a sentinel node, but no interview expects it. The idiom is to define your own struct:

```go
type ListNode struct {
    Val  int
    Next *ListNode
}
```

Points to notice:

- **`Next` is a pointer.** Embedding by value (`Next ListNode`) is a compile error because the struct would have infinite size. The pointer breaks the recursion.
- **The zero value is useful.** `&ListNode{}` is a valid dummy; you rarely need `new`.
- **No manual free.** The GC reclaims unreachable nodes. Drop the only reference to the head and the whole list becomes garbage on the next cycle. The flip side: an unbounded list held by a package-level variable will grow forever.
- **LeetCode's signature.** LeetCode Go problems use exactly this struct with a fixed name; your solution can reference `ListNode` directly.

The rest of this file assumes this struct.

## Core Operations

Four operations show up in nearly every linked-list problem.

### Reversal (Three-Pointer Iterative)

Walk the list once; at each node, redirect its `Next` to the previous node.

```
Initial:
  prev = nil    curr = head    next = ?

  nil    [1] -> [2] -> [3] -> nil
   ^      ^
  prev   curr

After step 1 (save next, rewire, advance both):

  nil <- [1]    [2] -> [3] -> nil
          ^      ^
         prev   curr

After step 2:

  nil <- [1] <- [2]    [3] -> nil
                 ^      ^
                prev   curr

After step 3:

  nil <- [1] <- [2] <- [3]    nil
                        ^      ^
                       prev   curr

Return prev.
```

The critical line is `next := curr.Next` *before* `curr.Next = prev`. Skip it and you lose the rest of the list on the first iteration.

### Finding the Middle (Slow/Fast Pointers)

`slow` moves one step, `fast` two. When `fast` reaches the end, `slow` is at the middle. Odd length: `slow` lands on the exact middle. Even length with loop `for fast != nil && fast.Next != nil`: `slow` lands on the **second** middle. For the *first* middle in the even case (which merge sort's split needs), use `for fast.Next != nil && fast.Next.Next != nil`.

Depth in `patterns/fast-slow-pointers.md`.

### Merging Two Sorted Lists

Dummy-head showcase. Keep a `tail` at a dummy, compare heads of the two input lists, splice the smaller onto `tail.Next`, advance. When one list empties, splice the remainder wholesale. Return `dummy.Next`. O(n + m) time, O(1) extra space because nodes are re-linked, not copied. This is why merge sort on linked lists is genuinely O(1) extra space — no scratch buffer.

### Cycle Detection (Floyd's Tortoise and Hare)

Two pointers; one moves one step, the other two. If a cycle exists they meet inside it in O(n) time and O(1) space. If none, `fast` hits `nil`. Finding the *entry point* needs a second phase (reset one pointer to the head, step both one at a time; they meet at the entry). Full proof in `patterns/fast-slow-pointers.md`.

## Go Template Code

```go
type ListNode struct {
    Val  int
    Next *ListNode
}

// Reverse a singly linked list in place. Returns the new head.
func Reverse(head *ListNode) *ListNode {
    var prev *ListNode
    curr := head
    for curr != nil {
        next := curr.Next // save before overwriting
        curr.Next = prev  // rewire
        prev = curr       // advance prev
        curr = next       // advance curr
    }
    return prev
}

// Middle returns the (second) middle node. For [1,2,3,4] returns 3.
func Middle(head *ListNode) *ListNode {
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }
    return slow
}

// MergeTwoSorted merges two sorted lists using a dummy head.
func MergeTwoSorted(a, b *ListNode) *ListNode {
    dummy := &ListNode{}
    tail := dummy
    for a != nil && b != nil {
        if a.Val <= b.Val {
            tail.Next = a
            a = a.Next
        } else {
            tail.Next = b
            b = b.Next
        }
        tail = tail.Next
    }
    if a != nil {
        tail.Next = a
    } else {
        tail.Next = b
    }
    return dummy.Next
}

// HasCycle: Floyd's tortoise and hare.
func HasCycle(head *ListNode) bool {
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast {
            return true
        }
    }
    return false
}
```

## Where JavaScript Differs

JavaScript has no struct type; a node is a plain object.

```js
class ListNode {
  constructor(val = 0, next = null) {
    this.val = val;
    this.next = next;
  }
}
```

Two idiom differences worth naming:

- **No pointer vs value distinction.** Every non-primitive assignment is a reference assignment, so `curr.next = prev` behaves the same way regardless. This removes the "why is `Next` a pointer" question but also removes the mental model that catches infinite-size-struct mistakes early.
- **End sentinel is `null`.** LeetCode's JS template uses `null`. `undefined` in a `.next` field is almost always a bug, not an intentional terminator.

Algorithms are identical.

## LeetCode Problems

### 1. Reverse Linked List — [LeetCode #206](https://leetcode.com/problems/reverse-linked-list/) · Easy

**Problem**

Given the `head` of a singly linked list, reverse the list and return the new head. The list has 0 to 5000 nodes with values in `[-5000, 5000]`.

**Brute Force**

Walk the list once collecting values into a slice, reverse the slice, rebuild a new list from it. O(n) time, O(n) extra space, plus the full allocation of a second list. The optimal below drops space to O(1) by rewiring the existing nodes in place.

<details>
<summary>Hint 1</summary>
Walk the list once. At each node, you want to make its `next` point backwards instead of forwards. What do you need to remember just before you overwrite `next`?
</details>

<details>
<summary>Hint 2</summary>
Keep three variables as you walk: the previous node, the current node, and the next node. Save the next node before you rewire the current node's pointer, then advance all three by one step.
</details>

<details>
<summary>Solution (Go)</summary>

```go
func reverseList(head *ListNode) *ListNode {
    var prev *ListNode
    curr := head
    for curr != nil {
        next := curr.Next
        curr.Next = prev
        prev = curr
        curr = next
    }
    return prev
}
```

**Why this works:** Each iteration retires one node: it will never be visited again, and its `Next` now points to the correct (reversed) neighbour. When `curr` becomes nil, `prev` is the old tail, which is the new head. O(n) time, O(1) space.
</details>

**Follow-Up: Implement it recursively.**

<details>
<summary>Solution (Go)</summary>

```go
func reverseList(head *ListNode) *ListNode {
    if head == nil || head.Next == nil {
        return head
    }
    newHead := reverseList(head.Next)
    head.Next.Next = head // the node that used to come after `head` now points back at it
    head.Next = nil       // break the forward link so the old head becomes the new tail
    return newHead
}
```

**Why this works:** Recurse all the way to the last node, which becomes the new head and bubbles unchanged up the stack. At each returning frame, the current `head` has a `Next` that already points to a fully-reversed suffix; flipping `head.Next.Next = head` attaches `head` to the tail of that suffix, and `head.Next = nil` ensures it terminates the list if the recursion unwinds no further. O(n) time, O(n) stack space — the trade vs the iterative version.
</details>

### 2. Add Two Numbers — [LeetCode #2](https://leetcode.com/problems/add-two-numbers/) · Medium

**Problem**

You are given two non-empty linked lists representing two non-negative integers. The digits are stored in **reverse order** (least-significant digit first), one digit per node. Add the two numbers and return the sum as a linked list in the same format. Each list has 1 to 100 nodes, no leading zeros (except the number 0 itself).

Example: `342 + 465 = 807` → input `[2,4,3] + [5,6,4]`, output `[7,0,8]`.

**Brute Force**

Walk both lists collecting digits into two big integers, add them, then build a new list digit by digit. Works for the toy examples but blows up on the real constraints: lists can have up to 100 digits each, overflowing `int64` and forcing a big-integer library. The streaming solution below never materialises either number, so it has no length limit.

<details>
<summary>Hint 1</summary>
Think about how you add two numbers on paper, from the ones place up. Both input lists are already stored least-significant-digit first — so the natural walking order matches the natural addition order. What's the one piece of state you have to carry between digit positions?
</details>

<details>
<summary>Hint 2</summary>
Use a dummy head and a running `carry`. At each step, pull the digit from each list (or 0 if that list is exhausted), add them plus carry, append `sum % 10` to the output, and set `carry = sum / 10`. Keep going until both lists are empty *and* carry is zero — don't forget the final carry-out ([9,9] + [1] = [0,0,1]).
</details>

<details>
<summary>Solution (Go)</summary>

```go
func addTwoNumbers(l1, l2 *ListNode) *ListNode {
    dummy := &ListNode{}
    tail := dummy
    carry := 0
    for l1 != nil || l2 != nil || carry != 0 {
        sum := carry
        if l1 != nil {
            sum += l1.Val
            l1 = l1.Next
        }
        if l2 != nil {
            sum += l2.Val
            l2 = l2.Next
        }
        carry = sum / 10
        tail.Next = &ListNode{Val: sum % 10}
        tail = tail.Next
    }
    return dummy.Next
}
```

**Why this works:** The three-way loop condition (either list has digits, or we still owe a carry) is the whole trick — it collapses the "lists of different lengths" and "final carry propagation" cases into the same code path. The dummy head means you never need a special case for "which list's first node starts the result". O(max(n, m)) time, O(max(n, m)) space for the result.
</details>

### 3. Merge Two Sorted Lists — [LeetCode #21](https://leetcode.com/problems/merge-two-sorted-lists/) · Easy

**Problem**

Given the heads of two sorted (non-decreasing) linked lists `list1` and `list2`, splice them into one sorted linked list by re-linking the existing nodes, and return its head. Each list has 0 to 50 nodes; values are in `[-100, 100]`.

**Brute Force**

Collect every value from both lists into a slice, sort it, rebuild a new list. O((n+m) log(n+m)) time, O(n+m) space. Throws away the fact that both inputs are already sorted.

<details>
<summary>Hint 1</summary>
You don't know in advance which list contributes the first node of the result. Is there a way to defer that decision, so you can write one uniform append step?
</details>

<details>
<summary>Hint 2</summary>
Create a dummy node and keep a `tail` pointer at it. At each step, splice the smaller head onto `tail.Next` and advance. When one list empties, attach the remaining list wholesale. Return the node after the dummy.
</details>

<details>
<summary>Solution (Go)</summary>

```go
func mergeTwoLists(a, b *ListNode) *ListNode {
    dummy := &ListNode{}
    tail := dummy
    for a != nil && b != nil {
        if a.Val <= b.Val {
            tail.Next = a
            a = a.Next
        } else {
            tail.Next = b
            b = b.Next
        }
        tail = tail.Next
    }
    if a != nil {
        tail.Next = a
    } else {
        tail.Next = b
    }
    return dummy.Next
}
```

**Why this works:** No allocation beyond the dummy; nodes are re-linked, not copied. The dummy removes the "which list starts the result" branch. O(n + m) time, O(1) extra space.
</details>

### 4. Remove Duplicates from Sorted List — [LeetCode #83](https://leetcode.com/problems/remove-duplicates-from-sorted-list/) · Easy

**Problem**

Given the `head` of a sorted (non-decreasing) linked list, delete all duplicates such that each value appears only once. Return the resulting (still sorted) list. The list has 0 to 300 nodes, values in `[-100, 100]`.

**Brute Force**

Collect values into a slice, dedupe with a hash set, rebuild a new list. O(n) time, O(n) space. Throws away the fact that duplicates are already adjacent.

<details>
<summary>Hint 1</summary>
Because the list is sorted, duplicates are always adjacent — you never need to look further than the next node. What's the minimum state you need to walk once and splice out matches?
</details>

<details>
<summary>Hint 2</summary>
A single pointer suffices. At each step, if `curr.Val == curr.Next.Val`, skip `curr.Next` by setting `curr.Next = curr.Next.Next` and *do not* advance `curr` yet (there may be more duplicates). Otherwise, advance.
</details>

<details>
<summary>Solution (Go)</summary>

```go
func deleteDuplicates(head *ListNode) *ListNode {
    curr := head
    for curr != nil && curr.Next != nil {
        if curr.Val == curr.Next.Val {
            curr.Next = curr.Next.Next
        } else {
            curr = curr.Next
        }
    }
    return head
}
```

**Why this works:** The head itself is never a candidate for deletion (we keep the first occurrence of each value), so no dummy node is needed. Not advancing `curr` on the deletion branch is what handles runs of three or more equal values in a single pass. O(n) time, O(1) space.
</details>

### 5. Remove Nth Node From End of List — [LeetCode #19](https://leetcode.com/problems/remove-nth-node-from-end-of-list/) · Medium

**Problem**

Given the `head` of a linked list of length `sz` and an integer `n` where `1 <= n <= sz`, remove the n-th node from the **end** of the list and return its head. Constraints: `1 <= sz <= 30`, node values in `[0, 100]`.

**Brute Force**

Two passes: walk once to count length `L`, walk again `L - n` steps to the predecessor, splice out. O(n) time, O(1) space but two passes. The two-pointer version does it in one.

<details>
<summary>Hint 1</summary>
You do not know the length up front. What if you kept two pointers exactly `n` nodes apart? When the leading pointer hits the end, where is the trailing one?
</details>

<details>
<summary>Hint 2</summary>
The deletion may target the head itself, which is the classic "dummy head" trigger. Start both pointers at a dummy that precedes the head, advance the fast pointer `n` steps, then advance both until fast reaches the last node. Slow is now at the node before the one to remove.
</details>

<details>
<summary>Solution (Go)</summary>

```go
func removeNthFromEnd(head *ListNode, n int) *ListNode {
    dummy := &ListNode{Next: head}
    slow, fast := dummy, dummy
    for i := 0; i < n; i++ {
        fast = fast.Next
    }
    for fast.Next != nil {
        slow = slow.Next
        fast = fast.Next
    }
    slow.Next = slow.Next.Next
    return dummy.Next
}
```

**Why this works:** Maintaining a constant gap of `n` between two pointers converts "nth from end" into "predecessor of the node to delete" in a single pass. The dummy handles the case where the head itself is being deleted. O(n) time, O(1) space.
</details>

**Follow-Up: Could you do it in one pass?** Already satisfied — the two-pointer gap trick above traverses the list exactly once. The Brute Force (count length, then walk) is the two-pass version this follow-up rules out.

### 6. Odd Even Linked List — [LeetCode #328](https://leetcode.com/problems/odd-even-linked-list/) · Medium

**Problem**

Given the `head` of a singly linked list, group all nodes at **odd indices** followed by all nodes at **even indices** (1-indexed by position, not by value). Preserve the relative order within each group. Return the reordered list. Constraints: 0 to 10⁴ nodes, values in `[-10⁶, 10⁶]`.

Example: `[1,2,3,4,5]` → `[1,3,5,2,4]` (odd-position nodes 1,3,5 first; then even-position 2,4).

**Brute Force**

Walk the list copying odd-indexed values into one slice and even-indexed into another, concatenate, and rebuild a new list from the result. O(n) time, O(n) extra space. The in-place rewiring version keeps the original nodes and uses no auxiliary storage.

<details>
<summary>Hint 1</summary>
You're not changing values, just reorder. The output is really two interleaved sub-chains, pulled apart and stitched end to end. Can you grow those two sub-chains while walking the list exactly once?
</details>

<details>
<summary>Hint 2</summary>
Keep two tails: `odd` starting at `head`, `even` starting at `head.Next`. Remember `evenHead` to splice later. At each step, pull `odd` forward to `even.Next` (the next odd node) and `even` forward to `odd.Next` (the next even node). When `even` runs out, attach `evenHead` after `odd`.
</details>

<details>
<summary>Solution (Go)</summary>

```go
func oddEvenList(head *ListNode) *ListNode {
    if head == nil || head.Next == nil {
        return head
    }
    odd, even := head, head.Next
    evenHead := even
    for even != nil && even.Next != nil {
        odd.Next = even.Next
        odd = odd.Next
        even.Next = odd.Next
        even = even.Next
    }
    odd.Next = evenHead
    return head
}
```

**Why this works:** Each iteration advances both chains by one node — `odd` jumps over the next even node to the one after, and `even` does the same the other way. The loop condition `even != nil && even.Next != nil` covers both parity cases cleanly: when it stops, `odd` is the last odd node, and `even` has already stopped extending itself with `nil`, so a single splice stitches the two chains. O(n) time, O(1) space.
</details>

### 7. Reorder List — [LeetCode #143](https://leetcode.com/problems/reorder-list/) · Medium

**Problem**

Given the `head` of a singly linked list `L0 → L1 → … → Ln-1 → Ln`, reorder it in place to `L0 → Ln → L1 → Ln-1 → L2 → Ln-2 → …`. You may not modify the values; only the nodes themselves may be re-linked. Constraints: 1 to 5·10⁴ nodes, values in `[1, 1000]`.

**Brute Force**

Copy node pointers into a slice; use two indices (front and back) to rewire in the interleaved order. O(n) time, O(n) space. The optimal composes split + reverse + merge to achieve O(1) space.

<details>
<summary>Hint 1</summary>
The target order interleaves the front of the list with the back. You cannot walk the list backwards. What structural change would let you consume both ends in one pass?
</details>

<details>
<summary>Hint 2</summary>
Split the list at the middle, reverse the second half in place, then merge the two halves by taking one node from each alternately.
</details>

<details>
<summary>Solution (Go)</summary>

```go
func reorderList(head *ListNode) {
    if head == nil || head.Next == nil {
        return
    }
    // Find first middle so the front half is >= the back half by one in odd cases.
    slow, fast := head, head
    for fast.Next != nil && fast.Next.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }
    second := reverse(slow.Next)
    slow.Next = nil
    first := head
    for second != nil {
        t1, t2 := first.Next, second.Next
        first.Next = second
        second.Next = t1
        first, second = t1, t2
    }
}

func reverse(head *ListNode) *ListNode {
    var prev *ListNode
    curr := head
    for curr != nil {
        next := curr.Next
        curr.Next = prev
        prev = curr
        curr = next
    }
    return prev
}
```

**Why this works:** Three known sub-operations composed. Each is O(n) and O(1) space, so the whole thing is O(n) time, O(1) space. Recognising this shape (split, reverse, merge) saves the whole problem.
</details>

### 8. Copy List with Random Pointer — [LeetCode #138](https://leetcode.com/problems/copy-list-with-random-pointer/) · Medium

**Problem**

A linked list of length `n` is given where each node holds a `Val`, a `Next` pointer, and an additional `Random` pointer that may point to **any node in the list or `nil`**. Construct a **deep copy**: `n` brand-new nodes with the same values, whose `Next` and `Random` pointers mirror the originals but reference only the new nodes (no pointer in the copy may point back into the original). Return the head of the copy. Constraints: 0 to 1000 nodes.

**Brute Force**

For each cloned node, to resolve its random, walk from the original head counting steps to find the target's index, then walk the cloned list the same number of steps. O(n²) time, O(n) space for the clones. The hash-map version drops time to O(n).

<details>
<summary>Hint 1</summary>
When you clone a node, its `random` pointer must eventually reference the *clone* of some other node, but that clone may not exist yet. How do you map an original node to its clone?
</details>

<details>
<summary>Hint 2</summary>
A hash map from original node to cloned node solves it in two passes: first pass creates all clones and populates the map; second pass fills in each clone's `next` and `random` by looking up the originals' targets in the map.
</details>

<details>
<summary>Solution (Go)</summary>

```go
type Node struct {
    Val    int
    Next   *Node
    Random *Node
}

func copyRandomList(head *Node) *Node {
    if head == nil {
        return nil
    }
    clone := make(map[*Node]*Node)
    for p := head; p != nil; p = p.Next {
        clone[p] = &Node{Val: p.Val}
    }
    for p := head; p != nil; p = p.Next {
        clone[p].Next = clone[p.Next]
        clone[p].Random = clone[p.Random]
    }
    return clone[head]
}
```

**Why this works:** The map decouples "create the clone" from "wire the clone", so the second pass can look up any target regardless of allocation order. `clone[nil]` returns the zero value `nil`, which is exactly the correct terminator. O(n) time, O(n) space.
</details>

### 9. LRU Cache — [LeetCode #146](https://leetcode.com/problems/lru-cache/) · Medium

**Problem**

Design a data structure that follows the **Least Recently Used (LRU)** eviction policy. Implement `LRUCache`:

- `LRUCache(capacity int)` — initialise with a positive capacity.
- `Get(key int) int` — return the value if the key exists, else `-1`. A successful `Get` counts as a use.
- `Put(key int, value int)` — insert or update. If inserting would exceed capacity, evict the least recently used key first.

Both `Get` and `Put` must run in **O(1) average time**. Constraints: `1 <= capacity <= 3000`, up to 2·10⁵ calls.

**Brute Force**

Hash map plus a slice tracking recency order. `Get` is O(1) for lookup but O(n) to move the key to the front of the slice (every element after it shifts). `Put` evicts the slice's tail, also O(n). The doubly linked list + hash map version makes both O(1).

<details>
<summary>Hint 1</summary>
Both `Get` and `Put` must be O(1). A hash map alone can't order entries by recency, and a list alone can't find an entry by key in O(1). What if you combined them?
</details>

<details>
<summary>Hint 2</summary>
Use a hash map from key to a node in a doubly linked list ordered by recency. The head end is most-recently-used, the tail end is least-recently-used. `Get` moves the node to the head. `Put` inserts at head and, if over capacity, deletes the tail. The `Prev` pointer is what makes move-to-head O(1); a singly linked list would force an O(n) walk to find each node's predecessor.
</details>

<details>
<summary>Solution (Go)</summary>

```go
type entry struct {
    key, val   int
    prev, next *entry
}

type LRUCache struct {
    cap        int
    m          map[int]*entry
    head, tail *entry // sentinel head and tail
}

func Constructor(capacity int) LRUCache {
    h, t := &entry{}, &entry{}
    h.next, t.prev = t, h
    return LRUCache{cap: capacity, m: make(map[int]*entry), head: h, tail: t}
}

func (c *LRUCache) unlink(e *entry) {
    e.prev.next = e.next
    e.next.prev = e.prev
}

func (c *LRUCache) pushFront(e *entry) {
    e.prev = c.head
    e.next = c.head.next
    c.head.next.prev = e
    c.head.next = e
}

func (c *LRUCache) Get(key int) int {
    e, ok := c.m[key]
    if !ok {
        return -1
    }
    c.unlink(e)
    c.pushFront(e)
    return e.val
}

func (c *LRUCache) Put(key, value int) {
    if e, ok := c.m[key]; ok {
        e.val = value
        c.unlink(e)
        c.pushFront(e)
        return
    }
    if len(c.m) == c.cap {
        lru := c.tail.prev
        c.unlink(lru)
        delete(c.m, lru.key)
    }
    e := &entry{key: key, val: value}
    c.pushFront(e)
    c.m[key] = e
}
```

**Why this works:** The map gives O(1) lookup by key; the doubly linked list gives O(1) reorder given a node reference. Sentinel head and tail nodes remove boundary branches. Both `Get` and `Put` are O(1) worst case.
</details>

## References

- `dsa/data-structures/arrays.md` — the O(1)-access counterpart; the tradeoffs only make sense in contrast.
- `dsa/patterns/fast-slow-pointers.md` — cycle detection, middle-finding, kth from end. Half the problems above rely on these techniques.
- `dsa/patterns/in-place-reversal.md` — generalises the three-pointer reverse to sublist reversal (`Reverse Nodes in k-Group`, `Reverse Linked List II`).
- `dsa/data-structures/hash-maps.md` — the LRU cache's other half.