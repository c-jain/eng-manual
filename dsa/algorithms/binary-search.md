---
Status: 🌳 Evergreen
Created: 2026-10-03
Last Updated: 2026-10-03
---

# Binary Search

## Table of Contents

1. [What It Is and Why It Exists](#what-it-is-and-why-it-exists)
2. [Why "Binary Search"](#why-binary-search)
3. [Monotonic Predicates: The Real Requirement](#monotonic-predicates-the-real-requirement)
4. [The Three Templates: Closed, Half-Open, Convergence](#the-three-templates-closed-half-open-convergence)
5. [The Overflow-Safe Midpoint](#the-overflow-safe-midpoint)
6. [Lower Bound and Upper Bound](#lower-bound-and-upper-bound)
7. [Search in Rotated Sorted Array](#search-in-rotated-sorted-array)
8. [Find Minimum in Rotated Sorted Array](#find-minimum-in-rotated-sorted-array)
9. [Search in a 2D Matrix](#search-in-a-2d-matrix)
10. [Binary Search on the Answer Space](#binary-search-on-the-answer-space)
11. [Complexity Reference](#complexity-reference)
12. [Recognizing the Pattern](#recognizing-the-pattern)
13. [Go Template Code](#go-template-code)
14. [Standard Library Binary Search in Go](#standard-library-binary-search-in-go)
15. [Where JavaScript Differs](#where-javascript-differs)
16. [LeetCode Problems](#leetcode-problems)
17. [References](#references)

## What It Is and Why It Exists

Binary search finds a target inside a **monotonic** search space by halving the space at every step. The default instinct for "is x in this list" is linear scan, O(n). Binary search exists because if the space has an ordering property, one comparison at the midpoint proves half the space can never contain the answer, so you throw that half away. Repeating this reduces n to 1 in `log2(n)` steps.

### The Problem It Solves

Any search over a monotonic space costs O(n) if scanned linearly. Halving costs O(log n). The bigger the space, the more valuable the trade: for n = 10^9, linear is a billion comparisons, binary is about 30.

## Why "Binary Search"

"Binary" is Latin for "of two." Each step splits the remaining space into two halves and discards one. Nothing more literary than that. It is sometimes called **half-interval search** or **logarithmic search** in older texts; both names describe the same operation. If a name helps, "the halving search" is the plainest.

## Monotonic Predicates: The Real Requirement

The most common misconception about binary search is that it requires a sorted array. It doesn't. It requires a **monotonic predicate** over some search space. Sorted arrays are just the most familiar case where such a predicate exists.

### What "Predicate" Means

A predicate is a yes/no question you ask about a point in the search space. Examples:

- Over an array of indices: "is `a[i] >= target`?"
- Over a range of possible speeds: "can Koko finish at speed `k` within `h` hours?"
- Over a matrix's row-major index: "is `m[i/cols][i%cols] >= target`?"

The search space is whatever set the predicate ranges over. It does not have to be a physical array.

### What "Monotonic" Means

The predicate's answer flips from false to true (or true to false) **exactly once** as you walk across the space. It never oscillates.

```
Space:       1   2   3   4   5   6   7   8   9   10
Predicate:   F   F   F   F   T   T   T   T   T   T
                             ^
                       one flip point
```

Binary search finds that flip point in O(log n), regardless of what the space physically is or how the predicate is computed.

### Why This Matters

Framing binary search as "sorted array + target" misses three common instances that look nothing like that on the surface:

- **Answer space (Koko, LC 875).** You're picking a speed `k` from `1..max(piles)`. Predicate: `canFinish(k)`. Faster speeds always finish, slower speeds sometimes don't. FFFF...TTTT holds over speed values. Binary search finds the smallest `k` where it flips to T. No array involved.

- **Rotated array (LC 33).** `[4,5,6,7,0,1,2]` is not globally sorted, so `a[i] >= target` isn't monotonic across the whole thing. But a different predicate is: at any midpoint, "is the target in the sorted half?" is decidable in O(1), and it partitions the remaining space cleanly. The monotonic structure is present, just phrased differently.

- **2D matrix flattened row-major (LC 74).** The matrix isn't a 1D array, but if you treat indices `0..rows*cols-1` as a virtual sorted 1D space with values fetched via `(i/cols, i%cols)`, the same predicate as classic binary search applies.

### The One-Sentence Version

Binary search works whenever you can define a yes/no question over some space that flips exactly once. Sorted arrays are the most familiar example of that, not the definition.

## The Three Templates: Closed, Half-Open, Convergence

There are three conventions for the loop bounds, and mixing them up is the single most common source of off-by-one bugs. Pick one per function and internalise its invariant. The three differ in how they define the live window and how the loop exits:

| Template | Bounds | Exit | Use when |
|----------|--------|------|----------|
| A — Closed | `[left, right]`, `right = len-1` | `left > right` (empty) | Exact-match lookup; answer is a known value or `-1`. |
| B — Half-Open | `[left, right)`, `right = len` | `left == right` (empty) | Boundary / insertion point; answer may be `len(a)` (past-the-end). |
| C — Convergence | `[left, right]`, `right = len-1` | `left == right` (one element) | Find the index of an element defined by a monotonic property, when the value isn't known in advance. |

### Template A — Closed Interval: `[left, right]`, `while left <= right`

The window under consideration is `[left, right]` inclusive on both ends. When `left > right` the window is empty, so the loop exits. Every non-answer index is excluded by `mid + 1` or `mid - 1`.

```
Scenario: Search 7 in [1, 3, 5, 7, 9, 11], Template A

initial:  left=0                          right=5   mid=2   a[mid]=5 < 7 -> left = mid+1
[ 1  3  5  7  9 11 ]
              ^--^
step 2:               left=3              right=5   mid=4   a[mid]=9 > 7 -> right = mid-1
[ 1  3  5  7  9 11 ]
              ^
step 3:               left=3   right=3               mid=3   a[mid]=7 == 7 -> return 3
```

Use Template A when the answer is guaranteed to be an existing element and you want the exact match or `-1`.

### Template B — Half-Open Interval: `[left, right)`, `while left < right`

The window is `[left, right)` — `right` is one past the last candidate. When `left == right` the window is empty. `left = mid + 1` says "I've looked at `mid` and rejected it, so start the next window strictly after `mid`." `right = mid` says "`mid` is not the answer as a value, but the answer might still be at position `mid` — keep it as the right edge." On exit `left == right` and that shared value is the answer position.

```
Scenario: lowerBound(2) in [1, 2, 2, 2, 3, 5], Template B

initial:  left=0                                right=6   mid=3   a[mid]=2 >= 2 -> right = mid
[ 1  2  2  2  3  5 ]
              ^     right moves down to 3
step 2:      left=0        right=3               mid=1   a[mid]=2 >= 2 -> right = mid
[ 1  2  2  2  3  5 ]
     ^              right moves down to 1
step 3:  left=0 right=1                          mid=0   a[mid]=1 < 2 -> left = mid+1
[ 1  2  2  2  3  5 ]
     ^              left moves to 1
exit:  left == right == 1                        answer: 1
```

A boundary problem is any question shaped as "find the index where something changes." Template B is designed for exactly that: it converges `left` and `right` onto the flip point of a monotonic predicate. Template A can't do this cleanly because it exits when `left > right`, so there's no single index both converge to.

### Template C — Convergence: `[left, right]`, `while left < right`

Closed-interval bounds (`right = len(nums) - 1`) with Template B's exit condition. The invariant is "the answer index is somewhere in `[left, right]` inclusive," and the two pointers *converge* onto a single index that must be the answer.

```go
left, right := 0, len(nums)-1
for left < right {
    mid := left + (right-left)/2
    if predicate(mid) {
        right = mid         // keep mid as a candidate
    } else {
        left = mid + 1      // discard mid
    }
}
return left                 // left == right, the answer index
```

Key differences from the other two:

- `right` always points at a real valid index, never past-the-end. This matters when `nums[right]` is read during the predicate (e.g., `nums[mid] > nums[right]` in LC 153 — can't do that if `right == len(nums)`).
- `right = mid` keeps `mid` as a live candidate instead of discarding it, so the answer index is never lost.
- The loop exits with the window collapsed to one element, not empty, so `nums[left]` is directly the answer.

### Rule of Thumb

| Use template | When |
|--------------|------|
| A — Closed | Exact-match lookup on a known value. Returns index or `-1`. |
| B — Half-Open | Boundary / insertion point / answer-space search. Answer may equal `len(a)` (past-the-end). |
| C — Convergence | Find the index of an element whose value isn't known in advance, defined by a monotonic property (pivot, peak, minimum in rotated array). |

Do not switch templates mid-function.

## The Overflow-Safe Midpoint

Never write `mid := (left + right) / 2`. In fixed-width integer types, `left + right` can overflow when both are large. In Go, `int` is 64-bit on modern targets so pure LeetCode inputs rarely trigger it, but the habit is wrong to carry into production code (32-bit systems, `int32` fields from a database row, etc.). Use:

```go
mid := left + (right-left)/2
```

`right - left` is a non-negative difference of two values already known to fit, so the addition cannot overflow. Same result mathematically, no overflow risk. This is the version to write everywhere.

## Lower Bound and Upper Bound

Two boundary primitives that most binary-search problems reduce to:

- **Lower bound** of target: smallest index `i` such that `a[i] >= target`.
- **Upper bound** of target: smallest index `i` such that `a[i] > target`.

Together they bracket every occurrence of `target`: the range `[lowerBound(target), upperBound(target))` contains exactly the copies of `target`. Count of occurrences is `upperBound(target) - lowerBound(target)`.

Any index in `[lowerBound(target), upperBound(target)]` is a valid insertion point that keeps the array sorted. `lowerBound` places the new copy **before** existing duplicates, `upperBound` places it **after**. Both are correct; the choice only matters when you care about the relative order of equal elements (stability). Go's `sort.SearchInts` returns `lowerBound` by convention, not because the other positions are wrong.

```
Array:              [ 1  2  2  2  3  5 ]
Index:                0  1  2  3  4  5

lowerBound(2) = 1     (first index with value >= 2)
upperBound(2) = 4     (first index with value >  2)

Occurrences of 2:  indices [1, 4)  ->  3 copies
```

If `target` is larger than every element, both bounds return `len(a)`. That's the sentinel meaning "would be inserted at the end."

## Search in Rotated Sorted Array

A sorted array rotated at some unknown pivot: `[4, 5, 6, 7, 0, 1, 2]`. The array is no longer globally sorted, but a key property holds: **at least one of the two halves around any midpoint is sorted.** Compare `a[left]` to `a[mid]`; if `a[left] <= a[mid]`, the left half is sorted; otherwise the right half is sorted. Once you know which half is sorted, you can decide in O(1) whether the target lies inside it (range check against its two ends) and shrink accordingly.

```
Scenario: Search 0 in [4, 5, 6, 7, 0, 1, 2]

initial: left=0, right=6, mid=3, a[mid]=7
  a[left]=4 <= a[mid]=7  ->  left half [4,5,6,7] is sorted
  is 0 in [4, 7)?  no  ->  discard left half, left = mid + 1 = 4

step 2:  left=4, right=6, mid=5, a[mid]=1
  a[left]=0 <= a[mid]=1  ->  left half [0,1] is sorted
  is 0 in [0, 1)?  yes  ->  discard right half, right = mid - 1 = 4

step 3:  left=4, right=4, mid=4, a[mid]=0  ->  match, return 4
```

This is the archetypal "binary search on structure, not values" problem.

**Why `<=` and not `<` in the distinct case:** when the window shrinks to two elements, `mid = left` (floor division), so `a[left] == a[mid]` even with all distinct values. The "left half" is then a single element, trivially sorted. Using `<=` correctly handles that degenerate window; strict `<` would send the loop down the wrong branch.

**Why duplicates still break the trick:** with distinct values, `a[left] == a[mid]` only happens when `left == mid` (the harmless degenerate case). With duplicates, `a[left] == a[mid]` can also happen when `left < mid` with the rotation point sitting *inside* the left half — e.g. in `[1, 0, 1, 1, 1]` with `left=0, mid=2`, both are `1`, but the "left half" `[1, 0, 1]` isn't sorted. The `<=` check wrongly declares it sorted, the range check excludes `target=0`, and the real answer gets discarded. LC 81's fix is to detect `a[left] == a[mid] == a[right]`, shrink both ends by one, and retry — giving O(n) worst case.

## Find Minimum in Rotated Sorted Array

Same setup, different question: where is the pivot? The minimum is the pivot. The trick is to compare `a[mid]` against `a[right]`, not `a[left]`:

- If `a[mid] > a[right]`, the minimum must be strictly to the right of `mid`. Set `left = mid + 1`.
- Otherwise, the minimum is at `mid` or to its left. Set `right = mid` (keep `mid` as a candidate).

Comparing against `a[right]` avoids an edge case that arises with `a[left]` when the array is not rotated at all. Loop until `left == right`, then `a[left]` is the minimum. This uses Template C — the convergence flavour, because `a[right]` must stay a readable index throughout.

## Search in a 2D Matrix

For a matrix where each row is sorted and the first element of each row is greater than the last element of the previous row (LC 74), the whole thing is really a flat sorted array laid out row-major. Treat indices `0` to `rows*cols - 1` as your search space, and map `mid` back to `(mid / cols, mid % cols)`. One binary search, O(log(rows * cols)).

For a matrix where each row is sorted and each column is sorted independently but the "flat" view is not sorted (LC 240), binary search doesn't apply directly. The staircase trick — start at top-right, move down if smaller, left if larger — is the standard O(rows + cols) solution. Do not force binary search onto it.

## Binary Search on the Answer Space

The most powerful application, and the one that separates people who "know binary search" from people who reach for it as a tool. The setup:

1. You need a value `k` (a speed, a capacity, a distance, a count) that is the smallest or largest satisfying some property.
2. The property `f(k)` is monotonic: if `f(k)` is true, then `f(k+1)` is also true (or the mirror image).
3. You don't have a direct formula for `k`, but given a candidate `k`, you can **check** `f(k)` in reasonable time (often linear).

Binary search the range of possible `k` values, calling `f(mid)` at each step. This turns "search for the answer" into "check if a guess works," which is often much easier to reason about.

The classic example is Koko Eating Bananas (LC 875). Given piles of bananas and `h` hours, find the minimum eating speed `k` such that eating `ceil(pile / k)` from each pile finishes within `h` hours.

- Search space: speeds `1` to `max(piles)`.
- Predicate `canFinish(k)`: true if summed hours at speed `k` is `<= h`.
- Monotonic: if speed `k` finishes in time, so does any speed `> k`. The predicate flips from false to true exactly once.
- Answer: lower bound of the "true" region — the smallest `k` for which `canFinish(k)` is true.

```
piles = [3, 6, 7, 11], h = 8
speed:      1    2    3    4    5    6    7    8    9   10   11
canFinish?  F    F    F    T    T    T    T    T    T    T    T
                          ^
                        boundary -> answer = 4
```

Structurally identical to lower bound, just over an integer range with a computed predicate instead of an array lookup.

### Common Answer-Space Patterns

- Minimum capacity to ship packages in D days (LC 1011).
- Split array largest sum minimum (LC 410).
- Minimum number of days to make M bouquets (LC 1482).
- Median of two sorted arrays (LC 4) — binary searches the partition point rather than a value.

## Complexity Reference

| Operation | Time | Space |
|-----------|------|-------|
| Classic binary search | O(log n) | O(1) |
| Lower / upper bound | O(log n) | O(1) |
| Rotated sorted array (distinct) | O(log n) | O(1) |
| Rotated sorted array (duplicates, LC 81) | O(n) worst | O(1) |
| 2D matrix, row-major sorted (LC 74) | O(log(mn)) | O(1) |
| 2D matrix, row+col sorted (LC 240) | O(m + n) staircase | O(1) |
| Answer space, k candidates, predicate cost `p` | O(p log k) | O(1) |

The `log` is base 2 always; log₂(10^9) ≈ 30, log₂(10^18) ≈ 60. Useful mental anchors.

## Recognizing the Pattern

- Input is sorted, or you can define a predicate over some space that flips exactly once.
- Problem asks for a specific value, a boundary, a count of occurrences, or an insertion position.
- Problem asks for the "minimum k such that…" or "maximum k such that…" with a checkable predicate.
- Problem gives you a rotated / partially-sorted structure where one half is always sorted.
- Naive solution is O(n) scan and constraints suggest that's too slow (n up to 10^5 or more, with multiple queries).

If none of these fit, binary search probably isn't the tool.

## Go Template Code

```go
// Template A: exact-match search in a sorted slice. Returns -1 if not found.
func binarySearch(a []int, target int) int {
    left, right := 0, len(a)-1
    for left <= right {
        mid := left + (right-left)/2
        if a[mid] == target {
            return mid
        }
        if a[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    return -1
}

// Template B: smallest index i such that a[i] >= target. Returns len(a) if none.
func lowerBound(a []int, target int) int {
    left, right := 0, len(a)
    for left < right {
        mid := left + (right-left)/2
        if a[mid] < target {
            left = mid + 1
        } else {
            right = mid
        }
    }
    return left
}

// Template B variant: smallest index i such that a[i] > target.
func upperBound(a []int, target int) int {
    left, right := 0, len(a)
    for left < right {
        mid := left + (right-left)/2
        if a[mid] <= target {
            left = mid + 1
        } else {
            right = mid
        }
    }
    return left
}

// Answer-space skeleton: find smallest k in [lo, hi] such that ok(k) is true.
// ok must be monotonic: if ok(k) then ok(k+1).
func binarySearchAnswer(lo, hi int, ok func(int) bool) int {
    for lo < hi {
        mid := lo + (hi-lo)/2
        if ok(mid) {
            hi = mid
        } else {
            lo = mid + 1
        }
    }
    return lo
}
```

Note the standard library also has `sort.Search(n, f)` which is exactly the answer-space skeleton over `[0, n)`. In production, prefer it. In interviews, write the skeleton yourself so the interviewer sees you understand the invariant.

### Standard Library Binary Search in Go

Go ships four binary-search primitives. Know which does what so you don't reach for the wrong one in production and so interview conversations stay accurate.

| Function | Package | Returns | Semantics |
|----------|---------|---------|-----------|
| `sort.SearchInts(a, x)` | `sort` | `int` | Lower bound: smallest `i` such that `a[i] >= x`. Returns `len(a)` if none. |
| `sort.SearchStrings(a, x)` | `sort` | `int` | Same as above, for `[]string`. |
| `sort.Search(n, f)` | `sort` | `int` | Smallest `i` in `[0, n)` where `f(i)` is true. The generic answer-space skeleton. `f` must be monotonic (false then true). |
| `slices.BinarySearch(s, x)` | `slices` (Go 1.21+) | `(int, bool)` | Lower bound **plus** a boolean saying whether `x` was actually present. |
| `slices.BinarySearchFunc(s, t, cmp)` | `slices` (Go 1.21+) | `(int, bool)` | Same as above but with a custom comparator — useful for sorted slices of structs. |

```go
a := []int{1, 2, 2, 2, 3, 5}

sort.SearchInts(a, 2)         // 1    (lower bound)
sort.SearchInts(a, 4)         // 5
sort.SearchInts(a, 9)         // 6    (= len(a), sentinel for "not present, insert at end")

// Upper bound isn't in the stdlib — build it from sort.Search:
sort.Search(len(a), func(i int) bool { return a[i] > 2 }) // 4

// slices.BinarySearch distinguishes "found at i" from "would insert at i":
i, found := slices.BinarySearch(a, 2) // 1, true
i, found = slices.BinarySearch(a, 4)  // 5, false
```

Pointers worth remembering:

- **Only `lowerBound` ships directly.** There is no `sort.UpperBound`. Build it with `sort.Search(n, func(i int) bool { return a[i] > x })`.
- **`sort.Search` is the general tool.** Any monotonic predicate — including answer-space searches like Koko — fits it. The input doesn't need to be a slice. Example: find the smallest capacity between 1 and 10^9 that can ship packages in `d` days: `sort.Search(1_000_000_001, canShip)`.
- **`slices.BinarySearch` is the modern form** for exact-match lookup on a sorted slice. The `(int, bool)` return is strictly more useful than `sort.SearchInts`'s single `int`, because you don't have to double-check `i < len(a) && a[i] == target` yourself.
- **The slice/array must already be sorted ascending.** None of these verify that; passing an unsorted slice silently returns garbage.
- **For custom types**, use `slices.BinarySearchFunc` or wrap `sort.Search` with your own index predicate. There's no need to implement the loop by hand in production.

In interviews, write the loop by hand unless the interviewer explicitly asks you to use the stdlib — showing you understand the invariant matters more than brevity. In production, use `slices.BinarySearch` for exact match and `sort.Search` for everything else.

## Where JavaScript Differs

JavaScript has no equivalent of `sort.SearchInts` in its standard library, so lower-bound and upper-bound must be handwritten. The bigger difference is numeric: JavaScript numbers are IEEE-754 doubles, so `(left + right) / 2` on values near 2^53 loses precision even before overflow becomes the concern. Using `Math.floor(left + (right - left) / 2)` avoids both. On typed arrays or BigInt work the overflow-safe form is still the right default.

## LeetCode Problems

### 1. Binary Search — [#704 (Easy)](https://leetcode.com/problems/binary-search/)

Given a sorted array of distinct integers and a target, return its index or `-1` if it's not present.

<details>
<summary>Brute Force</summary>

Scan left to right and return the first match. Time: O(n). Space: O(1).

The waste: sortedness is thrown away. At every step you learn only whether one element matches, not which half of the remaining array to eliminate.
</details>

<details>
<summary>Hint 1</summary>

You have a monotonic property (the array is sorted). What can one midpoint comparison prove about the rest?
</details>

<details>
<summary>Hint 2</summary>

Compare `a[mid]` to `target`. If they're equal, done. If `a[mid] < target`, the target (if present) must be in `[mid+1, right]`. Otherwise `[left, mid-1]`.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(log n) | Space: O(1)
func search(nums []int, target int) int {
    left, right := 0, len(nums)-1
    for left <= right {
        mid := left + (right-left)/2
        if nums[mid] == target {
            return mid
        }
        if nums[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    return -1
}
```

Template A: the answer is either an existing element or doesn't exist at all, so `[left, right]` inclusive is the natural window.
</details>

### 2. Search Insert Position — [#35 (Easy)](https://leetcode.com/problems/search-insert-position/)

Given a sorted array of distinct integers and a target, return the index where `target` is found, or the index where it would be inserted to keep the array sorted.

<details>
<summary>Brute Force</summary>

Scan left to right until you find the first index `i` with `nums[i] >= target`. If no such index, return `len(nums)`. Time: O(n). Space: O(1).

The waste: same as classic search — monotonicity is unused.
</details>

<details>
<summary>Hint 1</summary>

This is asking for the smallest index `i` such that `nums[i] >= target`. That phrasing should sound familiar.
</details>

<details>
<summary>Hint 2</summary>

It's lower bound. Use Template B: `[left, right)` half-open, `while left < right`. On exit, `left` is the answer, whether the target was found or not.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(log n) | Space: O(1)
func searchInsert(nums []int, target int) int {
    left, right := 0, len(nums)
    for left < right {
        mid := left + (right-left)/2
        if nums[mid] < target {
            left = mid + 1
        } else {
            right = mid
        }
    }
    return left
}
```

No special case for "not found" — Template B's exit condition covers both the found and the insertion-point case with a single return.
</details>

### 3. Find First and Last Position of Element in Sorted Array — [#34 (Medium)](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array/)

Given a sorted array (may contain duplicates) and a target, return the first and last indices of `target`, or `[-1, -1]` if not present.

<details>
<summary>Brute Force</summary>

Linear scan tracking first and last match indices. Time: O(n). Space: O(1).

The waste: two independent boundary queries (first and last occurrence) each solvable in O(log n).
</details>

<details>
<summary>Hint 1</summary>

The first index of `target` is exactly `lowerBound(target)` (if it points to a value equal to `target`). What's the last index in terms of the two bounds?
</details>

<details>
<summary>Hint 2</summary>

`upperBound(target)` gives the first index with value strictly greater. So the last index of `target` is `upperBound(target) - 1`. If `lowerBound == upperBound`, `target` isn't in the array.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(log n) | Space: O(1)
func searchRange(nums []int, target int) []int {
    lowerBound := func() int {
        l, r := 0, len(nums)
        for l < r {
            m := l + (r-l)/2
            if nums[m] < target {
                l = m + 1
            } else {
                r = m
            }
        }
        return l
    }
    upperBound := func() int {
        l, r := 0, len(nums)
        for l < r {
            m := l + (r-l)/2
            if nums[m] <= target {
                l = m + 1
            } else {
                r = m
            }
        }
        return l
    }
    lo, hi := lowerBound(), upperBound()
    if lo == hi {
        return []int{-1, -1}
    }
    return []int{lo, hi - 1}
}
```

Two independent Template B searches with different comparison operators (`<` vs `<=`). That single-character difference is the entire distinction between lower and upper bound.
</details>

### 4. Search in Rotated Sorted Array — [#33 (Medium)](https://leetcode.com/problems/search-in-rotated-sorted-array/)

A sorted array of distinct integers has been rotated at an unknown pivot. Given the rotated array and a target, return its index or `-1`.

<details>
<summary>Brute Force</summary>

Linear scan. Time: O(n). Space: O(1).

The waste: the array isn't globally sorted, so classical binary search doesn't apply directly. But at every midpoint, at least one of the two halves *is* sorted — that structure is enough to prune half the space per step.
</details>

<details>
<summary>Hint 1</summary>

At `mid`, one of `[left, mid]` or `[mid, right]` is guaranteed to be a normally sorted subarray. How do you tell which?
</details>

<details>
<summary>Hint 2</summary>

If `a[left] <= a[mid]`, the left half is sorted. Otherwise the right half is sorted. Once you know which half is sorted, a simple range check tells you whether `target` lies in it.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(log n) | Space: O(1)
func search(nums []int, target int) int {
    left, right := 0, len(nums)-1
    for left <= right {
        mid := left + (right-left)/2
        if nums[mid] == target {
            return mid
        }
        if nums[left] <= nums[mid] { // left half sorted
            if nums[left] <= target && target < nums[mid] {
                right = mid - 1
            } else {
                left = mid + 1
            }
        } else { // right half sorted
            if nums[mid] < target && target <= nums[right] {
                left = mid + 1
            } else {
                right = mid - 1
            }
        }
    }
    return -1
}
```

The predicate isn't "sorted array + comparison" any more — it's "target is in the sorted half." Same halving, different framing.
</details>

### 5. Find Minimum in Rotated Sorted Array — [#153 (Medium)](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array/)

A sorted array of distinct integers has been rotated at an unknown pivot. Return the minimum element.

<details>
<summary>Brute Force</summary>

Linear scan for the minimum. Time: O(n). Space: O(1).

The waste: the minimum is the pivot, and the pivot is exactly the boundary between the two sorted segments. That's a monotonic boundary you can binary-search.
</details>

<details>
<summary>Hint 1</summary>

Think of the array as split at the pivot into two sorted segments. Every element in the right segment is smaller than every element in the left segment. What comparison tells you which segment `mid` is in?
</details>

<details>
<summary>Hint 2</summary>

Compare `a[mid]` to `a[right]`. If `a[mid] > a[right]`, `mid` is in the left (larger) segment — the minimum is strictly to the right. Otherwise `mid` is in the right (smaller) segment or is the minimum itself — keep it as a candidate.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(log n) | Space: O(1)
func findMin(nums []int) int {
    left, right := 0, len(nums)-1
    for left < right {
        mid := left + (right-left)/2
        if nums[mid] > nums[right] {
            left = mid + 1
        } else {
            right = mid
        }
    }
    return nums[left]
}
```

This is **Template C — the convergence template**. `right = len(nums)-1` is a real valid index (needed so `nums[right]` can be read as the comparison anchor), `left < right` exits when the window collapses to one index, and `right = mid` keeps `mid` as a live candidate so the minimum is never discarded.

Comparing against `a[right]` instead of `a[left]` is also deliberate: with `a[left]`, a non-rotated array (already sorted) becomes a special case. Against `a[right]`, it just works.

An equally valid Template A phrasing tracks the minimum as a running variable, which lets you safely do `right = mid - 1` without losing the midpoint value:

```go
// Time: O(log n) | Space: O(1)
func findMin(nums []int) int {
    left, right := 0, len(nums)-1
    smallest := math.MaxInt
    for left <= right {
        mid := left + (right-left)/2
        if nums[mid] <= nums[right] {
            if nums[mid] < smallest {
                smallest = nums[mid]
            }
            right = mid - 1
        } else {
            left = mid + 1
        }
    }
    return smallest
}
```

Both are correct, same O(log n). The convergence template is more idiomatic for "find a boundary index" problems because it returns the answer directly without a running variable; the Template A version is more idiomatic when you want to keep the shrink-past-mid symmetry.
</details>

### 6. Search a 2D Matrix — [#74 (Medium)](https://leetcode.com/problems/search-a-2d-matrix/)

Given an `m x n` matrix where each row is sorted and the first element of each row is greater than the last of the previous row, determine whether `target` is in the matrix.

<details>
<summary>Brute Force</summary>

Scan every cell. Time: O(mn). Space: O(1).

The waste: the constraints say the matrix is really a flat sorted array wrapped as rows. All the sorted-array machinery applies as-is.
</details>

<details>
<summary>Hint 1</summary>

Pretend the matrix is a 1D sorted array of length `m*n`. Given a virtual index `i` in that array, which cell does it correspond to?
</details>

<details>
<summary>Hint 2</summary>

Cell at row `i / cols`, column `i % cols`. Now run a classic binary search over `[0, m*n - 1]`.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(log(m*n)) | Space: O(1)
func searchMatrix(matrix [][]int, target int) bool {
    if len(matrix) == 0 || len(matrix[0]) == 0 {
        return false
    }
    rows, cols := len(matrix), len(matrix[0])
    left, right := 0, rows*cols-1
    for left <= right {
        mid := left + (right-left)/2
        v := matrix[mid/cols][mid%cols]
        if v == target {
            return true
        }
        if v < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    return false
}
```

Nothing 2D-specific in the logic — only the index mapping. This is the "search space is virtual" flavour of binary search.
</details>

### 7. Koko Eating Bananas — [#875 (Medium)](https://leetcode.com/problems/koko-eating-bananas/)

Given `piles[i]` bananas per pile and `h` hours, find the minimum integer eating speed `k` such that eating `ceil(piles[i] / k)` from each pile finishes within `h` hours.

<details>
<summary>Brute Force</summary>

Try every speed from 1 upward until one works. Time: O(max(piles) · n). Space: O(1).

The waste: the "does speed `k` work?" predicate is monotonic — if `k` works, every faster speed also works. That's exactly what binary search on the answer space is for.
</details>

<details>
<summary>Hint 1</summary>

You can't compute `k` directly, but given a candidate `k` you can check whether it finishes in time in O(n). Where does that check sit in a monotonic pattern?
</details>

<details>
<summary>Hint 2</summary>

For speed `k`, hours needed = sum over piles of `ceil(pile / k)`. The predicate `hours <= h` is false for small `k` and true for large `k`, flipping exactly once. Binary-search the smallest `k` where it flips to true. Search range: `[1, max(piles)]`.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n * log(max(piles))) | Space: O(1)
func minEatingSpeed(piles []int, h int) int {
    maxPile := 0
    for _, p := range piles {
        if p > maxPile {
            maxPile = p
        }
    }
    canFinish := func(k int) bool {
        hours := 0
        for _, p := range piles {
            hours += (p + k - 1) / k // ceil(p / k) for positive ints
            if hours > h {
                return false
            }
        }
        return true
    }
    left, right := 1, maxPile
    for left < right {
        mid := left + (right-left)/2
        if canFinish(mid) {
            right = mid
        } else {
            left = mid + 1
        }
    }
    return left
}
```

Template B on an integer range with a computed predicate. The early exit inside `canFinish` when `hours > h` avoids overflow on adversarial inputs.
</details>

### 8. Capacity To Ship Packages Within D Days — [#1011 (Medium)](https://leetcode.com/problems/capacity-to-ship-packages-within-d-days/)

Given weights loaded onto a ship in order and a deadline of `days` days, return the minimum ship capacity such that all packages can be shipped in order within the deadline.

<details>
<summary>Brute Force</summary>

Try every capacity from `max(weights)` upward until one works. Time: O(sum(weights) · n) worst case. Space: O(1).

The waste: same monotonic answer-space structure as Koko.
</details>

<details>
<summary>Hint 1</summary>

What's the minimum possible capacity? What's the maximum you'd ever need?
</details>

<details>
<summary>Hint 2</summary>

Capacity must be at least `max(weights)` (else the largest package doesn't fit) and at most `sum(weights)` (single day, everything at once). The predicate `canShip(cap)` — greedily pack until you'd exceed `cap`, then start a new day — is monotonic in `cap`.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n * log(sum(weights))) | Space: O(1)
func shipWithinDays(weights []int, days int) int {
    lo, hi := 0, 0
    for _, w := range weights {
        if w > lo {
            lo = w
        }
        hi += w
    }
    canShip := func(cap int) bool {
        used, need := 0, 1
        for _, w := range weights {
            if used+w > cap {
                need++
                used = w
            } else {
                used += w
            }
        }
        return need <= days
    }
    for lo < hi {
        mid := lo + (hi-lo)/2
        if canShip(mid) {
            hi = mid
        } else {
            lo = mid + 1
        }
    }
    return lo
}
```

Same skeleton as Koko. Only the predicate changes. Once you spot the answer-space pattern, most of these problems collapse to "define the range, define `canDo(k)`, run Template B."
</details>

### 9. Search in Rotated Sorted Array II — [#81 (Medium)](https://leetcode.com/problems/search-in-rotated-sorted-array-ii/)

Same as LC 33 but the array may contain duplicates. Return whether `target` is present.

<details>
<summary>Brute Force</summary>

Linear scan. Time: O(n). Space: O(1).

The waste: most of the time the rotated-array trick still works. Duplicates only break it in a narrow case, and even then you can side-step them cheaply.
</details>

<details>
<summary>Hint 1</summary>

In LC 33, `a[left] <= a[mid]` proved the left half is sorted. What happens when `a[left] == a[mid]` and `a[mid] == a[right]` at the same time?
</details>

<details>
<summary>Hint 2</summary>

When `a[left] == a[mid] == a[right]`, you can't tell which half is sorted (e.g., `[1,1,1,0,1]` vs `[1,0,1,1,1]`). Sidestep by shrinking both ends by one and retrying. Otherwise the LC 33 logic applies unchanged — the problem's input guarantee (actually a rotation of a sorted array) means `a[left] <= a[mid]` still reliably implies the left half is sorted outside the three-way-equal case.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(log n) average, O(n) worst (all duplicates) | Space: O(1)
func search(nums []int, target int) bool {
    left, right := 0, len(nums)-1
    for left <= right {
        mid := left + (right-left)/2
        if nums[mid] == target {
            return true
        }
        if nums[left] == nums[mid] && nums[mid] == nums[right] {
            left++
            right--
        } else if nums[left] <= nums[mid] { // left half sorted
            if nums[left] <= target && target < nums[mid] {
                right = mid - 1
            } else {
                left = mid + 1
            }
        } else { // right half sorted
            if nums[mid] < target && target <= nums[right] {
                left = mid + 1
            } else {
                right = mid - 1
            }
        }
    }
    return false
}
```

**Why `a[left] <= a[mid]` is still sufficient here**, even with duplicates: the problem guarantees the input is a rotation of a sorted array. Under that guarantee, if `a[left] == a[mid]` and the three-way-equal shrink branch didn't trigger (i.e. `a[mid] != a[right]`), the left half is still provably sorted. If it weren't, the rotation would sit inside the left half, which would force some element strictly less than `a[left]` to appear in `a[left..mid]` — making `a[mid] < a[left]`, contradicting the equality. The all-three-equal check handles the one ambiguous configuration cleanly.

The O(n) worst case comes from arrays like `[1,1,1,1,1]` where every step triggers the shrink-both-ends branch. Can't do better without extra assumptions — that's an information-theoretic limit, not a coding failure.
</details>

### 10. Find Peak Element — [#162 (Medium)](https://leetcode.com/problems/find-peak-element/)

Given an array where `nums[-1]` and `nums[n]` are treated as `-infinity`, find any index `i` such that `nums[i] > nums[i-1]` and `nums[i] > nums[i+1]`. Adjacent elements are guaranteed distinct.

<details>
<summary>Brute Force</summary>

Scan until you find an element greater than both neighbours. Time: O(n). Space: O(1).

The waste: the array isn't sorted, but the *slope* between adjacent elements is a monotonic-like signal. You can navigate uphill and always land on a peak.
</details>

<details>
<summary>Hint 1</summary>

At `mid`, compare `nums[mid]` to `nums[mid+1]`. If `nums[mid] > nums[mid+1]`, is there guaranteed to be a peak on one particular side?
</details>

<details>
<summary>Hint 2</summary>

If `nums[mid] > nums[mid+1]`, a peak exists in `[left, mid]` (walking left from `mid` you must eventually hit a peak since the boundary is `-inf`). Otherwise a peak exists in `[mid+1, right]`. Template C on that principle — the window converges onto the peak index.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(log n) | Space: O(1)
func findPeakElement(nums []int) int {
    left, right := 0, len(nums)-1
    for left < right {
        mid := left + (right-left)/2
        if nums[mid] > nums[mid+1] {
            right = mid
        } else {
            left = mid + 1
        }
    }
    return left
}
```

The "predicate" here is `nums[mid] > nums[mid+1]` — false at the left of a peak and true at/after it. Not the FFFF...TTTT of a sorted array, but the same halving still isolates a valid index because the boundaries are `-inf`.
</details>

### 11. Find the Duplicate Number — [#287 (Medium)](https://leetcode.com/problems/find-the-duplicate-number/)

An array of `n+1` integers where each value is in `[1, n]`. Exactly one value is duplicated (possibly many times). Find it without modifying the array and in O(1) extra space.

<details>
<summary>Brute Force</summary>

Sort and scan for adjacent equal elements. Time: O(n log n). Space: O(1) if in-place sort is allowed — but the problem forbids modifying the array. Using a hash set is O(n) time / O(n) space.

The waste: neither approach exploits the fact that values are constrained to `[1, n]`. That constraint is the whole point.
</details>

<details>
<summary>Hint 1</summary>

Don't binary-search over indices. Binary-search over the possible *values* (the range `[1, n]`).
</details>

<details>
<summary>Hint 2</summary>

For a candidate value `v`, count how many array elements are `<= v`. Without duplicates the count would be exactly `v`. With a duplicate `d`, every `v >= d` gets a count strictly greater than `v`. That predicate flips once — binary-search the flip point.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n log n) | Space: O(1)
func findDuplicate(nums []int) int {
    left, right := 1, len(nums)-1
    for left < right {
        mid := left + (right-left)/2
        count := 0
        for _, v := range nums {
            if v <= mid {
                count++
            }
        }
        if count > mid {
            right = mid
        } else {
            left = mid + 1
        }
    }
    return left
}
```

Answer-space binary search where the "space" is a value range and the predicate is a full array scan. Floyd's cycle detection gives O(n) for the same problem, but this is the natural binary-search phrasing.
</details>

### 12. Search a 2D Matrix II — [#240 (Medium)](https://leetcode.com/problems/search-a-2d-matrix-ii/)

Given an `m x n` matrix where each row is sorted left-to-right and each column is sorted top-to-bottom (but the flat row-major view is *not* globally sorted), determine whether `target` is present.

<details>
<summary>Brute Force</summary>

Scan every cell. Time: O(mn). Space: O(1).

The waste: unlike LC 74, the matrix isn't flattenable into a single sorted 1D array, so classical binary search doesn't apply. But the two independent sort directions give a different kind of pruning.
</details>

<details>
<summary>Hint 1</summary>

Start at a corner where one direction increases and the other decreases. Which corner has that property?
</details>

<details>
<summary>Hint 2</summary>

Top-right (or bottom-left) works. From top-right: moving down increases, moving left decreases. Compare with target — one move eliminates an entire row or column.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(m + n) | Space: O(1)
func searchMatrix(matrix [][]int, target int) bool {
    if len(matrix) == 0 || len(matrix[0]) == 0 {
        return false
    }
    r, c := 0, len(matrix[0])-1
    for r < len(matrix) && c >= 0 {
        if matrix[r][c] == target {
            return true
        }
        if matrix[r][c] > target {
            c--
        } else {
            r++
        }
    }
    return false
}
```

Included in a binary-search file specifically as a **counter-example**: strictly, this isn't binary search — it doesn't halve the space each step. It eliminates one row or one column per step, giving O(m+n). Forcing a log-based approach here (binary-search each row) gives O(m log n), which is worse for square matrices. Knowing when *not* to reach for binary search is part of knowing the pattern.
</details>

### 13. Split Array Largest Sum — [#410 (Hard)](https://leetcode.com/problems/split-array-largest-sum/)

Given a non-negative integer array and an integer `k`, split the array into `k` non-empty contiguous subarrays such that the largest subarray sum is minimised. Return that minimised largest sum.

<details>
<summary>Brute Force</summary>

Try every way to partition into `k` subarrays and take the min-of-max. Time: exponential (or O(n^2 · k) with DP). Space: O(n · k) DP.

The waste: the answer is a single integer in a bounded range. Instead of computing it directly, guess it and verify.
</details>

<details>
<summary>Hint 1</summary>

If you're told the largest allowed subarray sum is `cap`, can you check in O(n) whether the array can be split into `<= k` parts each with sum `<= cap`?
</details>

<details>
<summary>Hint 2</summary>

Greedy: walk left to right, start a new part whenever adding the next element would exceed `cap`. If the number of parts is `<= k`, `cap` is feasible. That predicate is monotonic in `cap`. Search range: `[max(nums), sum(nums)]`.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n * log(sum(nums))) | Space: O(1)
func splitArray(nums []int, k int) int {
    lo, hi := 0, 0
    for _, v := range nums {
        if v > lo {
            lo = v
        }
        hi += v
    }
    canSplit := func(cap int) bool {
        parts, cur := 1, 0
        for _, v := range nums {
            if cur+v > cap {
                parts++
                cur = v
            } else {
                cur += v
            }
        }
        return parts <= k
    }
    for lo < hi {
        mid := lo + (hi-lo)/2
        if canSplit(mid) {
            hi = mid
        } else {
            lo = mid + 1
        }
    }
    return lo
}
```

Structurally identical to Koko (#875) and Ship (#1011). Once you see the answer-space skeleton, all three feel like the same problem.
</details>

### 14. Median of Two Sorted Arrays — [#4 (Hard)](https://leetcode.com/problems/median-of-two-sorted-arrays/)

Given two sorted arrays `A` and `B`, return the median of the combined sorted array in O(log(min(m, n))) time.

<details>
<summary>Brute Force</summary>

Merge the two arrays and pick the middle element(s). Time: O(m + n). Space: O(m + n).

The waste: you don't need to merge. The median is defined by a *partition* — a cut point in each array such that everything on the left of both cuts forms the lower half of the combined array. Binary-search the partition.
</details>

<details>
<summary>Hint 1</summary>

Suppose you cut `A` at index `i` and `B` at index `j`. What condition on `i + j` makes the left side exactly half of the combined length? What condition on the four boundary values (`A[i-1], A[i], B[j-1], B[j]`) makes the cut correct?
</details>

<details>
<summary>Hint 2</summary>

Enforce `i + j = (m + n + 1) / 2`. The cut is correct when `A[i-1] <= B[j]` and `B[j-1] <= A[i]`. Binary-search `i` in `[0, m]` on the shorter array, derive `j`. Adjust `i` up or down based on which inequality fails.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(log(min(m, n))) | Space: O(1)
func findMedianSortedArrays(nums1, nums2 []int) float64 {
    if len(nums1) > len(nums2) {
        nums1, nums2 = nums2, nums1
    }
    m, n := len(nums1), len(nums2)
    total := m + n
    half := (total + 1) / 2

    lo, hi := 0, m
    for lo <= hi {
        i := lo + (hi-lo)/2
        j := half - i

        leftA := math.MinInt
        if i > 0 {
            leftA = nums1[i-1]
        }
        rightA := math.MaxInt
        if i < m {
            rightA = nums1[i]
        }
        leftB := math.MinInt
        if j > 0 {
            leftB = nums2[j-1]
        }
        rightB := math.MaxInt
        if j < n {
            rightB = nums2[j]
        }

        if leftA <= rightB && leftB <= rightA {
            if total%2 == 1 {
                return float64(max(leftA, leftB))
            }
            return float64(max(leftA, leftB)+min(rightA, rightB)) / 2.0
        } else if leftA > rightB {
            hi = i - 1
        } else {
            lo = i + 1
        }
    }
    return 0
}
```

Search on the shorter array to guarantee `log(min(m, n))`. The `math.MinInt` / `math.MaxInt` sentinels handle the edge partitions (`i == 0` or `i == m`) without special-casing. This is the hardest common binary-search problem and worth solving once cleanly.
</details>

## References

- `dsa/patterns/two-pointers.md`
- `dsa/patterns/sliding-window.md`
- `dsa/patterns/prefix-sum.md`
- `dsa/data-structures/arrays.md`
- `dsa/fundamentals/complexity-analysis.md`