---
Status: 🌳 Evergreen
Created: 2026-09-03
Last Updated: 2026-09-04
---

# Prefix Sum

## Table Of Contents

1. [What It Is And Why It Exists](#what-it-is-and-why-it-exists)
2. [Why "Prefix Sum"](#why-prefix-sum)
3. [Building The Prefix Array](#building-the-prefix-array)
4. [Range Sum Queries In O(1)](#range-sum-queries-in-o1)
5. [Difference Array (The Inverse Trick)](#difference-array-the-inverse-trick)
6. [2D Prefix Sum](#2d-prefix-sum)
7. [Prefix Sum Plus Hash Map](#prefix-sum-plus-hash-map)
8. [Case Study: Monotonic Deque For Sliding Window Aggregates](#case-study-monotonic-deque-for-sliding-window-aggregates)
9. [Complexity Reference](#complexity-reference)
10. [Recognizing The Pattern](#recognizing-the-pattern)
11. [Go Template Code](#go-template-code)
12. [Where JavaScript Differs](#where-javascript-differs)
13. [LeetCode Problems](#leetcode-problems)
14. [References](#references)

## What It Is And Why It Exists

Prefix sum is a preprocessing pattern for static arrays. You precompute one auxiliary array where each cell holds a running total of everything up to that point in the source. With that in hand, the sum of any contiguous range `a[l..r]` reduces to a single subtraction of two prefix values, no scan needed. Preprocessing is O(n) done once; every subsequent range-sum query is O(1).

The pattern exists because the brute-force alternative is quadratic when queries are many. Answering Q range-sum queries on an array of length N by scanning the range each time costs O(N) per query and O(N·Q) total. That is the exact shape of problem where paying a linear upfront cost to build an index, then answering each query in constant time, is a strict win: total work drops from O(N·Q) to O(N + Q).

Prefix sum is the smallest, cheapest form of "index the data once, answer forever" thinking. The same idea, generalised to handle updates, is what Fenwick trees and segment trees do at O(log N) per operation. If the array never changes, prefix sum is the answer; if it changes often, prefix sum is the wrong tool and you climb the ladder to a BIT.

### The Problem It Solves

Repeated range-aggregate queries on a static array. Any operation that is invertible over the aggregate (sum, XOR, count-modulo-K) qualifies. Anything that is not cleanly invertible (max, min, gcd) does not, and belongs to sparse tables or segment trees instead.

> [!NOTE]
> **Invertible** means the operation has an "undo": given the aggregate and one part, you can recover the other. Addition is invertible via subtraction (`a + b = 10, a = 3 => b = 7`). XOR is its own inverse (`a XOR b = c => a = c XOR b`), so range-XOR works with a prefix-XOR array. Max is not: if `max(a, b) = 7` and `b = 5`, then `a` could be 7, 6, -100, anything ≤ 7. There is no way to recover it, so there is no meaningful `prefixMax[r+1] "minus" prefixMax[l]`. Prefix sum needs invertibility because the whole trick is `sum(a[l..r]) = prefix[r+1] - prefix[l]`, and the minus only exists if the operation has an inverse.

## Why "Prefix Sum"

`prefix[i]` is literally the sum of the *prefix* of the array up to index i, meaning the first i elements viewed as a leading segment. "Prefix" is the same word as in "prefix of a string": the leading slice, from the start up to some cut point. The array of these accumulated sums is the "prefix sum array". The name is the definition, not a metaphor.

## Building The Prefix Array

Two indexing conventions exist and almost every prefix-sum bug in an interview comes from mixing them. Pick one and stay with it.

**1-indexed with a leading zero (preferred).** Define `prefix[0] = 0` and `prefix[i] = a[0] + a[1] + ... + a[i-1]`. The prefix array has length `n+1`. The range formula is clean with no special case:

```
sum(a[l..r]) = prefix[r+1] - prefix[l]     // 0 <= l <= r < n
```

**0-indexed inclusive.** Define `prefix[i] = a[0] + ... + a[i]`. Length `n`. The range formula needs a guard:

```
sum(a[l..r]) = prefix[r] - (l > 0 ? prefix[l-1] : 0)
```

The 1-indexed form is preferred because the guard disappears, and empty-prefix (nothing summed yet) is a natural value at index 0.

```
Scenario: Building The 1-Indexed Prefix Array

a       = [ 3   1   4   1   5 ]
indices    0   1   2   3   4

prefix  = [ 0   3   4   8   9  14 ]
indices    0   1   2   3   4   5

prefix[3] = a[0]+a[1]+a[2] = 3+1+4 = 8
prefix[5] = a[0]+a[1]+a[2]+a[3]+a[4] = 14
```

## Range Sum Queries In O(1)

Once the prefix array is built, any range sum is one subtraction.

```
Scenario: Sum Of a[1..3] With The 1-Indexed Convention

a       = [ 3   1   4   1   5 ]
prefix  = [ 0   3   4   8   9  14 ]

sum(a[1..3]) = prefix[4] - prefix[1] = 9 - 3 = 6
verify:        a[1]+a[2]+a[3] = 1+4+1 = 6
```

The pattern extends without change to any invertible aggregate. Replace `+` and `-` with `XOR` and `XOR` (XOR is its own inverse) to answer range-XOR queries. Replace them with modular addition to answer "range sum mod M" queries. Anything with a well-defined inverse works.

> [!WARNING]
> Range max, range min, and range gcd are **not** invertible in this way. There is no `prefix[r] "minus" prefix[l]` for max. For those you need sparse tables (static, O(1) query) or segment trees (with updates, O(log n) query).

## Difference Array (The Inverse Trick)

Prefix sum and difference array are duals, but not in the update-vs-query axis. Neither handles interleaved reads and writes; both require an O(n) "freeze" step that sits on opposite sides of the batched O(1) operations.

- **Prefix sum**: **freeze then batch**. Freeze the source array, build in O(n), then batch many O(1) range reads. No writes after build.
- **Difference array**: **batch then freeze**. Batch many O(1) range writes into `diff`, then freeze by taking one O(n) prefix-sum pass to reconstruct the array. No reads until then.

If updates and queries need to interleave, both are the wrong tool. That is where Fenwick trees (O(log n) point update + O(log n) prefix-sum query) or segment trees take over.

The idea for the difference array: represent the array by the *changes* between consecutive positions. `diff[i] = a[i] - a[i-1]` (with `diff[0] = a[0]`). The original array is recovered by taking a prefix sum of `diff`. Both directions are O(n).

The unlock is what happens under a range update. "Add `v` to every element of `a[l..r]`" becomes two point updates on `diff`, in O(1):

```
diff[l]   += v      // v is now added to everything from l onward
diff[r+1] -= v      // cancels the addition starting at r+1
```

After all K range updates are recorded, one final prefix-sum pass over `diff` yields the updated array. Total cost is O(K + N) instead of the naive O(K·N).

```
Scenario: Two Range Updates Then Reconstruct

a       = [ 0   0   0   0   0 ]
diff    = [ 0   0   0   0   0 ]      (all zeros to start)

Update: add 3 to a[1..3]
  diff[1] += 3
  diff[4] -= 3
diff    = [ 0   3   0   0  -3 ]

Update: add 2 to a[2..4]
  diff[2] += 2
  diff[5] -= 2      (r+1 == n; the -2 falls off the end, safe)
diff    = [ 0   3   2   0  -3 ]

Prefix sum of diff:
final a = [ 0   3   5   5   2 ]
verify:      +0  +3  +3+2 +3+2 +2
```

> [!NOTE]
> The `diff[r+1] -= v` step is safe even when `r+1 == n` if you allocate `diff` of length `n+1` and ignore the last cell during reconstruction. Otherwise guard the write.

## 2D Prefix Sum

Extending prefix sum to a grid uses inclusion-exclusion. Define `P[i][j]` as the sum of the sub-rectangle from `(0, 0)` inclusive to `(i-1, j-1)` inclusive (the 1-indexed form, again with a leading zero row and column).

**Building:**

```
P[i][j] = grid[i-1][j-1]
        + P[i-1][j]         // rectangle above
        + P[i][j-1]         // rectangle to the left
        - P[i-1][j-1]       // subtract the top-left corner counted twice
```

**Querying** the sum of the rectangle from `(r1, c1)` to `(r2, c2)` inclusive:

```
sum = P[r2+1][c2+1]
    - P[r1][c2+1]           // strip above
    - P[r2+1][c1]           // strip to the left
    + P[r1][c1]             // add back the top-left corner subtracted twice
```

Inclusion-exclusion appears in both directions: build subtracts once (double-counted corner), query adds back once (double-subtracted corner). The two are mirror images.

```
Scenario: Query The Highlighted Sub-Rectangle

P[r2+1][c2+1]  (whole big rectangle from origin)
      minus
P[r1][c2+1]    (strip above the query, from origin down to row r1)
      minus
P[r2+1][c1]    (strip left of the query, from origin right to col c1)
      plus
P[r1][c1]      (top-left region subtracted twice above, add it back once)
```

Preprocessing is O(rows * cols), each query is O(1).

## Prefix Sum Plus Hash Map

This is the variant that shows up most in medium-hard interview questions. The setup: count or find contiguous subarrays whose sum equals K (or is divisible by K, or hits some other prefix-based property). Sliding window does not work here when the array can contain negative numbers, because the monotonicity that makes sliding window O(n) breaks.

The observation: `sum(a[l..r]) == K` iff `prefix[r+1] - prefix[l] == K` iff `prefix[l] == prefix[r+1] - K`. So at each right endpoint, the question "how many valid subarrays end here" becomes "how many earlier prefix values equal `currentPrefix - K`". A hash map from prefix-value to its number of occurrences answers that in O(1).

```
Algorithm: Count Subarrays With Sum Equal To K

seen         = { 0: 1 }        // empty prefix; needed so subarrays starting at 0 count
runningSum   = 0
count        = 0

for x in a:
    runningSum += x
    if (runningSum - K) in seen:
        count += seen[runningSum - K]
    seen[runningSum] += 1

return count
```

The `seen = {0: 1}` seed is the interview trap. Without it, any subarray whose sum from index 0 equals K is missed, because there is no earlier "prefix of value 0" recorded and the map lookup fails. The seed represents the empty prefix, which is a legitimate `l = 0` starting point.

**Divisibility variant.** For "count subarrays with sum divisible by K", the key is `runningSum mod K` instead of `runningSum` itself. The derivation, starting from the "sum equals K" identity:

```
sum(a[l..r]) is divisible by K
   iff  (prefix[r+1] - prefix[l]) mod K == 0
   iff  prefix[r+1] mod K == prefix[l] mod K
```

Two prefixes with the *same remainder mod K* are exactly the pairs whose difference is a multiple of K, and their difference is one qualifying subarray. So the algorithm is the same skeleton as "Subarray Sum Equals K", but hash the remainder rather than the raw prefix.

```
Trace: nums = [4, 5, 0, -2, -3, 1], K = 5

step | x  | running | r = running%5 | seen before      | count += seen[r] | seen after
-----+----+---------+---------------+------------------+------------------+----------------
init |    |    0    |      0        | {0:1}            |        -         | {0:1}
  1  |  4 |    4    |      4        | {0:1}            |        0         | {0:1, 4:1}
  2  |  5 |    9    |      4        | {0:1, 4:1}       |        1         | {0:1, 4:2}
  3  |  0 |    9    |      4        | {0:1, 4:2}       |        2         | {0:1, 4:3}
  4  | -2 |    7    |      2        | {0:1, 4:3}       |        0         | {0:1, 4:3, 2:1}
  5  | -3 |    4    |      4        | {0:1, 4:3, 2:1}  |        3         | {0:1, 4:4, 2:1}
  6  |  1 |    5    |      0        | {0:1, 4:4, 2:1}  |        1         | {0:2, 4:4, 2:1}
                                                            total = 7
```

Every earlier prefix with the same remainder pairs with the current one, so we add `seen[r]` *before* incrementing it. The Go modulo gotcha: `%` in Go is remainder, not modulo, and can be negative. Normalise with `((r % k) + k) % k`, otherwise two mathematically equivalent remainders (say `-2` and `3` when `k = 5`) land in different buckets and the count is wrong.

**Zero/one balance variant.** For "longest contiguous subarray with equal 0s and 1s", the reframing trick: replace every 0 with -1. Now a subarray has equal 0s and 1s iff its sum in the replaced array is zero, because each 1 contributes +1 and each 0 contributes -1, and they cancel exactly when the counts match. The problem becomes "longest subarray with sum 0", which is "Subarray Sum Equals K" with K = 0, but wanting *longest* instead of *count*.

Using the same identity: `sum(a[l..r]) == 0` iff `prefix[r+1] == prefix[l]`. So whenever the running prefix repeats, the subarray between the two occurrences sums to zero.

Two things change from the count variant. First, the map stores the *earliest* index each prefix value was seen (not a count), because for the widest pair at the current `r` you want the *earliest* matching `l`. Never overwrite once recorded. Second, the seed changes from `{0: 1}` to `{0: -1}`: if at index `i` the running prefix returns to 0, the whole subarray from index 0 through `i` has sum zero and length `i + 1`, which is `i - (-1)`. Seeding with `{0: 0}` would give width `i - 0 = i`, off by one.

```
Trace: nums = [0, 1, 0, 0, 1, 1, 0]  (interpret 0 as -1, 1 as +1)

 i | x | running | firstSeen before       | action                        | best
---+---+---------+------------------------+-------------------------------+-----
   |   |    0    | {0:-1}                 | seed                          |  0
 0 | 0 |   -1    | {0:-1}                 | not seen; store {-1:0}        |  0
 1 | 1 |    0    | {0:-1, -1:0}           | seen at -1; width 1-(-1) = 2  |  2
 2 | 0 |   -1    | {0:-1, -1:0}           | seen at 0;  width 2 - 0  = 2  |  2
 3 | 0 |   -2    | {0:-1, -1:0}           | not seen; store {-2:3}        |  2
 4 | 1 |   -1    | {0:-1, -1:0, -2:3}     | seen at 0;  width 4 - 0  = 4  |  4
 5 | 1 |    0    | {0:-1, -1:0, -2:3}     | seen at -1; width 5-(-1) = 6  |  6
 6 | 0 |   -1    | {0:-1, -1:0, -2:3}     | seen at 0;  width 6 - 0  = 6  |  6
                                                                 answer = 6
```

**The pattern behind both variants.** Both are "Subarray Sum Equals K" wearing a costume:

| Variant | Costume | Underneath |
|---|---|---|
| Sum equals K | none | count pairs where `prefix[r] - prefix[l] == K` |
| Sum divisible by K | hash the remainder | count pairs where `prefix[r] mod K == prefix[l] mod K` |
| Equal 0s and 1s | replace 0 with -1 | measure pairs where `prefix[r] == prefix[l]` |

Same hash-map trick, same seed for the empty prefix, same "look up before insert" ordering. Only the key function and what we store (count vs earliest index) change based on what the problem asks.

> [!TIP]
> The mental compression: hash-map lookup answers "have I seen a prefix value that, if subtracted from this one, gives me what the problem wants?" The problem statement dictates what "what the problem wants" is: exactly K, a multiple of K, zero, and so on. The mechanics of the map do not change.

## Case Study: Monotonic Deque For Sliding Window Aggregates

> [!NOTE]
> **"Aggregate"** here means any single value computed by reducing a range of elements: sum, max, min, count, XOR, and so on. Same sense as SQL's aggregate functions. "Sliding window aggregate" is the shape of problem where a window moves across the array and at each step you need the aggregate of whatever the window currently covers. This section uses sum (LC 862), but the same deque structure handles sliding max (LC 239) and sliding min with only the comparison flipped.

Consider LeetCode 862, "Shortest Subarray with Sum at Least K", where the array can contain negatives. Sliding window is out. The shrink step of sliding window relies on an invariant: removing an element from the left lowers the sum, so you can safely shrink until the sum drops below K and stop. With negatives, removing a negative element *raises* the sum (`sum -= (-3)` is `sum += 3`), which breaks the invariant. You can no longer trust "shrink until sum falls below K" as a stopping rule, and in the worst case the shortest valid window is behind an `l` you already skipped past, which sliding window cannot revisit.

Prefix sum reframes the problem. We want the smallest `r - l` such that `prefix[r] - prefix[l] >= K`, which reads: "for each right endpoint r, find the largest l < r (smallest window width) such that `prefix[l] <= prefix[r] - K`."

Two properties of the candidate `l` values matter:

1. If `l1 < l2` and `prefix[l1] >= prefix[l2]`, then `l1` is *dominated* by `l2`. Any future `r` that satisfies the threshold with `l1` also satisfies it with `l2` (same prefix or lower on `l2`), and `l2` gives a smaller window since it is closer to `r`. So `l1` can be discarded forever.

2. Once we consume `l` at some `r` (meaning `prefix[l] <= prefix[r] - K` and we record the width), we never need to consume it again for a later `r'`. Any width `r' - l` for `r' > r` is larger, so it cannot improve the answer.

**Why these two rules exist.** They are what shrinks the candidate set from "all earlier indices" (O(n²) work) down to "only the useful ones" (O(n) amortised).

- Rule 1 is the *space bound*: at any moment we only keep left endpoints that are not dominated by a later one. It justifies why the deque stays small, and specifically why we can safely evict from the back when a new smaller-or-equal prefix arrives.
- Rule 2 is the *time bound on the front*: each index leaves the deque at most once. It justifies why we can pop from the front the moment we consume a candidate, without worrying about needing it again.

Together they give O(n) amortised. Naive comparison of every `(l, r)` pair would be O(n²); this pair of observations is what buys us the linear complexity.

Both properties point to the same data structure: a **monotonic deque** of prefix indices, maintained so that the prefix values along the deque are strictly increasing from front to back. Push new indices at the back (evicting any back entries with `prefix >= currentPrefix`, since they are now dominated). Pop from the front while the front is a valid `l` for the current `r` (using each one once, recording widths).

```
Why It Is A Deque, Not A Stack Or Queue

- Back: needs push and pop, to maintain the increasing-prefix invariant
  when a new smaller-or-equal prefix arrives.
- Front: needs pop only, to consume valid l candidates from smallest
  index outward.

Two ends with different access patterns => double-ended queue.
```

Amortised complexity: each prefix index enters the deque at most once and leaves at most once. Total work across all `r` is O(n). Space is O(n) for both the prefix array and the deque.

```
Scenario: Threshold K=3, Sketch Of Deque State

prefix = [ 0   2  -1   1   4 ]
indices    0   1   2   3   4

r=0: deque=[0]                                          push 0
r=1: prefix[1]=2, no popped front satisfies; push 1     deque=[0,1]
r=2: prefix[2]=-1; evict back while prefix[back]>=-1
     evict 1 (prefix=2), evict 0 (prefix=0), push 2     deque=[2]
r=3: prefix[3]=1; front=2, prefix[3]-prefix[2]=2 < 3    push 3
     deque=[2,3]
r=4: prefix[4]=4; front=2, prefix[4]-prefix[2]=5 >= 3
     record width 4-2=2; pop front                      deque=[3]
     front=3, prefix[4]-prefix[3]=3 >= 3
     record width 4-3=1; pop front                      deque=[]

best width = 1
```

The deque here is *not* the monotonic stack from `monotonic-stack.md`, though the invariant idea is the same. Stacks answer "next greater element" style questions where you only ever grow one end and pop one end. Deques answer sliding-window aggregate questions where both ends need to move.

> [!IMPORTANT]
> Whenever you see "shortest subarray", "longest subarray", or "sliding maximum" combined with negative numbers or with an aggregate that is not monotonic under expansion, reach for prefix sum + monotonic deque, not sliding window.

## Complexity Reference

| Technique | Preprocess | Query / Update | Space | Replaces |
|---|---|---|---|---|
| 1D range sum | O(n) | O(1) query | O(n) | O(n·Q) naive range scans |
| 2D range sum | O(rows·cols) | O(1) query | O(rows·cols) | O(rows·cols·Q) naive |
| Difference array | O(n) build | O(1) range update, O(n) finalise | O(n) | O(K·N) applying K updates naively |
| Prefix sum + hash map | O(n) single pass | O(1) amortised per element | O(n) | O(n²) subarray enumeration |
| Prefix sum + monotonic deque | O(n) single pass | O(1) amortised per element | O(n) | O(n²) all-pairs prefix comparison |

## Recognizing The Pattern

Repeated range-sum queries on a static array signal 1D prefix sum. Repeated rectangle-sum queries signal 2D. "Add value v to every element in a range, then answer point queries" signals difference array. "Count subarrays whose sum equals / is divisible by / balances to a target" signals prefix sum plus hash map, especially when negatives rule out sliding window. "Shortest / longest subarray with a sum-based threshold, with negatives allowed" signals prefix sum plus monotonic deque.

The unifying tell across all five: the answer is expressible as a difference of two prefix values, and you need an efficient way to find or count the useful pairs.

## Go Template Code

```go
package prefixsum

// ===== 1D Prefix Sum (1-indexed with leading zero) =====

// Build returns the 1-indexed prefix array of length len(a)+1 such that
// prefix[i] = a[0]+a[1]+...+a[i-1]. prefix[0] is the empty-prefix sentinel.
// Time: O(n) | Space: O(n)
func Build(a []int) []int {
	prefix := make([]int, len(a)+1)
	for i, x := range a {
		prefix[i+1] = prefix[i] + x
	}
	return prefix
}

// RangeSum returns the sum of a[l..r] inclusive using a prefix array built
// by Build. Assumes 0 <= l <= r < len(a).
// Time: O(1) | Space: O(1)
func RangeSum(prefix []int, l, r int) int {
	return prefix[r+1] - prefix[l]
}

// ===== Difference Array =====

// ApplyRangeUpdates records K range updates in O(1) each and reconstructs
// the resulting array in one O(n) pass. Each update is {l, r, v} meaning
// "add v to a[l..r] inclusive".
// Time: O(n + k) | Space: O(n)
func ApplyRangeUpdates(n int, updates [][3]int) []int {
	diff := make([]int, n+1)
	for _, u := range updates {
		l, r, v := u[0], u[1], u[2]
		diff[l] += v
		diff[r+1] -= v // safe: diff has extra cell
	}
	result := make([]int, n)
	running := 0
	for i := 0; i < n; i++ {
		running += diff[i]
		result[i] = running
	}
	return result
}

// ===== 2D Prefix Sum =====

// Build2D returns a (rows+1) x (cols+1) prefix table with a leading zero
// row and column. P[i][j] is the sum of grid[0..i-1][0..j-1].
// Time: O(rows*cols) | Space: O(rows*cols)
func Build2D(grid [][]int) [][]int {
	rows := len(grid)
	if rows == 0 {
		return [][]int{{0}}
	}
	cols := len(grid[0])
	p := make([][]int, rows+1)
	for i := range p {
		p[i] = make([]int, cols+1)
	}
	for i := 1; i <= rows; i++ {
		for j := 1; j <= cols; j++ {
			p[i][j] = grid[i-1][j-1] + p[i-1][j] + p[i][j-1] - p[i-1][j-1]
		}
	}
	return p
}

// RectSum returns the sum of the sub-rectangle from (r1,c1) to (r2,c2)
// inclusive using a table built by Build2D. All coordinates are 0-indexed
// into the original grid.
// Time: O(1) | Space: O(1)
func RectSum(p [][]int, r1, c1, r2, c2 int) int {
	return p[r2+1][c2+1] - p[r1][c2+1] - p[r2+1][c1] + p[r1][c1]
}

// ===== Prefix Sum + Hash Map: Count Subarrays With Sum == K =====

// SubarraysWithSumK returns the number of contiguous subarrays whose sum
// equals k. Handles negative values.
// Time: O(n) | Space: O(n)
func SubarraysWithSumK(a []int, k int) int {
	seen := map[int]int{0: 1} // empty prefix: essential seed
	running, count := 0, 0
	for _, x := range a {
		running += x
		if c, ok := seen[running-k]; ok {
			count += c
		}
		seen[running]++
	}
	return count
}
```

## Where JavaScript Differs

The pattern is identical. Two small idiom differences:

```javascript
// Go: seen := map[int]int{0: 1}
// JS: use a Map to keep integer keys; a plain object stringifies keys
const seen = new Map([[0, 1]]);
seen.set(running, (seen.get(running) || 0) + 1);

// Go: negative-safe modulo
// v := ((running % k) + k) % k
// JS: same expression, since % is remainder (can be negative) not modulo
const bucket = ((running % k) + k) % k;
```

The 2D template translates directly, replacing `make([][]int, ...)` with nested `Array.from({length: rows+1}, () => new Array(cols+1).fill(0))`.

## LeetCode Problems

### 1. Range Sum Query - Immutable — [#303 (Easy)](https://leetcode.com/problems/range-sum-query-immutable/)

Given an integer array, implement `sumRange(l, r)` returning the sum of elements between indices `l` and `r` inclusive. The array does not change. Queries may be many.

<details>
<summary>Brute Force</summary>

Store the array. On every `sumRange(l, r)` call, loop from `l` to `r` and add.

Time: O(r-l+1) per query, O(N·Q) across Q queries. Space: O(1) beyond the input.

The waste: with Q queries on the same static array, the sum of the same overlapping regions is recomputed from scratch every time. This is the canonical setup for trading O(N) preprocessing for O(1) per query.
</details>

<details>
<summary>Hint 1</summary>

The constructor runs once, the query runs many times. What can you precompute so that each query is a fixed number of operations, regardless of range width?
</details>

<details>
<summary>Hint 2</summary>

Build a 1-indexed prefix array of length `n+1` where `prefix[i]` is the sum of the first `i` original elements. Any range sum is `prefix[r+1] - prefix[l]`.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Constructor: O(n) | Query: O(1) | Space: O(n)
type NumArray struct {
	prefix []int
}

func Constructor(nums []int) NumArray {
	prefix := make([]int, len(nums)+1)
	for i, x := range nums {
		prefix[i+1] = prefix[i] + x
	}
	return NumArray{prefix: prefix}
}

func (na *NumArray) SumRange(left int, right int) int {
	return na.prefix[right+1] - na.prefix[left]
}
```
</details>

### 2. Subarray Sum Equals K — [#560 (Medium)](https://leetcode.com/problems/subarray-sum-equals-k/)

Given an integer array `nums` and an integer `k`, return the number of contiguous subarrays whose sum equals `k`. `nums` can contain negatives.

<details>
<summary>Brute Force</summary>

Two nested loops: for each starting index, extend and accumulate; increment the counter whenever the running sum hits `k`.

Time: O(n²). Space: O(1).

The waste: every start index rebuilds the running sum from zero, ignoring the fact that any subarray sum is a difference of two prefix values. Negatives here also rule out sliding window; the shrink argument requires monotonicity.
</details>

<details>
<summary>Hint 1</summary>

Fix the right endpoint at index `r`. What condition on some earlier prefix value would make the subarray ending at `r` sum to exactly `k`?
</details>

<details>
<summary>Hint 2</summary>

`sum(a[l..r]) == k` iff `prefix[r+1] - prefix[l] == k` iff `prefix[l] == prefix[r+1] - k`. Maintain a hash map from prefix-value to count-of-occurrences. Seed it with `{0: 1}` so that subarrays starting at index 0 are counted. For each new prefix, look up how many earlier prefixes equal `current - k` and add to the answer.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(n)
func subarraySum(nums []int, k int) int {
	seen := map[int]int{0: 1}
	running, count := 0, 0
	for _, x := range nums {
		running += x
		if c, ok := seen[running-k]; ok {
			count += c
		}
		seen[running]++
	}
	return count
}
```

The `{0: 1}` seed represents the empty prefix. Without it, a subarray that starts at index 0 and sums to `k` is missed, because no prior prefix of value 0 has been recorded.
</details>

### 3. Contiguous Array — [#525 (Medium)](https://leetcode.com/problems/contiguous-array/)

Given a binary array (0s and 1s), find the maximum length of a contiguous subarray with an equal number of 0s and 1s.

<details>
<summary>Brute Force</summary>

For each pair `(l, r)`, count 0s and 1s in `a[l..r]` and update the best length when they match.

Time: O(n²) with a running per-pair check, O(n³) if fully naive. Space: O(1).

The waste: same as Subarray Sum Equals K, but disguised. Once you translate 0s into -1s, "equal 0s and 1s" becomes "sum equals 0", and it is the same problem.
</details>

<details>
<summary>Hint 1</summary>

Rewrite each 0 as -1. What does the sum of a subarray now represent?
</details>

<details>
<summary>Hint 2</summary>

With 0 -> -1, a subarray with equal counts has sum 0, meaning two prefixes with the same value. Maintain a hash map from prefix-value to the *first* index where it appeared. For each `r`, if the current running prefix has been seen before at index `l`, then `a[l+1..r]` sums to 0 and its length is `r - l`. Track the max.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(n)
func findMaxLength(nums []int) int {
	firstSeen := map[int]int{0: -1} // empty prefix seen "before" index 0
	running, best := 0, 0
	for i, x := range nums {
		if x == 0 {
			running--
		} else {
			running++
		}
		if l, ok := firstSeen[running]; ok {
			if width := i - l; width > best {
				best = width
			}
		} else {
			firstSeen[running] = i
		}
	}
	return best
}
```

Two key differences from the count variant: the map stores the *earliest* index (not a count, since we want the longest), and the seed value is `-1` (the position "just before" index 0), so that a subarray starting at index 0 with sum zero has width `i - (-1) = i + 1`.

**Why the width is `i - l` and not `i - l + 1`.** `firstSeen[running] = l` records the *right end* of the earlier prefix at that value, not a fresh starting point. `l` is already baked into the earlier prefix. For the two prefixes to match now, the elements between them must sum to zero, and those elements are `a[l+1], a[l+2], ..., a[i]` (index `l` itself is excluded). Count of elements: `i - (l+1) + 1 = i - l`. The `l = -1` seed then makes "subarray starting at index 0" fall out naturally: `i - (-1) = i + 1`, which is the count of elements from index 0 through index `i`.
</details>

### 4. Subarray Sums Divisible By K — [#974 (Medium)](https://leetcode.com/problems/subarray-sums-divisible-by-k/)

Given an integer array `nums` and an integer `k`, return the number of contiguous subarrays whose sum is divisible by `k`.

<details>
<summary>Brute Force</summary>

For every pair `(l, r)`, compute the subarray sum and check divisibility.

Time: O(n²). Space: O(1).

The waste: divisibility by `k` is a statement about prefix sums modulo `k`. Two prefixes with the same remainder differ by a multiple of `k`, and their difference is one qualifying subarray. Counting pairs of equal-remainder prefixes is O(n).
</details>

<details>
<summary>Hint 1</summary>

If `prefix[r+1] % k == prefix[l] % k`, what can you say about the sum of `a[l..r]`?
</details>

<details>
<summary>Hint 2</summary>

Same skeleton as Subarray Sum Equals K, but hash the *remainder* `runningSum mod k` rather than the raw running sum. Take care with negative remainders: normalise them with `((r % k) + k) % k`. Seed the map with `{0: 1}` for the empty prefix.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(k)
func subarraysDivByK(nums []int, k int) int {
	seen := map[int]int{0: 1}
	running, count := 0, 0
	for _, x := range nums {
		running += x
		r := ((running % k) + k) % k
		count += seen[r]
		seen[r]++
	}
	return count
}
```

Every earlier prefix with the same remainder pairs with the current one to form a valid subarray, so we add the current bucket size *before* incrementing it.
</details>

### 5. Range Sum Query 2D - Immutable — [#304 (Medium)](https://leetcode.com/problems/range-sum-query-2d-immutable/)

Given an `m x n` matrix, implement `sumRegion(r1, c1, r2, c2)` returning the sum of the sub-rectangle bounded by those corners. The matrix does not change.

<details>
<summary>Brute Force</summary>

On every `sumRegion` call, loop over the rectangle rows and add.

Time: O((r2-r1+1)·(c2-c1+1)) per query. Space: O(1) beyond input.

The waste: same shape as the 1D version, at a higher dimension. Overlapping rectangles have their inner cells re-summed on every call.
</details>

<details>
<summary>Hint 1</summary>

The 1D trick was "sum of range = difference of two prefixes". What is the 2D analogue when you extend `prefix` to a rectangle from the origin?
</details>

<details>
<summary>Hint 2</summary>

Build a `(m+1) x (n+1)` prefix table where `P[i][j]` is the sum of the sub-rectangle from `(0,0)` to `(i-1, j-1)`. The query sum is `P[r2+1][c2+1] - P[r1][c2+1] - P[r2+1][c1] + P[r1][c1]`. The `+ P[r1][c1]` corrects for the top-left corner that gets subtracted twice by the two strips.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Constructor: O(m*n) | Query: O(1) | Space: O(m*n)
type NumMatrix struct {
	p [][]int
}

func Constructor(matrix [][]int) NumMatrix {
	m := len(matrix)
	if m == 0 {
		return NumMatrix{p: [][]int{{0}}}
	}
	n := len(matrix[0])
	p := make([][]int, m+1)
	for i := range p {
		p[i] = make([]int, n+1)
	}
	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			p[i][j] = matrix[i-1][j-1] + p[i-1][j] + p[i][j-1] - p[i-1][j-1]
		}
	}
	return NumMatrix{p: p}
}

func (nm *NumMatrix) SumRegion(r1, c1, r2, c2 int) int {
	return nm.p[r2+1][c2+1] - nm.p[r1][c2+1] - nm.p[r2+1][c1] + nm.p[r1][c1]
}
```
</details>

### 6. Corporate Flight Bookings — [#1109 (Medium)](https://leetcode.com/problems/corporate-flight-bookings/)

Given `n` flights labelled 1 to `n` and a list of bookings where each `[first, last, seats]` means "add `seats` reserved seats on flights `first` through `last` inclusive", return an array of length `n` giving the total reserved seats per flight.

<details>
<summary>Brute Force</summary>

For each booking, loop from `first` to `last` and add `seats` to each element of the answer.

Time: O(K·N) for K bookings and N flights. Space: O(N).

The waste: each booking is a range update, and range updates are exactly what difference array optimises. K range updates should cost O(K), not O(K·N).
</details>

<details>
<summary>Hint 1</summary>

You have many range-update operations followed by a single "read the whole array" operation. Which pattern turns range updates into constant-time work?
</details>

<details>
<summary>Hint 2</summary>

Build a difference array `diff` of length `n+1`. For each booking `[first, last, seats]`, do `diff[first-1] += seats` and `diff[last] -= seats` (accounting for 1-indexed flight labels). Then take a prefix sum of `diff` to reconstruct the final array.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n + k) | Space: O(n)
func corpFlightBookings(bookings [][]int, n int) []int {
	diff := make([]int, n+1)
	for _, b := range bookings {
		first, last, seats := b[0]-1, b[1]-1, b[2] // convert to 0-indexed
		diff[first] += seats
		diff[last+1] -= seats
	}
	result := make([]int, n)
	running := 0
	for i := 0; i < n; i++ {
		running += diff[i]
		result[i] = running
	}
	return result
}
```

The `n+1`-length allocation lets `diff[last+1] -= seats` write safely even when `last == n-1`; the cell is simply not read during reconstruction.
</details>

### 7. Shortest Subarray With Sum At Least K — [#862 (Hard)](https://leetcode.com/problems/shortest-subarray-with-sum-at-least-k/)

Given an integer array `nums` (with possible negatives) and an integer `k`, return the length of the shortest non-empty contiguous subarray with sum at least `k`, or `-1` if none exists.

<details>
<summary>Brute Force</summary>

For every `(l, r)` pair, compute the subarray sum and track the shortest with sum >= k.

Time: O(n²) with running sums, O(n³) fully naive. Space: O(1).

The waste: sliding window looks tempting but is invalid here because negatives break monotonicity (shrinking from the left can *decrease* the sum). Prefix sum plus a monotonic deque of candidate left endpoints gets this to O(n) by keeping only prefixes that are useful and consuming each at most once.
</details>

<details>
<summary>Hint 1</summary>

Compute the prefix array. For each `r`, you want the largest `l < r` (smallest window) with `prefix[r] - prefix[l] >= k`. What can you throw away about "useless" values of `l`?
</details>

<details>
<summary>Hint 2</summary>

Two rules: (1) If `l1 < l2` and `prefix[l1] >= prefix[l2]`, discard `l1` forever, because `l2` is nearer to any future `r` *and* has a smaller-or-equal prefix. (2) Once you consume some `l` at `r` (recording its width), you never need it again for a later `r'`, since the width only grows. Both rules together mean the candidate list should be a deque with strictly increasing prefix values front-to-back: push at the back (evicting dominated entries), pop from the front (consuming used candidates).
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) amortised | Space: O(n)
func shortestSubarray(nums []int, k int) int {
	n := len(nums)
	prefix := make([]int, n+1)
	for i, x := range nums {
		prefix[i+1] = prefix[i] + x
	}
	dq := make([]int, 0, n+1) // stores indices into prefix
	best := n + 1
	for r := 0; r <= n; r++ {
		// consume front: any l where prefix[r] - prefix[l] >= k
		for len(dq) > 0 && prefix[r]-prefix[dq[0]] >= k {
			if width := r - dq[0]; width < best {
				best = width
			}
			dq = dq[1:]
		}
		// evict back: maintain increasing prefix values
		for len(dq) > 0 && prefix[dq[len(dq)-1]] >= prefix[r] {
			dq = dq[:len(dq)-1]
		}
		dq = append(dq, r)
	}
	if best == n+1 {
		return -1
	}
	return best
}
```

The `r <= n` loop bound (not `< n`) is intentional: `prefix` has length `n+1`, and `r = n` corresponds to a window whose right end is the last element of `nums`, which we must consider. The double-ended manipulation (front consume, back evict) is what the monotonic deque earns. The `>=` in the eviction check (not `>`) enforces a *strictly* increasing prefix sequence in the deque: when two indices tie on prefix value, the earlier one is dominated by the later one and can be evicted immediately.
</details>

## References

- [LeetCode: Prefix Sum Tag](https://leetcode.com/tag/prefix-sum/)
- [NeetCode: Prefix Sum roadmap](https://neetcode.io/roadmap)
- Cormen, Leiserson, Rivest, Stein — *Introduction to Algorithms* (4th ed.), §17.1–17.3 on amortized analysis (justifies the amortised O(n) claim for the monotonic deque variant)
- [Competitive Programmer's Handbook (Laaksonen)](https://cses.fi/book/book.pdf), §9.2 (Prefix Sum Arrays) and §29 (Segment Trees) for the "when to graduate to a segment tree" boundary