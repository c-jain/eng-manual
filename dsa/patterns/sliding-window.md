---
Status: 🌳 Evergreen
Created: 2026-08-09
Last Updated: 2026-08-09
---

# Sliding Window

## Table Of Contents

- [What It Is And Why It Exists](#what-it-is-and-why-it-exists)
- [Why "Sliding Window"](#why-sliding-window)
- [Fixed-Size Window](#fixed-size-window)
- [Variable-Size Window (Expand/Shrink)](#variable-size-window-expandshrink)
- [The "At Most K" Template And The "Exactly K" Trick](#the-at-most-k-template-and-the-exactly-k-trick)
- [Complexity Reference](#complexity-reference)
- [Recognizing The Pattern](#recognizing-the-pattern)
- [Go Template Code](#go-template-code)
- [Where JavaScript Differs](#where-javascript-differs)
- [LeetCode Problems](#leetcode-problems)
- [How To Remember This](#how-to-remember-this)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

## What It Is And Why It Exists

Sliding window applies to contiguous-range problems — "max sum of any subarray of size K," "longest substring with at most K distinct characters," "smallest subarray with sum ≥ target." The brute-force instinct is to check every possible window: for each start index, scan forward to compute the property (sum, distinct count, etc.). That's O(n) windows times O(n) work per window, O(n²) overall — sometimes O(n³) if the per-window check isn't O(1) either.

The waste is that window `[i, j]` and window `[i+1, j]` share almost everything. They differ by exactly one element leaving and (usually) one entering. Sliding window exists to reuse the previous window's computed aggregate and apply a small delta for the one element that changed, instead of recomputing the whole window from scratch. It converts an O(n²) nested scan into a single O(n) pass with two pointers marking the window's edges.

### The Problem It Solves

Any brute-force "for every window, recompute the whole thing" approach costs O(n) or worse per window. Maintaining a running aggregate (sum, frequency map) and updating it incrementally as the window's edges move drops that to O(1) amortized work per step, making the whole scan O(n).

## Why "Sliding Window"

"Window" is the current contiguous range `[left, right]` under consideration — literally a frame laid over part of the array or string. "Sliding" is moving that frame across the sequence: either shifting it wholesale by one position (fixed size), or stretching one edge forward and then dragging the other edge to catch up (variable size). The name is a direct visual description, not a metaphor that needs decoding.

## Fixed-Size Window

The window width `k` is given by the problem. Slide by exactly one position each step: add the incoming element on the right, and once the window exceeds size `k`, remove the outgoing element on the left. Every step after the initial fill, exactly one element enters and exactly one leaves.

```
Scenario: Window Slides From [0,2] To [1,3], k=3

[ 4  2  7  1  5 ]
  ^-----^
left=0  right=2   sum=4+2+7=13

Action:  right++ (include index 3, value 1 -> window size 4, too big)
         left++  (exclude index 0, value 4)

[ 4  2  7  1  5 ]
     ^-----^
    left=1  right=3   sum=13-4+1=10
```

## Variable-Size Window (Expand/Shrink)

There's no fixed width. Grow `right` to expand the window until some condition is met or violated, then shrink from `left` until the window is valid again. The width itself is usually the answer being tracked (max width, min width, or a count of valid windows).

### The Correctness Argument: Why `left` Never Moves Backward

This is the part that makes the pattern O(n) instead of O(n²), and it hinges on the window's condition being **monotonic**. A condition is monotonic here if growing or shrinking the window changes the tracked property in one consistent direction only — it never flips back and forth. "Sum ≥ target" over non-negative numbers is monotonic: adding elements can only hold the sum steady or grow it, never shrink it; removing elements can only hold it steady or shrink it, never grow it. "At most K distinct" is monotonic the same way: adding an element can only hold the distinct count steady or raise it, never lower it.

Given a monotonic condition, here's the claim, stated precisely: define `L(right)` as the smallest `left` that keeps the window `[left, right]` valid. The claim is that `L` never decreases as `right` increases — `L(right+1) >= L(right)` for every `right`. In plain terms: as `right` slides forward, the earliest point the window can start from only moves forward too, never backward.

Take "smallest subarray with sum ≥ target," all positive numbers. Suppose at `right = r`, the smallest valid `left` is `L` — meaning `sum(arr[L..r]) ≥ target`, but `sum(arr[L+1..r]) < target`.

```
Move to right = r+1. The window sum only grows (all elements
positive), so sum(arr[L..r+1]) >= target still holds.

Could some left' < L become newly required? No: a smaller left
only adds more positive elements, which can only increase the sum
further. If L was already the smallest valid left at r, nothing
about adding arr[r+1] creates a need to go below L.

=> left is safe to never reconsider positions before its current value.
```

Because `left` only advances and `right` only advances, each pointer makes at most `n` moves across the *entire* run, not per step — that's what makes the total work O(n) despite the shrink loop being nested inside the expand loop.

**This monotonicity is the load-bearing assumption.** It breaks if the array contains negative numbers: shrinking `left` no longer guarantees the sum only decreases, so "smallest valid left never needs to move back" stops being true. That case belongs to prefix-sum-plus-monotonic-deque territory, not plain sliding window.

```
Scenario A: expanding right (window not yet valid), target=7

[ 2  3  1  2  4  3 ]
  ^--^
 left=0 right=1   sum=5   (< 7, keep expanding)

Scenario B: shrinking left (window just became valid)

[ 2  3  1  2  4  3 ]
  ^-----------^
 left=0    right=4   sum=12  (>= 7, valid! now try shrinking)

  shrink: sum-arr[left]=12-2=10 >= 7 -> left++, record width
  shrink: sum-arr[left]=10-3=7  >= 7 -> left++, record width
  shrink: sum-arr[left]=7-1=6   < 7  -> stop, left stays
```

## The "At Most K" Template And The "Exactly K" Trick

Many variable-window problems are phrased as "at most K distinct values" (or "at most K zeros," etc.) — this is the natural fit for shrink-on-violation, because "at most K" is monotonic: once a window is invalid (more than K distinct), shrinking can only reduce the distinct count, never increase it.

"Exactly K" is not directly monotonic — a window can go from "fewer than K distinct" to "more than K distinct" without ever passing through "exactly K" in a way that's easy to track incrementally. The standard move is to avoid tracking "exactly K" directly and instead compute:

```
countExactlyK = countAtMostK(K) - countAtMostK(K-1)
```

Every window counted in `countAtMostK(K-1)` is also counted in `countAtMostK(K)`, so the subtraction leaves exactly the windows with precisely K distinct values. This turns a non-monotonic condition into two applications of a monotonic one.

## Complexity Reference

| Technique | Time | Space | Replaces |
|---|---|---|---|
| Fixed-size window | O(n) | O(1) | O(n·k) recompute-per-window brute force |
| Variable-size (shrink on violation) | O(n) amortized | O(1) or O(charset) | O(n²) recompute-per-start brute force |
| "Exactly K" via at-most-K subtraction | O(n) (two passes) | O(charset) | Direct exactly-K tracking (non-monotonic, error-prone) |

## Recognizing The Pattern

A fixed subarray/substring length in the problem statement ("size K") signals the fixed-size window. A phrase like "longest," "shortest," "at most K," or "contains all of" over a contiguous range signals variable-size expand/shrink. "Exactly K" phrasing, especially combined with "number of subarrays," signals the at-most-K subtraction trick rather than direct tracking. All-positive or all-non-negative constraints are what make the shrink argument valid in the first place — their absence is a signal to look elsewhere (prefix sum, monotonic deque).

## Go Template Code

```go
package slidingwindow

// ===== Fixed-Size Window =====

// MaxSumFixedWindow returns the maximum sum of any contiguous subarray of
// size k. Returns 0 if the input is shorter than k.
// Time: O(n) | Space: O(1)
func MaxSumFixedWindow(nums []int, k int) int {
	if len(nums) < k {
		return 0
	}
	windowSum := 0
	for i := 0; i < k; i++ {
		windowSum += nums[i]
	}
	best := windowSum
	for right := k; right < len(nums); right++ {
		windowSum += nums[right]   // element entering
		windowSum -= nums[right-k] // element leaving
		if windowSum > best {
			best = windowSum
		}
	}
	return best
}

// ===== Variable-Size Window: Shrink On Violation =====

// MinSubArrayLen returns the length of the smallest contiguous subarray with
// sum >= target, or 0 if no such subarray exists. Requires non-negative nums;
// the shrink argument relies on the sum being monotonic as the window grows.
// Time: O(n) | Space: O(1)
func MinSubArrayLen(target int, nums []int) int {
	left, sum := 0, 0
	best := len(nums) + 1
	for right := 0; right < len(nums); right++ {
		sum += nums[right]
		for sum >= target {
			if width := right - left + 1; width < best {
				best = width
			}
			sum -= nums[left]
			left++
		}
	}
	if best == len(nums)+1 {
		return 0
	}
	return best
}

// ===== Variable-Size Window: At-Most-K Distinct (General Template) =====

// LongestSubstringAtMostKDistinct returns the length of the longest
// substring containing at most k distinct characters.
// Time: O(n) | Space: O(k)
func LongestSubstringAtMostKDistinct(s string, k int) int {
	if k == 0 {
		return 0
	}
	freq := make(map[byte]int)
	left, best := 0, 0
	for right := 0; right < len(s); right++ {
		freq[s[right]]++
		for len(freq) > k {
			freq[s[left]]--
			if freq[s[left]] == 0 {
				delete(freq, s[left]) // must delete, not just leave a zero entry -- len(freq) counts keys
			}
			left++
		}
		if width := right - left + 1; width > best {
			best = width
		}
	}
	return best
}
```

## Where JavaScript Differs

The pattern itself is identical — the only meaningful divergence is that Go's fixed-size `[26]int` arrays (used below for uppercase-letter frequency counting) become plain objects or `Map` in JS, since JS has no fixed-size array literal for this purpose:

```javascript
// Go: var freq [26]int; freq[s[right]-'A']++
// JS equivalent:
const freq = new Map();
freq.set(ch, (freq.get(ch) || 0) + 1);
```

The expand/shrink loop structure, the `left`/`right` pointer roles, and the at-most-K subtraction trick are direct line-for-line translations — there's no Go-specific mechanic that changes the algorithm here.

## LeetCode Problems

### 1. Maximum Average Subarray I — [#643 (Easy)](https://leetcode.com/problems/maximum-average-subarray-i/)

Given an array and an integer `k`, find the contiguous subarray of length `k` with the maximum average.

<details>
<summary>Brute Force</summary>

For every starting index from `0` to `n-k`, sum that window's `k` elements directly and compare it to the best average found so far.

Time: O(n·k) — roughly `n` windows, each summed in O(k). Space: O(1).

The waste: up to `k-1` elements are shared between one window and the next, but this approach re-adds all of them from scratch every time instead of just adding the one new element and subtracting the one that left.
</details>

<details>
<summary>Hint 1</summary>

Recomputing the sum of every length-`k` window from scratch is O(n·k). What single value changes when the window slides by one position?
</details>

<details>
<summary>Hint 2</summary>

Compute the sum of the first `k` elements once. Then for each slide, add the element entering on the right and subtract the element leaving on the left — O(1) per step.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func findMaxAverage(nums []int, k int) float64 {
	sum := 0
	for i := 0; i < k; i++ {
		sum += nums[i]
	}
	best := sum
	for right := k; right < len(nums); right++ {
		sum += nums[right] - nums[right-k]
		if sum > best {
			best = sum
		}
	}
	return float64(best) / float64(k)
}
```
</details>

### 2. Permutation In String — [#567 (Medium)](https://leetcode.com/problems/permutation-in-string/)

Given two strings `s1` and `s2`, return true if `s2` contains a contiguous substring that is a permutation of `s1`.

<details>
<summary>Brute Force</summary>

For every window of length `len(s1)` in `s2`, build a fresh character-frequency count of that window from scratch and compare it against `s1`'s frequency count.

Time: O((n-m)·m) where `n = len(s2)`, `m = len(s1)` — roughly `n-m` windows, each costing O(m) to build and compare. Space: O(1), fixed 26-letter counters.

The waste: the frequency count is rebuilt entirely for every window instead of being updated incrementally — add the one character entering, remove the one character leaving.
</details>

<details>
<summary>Hint 1</summary>

A permutation of `s1` is just any arrangement with the same character frequencies. Does that suggest a fixed window size?
</details>

<details>
<summary>Hint 2</summary>

Use a fixed-size window of length `len(s1)` sliding across `s2`. Maintain a 26-count frequency array for the window and compare it to `s1`'s frequency array; a match means a permutation is present.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1) -- fixed 26-element arrays
func checkInclusion(s1 string, s2 string) bool {
	if len(s1) > len(s2) {
		return false
	}
	var need, window [26]int
	for i := 0; i < len(s1); i++ {
		need[s1[i]-'a']++
		window[s2[i]-'a']++
	}
	if need == window {
		return true
	}
	for right := len(s1); right < len(s2); right++ {
		window[s2[right]-'a']++
		left := right - len(s1)
		window[s2[left]-'a']--
		if need == window {
			return true
		}
	}
	return false
}
```
</details>

### 3. Longest Substring Without Repeating Characters — [#3 (Medium)](https://leetcode.com/problems/longest-substring-without-repeating-characters/)

Given a string, find the length of the longest substring without repeating characters.

<details>
<summary>Brute Force</summary>

Check every possible substring — all O(n²) of them — and for each one, verify it has no repeated character using a fresh set.

Time: O(n³) — O(n²) substrings, each requiring an O(n) uniqueness check. Space: O(n) for the set used per check.

The waste: substrings overlap heavily (the substring starting at `i` and the one starting at `i+1` share almost everything), but each check re-verifies that shared portion from zero. A first improvement — expand a window from each start index until the first duplicate, instead of checking every substring independently — drops this to O(n²), but it still throws away the tracking structure and restarts empty at every new start index. The O(n) version's real unlock is tracking each character's *last-seen index* once, across the entire string, so a duplicate tells you exactly where `left` needs to jump — no restart at all.
</details>

<details>
<summary>Hint 1</summary>

This is "at most 1 occurrence of each character" in disguise. What do you need to know to decide when to shrink the window?
</details>

<details>
<summary>Hint 2</summary>

Track the last seen index of each character. When you encounter a character already in the window (its last-seen index is `>= left`), jump `left` to just past that previous occurrence instead of shrinking one step at a time.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(charset)
func lengthOfLongestSubstring(s string) int {
	lastSeen := make(map[byte]int)
	left, best := 0, 0
	for right := 0; right < len(s); right++ {
		if idx, ok := lastSeen[s[right]]; ok && idx >= left {
			left = idx + 1
		}
		lastSeen[s[right]] = right
		if width := right - left + 1; width > best {
			best = width
		}
	}
	return best
}
```

This is a variant of shrink-on-violation: instead of shrinking one element at a time in a loop, the repeat's last-seen index tells you exactly where `left` needs to land, so the jump happens in one step.
</details>

### 4. Minimum Size Subarray Sum — [#209 (Medium)](https://leetcode.com/problems/minimum-size-subarray-sum/)

Given an array of positive integers and a target sum, find the length of the shortest contiguous subarray with sum ≥ target. Return 0 if none exists.

<details>
<summary>Brute Force</summary>

For every starting index, extend the subarray one element at a time, summing as you go, and record the length the moment the running sum first reaches `target`.

Time: O(n²) — up to `n` starting points, each potentially scanning up to `n` elements before hitting `target`. Space: O(1).

The waste: the running sum restarts from zero at every starting index, even though (for non-negative numbers) the earliest valid `left` for a given `right` never moves backward — see the correctness argument above. The O(n) version exploits exactly that fact to never redo work `left` has already done.
</details>

<details>
<summary>Hint 1</summary>

Expand the window until it's valid (sum ≥ target). Once valid, can it help to keep shrinking from the left while it's still valid, rather than stopping at the first valid state?
</details>

<details>
<summary>Hint 2</summary>

Every time the window sum is ≥ target, record the width, then try removing `arr[left]` and advancing `left` — as long as it's still ≥ target, that's a shorter valid window. Stop shrinking once the sum drops below target.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func minSubArrayLen(target int, nums []int) int {
	left, sum := 0, 0
	best := len(nums) + 1
	for right := 0; right < len(nums); right++ {
		sum += nums[right]
		for sum >= target {
			if width := right - left + 1; width < best {
				best = width
			}
			sum -= nums[left]
			left++
		}
	}
	if best == len(nums)+1 {
		return 0
	}
	return best
}
```
</details>

<details>
<summary>Follow-Up (LeetCode's Own): O(n log n) Solution</summary>

LeetCode states this follow-up directly on the problem: *"If you have figured out the O(n) solution, try coding another solution of which the time complexity is O(n log(n))."*

The angle is prefix sums instead of a moving window. Build `prefix[i]` = sum of the first `i` elements, so `prefix[0] = 0` and `prefix[j] - prefix[i] = sum(arr[i..j-1])`. Because all values are positive, `prefix` is strictly increasing — which means for any fixed starting index `i`, you can binary search for the smallest ending index `j` such that `prefix[j] - prefix[i] >= target`, instead of walking forward with a pointer.

This trades the two-pointer's amortized O(n) for O(log n) per starting index, O(n log n) overall — worse in practice, but it's a legitimate second technique, and articulating *why* it's worse (throwing away the monotonic-window structure that made O(n) possible in the first place) is itself a good thing to be able to say out loud in an interview.

```go
// Time: O(n log n) | Space: O(n)
func minSubArrayLenBinarySearch(target int, nums []int) int {
	n := len(nums)
	prefix := make([]int, n+1)
	for i := 0; i < n; i++ {
		prefix[i+1] = prefix[i] + nums[i]
	}
	best := n + 1
	for i := 0; i < n; i++ {
		needed := prefix[i] + target
		// smallest j in [i+1, n] such that prefix[j] >= needed
		k := sort.Search(n-i, func(k int) bool {
			return prefix[i+1+k] >= needed
		})
		j := i + 1 + k
		if j <= n {
			if width := j - i; width < best {
				best = width
			}
		}
	}
	if best == n+1 {
		return 0
	}
	return best
}
```
</details>

### 5. Longest Repeating Character Replacement — [#424 (Medium)](https://leetcode.com/problems/longest-repeating-character-replacement/)

Given a string of uppercase letters and an integer `k`, find the length of the longest substring you can make into all-one-character by replacing at most `k` characters.

<details>
<summary>Brute Force</summary>

Check every substring — O(n²) of them — and for each one, count the frequency of every letter to find the most frequent one, then test whether `windowSize - maxFreq <= k`.

Time: O(n³) — O(n²) substrings, each requiring O(n) work to scan and find the max frequency. Space: O(26), fixed.

The waste: the frequency table and the max-frequency value are rebuilt from scratch for every substring. The optimized version keeps a running frequency count and — the subtle part — lets `maxFreq` stay a stale-but-historical high across shrinks rather than recomputing it exactly every time, since a too-high `maxFreq` can never produce an over-counted answer.
</details>

<details>
<summary>Hint 1</summary>

A window is "achievable" if `windowSize - (count of the most frequent character in the window) <= k` — that's the number of replacements needed. What should trigger a shrink?
</details>

<details>
<summary>Hint 2</summary>

Track a frequency count per letter and the max frequency seen so far in the window. If `windowSize - maxFreq > k`, shrink from the left by one. Note: `maxFreq` doesn't need to be recalculated on shrink — it only ever needs to be a historical high to correctly bound the answer, not a live exact value.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1) -- fixed 26-element array
func characterReplacement(s string, k int) int {
	var freq [26]int
	left, maxFreq, best := 0, 0, 0
	for right := 0; right < len(s); right++ {
		freq[s[right]-'A']++
		if freq[s[right]-'A'] > maxFreq {
			maxFreq = freq[s[right]-'A']
		}
		windowSize := right - left + 1
		if windowSize-maxFreq > k {
			freq[s[left]-'A']--
			left++
		}
		if width := right - left + 1; width > best {
			best = width
		}
	}
	return best
}
```
</details>

### 6. Fruit Into Baskets — [#904 (Medium)](https://leetcode.com/problems/fruit-into-baskets/)

Given an array of fruit types, find the length of the longest contiguous subarray containing at most 2 distinct types.

<details>
<summary>Brute Force</summary>

For every starting tree, walk forward collecting fruit types into a set until a third type would be needed, then record how many trees were picked from.

Time: O(n²) — up to `n` starting points, each potentially walking up to `n` trees forward. Space: O(1), at most 3 types tracked before stopping.

The waste: the fruit-type tracking restarts from empty at every starting tree, instead of tracking composition incrementally across a single sweep and only adjusting the left edge when a third type actually appears.
</details>

<details>
<summary>Hint 1</summary>

Forget the fruit-basket framing for a second — the condition is really "at most 2 distinct values in the window at once." What should happen the moment a third distinct value shows up?
</details>

<details>
<summary>Hint 2</summary>

Track a frequency map of fruit types currently in the window. Whenever the map holds more than 2 distinct keys, shrink from the left — decrementing the outgoing type's count and deleting its key once that count hits zero — until you're back down to 2.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1) -- at most 3 keys in freq at any time
func totalFruit(fruits []int) int {
	freq := make(map[int]int)
	left, best := 0, 0
	for right := 0; right < len(fruits); right++ {
		freq[fruits[right]]++
		for len(freq) > 2 {
			freq[fruits[left]]--
			if freq[fruits[left]] == 0 {
				delete(freq, fruits[left])
			}
			left++
		}
		if width := right - left + 1; width > best {
			best = width
		}
	}
	return best
}
```

This is the general `LongestSubstringAtMostKDistinct` template from earlier in this file (see [Go Template Code](#go-template-code)) with `k = 2`, generalized from characters to arbitrary values via a `map[int]int` instead of `map[byte]int` — same shrink-on-violation mechanics, different key type.
</details>

### 7. Max Consecutive Ones III — [#1004 (Medium)](https://leetcode.com/problems/max-consecutive-ones-iii/)

Given a binary array and an integer `k`, find the length of the longest subarray of all 1s you can get by flipping at most `k` zeros to 1s.

<details>
<summary>Brute Force</summary>

For every starting index, walk forward flipping zeros as they're encountered, stopping once more than `k` zeros would be needed, and record the length reached.

Time: O(n²) — up to `n` starting points, each potentially walking up to `n` elements forward. Space: O(1).

The waste: the zero-count restarts from zero at every starting index instead of being tracked incrementally across one single sweep, the same "at most K" pattern as problems 3 and 6.
</details>

<details>
<summary>Hint 1</summary>

Reframe "at most k flips" as "at most k zeros allowed inside the window." What single counter do you need to track the violation condition?
</details>

<details>
<summary>Hint 2</summary>

Track a running count of zeros in the window. When it exceeds `k`, shrink from the left, decrementing the zero count only when the element leaving is itself a zero.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func longestOnes(nums []int, k int) int {
	left, zeroCount, best := 0, 0, 0
	for right := 0; right < len(nums); right++ {
		if nums[right] == 0 {
			zeroCount++
		}
		for zeroCount > k {
			if nums[left] == 0 {
				zeroCount--
			}
			left++
		}
		if width := right - left + 1; width > best {
			best = width
		}
	}
	return best
}
```
</details>

### 8. Subarrays With K Different Integers — [#992 (Hard)](https://leetcode.com/problems/subarrays-with-k-different-integers/)

Given an array and an integer `k`, count the number of contiguous subarrays with exactly `k` different integers.

<details>
<summary>Brute Force</summary>

For every pair of start and end indices — O(n²) subarrays — count the number of distinct integers in that subarray directly using a set, and check whether it equals `k`.

Time: O(n³) — O(n²) subarrays, each requiring O(n) to build a set and count distinct elements. Space: O(n) for the set per check.

The waste: distinct-counts are recomputed from scratch for every subarray. The optimized version reuses the monotonic "at most K" structure (see the template section above) to count every valid subarray ending at each `right` in a single sweep, and gets "exactly K" by subtracting two such sweeps rather than tracking exactness directly.
</details>

<details>
<summary>Hint 1</summary>

"Exactly K" isn't monotonic the way "at most K" is — shrinking a window can jump straight past exactly-K without ever stopping there. Can two applications of an "at most K" count combine into an exact count?
</details>

<details>
<summary>Hint 2</summary>

`countExactlyK = countAtMostK(K) - countAtMostK(K-1)`. Implement `atMostK(K)` by expanding `right` and, whenever the window holds more than `K` distinct values, shrinking from the left until it's back to at most `K` — the same expand/shrink loop you'd use for a plain "longest window with at most K distinct" problem. Instead of tracking a max width, add `(right - left + 1)` to a running count at every `right` — that's the number of valid subarrays ending exactly at `right`.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) -- two passes of atMostKDistinctCount | Space: O(k)
func subarraysWithKDistinct(nums []int, k int) int {
	return atMostKDistinctCount(nums, k) - atMostKDistinctCount(nums, k-1)
}

func atMostKDistinctCount(nums []int, k int) int {
	if k < 0 {
		return 0
	}
	freq := make(map[int]int)
	left, count := 0, 0
	for right := 0; right < len(nums); right++ {
		freq[nums[right]]++
		for len(freq) > k {
			freq[nums[left]]--
			if freq[nums[left]] == 0 {
				delete(freq, nums[left])
			}
			left++
		}
		count += right - left + 1 // every subarray [x, right] for left <= x <= right has <= k distinct
	}
	return count
}
```
</details>

## How To Remember This

**Sliding window = reuse, not rediscover.** Every step, ask what changed since the last window. Usually exactly one element in, at most one element out. Recomputing the whole aggregate from scratch means the pattern isn't actually being used.

**Fixed = conveyor belt, one in one out. Variable = accordion, stretch then squeeze.**

**Left Only Ever Creeps Forward.** For monotonic conditions, the smallest valid `left` never needs to move backward as `right` advances — that single fact is the entire reason the pattern is O(n) instead of O(n²).

**At Most K Is Monotonic, Exactly K Is Not.** When a problem says "exactly," reach for `atMost(K) - atMost(K-1)` rather than trying to track "exactly" directly.

## Interview Cheat Sheet

### Signal Phrases To Use

- "Since window `[i+1, j]` shares almost everything with `[i, j]`, I can maintain a running aggregate instead of recomputing the whole window — that drops this from O(n²) to O(n)."
- "This condition is monotonic — once the window is invalid, shrinking can only help, never hurt — so I can use shrink-on-violation with `left` never moving backward."
- "This is 'exactly K,' which isn't monotonic on its own, so I'll compute it as `atMost(K) - atMost(K-1)`."
- "I need to delete the key from the frequency map when its count hits zero, not just leave a zero entry, since I'm using `len(map)` as my distinct-count check."

### Red Flags To Avoid

| Mistake | Correct Approach |
|---|---|
| Using shrink-on-violation on an array with negative numbers | Sliding window's monotonicity argument requires non-negative values; use prefix sum + monotonic deque instead |
| Leaving a zero-count entry in the frequency map after decrementing | Delete the key once its count reaches zero if `len(map)` is used as the distinct-count check |
| Tracking "exactly K" directly with ad-hoc increment/decrement logic | Use `atMostK(K) - atMostK(K-1)`, which only relies on the monotonic "at most" condition |
| Recomputing `maxFreq` exactly on every shrink in Longest Repeating Character Replacement | `maxFreq` only needs to be a historical high — a stale but too-high value can never produce an invalid answer, since the window size bound still holds |
| Off-by-one on window width (`right - left` vs `right - left + 1`) | Decide once whether `[left, right]` is inclusive-inclusive and stay consistent; width is `right - left + 1` under that convention |

### Common Interviewer Probes

| Probe | What They Are Looking For |
|---|---|
| "Why doesn't `left` need to move backward?" | The monotonicity argument for the specific condition — not just "it usually works" |
| "What if the array has negative numbers?" | Recognizing that plain sliding window breaks, and naming the alternative (prefix sum, monotonic deque) |
| "How would you count subarrays with exactly K of something?" | The `atMostK(K) - atMostK(K-1)` decomposition |
| "Walk through what happens when you shrink the window." | Explicit description of what gets removed from the running aggregate and when the frequency map key gets deleted |
| "Fixed or variable window here?" | Recognizing whether the problem specifies a length (fixed) or asks for longest/shortest/at-most (variable) |

## References

- [LeetCode: Sliding Window Tag](https://leetcode.com/tag/sliding-window/)
- [NeetCode: Sliding Window roadmap](https://neetcode.io/roadmap)
- Sedgewick & Wayne — *Algorithms* (4th ed.), §1.4 (Analysis of Algorithms) for amortized-cost reasoning
- [Grokking the Coding Interview](https://www.designgurus.io/course/grokking-the-coding-interview) — original pattern-based framing for sliding window