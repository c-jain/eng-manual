---
Status: 🌳 Evergreen
Created: 2026-10-27
Last Updated: 2026-10-27
---

# In-Place Reversal

## Table Of Contents

1. [What It Is And Why It Exists](#what-it-is-and-why-it-exists)
2. [The Three-Pointer Move](#the-three-pointer-move)
3. [Reversing A Linked List](#reversing-a-linked-list)
4. [Reversing An Array](#reversing-an-array)
5. [Recognizing The Pattern](#recognizing-the-pattern)
6. [Where JavaScript Differs](#where-javascript-differs)
7. [LeetCode Problems](#leetcode-problems)
8. [References](#references)

## What It Is And Why It Exists

In-place reversal is a family of techniques for reversing the order of elements in a sequence without allocating a second sequence to hold the result. On a linked list it means rewiring `next` pointers so that traversal walks the other way. On an array it means swapping symmetric positions from the two ends inward. In both cases the space cost drops from O(n) (the copied sequence) to O(1) (a handful of pointers or indices).

The pattern exists because the obvious solution wastes memory. Reversing by copying is trivial: allocate a new list or slice, walk the original from the tail, append as you go. That works, but on constrained hardware or for hot inner loops it doubles the memory footprint of the data you were already storing. Interviewers ask for the in-place version because it forces you to reason about aliasing: when you overwrite `curr.Next`, the old value is lost unless you saved it first. That habit of thinking one step ahead of a mutation is exactly what the pattern trains.

> [!NOTE]
> **Hot inner loop** is performance-engineering slang for code that runs so often it dominates CPU time, usually the innermost loop of a nested traversal executing millions of times. Allocating inside a hot inner loop is expensive: every allocation is heap pressure the garbage collector must eventually reclaim, and the cost compounds with the iteration count. On a cold path (runs twice a day) nobody notices; on a hot one it shows up in profiles as GC spikes or allocation-rate stalls. In-place reversal exists precisely so this operation stays zero-allocation when dropped inside such a loop.

It is also a composable building block. Many mid-tier linked-list problems (`Reorder List`, `Palindrome Linked List`, `Rotate List`) decompose into "find some pivot, then reverse a chunk, then splice." Getting the reverse-a-chunk primitive rock-solid pays back everywhere else. The pattern applies whenever the reordering is a *true reversal*: position `i` in the affected range swaps with position `n-1-i`. Permutations that are not reversals (arbitrary shuffles, sorts, dedupes) need different machinery.

### Why The Name

"In-place" is the standard algorithmic term for "uses O(1) extra space beyond the input itself". The input is mutated; no parallel structure is allocated to hold the answer. "Reversal" is the operation being performed: index `i` in the affected range ends up where index `n-1-i` started (or, for a linked list, the node that was reachable in k hops ends up reachable in 0 and vice versa). The name is descriptive, not metaphorical: it says exactly what the pattern does and what it does not spend.

## The Three-Pointer Move

Every linked-list reversal in this file, from the full-list case to k-groups, is the same three-line loop under the hood. It is worth burning into muscle memory because a wrong ordering silently drops the tail.

```
prev = nil
curr = head
while curr != nil:
    next = curr.Next     // (1) save the reference we're about to overwrite
    curr.Next = prev     // (2) rewire: point backwards
    prev = curr          // (3a) advance prev
    curr = next          // (3b) advance curr
return prev              // new head
```

The invariant maintained across every iteration is: everything from `head` up to but not including `curr` has already been reversed; `prev` points at the newest head of that reversed prefix; `curr` points at the next node still in original order. When `curr` becomes nil the whole list has been consumed and `prev` is the new head.

The order of the three lines matters. Rewiring before saving is the classic bug:

```
curr.Next = prev         // BUG: original curr.Next is gone
next = curr.Next         // now reads prev, not the real next node
```

The list breaks at the first iteration and the loop either terminates early or spins in a two-node cycle.

```
Scenario: One Step Of Reversal

Before iteration (reversing 1 -> 2 -> 3 -> nil):

  prev = nil    curr = [1] -> [2] -> [3] -> nil

Step (1) save:              next = [2]
Step (2) rewire:            [1].Next = nil
Step (3) advance prev,curr: prev = [1], curr = [2]

After iteration:

  reversed prefix: nil <- [1]        (prev points here)
  remaining:       [2] -> [3] -> nil  (curr points at [2])
```

## Reversing A Linked List

The three-pointer move scales up cleanly. First on the whole list, then constrained to a window, then applied repeatedly across fixed-size chunks. Each subsection below builds on the one above.

### Full List

Drop the three-pointer move into a loop and you have it. There is no need for a dummy head because the head of the input is not a special case: it becomes the tail of the output, which is exactly what the loop already produces (its `Next` gets set to `nil` on the first iteration, since `prev` starts at `nil`).

```go
// Time: O(n) | Space: O(1)
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

A recursive form also exists and reads elegantly, but it costs O(n) stack frames. The key line is `head.Next.Next = head`: at that point `head.Next` still refers to the original second node, which the recursion has already made the tail of the reversed suffix; setting its `Next` to `head` extends the suffix by one, and `head.Next = nil` becomes the new tail's terminator. Prefer the iterative form unless the interviewer specifically asks for recursion.

```go
// Time: O(n) | Space: O(n) call stack
func reverseListRecursive(head *ListNode) *ListNode {
    if head == nil || head.Next == nil {
        return head
    }
    newHead := reverseListRecursive(head.Next)
    head.Next.Next = head
    head.Next = nil
    return newHead
}
```

### A Sublist

The generalisation: reverse only the nodes between positions `left` and `right` (1-indexed, inclusive), leaving everything outside untouched. The output must still be a single well-formed list.

The structural complication is that `left` may be 1, meaning the head itself moves. That is the standard signal for a dummy head node: allocate one whose `Next` is the real head, do all pointer manipulation relative to the dummy, return `dummy.Next` at the end. The head-moves-or-not distinction disappears.

The algorithm has three phases:

1. **Walk to the node just before position `left`.** Call it `preLeft`. This is the anchor that must eventually point at the reversed segment.
2. **Run the three-pointer reverse for exactly `right - left + 1` iterations,** starting from the node at position `left`. At the end, `prev` is the new head of the reversed segment and `curr` is the first node after it (the original node at position `right + 1`).
3. **Splice.** The original node at position `left` is now the tail of the reversed segment; its `Next` must become `curr`. The `preLeft` anchor's `Next` must become `prev`.

> [!NOTE]
> **Splice** comes from rope-work and film editing: cut two strands, then join the loose ends to make one continuous strand again. In pointer code, the "cut" is the moment you rewire pointers inside a segment, and the "splice" is the moment you reconnect that segment's ends to the surrounding untouched nodes. A splice is always about *two* seams (the left end and the right end of the affected segment), which is why the sublist algorithm saves two anchors — `preLeft` and `leftNode` — before touching anything. Save only one and half the list drops on the floor. The same idea shows up in `merge two sorted lists`, `Reorder List`, k-groups (each chunk splices to the previous chunk's tail), and Rust's `LinkedList.splice()` method in std.

```
Scenario: Reverse Sublist [2, 4] Of  1 -> 2 -> 3 -> 4 -> 5

After phase 1:
  preLeft = [1]
  leftNode (kept as pointer)   = [2]   // will become the segment tail

After phase 2 (three iterations):
  prev = [4]          // new segment head
  curr = [5]          // first node past the segment

After phase 3 (splice):
  preLeft.Next  = prev   => [1] -> [4]
  leftNode.Next = curr   => [2] -> [5]

Result:              1 -> 4 -> 3 -> 2 -> 5
```

```go
// Time: O(n) | Space: O(1)
func reverseBetween(head *ListNode, left, right int) *ListNode {
    if head == nil || left == right {
        return head
    }
    dummy := &ListNode{Next: head}
    preLeft := dummy
    for i := 1; i < left; i++ {
        preLeft = preLeft.Next
    }
    leftNode := preLeft.Next // will become tail after reverse

    var prev *ListNode
    curr := leftNode
    for i := 0; i < right-left+1; i++ {
        next := curr.Next
        curr.Next = prev
        prev = curr
        curr = next
    }

    preLeft.Next = prev
    leftNode.Next = curr
    return dummy.Next
}
```

The two saved pointers, `preLeft` and `leftNode`, are the whole difference between "reverse everything" and "reverse a window". They mark the two seams that need re-stitching after the local reverse mangles them.

### In K-Groups

Given `k`, reverse every consecutive block of `k` nodes; if the last block has fewer than `k` nodes, leave it in original order. The problem is the sublist algorithm applied repeatedly, with two subtleties:

1. **Count before you reverse.** Before running the three-pointer move on a group, walk `k` steps ahead to confirm at least `k` more nodes remain. If not, the tail must stay as-is per the problem's rules. Running the reverse first and then discovering the group was short leaves the list in a broken state.
2. **Chain the seams.** The tail of the previous reversed group must point at the head of the next reversed group. That means each iteration needs to remember which node was the tail of what it just reversed, because that node becomes the anchor for the next group.

```
Scenario: k = 2 On 1 -> 2 -> 3 -> 4 -> 5

Group 1 ([1, 2]):
  reverse -> 2 -> 1
  anchor previous group's tail's Next to new head (2)
  save [1] as the new anchor for group 2

Group 2 ([3, 4]):
  reverse -> 4 -> 3
  [1].Next = [4]
  save [3] as anchor

Group 3 ([5]): only 1 node < k, leave as-is.
  [3].Next = [5]

Result: 2 -> 1 -> 4 -> 3 -> 5
```

```go
// Time: O(n) | Space: O(1)
func reverseKGroup(head *ListNode, k int) *ListNode {
    dummy := &ListNode{Next: head}
    groupPrev := dummy

    for {
        // Check that k nodes remain starting after groupPrev.
        kth := groupPrev
        for i := 0; i < k && kth != nil; i++ {
            kth = kth.Next
        }
        if kth == nil {
            break
        }
        groupNext := kth.Next
        // Reverse the k-node segment [groupPrev.Next .. kth].
        prev := groupNext
        curr := groupPrev.Next
        for curr != groupNext {
            next := curr.Next
            curr.Next = prev
            prev = curr
            curr = next
        }
        // Splice: groupPrev now points at the new head (old kth),
        // and the new tail (old groupPrev.Next) has been saved.
        tail := groupPrev.Next
        groupPrev.Next = kth
        groupPrev = tail
    }
    return dummy.Next
}
```

The trick here is initialising `prev` to `groupNext` (the node after the group) instead of `nil`. That way, when the loop finishes, the segment's tail's `Next` is already correctly pointing at the next group's untouched region. No separate splice step is needed on the trailing seam.

The recursive form of k-group reversal is worth knowing because it reads more clearly than the iterative one, though it costs O(n/k) stack frames:

```go
// Time: O(n) | Space: O(n/k) call stack
func reverseKGroupRecursive(head *ListNode, k int) *ListNode {
    curr := head
    for i := 0; i < k; i++ {
        if curr == nil {
            return head // fewer than k nodes left; leave as-is
        }
        curr = curr.Next
    }
    // Reverse the first k nodes; curr is the (k+1)th, or the recursion's job.
    newHead := reverseFirstK(head, k)
    head.Next = reverseKGroupRecursive(curr, k)
    return newHead
}

func reverseFirstK(head *ListNode, k int) *ListNode {
    var prev *ListNode
    curr := head
    for i := 0; i < k; i++ {
        next := curr.Next
        curr.Next = prev
        prev = curr
        curr = next
    }
    return prev
}
```

## Reversing An Array

The array side of the pattern. First the plain end-to-end swap, then the rotation trick that composes three of those swaps into a cyclic shift.

### Full Range Swap

The array analog is simpler than the linked-list case: no pointer aliasing, just swap symmetric positions. Two indices march from the ends toward the middle, swapping as they go. They meet (odd length) or cross (even length) and the loop ends. Space is O(1); time is O(n) with exactly `n/2` swaps.

```go
// Time: O(n) | Space: O(1)
func reverseArray(a []int) {
    l, r := 0, len(a)-1
    for l < r {
        a[l], a[r] = a[r], a[l]
        l++
        r--
    }
}
```

The reversal-of-a-range variant takes explicit bounds and is the workhorse for the rotation trick in the next section:

```go
func reverseRange(a []int, l, r int) {
    for l < r {
        a[l], a[r] = a[r], a[l]
        l++
        r--
    }
}
```

There is a subtle correctness point: the loop condition is `l < r`, not `l <= r`. When they are equal the middle element is swapping with itself, which is a no-op but wastes a comparison. When they have crossed the loop must stop or the reversal undoes itself.

### Rotation As Three Reversals

Rotating an array right by `k` positions has an elegant O(1)-space solution built entirely out of in-place reversals. The trick is not obvious the first time but never forgotten once seen.

```
Given:      [1, 2, 3, 4, 5, 6, 7],  k = 3
Target:     [5, 6, 7, 1, 2, 3, 4]

Step 1: reverse the whole array.        [7, 6, 5, 4, 3, 2, 1]
Step 2: reverse the first k elements.   [5, 6, 7, 4, 3, 2, 1]
Step 3: reverse the remaining n-k.      [5, 6, 7, 1, 2, 3, 4]
```

Why does this work? A right rotation by `k` is the same as splitting the array at position `n - k`, then concatenating the two pieces in swapped order. Reversing the whole array flips both pieces and swaps their positions. Reversing each piece back in-place undoes the flip inside each piece without undoing the swap between them. The composition is exactly the desired rotation.

Total work: three reversals, at most `2n` element swaps, O(1) space. The naive alternative (allocate a new array, copy shifted) is O(n) space and often the first instinct to unlearn.

```go
// Time: O(n) | Space: O(1)
func rotate(nums []int, k int) {
    n := len(nums)
    k %= n
    reverseRange(nums, 0, n-1)
    reverseRange(nums, 0, k-1)
    reverseRange(nums, k, n-1)
}
```

The `k %= n` guard matters: without it, a caller passing `k = 10` on a length-7 array does 10 slots of work when it should do 3. Rotation by `n` is the identity, so only the remainder matters.

**Left rotation uses the same trick.** Two ways to express it:

1. **Reorder the three reversals.** Reverse the first `k`, reverse the remaining `n - k`, then reverse the whole array. For `[1, 2, 3, 4, 5, 6, 7]` left-rotated by 3: `[3, 2, 1, 4, 5, 6, 7]` -> `[3, 2, 1, 7, 6, 5, 4]` -> `[4, 5, 6, 7, 1, 2, 3]`.
2. **Delegate to the right rotator.** A left rotation by `k` is the same as a right rotation by `n - k`. If you only want one function in your toolkit, keep `rotate` as right rotation and call it with `n - k%n` for left.

```go
// Time: O(n) | Space: O(1)
func rotateLeft(nums []int, k int) {
    n := len(nums)
    if n == 0 {
        return
    }
    rotate(nums, n-k%n)
}
```

The `k%n` inside the subtraction (not `k` directly) handles callers passing `k >= n`, which would otherwise produce a negative argument to `rotate`.

## Recognizing The Pattern

Reach for in-place reversal when:

- The problem asks you to reverse a linked list or a contiguous portion of one.
- The problem asks to reverse a section of an array with an explicit space constraint.
- The problem is a rotation, cyclic shift, or "move the last k elements to the front" on an array.
- The problem is a composed shape: find a pivot with fast/slow pointers, then reverse the second half, then merge or compare (`Reorder List`, `Palindrome Linked List`).
- The problem constrains you to O(1) extra space and involves rearranging within a fixed structure.

Avoid it when:

- The reordering is not a reversal (e.g. arbitrary permutation, sort, dedupe). Reversal has the specific property that position `i` goes to position `n-1-i` inside the affected range; anything else needs different machinery.
- The structure is immutable in the problem's framing (persistent lists, functional style). Then a fresh reversed copy is the right answer even if it costs O(n) space.

## Where JavaScript Differs

The three-pointer move is identical in idiom. Two mechanical differences worth noting:

- **Destructuring swap.** Go has parallel assignment (`a[l], a[r] = a[r], a[l]`); JavaScript matches with array destructuring: `[a[l], a[r]] = [a[r], a[l]]`. The intent is the same, no temp variable.
- **No nil, use null.** JavaScript's linked-list terminator is `null`. The three-pointer loop reads `while (curr !== null)`, and the initial `prev` is `null`. No other structural change.

## LeetCode Problems

### 1. Remove Nodes From Linked List — [#2487 (Medium)](https://leetcode.com/problems/remove-nodes-from-linked-list/)

Given the head of a linked list, remove every node that has a strictly greater value somewhere to its right. Return the modified list.

<details>
<summary>Brute Force</summary>

For each node, scan the rest of the list to find the maximum value to its right. If that maximum exceeds the node's value, mark the node for removal. After scanning all nodes, do a second pass that unlinks the marked ones.

Time: O(n²) (nested scans). Space: O(n) for the removal marks.

The waste: every node redundantly re-scans the same suffix. The right-side maximum at position `i` and position `i+1` differ by at most one comparison, but the brute force throws that work away.
</details>

<details>
<summary>Hint 1</summary>

"Strictly greater to the right" is awkward in a singly linked list because you cannot walk backwards. What transformation on the list would turn that question into "strictly greater to the left"?
</details>

<details>
<summary>Hint 2</summary>

Reverse the list. Now the question becomes: remove every node that has a strictly greater value somewhere earlier. Walk the reversed list once tracking the running maximum; drop any node whose value is less than that maximum. Reverse the result back to restore the original order.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func removeNodes(head *ListNode) *ListNode {
    if head == nil {
        return nil
    }
    head = reverseList(head)
    maxVal := head.Val
    curr := head
    for curr.Next != nil {
        if curr.Next.Val < maxVal {
            curr.Next = curr.Next.Next // drop the smaller neighbor
        } else {
            curr = curr.Next
            maxVal = curr.Val
        }
    }
    return reverseList(head)
}
```

Three passes, each O(n): reverse, filter, reverse. Still O(n) total, O(1) extra space. The filter pass never advances past a node it just dropped, because `curr.Next` has been rewired to the node after the dropped one; the comparison on the next iteration sees the new neighbor.

An alternative solves it in one pass with a monotonic stack (O(n) space), but that belongs under the monotonic-stack pattern and would make this file about something else. The reverse-filter-reverse approach is the one that showcases reversal as a reusable preprocessing step.
</details>

### 2. Reverse Linked List II — [#92 (Medium)](https://leetcode.com/problems/reverse-linked-list-ii/)

Given the head and two 1-indexed positions `left <= right`, reverse the nodes between those positions inclusive and return the head.

<details>
<summary>Brute Force</summary>

Copy values into a slice, reverse the slice segment `[left-1, right-1]`, walk the list a second time writing values back.

Time: O(n). Space: O(n).

The waste: two full passes and an auxiliary slice, when a single pass can rewire the segment directly. This also modifies node values rather than the structure, which is fine here but breaks any downstream code that identifies nodes by pointer identity.
</details>

<details>
<summary>Hint 1</summary>

The nodes outside `[left, right]` do not move. What are the two seams where the reversed segment must be re-stitched into the rest of the list? Save one pointer for each seam before you start reversing.
</details>

<details>
<summary>Hint 2</summary>

Walk to the node just before position `left`; call it `preLeft`. Save `preLeft.Next` as `leftNode`; this will end up as the segment tail after reversal. Run the three-pointer reversal for exactly `right - left + 1` steps. Then `preLeft.Next = prev` (new head of segment), `leftNode.Next = curr` (first node past segment). Use a dummy head so `left == 1` is not a special case.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func reverseBetween(head *ListNode, left, right int) *ListNode {
    if head == nil || left == right {
        return head
    }
    dummy := &ListNode{Next: head}
    preLeft := dummy
    for i := 1; i < left; i++ {
        preLeft = preLeft.Next
    }
    leftNode := preLeft.Next
    var prev *ListNode
    curr := leftNode
    for i := 0; i < right-left+1; i++ {
        next := curr.Next
        curr.Next = prev
        prev = curr
        curr = next
    }
    preLeft.Next = prev
    leftNode.Next = curr
    return dummy.Next
}
```

The dummy earns itself precisely when `left == 1`: it means `preLeft = dummy` and the seam-fix `preLeft.Next = prev` also becomes the update to the returned head, with no separate branch.
</details>

### 3. Swap Nodes In Pairs — [#24 (Medium)](https://leetcode.com/problems/swap-nodes-in-pairs/)

Given a linked list, swap every two adjacent nodes and return the head. Do not swap values, swap the nodes themselves.

<details>
<summary>Brute Force</summary>

Walk the list writing values into a slice, swap adjacent pairs in the slice, walk again writing values back.

Time: O(n). Space: O(n). Same objection as before: the problem is about node identity, not values.
</details>

<details>
<summary>Hint 1</summary>

Two adjacent nodes `a -> b` become `b -> a`. What is the anchor node whose `Next` must change from `a` to `b`? What does `a.Next` become after the swap?
</details>

<details>
<summary>Hint 2</summary>

This is the k = 2 case of k-group reversal. Use a dummy head so that the swap of the very first pair is not a special case. Keep a `groupPrev` anchor whose `Next` will be the second node of each pair after the swap; advance it to the node that used to be first, then repeat.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func swapPairs(head *ListNode) *ListNode {
    dummy := &ListNode{Next: head}
    groupPrev := dummy
    for groupPrev.Next != nil && groupPrev.Next.Next != nil {
        first := groupPrev.Next
        second := first.Next
        // Rewire: groupPrev -> second -> first -> (rest)
        first.Next = second.Next
        second.Next = first
        groupPrev.Next = second
        // Advance anchor to what is now the tail of this pair.
        groupPrev = first
    }
    return dummy.Next
}
```

Three assignments do the swap. The order matters: `first.Next = second.Next` must happen before `second.Next = first`, otherwise `second.Next` is overwritten while `first.Next` still needs it.
</details>

### 4. Reverse Nodes In K-Group — [#25 (Hard)](https://leetcode.com/problems/reverse-nodes-in-k-group/)

Given a linked list and integer `k`, reverse every consecutive block of `k` nodes. If fewer than `k` nodes remain at the end, leave them in original order.

<details>
<summary>Brute Force</summary>

Copy all values into a slice. Reverse each length-k window in place, leaving any short tail alone. Copy values back.

Time: O(n). Space: O(n).

The waste: same as pair-swap. The structural constraint matters (rearrange nodes, not values) and the space cost is avoidable.
</details>

<details>
<summary>Hint 1</summary>

Before reversing a group, verify that at least `k` more nodes exist starting from the current anchor. If not, the tail must stay untouched. What is the cleanest way to check that without walking twice?
</details>

<details>
<summary>Hint 2</summary>

Keep a `groupPrev` anchor. In each iteration: walk `k` steps forward from `groupPrev` to find the `k`th node (or exit if you hit nil early). Reverse the segment between `groupPrev.Next` and that `k`th node. Then update `groupPrev` to the node that was originally first (now the segment's tail). Initialise the reversal's `prev` to the node just past the group so the segment's tail auto-connects to the next chunk.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func reverseKGroup(head *ListNode, k int) *ListNode {
    dummy := &ListNode{Next: head}
    groupPrev := dummy
    for {
        kth := groupPrev
        for i := 0; i < k && kth != nil; i++ {
            kth = kth.Next
        }
        if kth == nil {
            break
        }
        groupNext := kth.Next
        prev := groupNext
        curr := groupPrev.Next
        for curr != groupNext {
            next := curr.Next
            curr.Next = prev
            prev = curr
            curr = next
        }
        tail := groupPrev.Next
        groupPrev.Next = kth
        groupPrev = tail
    }
    return dummy.Next
}
```
</details>

### 5. Reverse String — [#344 (Easy)](https://leetcode.com/problems/reverse-string/)

Reverse an array of characters in place. Do not allocate a second array.

<details>
<summary>Brute Force</summary>

Allocate a second slice of the same length, walk the original in reverse, copy each character in. Assign back.

Time: O(n). Space: O(n).

The waste: the problem's explicit constraint is O(1) extra space, and the two-pointer swap achieves it.
</details>

<details>
<summary>Hint 1</summary>

Two indices, one at each end. What happens if they walk toward each other, swapping as they pass?
</details>

<details>
<summary>Hint 2</summary>

Standard in-place reversal: `l = 0`, `r = len - 1`. While `l < r`, swap `s[l]` and `s[r]`, then advance `l` and retreat `r`. The loop stops when they meet (odd length) or cross (even length).
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func reverseString(s []byte) {
    l, r := 0, len(s)-1
    for l < r {
        s[l], s[r] = s[r], s[l]
        l++
        r--
    }
}
```
</details>

### 6. Rotate Array — [#189 (Medium)](https://leetcode.com/problems/rotate-array/)

Rotate the array to the right by `k` steps. The follow-up asks for an O(1) extra-space solution.

<details>
<summary>Brute Force</summary>

Allocate a new array. For each index `i`, place `nums[i]` at position `(i + k) % n` in the new array. Copy back.

Time: O(n). Space: O(n).

The waste: the O(1)-space follow-up rules this out. Two other O(1)-space approaches exist (cyclic replacements, three-reversals); the three-reversals variant is the pattern-native one.
</details>

<details>
<summary>Hint 1</summary>

If you split the array at position `n - k`, you get two chunks that need to swap positions. Can you do that swap using only in-place reversals?
</details>

<details>
<summary>Hint 2</summary>

Reverse the whole array. Reverse the first `k` elements. Reverse the last `n - k` elements. Guard against `k > n` with `k %= n` first.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func rotate(nums []int, k int) {
    n := len(nums)
    if n == 0 {
        return
    }
    k %= n
    reverseRange(nums, 0, n-1)
    reverseRange(nums, 0, k-1)
    reverseRange(nums, k, n-1)
}

func reverseRange(a []int, l, r int) {
    for l < r {
        a[l], a[r] = a[r], a[l]
        l++
        r--
    }
}
```

Total swaps are at most `n/2 + k/2 + (n-k)/2 = n`, so the constant factor is one full pass of swaps. That is better than the O(n) space alternative and, for cache reasons, often faster than the cyclic-replacements variant on modern hardware.
</details>

### 7. Reverse Words In A String III — [#557 (Easy)](https://leetcode.com/problems/reverse-words-in-a-string-iii/)

Given a string containing words separated by single spaces, reverse the characters within each word while preserving word order and spaces.

<details>
<summary>Brute Force</summary>

Split on spaces, reverse each substring, join with spaces.

Time: O(n). Space: O(n) for the split slice and the joined result.

The waste: allocating substrings and a join buffer for what is fundamentally a sequence of in-place reversals on ranges of a byte slice. If the input is already mutable bytes, we can hit each range with the two-pointer swap directly.
</details>

<details>
<summary>Hint 1</summary>

Walk the string tracking the start of the current word. When you hit a space or the end, you know the range of the just-finished word. What operation on that range gives you the answer?
</details>

<details>
<summary>Hint 2</summary>

Convert to a byte slice. Scan for word boundaries. For each range `[start, end]` bounding a word, run the two-pointer reverse. Convert back to string at the end.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(n) for the byte slice conversion; the reversal itself is O(1) extra.
func reverseWords(s string) string {
    b := []byte(s)
    start := 0
    for i := 0; i <= len(b); i++ {
        if i == len(b) || b[i] == ' ' {
            reverseRange(b, start, i-1)
            start = i + 1
        }
    }
    return string(b)
}

func reverseRange(b []byte, l, r int) {
    for l < r {
        b[l], b[r] = b[r], b[l]
        l++
        r--
    }
}
```

The `i <= len(b)` bound is the trick that handles the final word without a separate flush step. When `i` reaches `len(b)`, the sentinel branch runs and reverses the last word's range.

The O(n) space caveat is Go-specific: strings are immutable, so the byte-slice conversion is unavoidable. In a language with mutable strings (C++, Rust `&mut str` via bytes) the whole operation is O(1) extra space.
</details>

## References

- [LeetCode: Linked List Tag](https://leetcode.com/tag/linked-list/)
- [NeetCode: Linked List roadmap](https://neetcode.io/roadmap) - the k-group and swap-pairs walkthroughs are especially good in Go
- Sedgewick, Wayne - *Algorithms* (4th ed.), §1.3 (Bags, Queues, and Stacks) for the classical iterative-vs-recursive reversal comparison
- `dsa/data-structures/linked-lists.md` - the three-pointer move as an operation on the underlying data structure, plus the aliasing intuition
- `dsa/patterns/fast-slow-pointers.md` - the sibling primitive most often composed with in-place reversal (`Reorder List`, `Palindrome Linked List`: find middle then reverse then merge/compare)
- `dsa/patterns/two-pointers.md` - the array-side counterpart; the two-pointer swap that powers `reverseRange` is a direct application