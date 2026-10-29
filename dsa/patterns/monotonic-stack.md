---
Status: 🌳 Evergreen
Created: 2026-10-29
Last Updated: 2026-10-29
---

# Monotonic Stack

## Table Of Contents

1. [What It Is And Why It Exists](#what-it-is-and-why-it-exists)
2. [The Four Variants](#the-four-variants)
3. [Why It Is O(n): Amortised Analysis](#why-it-is-on-amortised-analysis)
4. [Walkthrough: Next Greater Element](#walkthrough-next-greater-element)
5. [The Sentinel Trick](#the-sentinel-trick)
6. [Storing Indices vs Values](#storing-indices-vs-values)
7. [Recognizing The Pattern](#recognizing-the-pattern)
8. [Where JavaScript Differs](#where-javascript-differs)
9. [LeetCode Problems](#leetcode-problems)
10. [References](#references)

## What It Is And Why It Exists

A monotonic stack is a regular LIFO stack with one extra rule: before pushing a new element, pop everything on top that violates the desired ordering (strictly or non-strictly increasing, strictly or non-strictly decreasing) with respect to the incoming element. What remains on the stack, top to bottom, is always sorted in the chosen direction.

The pattern exists to answer a single family of questions in linear time: for each position `i` in an array, what is the nearest position on the left (or right) whose value is greater than (or less than) `nums[i]`? The brute-force answer is a nested loop scanning outward from every index, which costs O(n^2). A monotonic stack folds the same work into one pass by exploiting a simple observation: if `nums[j] <= nums[k]` and `j < k`, then `nums[j]` can never be the "next greater" for anything to the right of `k`, because `nums[k]` will always block the view. So `nums[j]` can be discarded the moment a larger `nums[k]` appears. The stack is the data structure that carries the still-relevant candidates forward.

The problems it solves cluster around one shape: "for each element, find the nearest neighbour on one side that satisfies a comparison predicate". Next Greater Element, Previous Smaller Element, Daily Temperatures, Stock Span, Largest Rectangle In Histogram, Trapping Rain Water. All are the same trick with different bookkeeping.

### Why The Name

"Monotonic" describes the state of the stack, not the input. The input can be arbitrary. What is monotonic is the residual set of indices the algorithm keeps around: they form an increasing (or decreasing) sequence of values when read from the bottom of the stack to the top. The word is the same one used in calculus for a function that never changes direction. Calling it a "monotonic stack" is naming the data structure by its invariant, the same way a "min-heap" is named by the invariant that the root is the minimum.

## The Four Variants

There are exactly four canonical variants, one for each combination of (which side you scan from) and (which comparison you want).

```
                +--------------------+-------------------------+
                | Nearest on RIGHT   | Nearest on LEFT         |
+---------------+--------------------+-------------------------+
| GREATER than  | Scan L to R.       | Scan L to R.            |
| current       | Stack decreasing   | Stack decreasing        |
|               | (bot -> top).      | (bot -> top).           |
|               | Pop while top < x. | Pop while top <= x.     |
|               | Popped element's   | Remaining top is x's    |
|               | answer is i.       | answer.                 |
+---------------+--------------------+-------------------------+
| SMALLER than  | Scan L to R.       | Scan L to R.            |
| current       | Stack increasing   | Stack increasing        |
|               | (bot -> top).      | (bot -> top).           |
|               | Pop while top > x. | Pop while top >= x.     |
|               | Popped element's   | Remaining top is x's    |
|               | answer is i.       | answer.                 |
+---------------+--------------------+-------------------------+
```

Two mental shortcuts collapse this table:

1. **Which direction the stack is monotonic in is the opposite of what you are looking for.** Looking for a *greater* neighbour, keep a *decreasing* stack (so the incoming greater element can eat through the smaller candidates on top). Looking for a *smaller* neighbour, keep an *increasing* stack.
2. **Scan direction picks between "answer on pop" and "answer on push".** For a "nearest on the right" query, you learn the answer for element `x` at the moment something later pops `x`. For a "nearest on the left" query, you learn the answer for the incoming `x` from whatever is left on top of the stack after the pops.

Strict vs non-strict comparison (`<` vs `<=`) matters when the input has duplicates. Which one to use is problem-defined: "next strictly greater" pops on `<`, "next greater or equal" pops on `<=`. Get this wrong and duplicates either double-count or vanish.

## Why It Is O(n): Amortised Analysis

The inner `for` loop that pops the stack looks quadratic. It is not. Each index enters the stack at most once, and each index leaves the stack at most once. That is 2n stack operations across the whole scan, regardless of what the outer loop is doing.

This is a textbook amortised argument. Any single outer iteration might pop many elements, but those pops are paid for by the earlier pushes. The total work is bounded by the total number of pushes, which is `n`. So the whole algorithm is O(n) even though a single iteration can look expensive.

```
Amortised bookkeeping:

Each index i:
  - is pushed exactly once  (1 push)
  - is popped at most once  (0 or 1 pop)

Total operations across all iterations:
  <= n pushes + n pops = 2n = O(n)
```

The same logic explains why the pattern only works with a stack, not with, say, a scan that has to look at all previous elements. The stack lets you discard evidence permanently as soon as it becomes irrelevant, and the amortised bound depends on that discarding being final.

## Walkthrough: Next Greater Element

Take `nums = [2, 1, 2, 4, 3]`. We want `res[i]` to be the index of the first element to the right of `i` that is strictly greater than `nums[i]`, or `-1` if none exists.

The stack holds indices, and the values at those indices form a strictly decreasing sequence from bottom to top. When we visit index `i`, we pop every top whose value is less than `nums[i]`, because `i` is that top's answer, then push `i`.

```
Scenario: Next Greater Element on nums = [2, 1, 2, 4, 3]

i=0, x=2:
  stack empty, no pops. push 0.
  stack indices: [0]     values on stack (bot->top): [2]
  res so far:    [_, _, _, _, _]

i=1, x=1:
  top value 2 is not < 1, no pops. push 1.
  stack indices: [0, 1]  values: [2, 1]
  res so far:    [_, _, _, _, _]

i=2, x=2:
  top value 1 < 2, pop index 1. res[1] = 2.
  top value 2 is not < 2 (strict), stop. push 2.
  stack indices: [0, 2]  values: [2, 2]
  res so far:    [_, 2, _, _, _]

i=3, x=4:
  top value 2 < 4, pop index 2. res[2] = 3.
  top value 2 < 4, pop index 0. res[0] = 3.
  stack empty, stop. push 3.
  stack indices: [3]     values: [4]
  res so far:    [3, 2, 3, _, _]

i=4, x=3:
  top value 4 is not < 3, no pops. push 4.
  stack indices: [3, 4]  values: [4, 3]
  res so far:    [3, 2, 3, _, _]

End of scan. Everything still on the stack has no greater element to
its right: res[3] = -1, res[4] = -1.

Final res: [3, 2, 3, -1, -1]
```

Two things to notice. First, index 0 is popped from the *bottom* of the surviving stack when `i=3` arrives, not because we searched for it but because the stack was already sorted, so once 4 clears its way through 2, everything below is guaranteed smaller and gets swept in the same pass. Second, the answer for a given index is written exactly at the moment it is popped, never later. That is the whole trick.

## The Sentinel Trick

Several problems (Largest Rectangle In Histogram, most notably) leave elements on the stack at the end of the scan, and those elements need their answer computed too. The general fix is to pretend the array is followed by a sentinel value that guarantees every remaining stack entry gets popped.

For a *decreasing* stack answering "next greater on the right", the sentinel is `+infinity` (or any value larger than every input). For an *increasing* stack answering "next smaller on the right", the sentinel is `-infinity` or `0` (when inputs are known non-negative, as in histograms).

You do not have to actually append the sentinel to the input; running the outer loop one extra iteration with a synthesised value is enough. The largest-rectangle solution below does exactly this.

## Storing Indices vs Values

Almost always, push indices. From an index you can read the value in O(1), but from a value alone you cannot compute distances, and most monotonic-stack problems care about distances (Daily Temperatures wants `res[j] = i - j`, Largest Rectangle wants a width). Storing values is a trap that works for a few toy problems and then fails.

The one exception is when the values are what you push *into a hash map* and you never need distances (LC 496 Next Greater Element I is the clean example). There, values are enough because the mapping from value to next-greater is the entire deliverable.

## Recognizing The Pattern

Reach for a monotonic stack when the problem has all three of these traits:

1. **Per-index nearest-boundary answer.** The output has one entry per input position, and each entry is a distance, an index, or a value derived from the closest neighbour in one direction that satisfies a comparison.
2. **Static sequence, single pass acceptable.** The input is not being edited between queries. If it were, you would climb to a segment tree.
3. **The comparison discards permanently.** Once a candidate is beaten by something taller (or shorter), it will never be an answer for anything further along in the scan. This is the signature that a stack, not a queue, is the right container.

Signal phrases in the prompt: "next greater", "previous smaller", "nearest larger element", "how many days until", "span of stock prices", "largest rectangle", "how much water is trapped". Any of these is a monotonic stack until proven otherwise.

Counter-signal: if the problem allows *both* sides to be relevant simultaneously (e.g. "for each element, find the product of the nearest larger on left and right"), you often still use a monotonic stack, just twice, once per side.

## Where JavaScript Differs

JavaScript arrays give you `push` and `pop` directly and both are O(1) amortised, so the template ports cleanly:

```javascript
function nextGreaterRight(nums) {
    const res = new Array(nums.length).fill(-1);
    const stack = [];
    for (let i = 0; i < nums.length; i++) {
        while (stack.length && nums[stack[stack.length - 1]] < nums[i]) {
            res[stack.pop()] = i;
        }
        stack.push(i);
    }
    return res;
}
```

Two gotchas. First, `stack[stack.length - 1]` is the idiomatic "peek"; there is no built-in. Second, comparing numbers is fine, but if the input is ever strings, use explicit numeric conversion (`Number(x)`) because `<` on strings compares lexicographically and will silently break the invariant.

## LeetCode Problems

### 1. Next Greater Element I - [#496 (Easy)](https://leetcode.com/problems/next-greater-element-i/)

You are given two arrays `nums1` and `nums2` where `nums1` is a subset of `nums2` and all values are distinct. For each element of `nums1`, find the next greater element to its right in `nums2`, or `-1` if none exists.

<details>
<summary>Brute Force</summary>

For every `x` in `nums1`, scan `nums2` to find `x`, then scan forward from that position looking for a value greater than `x`.

Time: O(m * n) where `m = len(nums1)`, `n = len(nums2)`. Space: O(1) beyond output.

The waste: the "next greater" answer for every value in `nums2` is a fixed property of `nums2`, independent of `nums1`. Recomputing it per query throws away work you could have done once.
</details>

<details>
<summary>Hint 1</summary>

Precompute an answer table for every value in the larger array. Once that table exists, each query in the smaller array is O(1).
</details>

<details>
<summary>Hint 2</summary>

Sweep the larger array left to right, maintaining a stack of values that are still waiting for their next-greater answer. Whenever the incoming value beats the top of the stack, the top's answer is the incoming value. Record answers in a hash map keyed by value (safe here because values are distinct).
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n + m). Space: O(n) for the map and stack.
func nextGreaterElement(nums1, nums2 []int) []int {
    ngr := make(map[int]int, len(nums2))
    stack := make([]int, 0, len(nums2))
    for _, x := range nums2 {
        for len(stack) > 0 && stack[len(stack)-1] < x {
            ngr[stack[len(stack)-1]] = x
            stack = stack[:len(stack)-1]
        }
        stack = append(stack, x)
    }
    for _, x := range stack {
        ngr[x] = -1
    }
    res := make([]int, len(nums1))
    for i, x := range nums1 {
        res[i] = ngr[x]
    }
    return res
}
```

This is the one problem where pushing values (not indices) is the right call: distances are not asked for, and the hash map wants values as keys anyway.
</details>

<details>
<summary>Follow-Up (from LeetCode)</summary>

*Could you find an O(nums1.length + nums2.length) solution?*

The solution above is already O(n + m): one pass over `nums2` to build the map, one pass over `nums1` to read it. The naive nested-loop approach is O(n * m); the stack removes the inner scan by remembering unresolved candidates instead of rescanning them.
</details>

### 2. Daily Temperatures - [#739 (Medium)](https://leetcode.com/problems/daily-temperatures/)

Given a list of daily temperatures, return an array where `answer[i]` is the number of days you have to wait after day `i` for a warmer temperature. If there is no future day with a warmer temperature, put `0`.

<details>
<summary>Brute Force</summary>

For each day `i`, scan forward until you find a warmer day; record `j - i`. If none is found, record `0`.

Time: O(n^2). Space: O(1) beyond output.

The waste: cold days that were already skipped over by a previous warmer day never turn out useful for later days either. The scan repeats that work.
</details>

<details>
<summary>Hint 1</summary>

You need the nearest future position with a strictly greater value. Any candidate that is beaten by a later warmer day can be discarded immediately; a still-warmer day will always overshadow it.
</details>

<details>
<summary>Hint 2</summary>

Keep a stack of day-indices whose answers you have not yet written. When today's temperature exceeds the temperature at the top of the stack, today is the answer for that day: pop and record the day-difference.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) amortised. Space: O(n).
func dailyTemperatures(temperatures []int) []int {
    n := len(temperatures)
    res := make([]int, n)
    stack := make([]int, 0, n)
    for i, t := range temperatures {
        for len(stack) > 0 && temperatures[stack[len(stack)-1]] < t {
            j := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            res[j] = i - j
        }
        stack = append(stack, i)
    }
    return res
}
```

`res` is zero-initialised in Go, so days that never find a warmer future are already correct with no extra bookkeeping. This is the "answer written on pop" side of the symmetry: the stack is decreasing bottom to top, and the incoming warmer day resolves every entry it beats.
</details>

### 3. Next Greater Element II - [#503 (Medium)](https://leetcode.com/problems/next-greater-element-ii/)

Same as Next Greater Element I, but the array is *circular*: after the last element, indices wrap to the first. Return the next greater number for every element, or `-1` if no such number exists in the circular traversal.

<details>
<summary>Brute Force</summary>

For each `i`, scan `i+1, i+2, ..., i+n-1` modulo `n` looking for a strictly greater value.

Time: O(n^2). Space: O(1) beyond output.
</details>

<details>
<summary>Hint 1</summary>

The circular wraparound can be simulated by walking the array twice back to back. Any element that has no next-greater in the first pass either finds one in the second pass or has none at all.
</details>

<details>
<summary>Hint 2</summary>

Run the standard next-greater sweep for `2n` iterations, indexing by `i % n`. Only push during the first `n` iterations so no index gets a new answer twice. Entries still on the stack at the end have no greater element anywhere in the circle.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) (2n iterations, each index popped at most once).
// Space: O(n).
func nextGreaterElements(nums []int) []int {
    n := len(nums)
    res := make([]int, n)
    for i := range res {
        res[i] = -1
    }
    stack := make([]int, 0, n)
    for i := 0; i < 2*n; i++ {
        x := nums[i%n]
        for len(stack) > 0 && nums[stack[len(stack)-1]] < x {
            res[stack[len(stack)-1]] = x
            stack = stack[:len(stack)-1]
        }
        if i < n {
            stack = append(stack, i)
        }
    }
    return res
}
```

Doubling the loop instead of physically duplicating the array keeps space at O(n). The guard `if i < n` ensures each original index is a valid stack candidate at most once; second-pass iterations only *resolve* pending answers, they never add new questions.
</details>

### 4. Online Stock Span - [#901 (Medium)](https://leetcode.com/problems/online-stock-span/)

Design a class that collects daily stock prices one at a time. On each call `next(price)`, return the span of the stock's price for that day: the maximum number of consecutive days (ending on today, going backwards) for which the price was less than or equal to today's price.

<details>
<summary>Brute Force</summary>

Store every price. On each `next(price)` call, walk backwards from the newest price, counting days whose price is less than or equal to `price`, and stop at the first day that exceeds it.

Time: O(n) per call, O(n^2) across `n` calls. Space: O(n).

The waste: a price that has already been "eaten" by a later, higher price will never be the limiting boundary again. Rescanning past it every day is pure duplication.
</details>

<details>
<summary>Hint 1</summary>

For each day still on the stack, remember not just the price but the span that day resolved to. That way, when a new price swallows a previous day, you can absorb its span in one addition instead of counting days one by one.
</details>

<details>
<summary>Hint 2</summary>

Keep a stack of `(price, span)` entries whose prices strictly exceed the newest day's price. On `next(price)`, pop every entry with a price `<= price`, adding its span into a running total. Push `(price, total_span)` at the end.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Amortised O(1) per Next call. Space O(n) worst case.
type StockSpanner struct {
    stack []stockEntry
}

type stockEntry struct {
    price, span int
}

func Constructor() StockSpanner { return StockSpanner{} }

func (s *StockSpanner) Next(price int) int {
    span := 1
    for len(s.stack) > 0 && s.stack[len(s.stack)-1].price <= price {
        span += s.stack[len(s.stack)-1].span
        s.stack = s.stack[:len(s.stack)-1]
    }
    s.stack = append(s.stack, stockEntry{price, span})
    return span
}
```

This is the "streaming" flavour of the pattern: the input arrives one element at a time, but the amortised argument still holds because each `stockEntry` is pushed exactly once and popped at most once across the lifetime of the object.
</details>

### 5. Trapping Rain Water - [#42 (Hard)](https://leetcode.com/problems/trapping-rain-water/)

Given `n` non-negative integers representing an elevation map where the width of each bar is 1, compute how much water it can trap after raining.

<details>
<summary>Brute Force</summary>

For each index `i`, find the tallest bar to its left and the tallest bar to its right. The water above `i` is `min(leftMax, rightMax) - height[i]` if positive, else 0.

Time: O(n^2). Space: O(1) beyond output.

Precomputing `leftMax` and `rightMax` arrays gets it to O(n) time, O(n) space, which is the standard two-array solution. The monotonic-stack version reaches the same asymptotic complexity but computes water "row by row" horizontally, which is a genuinely different mental model and is what interviewers usually want when they say "solve it with a stack".
</details>

<details>
<summary>Hint 1</summary>

Water gets trapped when there is a lower bar between two taller bars. Sweep left to right and keep a stack of bar indices whose bounded water potential is still undetermined.
</details>

<details>
<summary>Hint 2</summary>

Maintain a decreasing stack of indices. When the incoming bar is taller than the top, pop the top as the "bottom" of a puddle, then use the *new* top as the left wall and the incoming bar as the right wall. The trapped water is `(right - left - 1) * (min(leftHeight, rightHeight) - bottomHeight)`.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n). Space: O(n).
func trap(height []int) int {
    water := 0
    stack := make([]int, 0, len(height))
    for i, h := range height {
        for len(stack) > 0 && height[stack[len(stack)-1]] < h {
            bottom := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            if len(stack) == 0 {
                break
            }
            left := stack[len(stack)-1]
            width := i - left - 1
            bounded := min(height[left], h) - height[bottom]
            water += width * bounded
        }
        stack = append(stack, i)
    }
    return water
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}
```

Each pop resolves one horizontal layer of water bounded by the two surviving walls. Layers add up correctly because taller layers get resolved later, each time the bottom is popped and the left wall becomes whatever remains. Notice water is added only when there *is* a left wall; if the popped bottom was the leftmost bar, water spills off the edge and contributes zero.

The pop condition is strict `<`, so equal heights stay on the stack rather than being eagerly cleared. The stack is non-strictly decreasing bottom to top. This is safe: when a duplicate is later popped against a taller incoming bar, the new left wall equals the popped bottom's height, so `min(leftHeight, h) - bottomHeight` resolves to zero and no spurious water is added. Using `<=` would collapse those zero-contribution layers immediately; the final total is identical either way.

Go 1.21+ has `min` in the built-in library. The helper above is kept for portability with older versions.
</details>

### 6. Largest Rectangle In Histogram - [#84 (Hard)](https://leetcode.com/problems/largest-rectangle-in-histogram/)

Given an array of bar heights (each of width 1), find the area of the largest rectangle that can be formed within the histogram.

<details>
<summary>Brute Force</summary>

For each bar `i`, expand outward left and right while heights stay `>= height[i]`. Area is `height[i] * width`. Take the max.

Time: O(n^2). Space: O(1).

The waste: the "previous smaller" and "next smaller" boundaries for each bar are exactly what the expansion recomputes. A monotonic stack finds both in a single pass.
</details>

<details>
<summary>Hint 1</summary>

For each bar `i`, the largest rectangle whose *shortest* bar is `height[i]` extends from just after the previous strictly smaller bar to just before the next strictly smaller bar. Compute both boundaries and the answer is straightforward.
</details>

<details>
<summary>Hint 2</summary>

Sweep left to right with an increasing stack. When the incoming bar is shorter than the top, the top's rectangle ends here: its width is bounded on the right by the current index and on the left by the new stack top after the pop. Append a sentinel bar of height 0 at the end so every entry gets popped.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) amortised. Space: O(n).
func largestRectangleArea(heights []int) int {
    n := len(heights)
    stack := make([]int, 0, n+1)
    best := 0
    for i := 0; i <= n; i++ {
        var cur int
        if i == n {
            cur = 0 // sentinel: forces everything to be popped
        } else {
            cur = heights[i]
        }
        for len(stack) > 0 && heights[stack[len(stack)-1]] > cur {
            top := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            var left int
            if len(stack) == 0 {
                left = -1
            } else {
                left = stack[len(stack)-1]
            }
            width := i - left - 1
            area := heights[top] * width
            if area > best {
                best = area
            }
        }
        stack = append(stack, i)
    }
    return best
}
```

The sentinel `cur = 0` at `i == n` is what makes the "leftover on the stack" case fall out for free: it is strictly smaller than every real bar (heights are non-negative and a zero-height bar is not the answer), so it drains the entire stack, resolving each remaining bar with the correct right boundary of `n`.

The width formula `i - left - 1` is worth staring at until it clicks. `top` is the bar whose rectangle we are pricing. `i` is the first index to the right that is strictly shorter, so the rectangle stops at `i - 1`. `left` is the last index to the left that is strictly shorter (or `-1` if none), so the rectangle starts at `left + 1`. Width is `(i - 1) - (left + 1) + 1 = i - left - 1`.
</details>

## References

- [LeetCode: Monotonic Stack Tag](https://leetcode.com/tag/monotonic-stack/)
- [NeetCode: Stack roadmap](https://neetcode.io/roadmap)
- Cormen, Leiserson, Rivest, Stein, *Introduction to Algorithms* (4th ed.), §17.1 on the aggregate method of amortised analysis (formal justification of the O(n) claim)
- [Competitive Programmer's Handbook (Laaksonen)](https://cses.fi/book/book.pdf), §8.3 (Nearest smaller elements) for the canonical previous-smaller-element derivation