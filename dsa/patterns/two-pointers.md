---
Status: 🌳 Evergreen
Created: 2026-07-02
Last Updated: 2026-07-02
---

# Two Pointers

## Table of Contents

- [What It Is and Why It Exists](#what-it-is-and-why-it-exists)
- [Why "Two Pointers"](#why-two-pointers)
- [Opposite-End (Converging) Pointers](#opposite-end-converging-pointers)
- [Same-Direction (Read/Write) Pointers](#same-direction-readwrite-pointers)
- [Partition Problems (Dutch National Flag)](#partition-problems-dutch-national-flag)
- [Complexity Reference](#complexity-reference)
- [Recognizing the Pattern](#recognizing-the-pattern)
- [Go Template Code](#go-template-code)
- [Where JavaScript Differs](#where-javascript-differs)
- [LeetCode Problems](#leetcode-problems)
- [How to Remember This](#how-to-remember-this)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

## What It Is and Why It Exists

Two pointers means tracking **two index variables** over the same sequence — instead of the usual one — and moving them according to rules that exploit a structural property of the data (most often sortedness, or an invariant you maintain yourself) to collapse a nested-loop search into a single linear pass.

The default instinct for "find a pair (or triplet) satisfying some condition" is to check every pair: two nested loops, O(n²). Two pointers exists because, given the right structural property, you can *prove* that large regions of that search space can never contain the answer — so you never check them. The pointers aren't a stylistic trick; each pointer movement encodes a small proof of which candidates are safe to discard.

### The Problem It Solves

Brute force over an array of size `n` for a pair-condition problem costs O(n²) time. Sorting first and using two pointers reduces the pair-search itself to O(n) — the dominant cost becomes the O(n log n) sort, or O(n) if the array is already sorted. For in-place compaction/partitioning problems, the brute-force alternative is usually "allocate a new array and copy what you want into it" — O(n) extra space. Two pointers (same-direction flavor) removes that extra space entirely, which is exactly why interviewers attach an "O(1) extra space" constraint to these problems: it's a direct signal to use this pattern.

## Why "Two Pointers"

"Pointer" here means **index**, not a memory address — don't confuse this with Go's `*T`. The name is literal: two index variables walk the same array (or string), as opposed to the single index of an ordinary `for` loop. The term is generic across languages and predates any particular one; it shows up identically in C, Java, Python, and Go interview contexts.

## Opposite-End (Converging) Pointers

Start `lo = 0`, `hi = n-1`. Move them toward each other until they meet.

### The Correctness Argument: Elimination by Domination

Every converging two-pointer proof has the same shape, regardless of the specific problem: at each step, compare the two current candidates by whatever metric the problem cares about. Whichever pointer is **provably worse than what has already been considered** gets discarded — permanently. Since each step retires exactly one index and never reconsiders it, the total work is O(n).

**Instantiation 1 — sum vs. target (sorted array).**

```
sum = numbers[lo] + numbers[hi]

sum < target → numbers[lo] is the bottleneck.
  Proof: for any k where lo < k < hi, numbers[k] ≤ numbers[hi] (sorted ascending),
  so numbers[lo] + numbers[k] ≤ numbers[lo] + numbers[hi] < target.
  numbers[lo] cannot reach target paired with anything still in range.
  Safe to discard → lo++

sum > target → symmetric argument on numbers[hi]. Safe to discard → hi--
```

Traced on `numbers = [2, 7, 11, 15]`, `target = 9`:

```
Step 1:  [2, 7, 11, 15]   lo=0  hi=3   sum = 2+15 = 17 > 9  → hi--
Step 2:  [2, 7, 11, 15]   lo=0  hi=2   sum = 2+11 = 13 > 9  → hi--
Step 3:  [2, 7, 11, 15]   lo=0  hi=1   sum = 2+7  = 9  == 9 → found, return [lo, hi]
```

**Instantiation 2 — maximize area (Container With Most Water).** Here there's no target to compare against; the metric is `area = width × min(height[lo], height[hi])`. The domination argument changes shape slightly but keeps the same structure: whichever wall is **shorter** is the bottleneck, because pairing that wall with anything further inside can only shrink the width while the height ceiling stays the same or gets worse. So the shorter wall is always safe to discard — it has already contributed its best possible area against the current opposite wall.

```
For lo with height[lo] < height[hi], and any k with lo < k < hi:
  area(lo, k)  = (k - lo)        × min(height[lo], height[k])
              ≤ (k - lo)        × height[lo]                  (min ≤ height[lo] always)
              ≤ (hi - lo)       × height[lo]                  (k - lo < hi - lo)
              = area(lo, hi)
→ lo can never beat the area already computed at (lo, hi). Discard lo.
```

The two instantiations look different on the surface — one compares against a fixed target, the other maximizes a derived quantity — but they're the same proof template: identify which pointer's current position is dominated, discard it, never look back.

## Same-Direction (Read/Write) Pointers

Both pointers move forward at the same rate, but with different roles: `read` scans every element; `write` marks the boundary of what has already been committed to keep. This is used for in-place compaction, filtering, and deduplication.

### The Correctness Argument: The `write ≤ read` Invariant

The entire technique rests on one invariant, maintained at every step: **`write ≤ read`.**

- Discard the current element: `read++`, `write` unchanged → invariant trivially holds (it only got slacker).
- Keep the current element: copy `arr[read]` into `arr[write]`, then `write++; read++` → still `write ≤ read`, since both advanced together.

Because `write` can never get ahead of `read`, you never overwrite a value before you've had the chance to read it. That's the entire reason this works with O(1) extra space: a naive filter would allocate a second array and copy kept elements into it. Here, the same array serves as both source and destination — `read` reads the original, unmodified values ahead of `write`, while `write` overwrites already-consumed slots behind it. The `write ≤ read` invariant is what guarantees those two roles never collide.

Traced on Remove Duplicates from Sorted Array, `nums = [1, 1, 2, 2, 3]` — here the `keep` rule is "not a duplicate of the last kept value":

```
write=0  read=0

read=0  val=1  new (first element) → nums[0]=1, write=1
read=1  val=1  duplicate            → skip
read=2  val=2  new                  → nums[1]=2, write=2
read=3  val=2  duplicate            → skip
read=4  val=3  new                  → nums[2]=3, write=3

Kept region [0, write) = [1, 2, 3]
```

### A Filtering Trace (Distinct From Deduplication)

Deduplication is one specific instance of same-direction filtering — the `keep` rule happens to be "not equal to the last kept value." The technique itself is more general: `keep` can be *any* predicate on the current element, with no reference to neighboring elements at all. The `write ≤ read` invariant doesn't change either way.

Traced on "keep only positive numbers," `nums = [-1, 3, -2, 5, 4]`:

```
write=0  read=0

read=0  val=-1  keep(-1)=false → skip
read=1  val=3   keep(3)=true   → nums[0]=3, write=1
read=2  val=-2  keep(-2)=false → skip
read=3  val=5   keep(5)=true   → nums[1]=5, write=2
read=4  val=4   keep(4)=true   → nums[2]=4, write=3

Kept region [0, write) = [3, 5, 4]
```

Notice `keep` here never looks at `write-1` or any neighboring element — each decision is entirely local to `nums[read]`. That's the general case; dedup's "compare to the previous kept value" is a special case of it. The `KeepIf` function in the Go template below implements this general form directly.

### Not to Be Confused With Fast & Slow Pointers

A pattern you'll meet later (`patterns/fast-slow-pointers.md`) also uses two index variables and sounds similar — but it's a different technique entirely: there, both pointers move through a **linked list** at **different speeds** (Floyd's cycle detection), and the goal is finding cycles or midpoints, not partitioning an array. Here, both pointers move at the *same* speed; the distinction is *role* (reader vs. boundary marker), not speed. Worth keeping these mentally separate, since the names are easy to conflate later.

## Partition Problems (Dutch National Flag)

Same-direction compaction generalizes to **three** pointers when you need three output regions instead of two (keep/discard). This is the Dutch National Flag problem: partition an array of three distinct values into three contiguous blocks in a single pass, O(1) space.

At any point mid-scan, the array holds four regions, but only three pointers are needed because each pointer marks a *boundary* between two adjacent regions:

```
[ 0 0 0 | 1 1 1 | ? ? ? | 2 2 2 ]
        low     mid     high
```

- **`low`** is the boundary between the confirmed-0 region and the confirmed-1 region. Everything to its left is a settled 0.
- **`mid`** is the boundary between the confirmed-1 region and the unclassified region. It is also the *active scanner* — `nums[mid]` is the element being examined right now.
- **`high`** is the boundary between the unclassified region and the confirmed-2 region. Everything to its right is a settled 2.

The **confirmed-1 region** is simply the span `[low, mid)` — every element that `mid` has already examined and determined to be a `1`. Nothing further happens to this span; it just grows as `mid` moves forward. It's "confirmed" in the same sense as the 0s and 2s: settled, and never revisited.

The three cases on `nums[mid]`:

- **`== 0`**: swap `nums[low]` and `nums[mid]`, then `low++; mid++`. Both advance because the value that was sitting at `low` — and is now swapped into `mid`'s old slot — was itself a confirmed `1` (it came from inside the confirmed-1 region, which starts exactly at `low`). Since it's already classified, `mid` is safe to move past it without re-examining it.
- **`== 2`**: swap `nums[mid]` and `nums[high]`, then `high--` **only**. `mid` does *not* advance, because the value swapped into `mid`'s slot came from the *unclassified* region (from position `high`, which was still unexamined) — it must be examined next, not skipped.
- **`== 1`**: it already belongs where it is, at the front of the unclassified region. Just `mid++`, extending the confirmed-1 region by one.

The asymmetry between the `0` case and the `2` case — whether `mid` advances — is the one detail that trips people up, and it follows directly from *where the swapped-in value came from*: from `low`, it's pre-classified (safe to skip); from `high`, it's still unknown (must stay in front of `mid`).

## Complexity Reference

| Technique | Time | Space | Replaces |
|---|---|---|---|
| Converging (pair condition, sorted) | O(n) | O(1) | O(n²) brute-force pair check |
| Converging (with prior sort) | O(n log n) | O(1) extra | O(n²) or O(n³) (e.g. 3Sum) |
| Same-direction (read/write) | O(n) | O(1) | O(n) extra-array filter/copy |
| Dutch flag (3-way partition) | O(n), one pass | O(1) | Two-pass counting sort |

## Recognizing the Pattern

Sorted input — or input you're explicitly allowed to sort because order doesn't matter for the answer — combined with a pair or triplet condition is the strongest signal for converging pointers. An "in-place" or "O(1) extra space" constraint on a filtering, deduplication, or partitioning problem signals same-direction. Three output buckets instead of two (not just keep/discard) signals the Dutch flag generalization.

## Go Template Code

```go
package twopointers

// ===== Opposite-End (Converging) Pointers =====

// TwoSumSorted returns the 1-indexed positions of two numbers in a sorted
// slice that add up to target, or nil if no such pair exists.
// Time: O(n) | Space: O(1)
func TwoSumSorted(numbers []int, target int) []int {
	lo, hi := 0, len(numbers)-1
	for lo < hi {
		sum := numbers[lo] + numbers[hi]
		switch {
		case sum == target:
			return []int{lo + 1, hi + 1}
		case sum < target:
			lo++ // numbers[lo] can't reach target with anything <= numbers[hi]; discard it
		default:
			hi-- // numbers[hi] overshoots even with the smallest remaining lo; discard it
		}
	}
	return nil
}

// MaxArea returns the largest area enclosed between two walls in height,
// using the elimination argument: the shorter wall is always the bottleneck.
// Time: O(n) | Space: O(1)
func MaxArea(height []int) int {
	lo, hi := 0, len(height)-1
	best := 0
	for lo < hi {
		h := minOf(height[lo], height[hi])
		area := (hi - lo) * h
		if area > best {
			best = area
		}
		if height[lo] < height[hi] {
			lo++ // lo is the bottleneck; nothing it pairs with inside [lo,hi] can beat this area
		} else {
			hi--
		}
	}
	return best
}

// ===== Same-Direction (Read/Write) Pointers =====

// KeepIf compacts arr in-place, keeping only elements that satisfy keep,
// preserving relative order. Returns the count of kept elements.
// Time: O(n) | Space: O(1)
func KeepIf(arr []int, keep func(int) bool) int {
	write := 0
	for read := 0; read < len(arr); read++ {
		if keep(arr[read]) {
			arr[write] = arr[read]
			write++
		}
	}
	return write
}

// ===== Partition (Dutch National Flag) =====

// SortColors sorts a slice of 0s, 1s, and 2s in-place in one pass using
// three pointers: low marks the end of the 0-region, high marks the start
// of the 2-region, and mid scans the unclassified middle.
// Time: O(n) | Space: O(1)
func SortColors(nums []int) {
	low, mid, high := 0, 0, len(nums)-1
	for mid <= high {
		switch nums[mid] {
		case 0:
			nums[low], nums[mid] = nums[mid], nums[low]
			low++
			mid++ // swapped-in value at mid is a known 1 — already classified
		case 1:
			mid++ // already in the correct region, just extend the boundary
		default: // 2
			nums[mid], nums[high] = nums[high], nums[mid]
			high--
			// mid does NOT advance — the swapped-in value is unclassified
		}
	}
}

// ===== Helpers =====

func minOf(a, b int) int {
	if a < b {
		return a
	}
	return b
}

func absOf(x int) int {
	if x < 0 {
		return -x
	}
	return x
}
```

## Where JavaScript Differs

The pattern itself is identical — the only meaningful divergence is idiomatic swapping and the lack of a typed numeric slice. Go's tuple assignment (`a[i], a[j] = a[j], a[i]`) becomes array destructuring in JS:

```javascript
// Same-direction swap idiom (used inside Sort Colors / Dutch flag)
[nums[low], nums[mid]] = [nums[mid], nums[low]];
```

Everything else — the `lo`/`hi` convergence loop, the `write`/`read` compaction loop, the three-pointer partition — is a direct line-for-line translation; there's no Go-specific mechanic (slice headers, aliasing) that changes the algorithm here, unlike in `arrays.md`.

## LeetCode Problems

### 1. Valid Palindrome — [#125 (Easy)](https://leetcode.com/problems/valid-palindrome/)

Given a string, determine whether it reads the same forwards and backwards after considering only letters and digits and ignoring case.

<details>
<summary>Hint 1</summary>

You don't need to build a cleaned-up copy of the string first. Can two pointers starting at each end simply skip over non-alphanumeric characters as they move inward?
</details>

<details>
<summary>Hint 2</summary>

Move `lo` forward and `hi` backward, skipping non-alphanumeric characters at each position. Once both land on alphanumeric characters, compare them case-insensitively — a mismatch means it's not a palindrome.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func isPalindrome(s string) bool {
	isAlnum := func(b byte) bool {
		return (b >= 'a' && b <= 'z') || (b >= 'A' && b <= 'Z') || (b >= '0' && b <= '9')
	}
	toLower := func(b byte) byte {
		if b >= 'A' && b <= 'Z' {
			return b + ('a' - 'A')
		}
		return b
	}
	lo, hi := 0, len(s)-1
	for lo < hi {
		for lo < hi && !isAlnum(s[lo]) {
			lo++
		}
		for lo < hi && !isAlnum(s[hi]) {
			hi--
		}
		if toLower(s[lo]) != toLower(s[hi]) {
			return false
		}
		lo++
		hi--
	}
	return true
}
```

This is byte-based, which is correct for ASCII input. For full Unicode correctness you'd convert to a `[]rune` first, since a multi-byte rune can't be indexed by single bytes.
</details>

### 2. Two Sum II — Input Array Is Sorted — [#167 (Medium)](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/)

Given an array sorted in non-decreasing order and a target, return the 1-indexed positions of the two numbers that sum to target, using only constant extra space.

<details>
<summary>Hint 1</summary>

A hash map solves this in O(n) space, but ignores the fact that the array is sorted. What does sortedness let you do instead of a lookup table?
</details>

<details>
<summary>Hint 2</summary>

Start `lo` at the beginning, `hi` at the end. If the sum is too small, `numbers[lo]` is the bottleneck — move it up. If too large, `numbers[hi]` is the bottleneck — move it down.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func twoSum(numbers []int, target int) []int {
	lo, hi := 0, len(numbers)-1
	for lo < hi {
		sum := numbers[lo] + numbers[hi]
		switch {
		case sum == target:
			return []int{lo + 1, hi + 1}
		case sum < target:
			lo++
		default:
			hi--
		}
	}
	return nil
}
```
</details>

### 3. Container With Most Water — [#11 (Medium)](https://leetcode.com/problems/container-with-most-water/)

Given an array of heights representing vertical lines at each index, choose two lines that, together with the x-axis, form a container holding the most water. Return the maximum area.

<details>
<summary>Hint 1</summary>

Brute force checks every pair — O(n²). Starting from the widest possible container (the two ends) and shrinking inward, which wall should you move first?
</details>

<details>
<summary>Hint 2</summary>

The shorter of the two current walls is always the bottleneck on area. Moving the *taller* wall inward can only shrink the width while the height ceiling stays the same or gets worse. Always move the pointer at the shorter wall.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func maxArea(height []int) int {
	lo, hi := 0, len(height)-1
	best := 0
	for lo < hi {
		h := minOf(height[lo], height[hi])
		if area := (hi - lo) * h; area > best {
			best = area
		}
		if height[lo] < height[hi] {
			lo++
		} else {
			hi--
		}
	}
	return best
}

func minOf(a, b int) int {
	if a < b {
		return a
	}
	return b
}
```
</details>

### 4. 3Sum — [#15 (Medium)](https://leetcode.com/problems/3sum/)

Given an array, find all unique triplets that sum to zero.

<details>
<summary>Hint 1</summary>

Brute force is O(n³). If you fix one number, the remaining problem becomes "find two numbers that sum to a target" in the rest of the array — a problem you already have a linear-time solution for.
</details>

<details>
<summary>Hint 2</summary>

Sort the array. For each index `i`, run the converging two-pointer scan on the subarray after `i`, searching for pairs that sum to `-nums[i]`. Skip duplicate values for `i`, and after finding a match, skip duplicate values for `lo`/`hi` too, to avoid emitting duplicate triplets.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n^2) | Space: O(1) extra, excluding the sort and output
func threeSum(nums []int) [][]int {
	sort.Ints(nums)
	var result [][]int
	n := len(nums)
	for i := 0; i < n-2; i++ {
		if i > 0 && nums[i] == nums[i-1] {
			continue // skip duplicate anchors
		}
		lo, hi := i+1, n-1
		target := -nums[i]
		for lo < hi {
			sum := nums[lo] + nums[hi]
			switch {
			case sum == target:
				result = append(result, []int{nums[i], nums[lo], nums[hi]})
				lo++
				hi--
				for lo < hi && nums[lo] == nums[lo-1] {
					lo++ // skip duplicate lo
				}
				for lo < hi && nums[hi] == nums[hi+1] {
					hi-- // skip duplicate hi
				}
			case sum < target:
				lo++
			default:
				hi--
			}
		}
	}
	return result
}
```
</details>

### 5. Remove Duplicates from Sorted Array — [#26 (Easy)](https://leetcode.com/problems/remove-duplicates-from-sorted-array/)

Given a sorted array, remove duplicates in-place so each unique element appears once, and return the count of unique elements.

<details>
<summary>Hint 1</summary>

Since the array is sorted, every duplicate sits immediately next to the value it duplicates. Can a write pointer track the last unique value placed, and compare only against that?
</details>

<details>
<summary>Hint 2</summary>

A read pointer scans every element. Whenever `nums[read] != nums[write-1]`, it's a new unique value — copy it to `nums[write]` and advance `write`.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func removeDuplicates(nums []int) int {
	if len(nums) == 0 {
		return 0
	}
	write := 1
	for read := 1; read < len(nums); read++ {
		if nums[read] != nums[write-1] {
			nums[write] = nums[read]
			write++
		}
	}
	return write
}
```
</details>

### 6. Sort Colors — [#75 (Medium)](https://leetcode.com/problems/sort-colors/)

Given an array containing only 0s, 1s, and 2s, sort it in-place in a single pass, using O(1) extra space.

<details>
<summary>Hint 1</summary>

A two-pass counting sort works, but isn't a single pass. Can three pointers track three regions — confirmed 0s, confirmed 1s, and an unexamined middle — while scanning once?
</details>

<details>
<summary>Hint 2</summary>

`low` marks where the next 0 should go, `high` marks where the next 2 should go, `mid` scans the unknown middle. On seeing `0`, swap with `low` and advance both `low` and `mid`. On seeing `2`, swap with `high` and advance only `high` — the swapped-in value is still unclassified. On seeing `1`, just advance `mid`.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func sortColors(nums []int) {
	low, mid, high := 0, 0, len(nums)-1
	for mid <= high {
		switch nums[mid] {
		case 0:
			nums[low], nums[mid] = nums[mid], nums[low]
			low++
			mid++
		case 1:
			mid++
		default:
			nums[mid], nums[high] = nums[high], nums[mid]
			high--
		}
	}
}
```
</details>

### 7. Squares of a Sorted Array — [#977 (Easy)](https://leetcode.com/problems/squares-of-a-sorted-array/)

Given an array sorted in ascending order (which may include negative numbers), return an array of the squares of each number, also sorted in ascending order.

<details>
<summary>Hint 1</summary>

Squaring breaks the original order, since negative numbers become positive. But the largest squares must come from the values farthest from zero — where do those live in a sorted array?
</details>

<details>
<summary>Hint 2</summary>

The largest absolute values sit at one of the two ends. Use two pointers from both ends, compare absolute values, and fill the result array **from the back** — the largest square is placed last each time.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(n) for the result, required by the problem
func sortedSquares(nums []int) []int {
	n := len(nums)
	result := make([]int, n)
	lo, hi := 0, n-1
	for i := n - 1; i >= 0; i-- {
		if absOf(nums[lo]) > absOf(nums[hi]) {
			result[i] = nums[lo] * nums[lo]
			lo++
		} else {
			result[i] = nums[hi] * nums[hi]
			hi--
		}
	}
	return result
}

func absOf(x int) int {
	if x < 0 {
		return -x
	}
	return x
}
```
</details>

## How to Remember This

**Two Pointers = A Proof, Not a Trick.** Every converging-pointer move discards a candidate because it's *provably dominated* — by a target comparison, by a height ceiling, by whatever the problem's metric is. If you ever forget why moving a pointer is "safe," re-derive the domination argument for that specific metric.

**Converging: Worse Pointer Moves.** Whichever side is currently the bottleneck is the one that advances. Sum too small → the small side moves. Wall too short → the short side moves.

**Same-Direction: `write` Never Outruns `read`.** That single inequality is the entire correctness proof for every read/write compaction problem — dedup, filter, move-to-front, all of it.

**Dutch Flag: `mid` Only Trusts What `low` Gives It.** Swapping in from `low` is always safe to skip past (already classified); swapping in from `high` is never safe to skip past (still unknown). That asymmetry is the one fact to memorize about the algorithm.

**Not Fast & Slow.** Same-speed, different-role (this file) vs. different-speed, same-role (cycle detection, later file). If the problem involves a *linked list* and *cycles*, you're in the wrong file.

## Interview Cheat Sheet

### Signal Phrases to Use

- "Since the array is sorted, I can use two pointers from both ends instead of a hash map — that drops the space from O(n) to O(1)."
- "The shorter wall is always the bottleneck here, so I can safely discard it and move that pointer inward."
- "I'll fix one element and reduce the rest to a two-pointer sum problem, turning the O(n³) brute force into O(n²)."
- "I'll maintain a write pointer that never gets ahead of the read pointer, so the compaction happens safely in-place."
- "Since there are three categories instead of two, I need three pointers — Dutch National Flag — not just read/write."

### Red Flags to Avoid

| Mistake | Correct Approach |
|---|---|
| Using converging pointers on an unsorted array without sorting first (when order doesn't matter for the answer) | Sort first if order isn't part of the output requirement; converging pointers require the sortedness invariant to be valid |
| Moving both `lo` and `hi` on a tie (`sum == target` in a problem needing *all* pairs, not just one) | Decide explicitly what a match should do — return immediately, or advance both and continue scanning for more pairs |
| Forgetting duplicate-skipping in 3Sum, producing duplicate triplets | After a match, advance `lo`/`hi` past any repeated values before continuing |
| Letting `mid` advance after swapping with `high` in Dutch flag | The swapped-in value from `high` is unclassified — `mid` must examine it next, don't advance |
| Treating "two pointers" and "fast & slow pointers" as the same pattern | Different roles entirely — same-speed compaction here vs. different-speed cycle detection there |

### Common Interviewer Probes

| Probe | What They Are Looking For |
|---|---|
| "Why is it safe to move that pointer and never come back to it?" | The domination/elimination argument for the specific metric — not just "it usually works" |
| "Can you do this without sorting?" | Recognize whether the problem's output requires preserving original order; if not, sorting is free game |
| "What if there are duplicate values?" | For 3Sum-style problems: explicit duplicate-skipping logic after a match; for Two Sum II: usually unaffected since indices are still unique |
| "Walk through your invariant." | State `write ≤ read` (or the region boundaries for Dutch flag) explicitly, not just narrate the code |
| "What's the difference between this and what you'd use for a linked list cycle?" | Same-speed/role-based two pointers here vs. different-speed fast & slow pointers there |

## References

- [LeetCode: Two Pointers Tag](https://leetcode.com/tag/two-pointers/)
- [NeetCode: Two Pointers roadmap](https://neetcode.io/roadmap)
- [CP-Algorithms: Two Pointers technique](https://cp-algorithms.com/sequences/sparse-table.html)
- Sedgewick & Wayne — *Algorithms* (4th ed.), §1.4 (Analysis of Algorithms) for the elimination-argument style of proof
- [Dutch National Flag problem — Wikipedia](https://en.wikipedia.org/wiki/Dutch_national_flag_problem) — origin of the name and Dijkstra's original framing