---
Status: 🌳 Evergreen
Created: 2026-10-26
Last Updated: 2026-10-26
---

# Fast And Slow Pointers

## Table Of Contents

1. [What It Is And Why It Exists](#what-it-is-and-why-it-exists)
2. [Why "Fast And Slow Pointers"](#why-fast-and-slow-pointers)
3. [Floyd's Cycle Detection](#floyds-cycle-detection)
4. [Finding The Cycle Entry](#finding-the-cycle-entry)
5. [Cycle Length](#cycle-length)
6. [Middle Of A Linked List](#middle-of-a-linked-list)
7. [Kth Node From The End](#kth-node-from-the-end)
8. [Cycle Detection On A Function (Happy Number)](#cycle-detection-on-a-function-happy-number)
9. [Recognizing The Pattern](#recognizing-the-pattern)
10. [Go Template Code](#go-template-code)
11. [Where JavaScript Differs](#where-javascript-differs)
12. [LeetCode Problems](#leetcode-problems)
13. [References](#references)

## What It Is And Why It Exists

Fast and slow pointers means walking the same sequence with two indices (or node references) that advance at **different rates** on every step, most commonly `slow += 1` and `fast += 2`. The rate gap is a proof device. Every use of this pattern reduces to one of two consequences:

- If the sequence is finite and non-cyclic, `fast` reaches the end first. Wherever `slow` is at that moment is a **positional landmark** (the midpoint, an offset from the end).
- If the sequence is cyclic, `fast` cannot fall off the end and must eventually catch `slow` from behind. That collision is the **cycle-detected** signal.

### The Problem It Solves

The natural cycle-detection solution is a hash set of visited nodes: O(n) time, O(n) space. Floyd's algorithm does the same in O(n) time, O(1) space. The space guarantee is why interviewers ask about it. Whenever a linked-list problem attaches an "O(1) extra space" constraint to cycle detection, midpoint, or nth-from-end, this is the intended solution. The alternative for the positional-landmark variants is a two-pass approach (measure length, then walk again), which works but gives up the single-pass property.

## Why "Fast And Slow Pointers"

"Fast" and "slow" literally describe how many nodes each pointer advances per iteration. Alternative names for the same pattern:

- **Tortoise and hare** — Floyd's own metaphor. Tortoise is slow, hare is fast.
- **Floyd's cycle-finding algorithm** — the specific application to cycle detection, published by Robert W. Floyd.
- **Runner technique** — occasionally used for the midpoint and kth-from-end variants.

## Floyd's Cycle Detection

Both pointers start at the head. Each step, `slow` advances one node, `fast` advances two.

```
Non-cyclic:  H -> A -> B -> C -> D -> nil
             fast reaches nil first.

Cyclic:      H -> A -> B -> C -> D
                            ^         |
                            |_________|
             fast eventually lands on the same node as slow.
```

### Why This Guarantees A Meeting

Once both pointers are in the cycle, define the **gap** as the forward distance from `slow` to `fast` around the loop, a value in `[0, L)` where `L` is the cycle length. Gap = 0 means they are on the same node, a meeting.

Each step the gap grows by `2 - 1 = 1` (mod `L`). Adding 1 repeatedly mod `L` cycles through every residue `0, 1, ..., L-1`, so the gap must hit 0 within `L` steps. Guaranteed meeting.

### Why Speeds 1 And 2 Specifically

With speeds 1 and 3 the gap grows by 2 per step. If `L` is even and the pointers enter the cycle with an odd gap, the gap stays odd forever and never reaches 0. No meeting. Concrete example, `L = 4`, initial gap 1:

```
gap sequence: 1, 3, 1, 3, ...   (mod 4)
```

An increment of 1 is the only value coprime to every possible `L`, which is why 1 vs 2 is canonical.

## Finding The Cycle Entry

Once a meeting is detected, a second phase locates the exact node where the cycle begins. Reset one pointer to the head; leave the other at the meeting point. Move both one step at a time. The node where they meet again is the cycle entry.

### Why This Works

Let:
- `μ` (mu) = distance from head to cycle entry
- `L` = cycle length
- `k` = distance from cycle entry to the meeting point, measured forward around the cycle (so `0 ≤ k < L`)

At the meeting point:
- `slow` has walked `μ + k` nodes total.
- `fast` has walked `2(μ + k)` nodes total.
- Both are on the same cycle node, so fast walked `μ + k` more steps than slow, and those extra steps happened inside the cycle. To end at the same cycle position after extra walking, that walking must wrap around a whole number of times: `μ + k = n·L` for some integer `n ≥ 1`. (`n ≥ 1` because fast moves strictly faster, so by the first checked meeting it has walked at least one more step than slow.)

Rearranging: `μ = n·L - k`. Read that as: **starting from the meeting point, walking `μ` steps forward lands you at the cycle entry** (because `n·L - k` steps forward from position `k` around the cycle brings you to position `0`, which is the entry).

So if you also walk `μ` steps forward from the head, you land at the entry too. Two pointers walking at the same speed, one from head, one from meeting point: they meet at the entry.

### Trace

```
List:   1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7
                       ^              |
                       |______________|

μ = 3 (head to node 4), L = 4.

Phase 1 (Floyd):
  step 0: slow=1  fast=1
  step 1: slow=2  fast=3
  step 2: slow=3  fast=5
  step 3: slow=4  fast=7
  step 4: slow=5  fast=5   <- meeting at node 5, so k = 1

Verify: μ + k = 3 + 1 = 4 = 1·L. ✓

Phase 2:
  p1 = head (node 1), p2 = meeting (node 5)
  step 1: p1=2, p2=6
  step 2: p1=3, p2=7
  step 3: p1=4, p2=4   <- entry
```

## Cycle Length

After phase 1, keep `slow` fixed and walk `fast` (or a fresh pointer) around the loop one step at a time until it returns to `slow`. The number of steps is `L`. This is occasionally asked as a follow-up and is trivially O(L) time.

## Middle Of A Linked List

Same setup, no cycle assumed. When `fast` reaches the end, `slow` is at the middle.

```
Odd length:   1 -> 2 -> 3 -> 4 -> 5 -> nil
              slow ends at 3.  (exact middle)

Even length:  1 -> 2 -> 3 -> 4 -> 5 -> 6 -> nil
              slow ends at 4.  (second of the two middles)
```

The termination condition determines which middle you get on an even-length list:

- `for fast != nil && fast.Next != nil` → `slow` lands on the **second** middle (4 in the example above). This is the LeetCode convention.
- `for fast.Next != nil && fast.Next.Next != nil` → `slow` lands on the **first** middle (3 in the example above). Useful when you need to split the list into two halves and want the left half to end cleanly.

The two variants are not interchangeable; pick based on what the caller needs to do with `slow` next.

## Kth Node From The End

Not really "fast and slow" in the ratio sense (both eventually move at speed 1), but a runner-technique variant that uses a controlled gap between two pointers to derive a positional landmark in one pass.

1. Advance `fast` by `k` nodes.
2. Then advance `slow` and `fast` together until `fast` reaches the end.
3. `slow` is now at the kth node from the end, because it stayed exactly `k` nodes behind `fast` throughout.

## Cycle Detection On A Function (Happy Number)

Floyd's trick generalises beyond linked lists. If `f` is a **deterministic function** (same input always gives the same output) on a **finite set of values** (the sequence cannot explore infinitely many distinct values), then the sequence `x, f(x), f(f(x)), ...` must eventually repeat and from there cycle forever (pigeonhole). Treat the sequence as if each value "points" to its successor and run the same two-speed scheme — `slow = f(slow)`, `fast = f(f(fast))` — to detect the cycle in O(1) space.

For linked lists both conditions are automatic: `node.Next` is deterministic (field read) and the list has finitely many nodes.

Happy Number (LeetCode #202) is the canonical instance: `f(n)` = sum of squares of digits. A number is "happy" if the sequence reaches 1; otherwise it cycles without hitting 1. Floyd detects the non-happy case without the hash set of seen values that the naive solution needs.

### Two Ways To Seed The Pointers

Two correct idioms differ in where the equality check sits.

**Option A — seed `fast` one step ahead, check at top of loop:**

```go
slow, fast := n, sumSquares(n)
for fast != 1 && slow != fast {
    slow = sumSquares(slow)
    fast = sumSquares(sumSquares(fast))
}
return fast == 1
```

**Option B — both start at `n`, check after moving:**

```go
slow, fast := n, n
for {
    slow = sumSquares(slow)
    fast = sumSquares(sumSquares(fast))
    if fast == 1 {
        return true
    }
    if slow == fast {
        return false
    }
}
```

Option A needs the seed because `slow != fast` is checked before any move. Starting with both at `n` would exit on iteration 0 with `fast == n`, wrong for any `n > 1`. Pre-advancing `fast` puts them at different positions so the top-of-loop check does not spuriously fire.

Option B moves first and checks after, mirroring the `HasCycle` template exactly.

**For interviews**, prefer Option B. One shape to memorise across linked lists and function iteration, less to get wrong under pressure.

## Recognizing The Pattern

- Linked list cycle question → Floyd's.
- Linked list middle, nth from end, or any one-pass positional landmark → runner variant.
- "O(1) extra space" on a linked list problem → almost always this pattern.
- Deterministic function on a finite domain, need to know if it cycles → generalised Floyd's.

Strongest single signal: "linked list" plus "constant extra space."

## Go Template Code

```go
package fastslow

// ListNode is the standard singly linked list node.
type ListNode struct {
	Val  int
	Next *ListNode
}

// HasCycle returns true if the list contains a cycle.
// Time: O(n) | Space: O(1)
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

// DetectCycle returns the node where the cycle begins, or nil if none.
// Time: O(n) | Space: O(1)
func DetectCycle(head *ListNode) *ListNode {
	slow, fast := head, head
	for fast != nil && fast.Next != nil {
		slow = slow.Next
		fast = fast.Next.Next
		if slow == fast {
			// Phase 2: walk one pointer from head, one from meeting point,
			// both at speed 1. They collide at the cycle entry.
			p := head
			for p != slow {
				p = p.Next
				slow = slow.Next
			}
			return p
		}
	}
	return nil
}

// CycleLength returns the length of the cycle, or 0 if none.
// Time: O(n) | Space: O(1)
func CycleLength(head *ListNode) int {
	slow, fast := head, head
	for fast != nil && fast.Next != nil {
		slow = slow.Next
		fast = fast.Next.Next
		if slow == fast {
			length := 1
			curr := slow.Next
			for curr != slow {
				curr = curr.Next
				length++
			}
			return length
		}
	}
	return 0
}

// MiddleNode returns the middle node of the list. For even length, returns
// the second of the two middles (LeetCode convention).
// Time: O(n) | Space: O(1)
func MiddleNode(head *ListNode) *ListNode {
	slow, fast := head, head
	for fast != nil && fast.Next != nil {
		slow = slow.Next
		fast = fast.Next.Next
	}
	return slow
}

// MiddleNodeLeft returns the first of the two middles for even-length lists.
// Useful when splitting the list in two and the left half must end cleanly.
// Time: O(n) | Space: O(1)
func MiddleNodeLeft(head *ListNode) *ListNode {
	if head == nil {
		return nil
	}
	slow, fast := head, head
	for fast.Next != nil && fast.Next.Next != nil {
		slow = slow.Next
		fast = fast.Next.Next
	}
	return slow
}

// RemoveNthFromEnd removes the kth node from the end and returns the new head.
// Uses a dummy head to handle removal of the first node uniformly.
// Time: O(n) | Space: O(1)
func RemoveNthFromEnd(head *ListNode, k int) *ListNode {
	dummy := &ListNode{Next: head}
	slow, fast := dummy, dummy
	// Move fast k+1 steps so slow ends up one before the target.
	for i := 0; i <= k; i++ {
		if fast == nil {
			return head // k larger than list length
		}
		fast = fast.Next
	}
	for fast != nil {
		slow = slow.Next
		fast = fast.Next
	}
	slow.Next = slow.Next.Next
	return dummy.Next
}

// IsHappy detects whether repeatedly summing squares of digits reaches 1,
// using Floyd's algorithm on the sequence x, f(x), f(f(x)), ...
// Time: O(log n) per step, small constant number of steps | Space: O(1)
func IsHappy(n int) bool {
	sumSquares := func(x int) int {
		s := 0
		for x > 0 {
			d := x % 10
			s += d * d
			x /= 10
		}
		return s
	}
	slow, fast := n, n
	for {
		slow = sumSquares(slow)
		fast = sumSquares(sumSquares(fast))
		if fast == 1 {
			return true
		}
		if slow == fast {
			return false
		}
	}
}
```

## Where JavaScript Differs

The algorithm is identical. The only surface difference is that JavaScript has no dedicated pointer type — you follow the `next` field on a plain object, which reads the same as Go's `node.Next`:

```javascript
function hasCycle(head) {
    let slow = head, fast = head;
    while (fast !== null && fast.next !== null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow === fast) return true;
    }
    return false;
}
```

Reference equality (`===`) is what compares the two node references, matching Go's `==` on pointer values. No semantics change.

## LeetCode Problems

### 1. Linked List Cycle — [LeetCode #141](https://leetcode.com/problems/linked-list-cycle/) · Easy

**Problem**

Given the `head` of a linked list, determine whether the list contains a cycle. A cycle exists if some node can be reached again by continuously following the `Next` pointer. Constraints: 0 to 10⁴ nodes.

**Brute Force**

Walk the list, inserting each visited node pointer into a hash set. If a node reappears, there is a cycle; if `nil` is reached first, there isn't. O(n) time, O(n) space.

<details>
<summary>Hint 1</summary>

A hash set of visited nodes solves this in O(n) space. Is there a way to detect a cycle using only two node references and constant extra space?
</details>

<details>
<summary>Hint 2</summary>

Advance two references through the list at different rates, one node per step and two nodes per step. If the list has no cycle, the faster one reaches nil. If it has a cycle, the faster one must eventually land on the same node as the slower one.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func hasCycle(head *ListNode) bool {
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
</details>

**Follow-Up: Can you solve it using O(1) (i.e. constant) memory?** Already satisfied by the solution above. The hash-set approach from Brute Force is the O(n)-space version this rules out; Floyd's two-speed walk needs only the `slow` and `fast` references.

### 2. Linked List Cycle II — [LeetCode #142](https://leetcode.com/problems/linked-list-cycle-ii/) · Medium

**Problem**

Given the `head` of a linked list, return the node where the cycle begins, or `nil` if there is no cycle. Do not modify the list. Constraints: 0 to 10⁴ nodes.

**Brute Force**

Hash set of visited node pointers; the first revisit is the cycle entry. O(n) time, O(n) space.

<details>
<summary>Hint 1</summary>

First detect that a cycle exists using the two-speed approach. What extra information is available at the meeting point that pins down where the cycle actually starts?
</details>

<details>
<summary>Hint 2</summary>

Let μ be the distance from head to cycle entry and L the cycle length. At the meeting point, the faster reference has taken exactly one extra lap. Algebra on the distances shows that starting a fresh reference at head and walking the meeting-point reference forward at the same speed makes them collide at the cycle entry.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func detectCycle(head *ListNode) *ListNode {
	slow, fast := head, head
	for fast != nil && fast.Next != nil {
		slow = slow.Next
		fast = fast.Next.Next
		if slow == fast {
			p := head
			for p != slow {
				p = p.Next
				slow = slow.Next
			}
			return p
		}
	}
	return nil
}
```
</details>

**Follow-Up: Can you solve it using O(1) (i.e. constant) memory?** Already satisfied. Floyd's two-phase algorithm uses only two pointers across both phases, no auxiliary set.

### 3. Middle Of The Linked List — [LeetCode #876](https://leetcode.com/problems/middle-of-the-linked-list/) · Easy

**Problem**

Given the `head` of a singly linked list, return the middle node. If there are two middle nodes (even length), return the **second** one. Constraints: 1 to 100 nodes.

**Brute Force**

Walk the list once to count the length `n`, then walk again from the head `n/2` steps to reach the middle. O(n) time, O(1) space, but two passes.

<details>
<summary>Hint 1</summary>

The obvious solution counts the length first and then walks again to the halfway index, two passes. Can a single pass do it by moving two references through the list at different rates?
</details>

<details>
<summary>Hint 2</summary>

Advance one reference by one node and another by two nodes per iteration. When the faster one runs off the end, the slower one is exactly at the halfway point. The loop condition determines which of the two middles you get on even-length lists.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func middleNode(head *ListNode) *ListNode {
	slow, fast := head, head
	for fast != nil && fast.Next != nil {
		slow = slow.Next
		fast = fast.Next.Next
	}
	return slow
}
```
</details>

### 4. Remove Nth Node From End Of List — [LeetCode #19](https://leetcode.com/problems/remove-nth-node-from-end-of-list/) · Medium

**Problem**

Given the `head` of a linked list of length `sz` and an integer `n` with `1 <= n <= sz`, remove the n-th node from the **end** of the list and return the head. Constraints: `1 <= sz <= 30`, node values in `[0, 100]`.

**Brute Force**

Walk the list once to count the length `L`, then walk again from the head to position `L - n` and splice out that node. O(n) time, O(1) space, but two passes.

<details>
<summary>Hint 1</summary>

Two passes work: measure the length, then walk to `length - n`. Can a single pass do the same by maintaining a fixed gap between two references?
</details>

<details>
<summary>Hint 2</summary>

Advance one reference n+1 steps ahead of the other, then advance both together until the leading reference falls off the end. The trailing reference is now positioned just before the node to delete. A dummy head simplifies the case where the node to delete is the first one.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func removeNthFromEnd(head *ListNode, n int) *ListNode {
	dummy := &ListNode{Next: head}
	slow, fast := dummy, dummy
	for i := 0; i <= n; i++ {
		fast = fast.Next
	}
	for fast != nil {
		slow = slow.Next
		fast = fast.Next
	}
	slow.Next = slow.Next.Next
	return dummy.Next
}
```
</details>

**Follow-Up: Could you do this in one pass?** Already satisfied. The two-pointer gap technique above visits each node exactly once; the brute-force length-then-walk approach is the two-pass version this rules out.

### 5. Happy Number — [LeetCode #202](https://leetcode.com/problems/happy-number/) · Easy

**Problem**

A positive integer `n` is **happy** if, starting from `n` and repeatedly replacing the number with the sum of the squares of its digits, the sequence eventually reaches `1`. Otherwise it loops endlessly in a cycle that never includes `1`. Return `true` if `n` is happy, else `false`. Constraints: `1 <= n <= 2^31 - 1`.

Example: `19 → 1² + 9² = 82 → 8² + 2² = 68 → 6² + 8² = 100 → 1² + 0² + 0² = 1`, so `19` is happy.

**Brute Force**

Hash set of seen values. Repeatedly apply `f` and insert each result; if `1` is reached, happy; if a repeat is seen before `1`, unhappy. O(k) time and O(k) space where `k` is the sequence length before termination.

<details>
<summary>Hint 1</summary>

The sequence of transformations must eventually revisit a value (there are only finitely many reachable states). The unhappy case is exactly the case where that repetition happens without ever hitting 1. Detecting the repetition in constant space is the key.
</details>

<details>
<summary>Hint 2</summary>

Treat the sequence x, f(x), f(f(x)), ... as if it were a linked list where each node points to its successor. Cycle detection with two references — one applying f once per step, the other applying it twice — either meets at a value equal to 1 (happy) or meets at some other value inside a cycle (unhappy).
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(log n) per step, terminates quickly | Space: O(1)
// Uses the move-first-then-check shape (Option B) to mirror the HasCycle template.
func isHappy(n int) bool {
	sumSquares := func(x int) int {
		s := 0
		for x > 0 {
			d := x % 10
			s += d * d
			x /= 10
		}
		return s
	}
	slow, fast := n, n
	for {
		slow = sumSquares(slow)
		fast = sumSquares(sumSquares(fast))
		if fast == 1 {
			return true
		}
		if slow == fast {
			return false
		}
	}
}
```
</details>

### 6. Palindrome Linked List — [LeetCode #234](https://leetcode.com/problems/palindrome-linked-list/) · Easy

**Problem**

Given the `head` of a singly linked list, return `true` if the sequence of values reads the same forwards and backwards. Constraints: 1 to 10⁵ nodes, values in `[0, 9]`.

**Brute Force**

Copy all node values into a slice, then check the slice with two pointers converging from both ends. O(n) time, O(n) space.

<details>
<summary>Hint 1</summary>

The O(n) space solution copies values into a slice and checks the slice with two pointers. The O(1) space solution needs to compare the first half against the second half without an auxiliary array. What structural operations on the list itself would make that comparison possible?
</details>

<details>
<summary>Hint 2</summary>

Find the middle of the list with two references at different speeds, reverse the second half in place, then walk one reference from the head and another from the reversed midpoint, comparing values. Optionally restore the second half before returning if the input must not be mutated.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func isPalindrome(head *ListNode) bool {
	// Find first middle (so left half ends cleanly for odd length).
	slow, fast := head, head
	for fast.Next != nil && fast.Next.Next != nil {
		slow = slow.Next
		fast = fast.Next.Next
	}
	// Reverse second half.
	var prev *ListNode
	curr := slow.Next
	for curr != nil {
		next := curr.Next
		curr.Next = prev
		prev = curr
		curr = next
	}
	// Compare halves.
	left, right := head, prev
	for right != nil {
		if left.Val != right.Val {
			return false
		}
		left = left.Next
		right = right.Next
	}
	return true
}
```
</details>

**Follow-Up: Could you do it in O(n) time and O(1) space?** Already satisfied. The solution above finds the midpoint, reverses the second half in place, and compares. The slice-and-two-pointer brute force is the O(n)-space version this rules out.

### 7. Reorder List — [LeetCode #143](https://leetcode.com/problems/reorder-list/) · Medium

**Problem**

Given the head of a singly linked list `L0 → L1 → ... → Ln-1 → Ln`, reorder it to `L0 → Ln → L1 → Ln-1 → L2 → Ln-2 → ...`. The reordering must be done in place by rearranging node links, not by swapping values. Constraints: 1 to 5·10⁴ nodes, values in `[-1000, 1000]`.

**Brute Force**

Copy node references into a slice, then build a new linking order by alternately taking from the front and back of the slice. O(n) time, O(n) space.

<details>
<summary>Hint 1</summary>

The interleaving of the front and back suggests splitting the list into two halves and merging them. How do you find the split point in one pass without knowing the length?
</details>

<details>
<summary>Hint 2</summary>

Find the middle using the two-speed technique, reverse the second half in place, then merge the two halves node by node, alternating between them.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func reorderList(head *ListNode) {
	if head == nil || head.Next == nil {
		return
	}
	// Find first middle.
	slow, fast := head, head
	for fast.Next != nil && fast.Next.Next != nil {
		slow = slow.Next
		fast = fast.Next.Next
	}
	// Reverse second half.
	var prev *ListNode
	curr := slow.Next
	slow.Next = nil
	for curr != nil {
		next := curr.Next
		curr.Next = prev
		prev = curr
		curr = next
	}
	// Merge.
	first, second := head, prev
	for second != nil {
		t1, t2 := first.Next, second.Next
		first.Next = second
		second.Next = t1
		first, second = t1, t2
	}
}
```
</details>

## References

- `dsa/patterns/two-pointers.md`
- `dsa/data-structures/linked-lists.md`
- `dsa/patterns/in-place-reversal.md`