---
Status: 🌳 Evergreen
Created: 2026-06-30
Last Updated: 2026-06-30
---

# Arrays

## Table of Contents

- [What It Is and Why It Exists](#what-it-is-and-why-it-exists)
- [Why "Array"](#why-array)
- [Memory Internals](#memory-internals)
- [Static vs Dynamic Arrays](#static-vs-dynamic-arrays)
- [Arrays in Go](#arrays-in-go)
- [2D Arrays](#2d-arrays)
- [In-Place Operations](#in-place-operations)
- [Complexity Reference](#complexity-reference)
- [Kadane's Algorithm](#kadanes-algorithm)
- [Go Template Code](#go-template-code)
- [LeetCode Problems](#leetcode-problems)
- [How to Remember This](#how-to-remember-this)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

## What It Is and Why It Exists

An array is a **contiguous block of memory** holding a fixed number of elements of the same type, each accessible by an integer index starting at zero.

The reason arrays exist is pure CPU arithmetic. A processor accesses memory by address — a number in a flat address space. If you want to store `n` values and retrieve any of them without scanning, you need a formula that maps an index directly to a memory address. Contiguity gives exactly that:

```
address(i) = base_address + (i × sizeof(element))
```

One multiplication, one addition. O(1) random access regardless of array size. No pointer chains, no searching. This is the minimal structure that makes indexed access possible — every other data structure is built on top of it.

### The Problems It Introduces

The same contiguity that enables O(1) access creates the defining cost: **insertion and deletion are O(n)**. To insert at position `i`, every element from `i` onwards must shift right one slot:

```
Insert X at index 1 in [A, B, C, D]:

Before: [ A | B | C | D |   ]
After:  [ A | X | B | C | D ]
               ↑
               every element right of X had to move
```

Two further limitations:

- **Memory fragmentation**: the OS must find a single contiguous free block at allocation time. A 1 GB array can fail to allocate even if 1.5 GB of total free memory exists, because it is fragmented across non-adjacent regions.
- **Slice aliasing** (Go-specific): two slices can share the same backing array. Mutations through one are visible through the other, leading to subtle bugs covered in the Go section below.

## Why "Array"

From Old French *areer* — "to arrange in order, to put in array". In everyday English, an "array" is an ordered arrangement of things. The word entered computing through FORTRAN (1957), where the concept was called *subscripted variables* — a variable `A` at position `i` written as `A(i)`. By the 1960s "array" had become the standard term across languages.

The name directly reflects the defining property: elements arranged in a strict linear sequence, each position unambiguously identified by its numeric index.

## Memory Internals

For an `int32` array `[10, 20, 30, 40]` at base address `0x1000`:

```
Index:    0         1         2         3
          +----+----+----+----+----+----+----+----+
Value:    |    10   |    20   |    30   |    40   |
          +----+----+----+----+----+----+----+----+
Address:  0x1000    0x1004    0x1008    0x100C

Element size: 4 bytes (int32)
Formula: address(i) = 0x1000 + (i × 4)
```

**Cache line effect.** A CPU cache line is typically 64 bytes. Accessing `arr[0]` loads 64 contiguous bytes into the L1 cache — meaning `arr[1]` through `arr[15]` (for int32) are already warm. Sequential array traversal is one of the fastest possible memory access patterns for this reason.

**Contrast with linked structures.** A linked list node at index `i` can live anywhere in memory. Traversing to index `i` requires following `i` pointer dereferences, each potentially causing a cache miss. Arrays pay nothing for sequential traversal; linked lists pay dearly.

## Static vs Dynamic Arrays

Every array implementation in every language is one of two kinds, and the distinction explains why "append" sometimes silently costs O(n).

**Static array** — size fixed at the moment of allocation. The compiler or runtime reserves exactly that many contiguous slots and never moves them. C's `int arr[10]`, Java's `int[] arr = new int[10]`, and Go's `[10]int` are static arrays. You cannot grow one — you can only allocate a new, larger array and copy the contents across.

**Dynamic array** — a static array wrapped with bookkeeping that makes it *appear* resizable. Internally it still allocates a fixed-size block, but it tracks how many slots are in use (`length`) versus how many are reserved (`capacity`). Adding an element past capacity triggers: allocate a new, larger static array → copy everything across → discard the old block. Python's `list`, Java's `ArrayList`, C++'s `vector`, and Go's slice are all dynamic arrays built this way.

```
Dynamic array = static array + { length, capacity, growth policy }
```

The reason dynamic arrays still claim O(1) *amortized* append, despite the occasional O(n) copy, is that expensive copies become exponentially rarer as the array grows — a 2× growth factor means the total cost across n appends is bounded by a constant multiple of n, not n². (Full amortized-analysis derivation lives in `fundamentals/complexity-analysis.md`.)

Go gives you a direct hands-on example of both ends of this spectrum, covered next.

## Arrays in Go

Go has two distinct array-like types that behave fundamentally differently.

### Fixed-Size Arrays

```go
var a [5]int            // zero-initialised: [0 0 0 0 0]
b := [3]int{1, 2, 3}    // literal initialisation
c := [...]int{4, 5, 6}  // compiler infers size as 3
```

Key properties:
- The size is **part of the type**: `[5]int` and `[6]int` are incompatible types.
- Arrays are **value types**: assigning one copies all elements into a new allocation.
- Small fixed arrays are stack-allocated by the compiler.

Fixed arrays are used for small, known-size structures (coordinates, checksums, lookup tables). For general-purpose sequential data, slices are idiomatic.

### Slices: The Idiomatic Go Array

A slice is not an array. It is a **3-field struct describing a view into an underlying backing array**:

```
Slice Header
+----------+
|  ptr     |  pointer to element 0 of this view in the backing array
+----------+
|  len     |  number of accessible elements through this slice
+----------+
|  cap     |  total slots from ptr to end of backing array
+----------+
```

`s := make([]int, 3, 5)` produces len=3, cap=5:

```
Slice Header           Backing Array (cap=5 slots)
+---------+            +---+---+---+---+---+
| ptr ----+----------> | 0 | 0 | 0 |   |   |
+---------+            +---+---+---+---+---+
| len: 3  |              0   1   2   3   4   (indices)
+---------+
| cap: 5  |            visible:  slots 0–2 (len)
+---------+            reserved: slots 3–4 (cap - len)
```

### Slice Growth and Append

`append` always returns a new slice header. If `len < cap`, it writes into the existing backing array and increments len. If `len == cap`, it allocates a larger backing array, copies all elements, then writes the new element.

Go's growth policy (approx, Go 1.18+):
- len < 256: new cap ≈ 2× old cap
- len ≥ 256: growth factor decreases toward 1.25×

```go
s := []int{1, 2, 3}  // len=3, cap=3
s = append(s, 4)      // cap exceeded → new backing array, cap≈6
                      // old backing array is now unreferenced
```

Because the backing array changes, `append` **must** be assigned back: `s = append(s, v)`. Discarding the return value is a common bug.

### Slice Aliasing

The most important Go-specific array hazard. Two slices sharing the same backing array see each other's mutations.

**Scenario A — append within capacity (shared backing array)**

```
a := make([]int, 3, 6)   // len=3, cap=6
b := append(a, 99)        // len < cap, no reallocation

Backing: [ 0 | 0 | 0 | 99 |   |   ]
           ↑
         a (len=3) and b (len=4) share this array

b[0] = 7  →  a[0] is now also 7
```

**Scenario B — append exceeds capacity (independent backing arrays)**

```
a := []int{1, 2, 3}   // len=3, cap=3
b := append(a, 4)      // cap exceeded → new allocation

Backing A: [ 1 | 2 | 3 ]          ←  a points here
Backing B: [ 1 | 2 | 3 | 4 ]      ←  b points to new memory

b[0] = 7  →  a[0] is unchanged
```

Whether you land in scenario A or B depends on the current capacity — which is invisible at the call site. **Never rely on sharing or independence after an append.** When you need a guaranteed independent copy:

```go
independent := make([]int, len(a))
copy(independent, a)

// or, more concisely:
independent := append([]int(nil), a...)
```

## 2D Arrays

A 2D array models a grid: rows and columns, accessed as `arr[i][j]`. There are two genuinely different memory layouts, and conflating them is a common source of bugs.

### Contiguous (Row-Major) Layout

A true 2D array is one flat block of memory, with row `i` placed immediately after row `i-1`. A single formula computes any cell's address:

```
address(i, j) = base + (i × cols + j) × element_size
```

For a 3×4 matrix:

```
Logical:                  Physical (row-major, flattened):
[ 0  1  2  3]              +---+---+---+---+---+---+---+---+---+---+---+---+
[ 4  5  6  7]      →       | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |10 |11 |
[ 8  9 10 11]              +---+---+---+---+---+---+---+---+---+---+---+---+
                              row 0          row 1          row 2
```

Row `i`, column `j` lives at flat index `i × 4 + j`. This is what `int arr[3][4]` gives you in C.

### Slice of Slices — Go's Default

Go has no built-in contiguous 2D type. `[][]int` is a **slice of slices**: an outer slice whose elements are themselves independent slice headers, each pointing to its own separately allocated backing array.

```
Stack                      Heap

Outer slice header         Outer backing array — stores inner slice headers inline
+-----------+              +------------------+------------------+------------------+
| ptr ------+----------->  | ptr | len | cap  | ptr | len | cap  | ptr | len | cap  |
| len: 3    |              +--+---------------+--+---------------+--+---------------+
| cap: 3    |                 |                  |                  |
+-----------+                 |                  |                  |
                              ↓ Row 0            ↓ Row 1            ↓ Row 2
                         +---+---+---+---+  +---+---+---+---+  +---+---+---+---+
                         | 0 | 1 | 2 | 3 |  | 4 | 5 | 6 | 7 |  | 8 | 9 |10 |11 |
                         +---+---+---+---+  +---+---+---+---+  +---+---+---+---+
                         (separate heap allocation per row)
```

The rows are **not guaranteed to be adjacent in memory** — each is its own allocation. Two consequences:

- **Jagged arrays are free.** Rows can have different lengths, since each is an independent slice — useful for adjacency lists or triangular matrices.
- **Cache locality holds within a row, not across rows.** Row-major traversal (outer loop over `i`, inner loop over `j`) stays cache-friendly because each row is contiguous. Row 1 is not guaranteed to sit anywhere near row 0 physically — unlike a true contiguous 2D array.

### Building a 2D Slice in Go

```go
rows, cols := 3, 4
grid := make([][]int, rows)
for i := range grid {
	grid[i] = make([]int, cols)
}
```

For guaranteed contiguous memory (matching the row-major model above), allocate one flat backing slice and window it into rows:

```go
flat := make([]int, rows*cols)
grid := make([][]int, rows)
for i := range grid {
	grid[i] = flat[i*cols : (i+1)*cols]
}
```

Every row is now a window into the same backing array — true row-major layout with the ergonomics of `grid[i][j]` indexing.

## In-Place Operations

An operation is **in-place** if it modifies the array using O(1) (or O(log n), by convention) auxiliary space — no second array, no copy. Interviewers ask for in-place solutions specifically to test direct index manipulation instead of reaching for an extra data structure.

Three mechanical primitives cover most in-place array operations:

**Shifting** — covered earlier in [The Problems It Introduces](#the-problems-it-introduces): inserting or deleting at position `i` shifts every element after `i` by one slot. O(n) per operation, O(1) space.

**Swapping** — exchanging two elements directly, as in `Reverse` (see Go Template Code below): `arr[lo], arr[hi] = arr[hi], arr[lo]`. No temporary needed.

**Write-pointer compaction** — to remove or reorder elements by a condition, walk the array once with a `read` pointer, advancing a separate `write` pointer only when an element should be kept:

```
Compact non-zero elements from [0, 1, 0, 3, 12]:

write=0, read scans left to right:
read=0: arr[0]=0  → skip (zero)
read=1: arr[1]=1  → arr[write]=1, write++ → write=1
read=2: arr[2]=0  → skip (zero)
read=3: arr[3]=3  → arr[write]=3, write++ → write=2
read=4: arr[4]=12 → arr[write]=12, write++ → write=3

After compaction: [1, 3, 12, ...], write=3
Remaining slots [write:] get filled with the skipped value (0 here).
```

This is a *mechanical primitive*, not the full two-pointer *pattern* (pair-finding, partitioning, and more across problem types) — that gets its own dedicated treatment in `patterns/two-pointers.md`.

## Complexity Reference

| Operation | Time | Space | Notes |
|---|---|---|---|
| Read / write by index | O(1) | O(1) | Direct address computation |
| 2D array access (row-major) | O(1) | O(1) | `address(i,j) = base + (i×cols+j)×size` |
| Append (amortized) | O(1) | O(1) | Growth events are rare on average |
| Append (worst case) | O(n) | O(n) | Full copy into new backing array |
| Insert at front or middle | O(n) | O(1) | Shift elements right |
| Delete from front or middle | O(n) | O(1) | Shift elements left |
| Linear search (unsorted) | O(n) | O(1) | |
| Binary search (sorted) | O(log n) | O(1) | |
| Sort (comparison-based) | O(n log n) | O(log n) | Go uses pdqsort |

## Kadane's Algorithm

Maximum subarray sum in O(n) time, O(1) space. At each index: the best sum of any subarray ending here is either `arr[i]` alone (start fresh) or `maxEndingHere + arr[i]` (extend). If the running sum has gone negative, it is dead weight — any subarray that includes it would do better starting fresh.

```
arr = [-2, 1, -3, 4, -1, 2, 1, -5, 4]

 i    arr[i]   maxEndingHere   maxSoFar
 0      -2          -2            -2
 1       1           1             1    ← started fresh (1 > -2+1)
 2      -3          -2             1
 3       4           4             4    ← started fresh (4 > -2+4)
 4      -1           3             4
 5       2           5             5
 6       1           6             6    ← global maximum
 7      -5           1             6
 8       4           5             6
```

The answer is 6, from subarray `[4, -1, 2, 1]`.

## Go Template Code

```go
package arrays

// ===== Foundational Operations =====

// Reverse reverses a slice in-place using two pointers.
// Time: O(n) | Space: O(1)
func Reverse(arr []int) {
	lo, hi := 0, len(arr)-1
	for lo < hi {
		arr[lo], arr[hi] = arr[hi], arr[lo]
		lo++
		hi--
	}
}

// BinarySearch returns the index of target in a sorted slice, or -1 if absent.
// Uses lo + (hi-lo)/2 to avoid integer overflow in languages where int can overflow;
// Go's int is 64-bit on modern platforms but the habit is worth keeping.
// Time: O(log n) | Space: O(1)
func BinarySearch(arr []int, target int) int {
	lo, hi := 0, len(arr)-1
	for lo <= hi {
		mid := lo + (hi-lo)/2
		switch {
		case arr[mid] == target:
			return mid
		case arr[mid] < target:
			lo = mid + 1
		default:
			hi = mid - 1
		}
	}
	return -1
}

// ===== In-Place Operations =====

// CompactNonZero moves all non-zero elements to the front, preserving order,
// and fills the remaining tail with zero. Write-pointer compaction.
// Time: O(n) | Space: O(1)
func CompactNonZero(arr []int) {
	write := 0
	for read := 0; read < len(arr); read++ {
		if arr[read] != 0 {
			arr[write] = arr[read]
			write++
		}
	}
	for ; write < len(arr); write++ {
		arr[write] = 0
	}
}

// ===== Kadane's Algorithm =====

// MaxSubarraySum returns the maximum contiguous subarray sum (Kadane's).
// Handles all-negative arrays correctly — returns the least-negative element.
// Time: O(n) | Space: O(1)
func MaxSubarraySum(arr []int) int {
	if len(arr) == 0 {
		return 0
	}
	maxHere := arr[0]
	maxSoFar := arr[0]
	for _, v := range arr[1:] {
		extended := maxHere + v
		if v > extended {
			maxHere = v // start fresh
		} else {
			maxHere = extended // extend
		}
		if maxHere > maxSoFar {
			maxSoFar = maxHere
		}
	}
	return maxSoFar
}

// ===== Helpers =====

// maxOf returns the larger of two ints.
// Time: O(1) | Space: O(1)
func maxOf(a, b int) int {
	if a > b {
		return a
	}
	return b
}

// minOf returns the smaller of two ints.
// Time: O(1) | Space: O(1)
func minOf(a, b int) int {
	if a < b {
		return a
	}
	return b
}
```

## LeetCode Problems

### 1. Two Sum — [#1 (Easy)](https://leetcode.com/problems/two-sum/)

Given an integer array and a target, return indices of the two numbers that add up to the target. Exactly one solution exists; you may not use the same element twice.

<details>
<summary>Hint 1</summary>

For each element `v`, the value that pairs with it to reach `target` is `target - v`. Can you check in O(1) whether that complement has already been seen?
</details>

<details>
<summary>Hint 2</summary>

Use a hash map from value → index. For each element, check if its complement is already in the map. If yes, you have your pair. If no, store the current element.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(n)
func twoSum(nums []int, target int) []int {
	seen := make(map[int]int) // value → index
	for i, v := range nums {
		if j, ok := seen[target-v]; ok {
			return []int{j, i}
		}
		seen[v] = i
	}
	return nil
}
```
</details>

### 2. Best Time to Buy and Sell Stock — [#121 (Easy)](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/)

Given daily prices, find the maximum profit from a single buy-sell transaction. You must buy before you sell.

<details>
<summary>Hint 1</summary>

The profit from selling on day `i` is `prices[i] - cheapest_price_seen_so_far`. Scan left to right, tracking the running minimum.
</details>

<details>
<summary>Hint 2</summary>

At each index, compute `prices[i] - minPrice`. Update `maxProfit` if this beats the current best, then update `minPrice` if `prices[i]` is a new low.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func maxProfit(prices []int) int {
	if len(prices) == 0 {
		return 0
	}
	minPrice := prices[0]
	maxProfit := 0
	for _, p := range prices[1:] {
		profit := p - minPrice
		if profit > maxProfit {
			maxProfit = profit
		}
		if p < minPrice {
			minPrice = p
		}
	}
	return maxProfit
}
```
</details>

### 3. Contains Duplicate — [#217 (Easy)](https://leetcode.com/problems/contains-duplicate/)

Return `true` if any value appears at least twice in the array.

<details>
<summary>Hint</summary>

Use a hash set (in Go: `map[int]struct{}`). For each element, if it is already in the set, return true. Otherwise add it.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(n)
func containsDuplicate(nums []int) bool {
	seen := make(map[int]struct{})
	for _, v := range nums {
		if _, ok := seen[v]; ok {
			return true
		}
		seen[v] = struct{}{}
	}
	return false
}
```
</details>

### 4. Move Zeroes — [#283 (Easy)](https://leetcode.com/problems/move-zeroes/)

Given an integer array, move all zeroes to the end while preserving the relative order of non-zero elements. Must be done in-place, no copy.

<details>
<summary>Hint 1</summary>

A single forward pass can do this with two roles: a pointer reading every element, and a separate pointer marking where the next non-zero value should land.
</details>

<details>
<summary>Hint 2</summary>

Use a write pointer starting at 0. For each element, if it's non-zero, place it at `arr[write]` and increment `write`. After the scan, fill every position from `write` to the end with zero.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func moveZeroes(nums []int) {
	write := 0
	for read := 0; read < len(nums); read++ {
		if nums[read] != 0 {
			nums[write] = nums[read]
			write++
		}
	}
	for ; write < len(nums); write++ {
		nums[write] = 0
	}
}
```
</details>

### 5. Maximum Subarray — [#53 (Medium)](https://leetcode.com/problems/maximum-subarray/)

Find the contiguous subarray with the largest sum. Classic Kadane's application.

<details>
<summary>Hint 1</summary>

At each element you have a binary choice: start a new subarray here, or extend the running one. Which gives a larger sum ending at this position?
</details>

<details>
<summary>Hint 2</summary>

`maxEndingHere = max(nums[i], maxEndingHere + nums[i])`. If `maxEndingHere` has gone negative, starting fresh (`nums[i]` alone) is always better. Update `maxSoFar` at each step.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func maxSubArray(nums []int) int {
	maxHere := nums[0]
	maxSoFar := nums[0]
	for _, v := range nums[1:] {
		if v > maxHere+v {
			maxHere = v
		} else {
			maxHere += v
		}
		if maxHere > maxSoFar {
			maxSoFar = maxHere
		}
	}
	return maxSoFar
}
```
</details>

### 6. Maximum Product Subarray — [#152 (Medium)](https://leetcode.com/problems/maximum-product-subarray/)

Find the contiguous subarray with the largest product.

<details>
<summary>Hint 1</summary>

Unlike sum, a large negative product can become a large positive if multiplied by another negative. Tracking only the running maximum is insufficient — a large negative now could become the best later.
</details>

<details>
<summary>Hint 2</summary>

Track both `maxHere` and `minHere` at each step. At each element, the new max could come from any of: the element alone, `maxHere × v`, or `minHere × v` (if `v` is negative, max and min swap). Update the global result from `maxHere`.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func maxProduct(nums []int) int {
	maxHere := nums[0]
	minHere := nums[0]
	result := nums[0]
	for _, v := range nums[1:] {
		// Compute both candidates before updating (avoid using partially updated values)
		newMax := maxOf(v, maxOf(maxHere*v, minHere*v))
		newMin := minOf(v, minOf(maxHere*v, minHere*v))
		maxHere, minHere = newMax, newMin
		if maxHere > result {
			result = maxHere
		}
	}
	return result
}

func maxOf(a, b int) int {
	if a > b {
		return a
	}
	return b
}

func minOf(a, b int) int {
	if a < b {
		return a
	}
	return b
}
```
</details>

### 7. Set Matrix Zeroes — [#73 (Medium)](https://leetcode.com/problems/set-matrix-zeroes/)

Given an m×n matrix, if a cell is 0, set its entire row and column to 0. Do it in-place with O(1) extra space.

<details>
<summary>Hint 1</summary>

The straightforward approach records which rows/columns to zero using two separate sets — O(m+n) space. Can the matrix's own first row and first column serve as that marker storage instead?
</details>

<details>
<summary>Hint 2</summary>

Use `matrix[i][0]` and `matrix[0][j]` as markers for row `i` and column `j`. Since `matrix[0][0]` is shared by both the first row's and first column's markers, track whether the first row itself needs zeroing with one separate boolean before you start overwriting it.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(m*n) | Space: O(1)
func setZeroes(matrix [][]int) {
	rows, cols := len(matrix), len(matrix[0])

	firstRowHasZero := false
	for j := 0; j < cols; j++ {
		if matrix[0][j] == 0 {
			firstRowHasZero = true
		}
	}
	firstColHasZero := false
	for i := 0; i < rows; i++ {
		if matrix[i][0] == 0 {
			firstColHasZero = true
		}
	}

	// Use first row/col as markers, derived from the rest of the matrix
	for i := 1; i < rows; i++ {
		for j := 1; j < cols; j++ {
			if matrix[i][j] == 0 {
				matrix[i][0] = 0
				matrix[0][j] = 0
			}
		}
	}

	// Apply markers to the inner matrix (row/col 0 handled last)
	for i := 1; i < rows; i++ {
		for j := 1; j < cols; j++ {
			if matrix[i][0] == 0 || matrix[0][j] == 0 {
				matrix[i][j] = 0
			}
		}
	}

	if firstRowHasZero {
		for j := 0; j < cols; j++ {
			matrix[0][j] = 0
		}
	}
	if firstColHasZero {
		for i := 0; i < rows; i++ {
			matrix[i][0] = 0
		}
	}
}
```
</details>

## How to Remember This

**Array = Formula Access.**
O(1) random access exists because `address = base + index × size`. If you ever forget why arrays are fast, derive the formula. Everything else follows.

**Slice = Ptr + Len + Cap.**
When confused about aliasing or append behaviour, draw the 3-field struct on paper. Once you can see the backing array and where each slice header points, the answer is always clear.

**Dynamic Array = Static Array + Bookkeeping.**
Resizing isn't magic — it's "allocate bigger, copy everything, discard the old block," wrapped in `len`/`cap` so you never see it happen directly.

**2D Slice = Pointers to Pointers.**
Go's `[][]int` is not one block — it's an outer slice of independently allocated row slices. Only a manually flattened single slice gives you true contiguous row-major memory.

**In-Place = Write Pointer Lags Read Pointer.**
Whenever you need to compact or filter an array without extra space, a read pointer scans ahead and a write pointer only advances when something is worth keeping.

**Kadane's: "Cut Negative Ballast."**
If the running sum goes negative, it drags down every future subarray that includes it. Cut it loose and start fresh. The algorithm is this observation applied at every index.

## Interview Cheat Sheet

### Signal Phrases to Use

- "I'll use a hash map for O(1) complement lookup, making the overall solution O(n) time and O(n) space."
- "A Go slice is a dynamic array — a fixed-size backing array plus length/capacity bookkeeping that handles resizing."
- "I'll keep the inner loop over columns for row-major traversal — that keeps memory access cache-friendly."
- "I can use the matrix's own first row and column as O(1) markers instead of a separate set."
- "This is Kadane's — at each position I either extend the running subarray or start fresh, whichever is larger."
- "Slice append in Go is amortized O(1). The occasional O(n) copy is paid for by all the cheap appends preceding it."
- "I need to `copy` the slice here to avoid aliasing — both would otherwise share the same backing array."

### Red Flags to Avoid

| Mistake | Correct Approach |
|---|---|
| `mid = (lo + hi) / 2` | Use `lo + (hi-lo)/2`; the habit prevents overflow in other languages |
| Allocating a new array when the problem says "in-place" | Reuse the existing array via shifting, swapping, or write-pointer compaction |
| Column-major traversal of a `[][]int` (`for j { for i {...} }`) | Row-major (`for i { for j {...} }`) keeps the inner loop cache-friendly; Go's slice-of-slices rows aren't guaranteed adjacent |
| Mutating a slice parameter assuming the caller is unaffected | Go passes the slice header by value, but the backing array is shared — callers see mutations |
| Assuming `append` always shares (or always copies) the backing array | Depends on remaining capacity; never rely on either without explicit `copy` |
| Kadane's returning 0 for all-negative input | The algorithm correctly returns the least-negative element; only return 0 if the problem explicitly says the subarray can be empty |

### Common Interviewer Probes

| Probe | What They Are Looking For |
|---|---|
| "Can you do this in O(1) space?" | Eliminate auxiliary structures; use rolling variables or write-pointer compaction instead of extra arrays |
| "Is your `[][]int` actually contiguous in memory?" | No — it's independently allocated row slices; only a manually flattened single slice is truly contiguous |
| "Can you avoid the O(m+n) extra space for row/column tracking?" | Reuse the matrix's own first row and column as in-place markers |
| "How would you handle integer overflow in the product?" | Cast to int64 before multiplying, or check bounds before each multiplication |
| "Trace your algorithm step by step on this input." | Walk through the trace table (i, value, state variables) row by row |
| "Why track both max and min in Maximum Product Subarray?" | A large negative can flip to the best positive when multiplied by another negative — tracking only max loses that case |

## References

- Cormen, Leiserson, Rivest, Stein — *Introduction to Algorithms* (CLRS), Ch. 2 (Insertion Sort and Loop Invariants)
- Sedgewick & Wayne — *Algorithms* (4th ed.), §1.1 (Primitive Types and Arrays)
- [Go Specification: Slice types](https://go.dev/ref/spec#Slice_types)
- [Go Blog: Arrays, slices, strings — the mechanics of append](https://go.dev/blog/slices-intro)
- [NeetCode: Arrays and Hashing roadmap](https://neetcode.io/roadmap)
- [LeetCode: Top Interview 150](https://leetcode.com/studyplan/top-interview-150/)