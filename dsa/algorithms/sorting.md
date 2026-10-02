---
Status: 🌳 Evergreen
Created: 2026-10-02
Last Updated: 2026-10-02
---

# Sorting

## Table Of Contents

1. [What It Is And Why It Exists](#what-it-is-and-why-it-exists)
2. [Classification Axes](#classification-axes)
3. [The Ω(n log n) Lower Bound For Comparison Sorts](#the-ω-n-log-n-lower-bound-for-comparison-sorts)
4. [Bubble Sort](#bubble-sort)
5. [Selection Sort](#selection-sort)
6. [Insertion Sort](#insertion-sort)
7. [Merge Sort](#merge-sort)
8. [Quick Sort](#quick-sort)
9. [Heap Sort](#heap-sort)
10. [Counting Sort](#counting-sort)
11. [Radix Sort](#radix-sort)
12. [Complexity Reference](#complexity-reference)
13. [Sorting In Go](#sorting-in-go)
14. [Recognizing Which Sort To Use](#recognizing-which-sort-to-use)
15. [LeetCode Problems](#leetcode-problems)
16. [References](#references)

## What It Is And Why It Exists

Sorting arranges a sequence of elements into a total order defined by some comparison rule (usually `<` on the key). The point is rarely the sorted output itself. The point is that sortedness is a **precondition** that unlocks cheaper downstream operations: binary search becomes O(log n), duplicate detection becomes O(n), two-pointer techniques on pair-sum problems become O(n), median and percentiles become O(1) lookups, and interval or event-line sweeps become linear.

Almost every algorithm in this file existed before computers. They are catalogued together because they represent fundamentally different **strategies for exploiting structure**: pairwise-swap adjacent inversions (bubble), pick the extremum each round (selection), grow a sorted prefix one element at a time (insertion), divide-and-conquer with a merge (merge), divide-and-conquer with a partition (quick), heap-based extraction (heap), key-histogram (counting), and digit-by-digit stable passes (radix). Each of those strategies has a specific shape of input where it is the right answer and a specific shape where it is embarrassingly bad, which is why "just use one sort" is not the industry consensus.

### Why "Sort"

From the Latin *sortiri* meaning "to draw lots, to allocate by category". Same root as "sort" meaning "type" or "kind". A sorted sequence is one where every element has been assigned its correct position relative to a rule. The verb predates computing by centuries and was borrowed wholesale.

## Classification Axes

Every sorting algorithm sits at one point on each of these five axes. Interviewers ask "which sort would you use" as a shorthand for "walk me through these axes and defend your choice."

**Comparison vs non-comparison.** A comparison sort only ever asks "is a < b?" and treats keys as opaque. Bubble, selection, insertion, merge, quick, and heap are comparison sorts. Counting and radix are not: they inspect the key's structure (its integer value, its digits, its bytes). This distinction determines whether the Ω(n log n) lower bound applies. Comparison sorts are bounded below by it. Non-comparison sorts escape it by using extra information about the key domain.

**Stable vs unstable.** Stable means: two elements with the same key preserve their original relative order. This matters when sorting on a secondary key after a primary sort, since the primary order must survive the second pass. Merge, insertion, bubble, counting, and radix are stable. Quick and heap are not (in their standard forms). Selection is not stable in the usual swap-based implementation.

**In-place vs out-of-place.** In-place means O(1) or O(log n) auxiliary memory beyond the input. Bubble, selection, insertion, quick, and heap are in-place. Merge is not (O(n) auxiliary in the standard top-down version). Counting and radix use O(n + k) auxiliary. When you have a 10 GB file to sort and 8 GB of RAM, this axis is the whole question.

**Adaptive vs non-adaptive.** Adaptive means the algorithm gets faster on already-sorted or nearly-sorted input. Insertion is the canonical adaptive sort (O(n) on sorted input). Bubble with early-exit is adaptive. Merge and heap are not adaptive: they do the same work regardless of input order. This is why Go's standard library uses a hybrid that falls through to insertion on small or nearly-sorted subarrays.

**Internal vs external.** Internal sorting assumes the entire input fits in memory. External sorting is what you do when it doesn't: merge sort variants dominate here, because merging streams from disk is the natural fit. Not covered further in this file, but worth naming.

## The Ω(n log n) Lower Bound For Comparison Sorts

Any algorithm that only compares pairs of elements (never looks at their value directly) needs at least ⌈log₂(n!)⌉ comparisons in the worst case. Since log₂(n!) = Θ(n log n) by Stirling's approximation, no comparison sort can do better than Θ(n log n) worst case. This is a property of the **problem**, not of any specific algorithm.

**Argument.** Model any comparison sort as a decision tree. Each internal node is one comparison (`a[i] < a[j]?`), each leaf is one of the n! possible input permutations the algorithm might be facing. The algorithm's path through the tree is determined by the outcomes of its comparisons, and it must reach a distinct leaf for every distinct input permutation (otherwise it would produce the same output for two inputs that need different rearrangements). A binary tree with n! leaves has depth ≥ ⌈log₂(n!)⌉. The worst-case path length equals the worst-case number of comparisons. Done.

**Why this matters.** It tells you merge sort and heap sort are asymptotically optimal for comparison sorts. It also tells you counting sort and radix sort don't violate any law of physics when they beat that bound: they aren't comparison sorts, so the theorem does not apply to them.

## Bubble Sort

Repeatedly walk the array; on each pass, compare adjacent pairs and swap if out of order. After the k-th full pass, the k largest elements are guaranteed to have "bubbled" to the correct positions at the end. Stop when a full pass makes zero swaps (the early-exit optimization).

The name comes from the visual: larger elements rise to the top of the array like bubbles in water, one pass at a time.

```
Input: [5, 1, 4, 2, 8]

Pass 1: compare each adjacent pair, swap if out of order
  [5,1,4,2,8] -> [1,5,4,2,8] -> [1,4,5,2,8] -> [1,4,2,5,8] -> [1,4,2,5,8]
  (8 was already in place; largest is now at the end)

Pass 2:
  [1,4,2,5,8] -> [1,4,2,5,8] -> [1,2,4,5,8] -> [1,2,4,5,8]
  (5 was already correct; next-largest settled)

Pass 3: no swaps happen, early exit.
Result: [1,2,4,5,8]
```

```go
func BubbleSort(a []int) {
    n := len(a)
    for i := 0; i < n-1; i++ {
        swapped := false
        for j := 0; j < n-1-i; j++ {
            if a[j] > a[j+1] {
                a[j], a[j+1] = a[j+1], a[j]
                swapped = true
            }
        }
        if !swapped {
            return // already sorted, bail
        }
    }
}
```

**Complexity.** Worst and average: O(n²) comparisons and swaps. Best (already sorted, with early-exit): O(n). Space: O(1) auxiliary. Stable, in-place, adaptive.

**Why it exists in the syllabus but not in production.** Bubble sort is pedagogically the simplest sort to explain and to prove correct. In real code it is strictly dominated by insertion sort, which does the same amount of work but with fewer swaps and a lower constant factor. No serious library uses bubble sort. Its job is to teach the invariant "after pass k, the tail k elements are final."

## Selection Sort

Divide the array into a sorted prefix (initially empty) and an unsorted suffix. Each round, scan the suffix to find the minimum, then swap it into the first position of the suffix, growing the sorted prefix by one. After n-1 rounds, the whole array is sorted.

The name is direct: on each round, you **select** the smallest element from what remains and place it.

```
Input: [64, 25, 12, 22, 11]

Round 1: min of [64,25,12,22,11] is 11 at index 4. Swap index 0 and 4.
         [11 | 25, 12, 22, 64]

Round 2: min of [25,12,22,64] is 12 at index 2. Swap index 1 and 2.
         [11, 12 | 25, 22, 64]

Round 3: min of [25,22,64] is 22 at index 3. Swap index 2 and 3.
         [11, 12, 22 | 25, 64]

Round 4: min of [25,64] is 25. Already in place, self-swap.
         [11, 12, 22, 25 | 64]
```

```go
func SelectionSort(a []int) {
    n := len(a)
    for i := 0; i < n-1; i++ {
        minIdx := i
        for j := i + 1; j < n; j++ {
            if a[j] < a[minIdx] {
                minIdx = j
            }
        }
        a[i], a[minIdx] = a[minIdx], a[i]
    }
}
```

**Complexity.** Always O(n²) comparisons (the inner scan doesn't shrink meaningfully in cost until n is exhausted), but only O(n) swaps total (one per outer round). Space: O(1). Not stable in the swap-based version above (the swap can jump an equal-key element over another). Not adaptive.

**Why not stable, concretely.** Take `[5a, 3, 5b, 2]` where `5a` and `5b` share the sort key but carry different secondary data. Round 1 finds the min (`2` at index 3) and swaps it with index 0, giving `[2, 3, 5b, 5a]`. `5a` originally preceded `5b`; now it follows it, and no later round will fix that. The mechanism is that selection swaps two non-adjacent indices, so the element leaving position `i` can jump over any equal-key element sitting between `i` and the min-index. Bubble and insertion avoid this because they only swap adjacent pairs with strict `>`. A stable variant shifts the intervening elements right by one instead of swapping, trading O(n) swaps for O(n²) writes.

**Where it wins.** When writes are far more expensive than reads (writing to flash memory, wearing out EEPROM cells, or updating a slow persistent structure), selection sort's O(n) swap count is genuinely valuable. Nowhere else.

## Insertion Sort

Walk the array left to right. For each element, treat everything to its left as a sorted prefix and **insert** the current element into its correct position in that prefix by shifting larger elements one slot to the right.

The name mirrors what a card player does when picking up cards one at a time: hold the sorted hand, insert each new card into position.

```
Input: [5, 2, 4, 6, 1, 3]

i=1, key=2: prefix [5]. Shift 5 right, insert 2 at index 0.
            [2, 5 | 4, 6, 1, 3]

i=2, key=4: prefix [2, 5]. Shift 5, insert 4 at index 1.
            [2, 4, 5 | 6, 1, 3]

i=3, key=6: prefix [2,4,5]. 5 <= 6, no shift, insert 6 at index 3.
            [2, 4, 5, 6 | 1, 3]

i=4, key=1: prefix [2,4,5,6]. Shift all four right, insert 1 at index 0.
            [1, 2, 4, 5, 6 | 3]

i=5, key=3: shift 6,5,4, insert 3 at index 2.
            [1, 2, 3, 4, 5, 6]
```

```go
func InsertionSort(a []int) {
    for i := 1; i < len(a); i++ {
        key := a[i]
        j := i - 1
        // Shift larger elements one slot right; strict > keeps stability.
        for j >= 0 && a[j] > key {
            a[j+1] = a[j]
            j--
        }
        a[j+1] = key
    }
}
```

**Complexity.** Worst (reverse sorted): O(n²). Best (already sorted): O(n), because the inner loop exits on the first comparison. Average: O(n²) but with a small constant. Space: O(1). Stable, in-place, adaptive.

**Where it wins.** Small n (say n ≤ 32 or so). Nearly-sorted input. This is why every industrial-strength sort (introsort, pdqsort, Timsort) falls through to insertion sort for small subarrays or nearly-sorted runs. It has the lowest constant factor of any comparison sort on such inputs.

## Merge Sort

Divide-and-conquer. Split the array in half, recursively sort each half, then merge the two sorted halves into one sorted array.

The name is literal: the interesting work happens in the **merge** step. The split is trivial (compute a midpoint); the recursion is bookkeeping; the merge is where sortedness is actually assembled.

```
Split phase (top-down):
             [38, 27, 43, 3, 9, 82, 10]
              /                       \
        [38, 27, 43, 3]          [9, 82, 10]
         /          \             /        \
     [38, 27]     [43, 3]      [9, 82]    [10]
      /    \      /    \       /    \
   [38]  [27]  [43]   [3]   [9]  [82]

Merge phase (bottom-up), each merge walks two sorted lists with two pointers:
   [38]+[27]  -> [27, 38]
   [43]+[3]   -> [3, 43]
   [9]+[82]   -> [9, 82]
   [27,38]+[3,43]  -> [3, 27, 38, 43]
   [9,82]+[10]     -> [9, 10, 82]
   [3,27,38,43]+[9,10,82] -> [3, 9, 10, 27, 38, 43, 82]
```

```go
func MergeSort(a []int) {
    if len(a) < 2 {
        return
    }
    mid := len(a) / 2
    left := append([]int(nil), a[:mid]...)
    right := append([]int(nil), a[mid:]...)
    MergeSort(left)
    MergeSort(right)
    merge(a, left, right)
}

func merge(dst, left, right []int) {
    i, j, k := 0, 0, 0
    for i < len(left) && j < len(right) {
        // <= (not <) is what makes this stable: on ties, take from left first.
        if left[i] <= right[j] {
            dst[k] = left[i]
            i++
        } else {
            dst[k] = right[j]
            j++
        }
        k++
    }
    for i < len(left) {
        dst[k], k, i = left[i], k+1, i+1
    }
    for j < len(right) {
        dst[k], k, j = right[j], k+1, j+1
    }
}
```

**Complexity.** T(n) = 2T(n/2) + O(n). By the master theorem, O(n log n) in all cases (best, average, worst). Space: O(n) auxiliary for the two half-copies at each level. Stable. Not in-place.

**Why the recurrence gives n log n.** The recursion tree has log₂ n levels (halving each time). At every level, the merge work across all sub-arrays sums to O(n) (every element is touched once per merge). Total = O(n) per level × log n levels = O(n log n).

**Where it wins.** Guaranteed O(n log n) worst case with no dependence on input distribution: predictable latency. Stable, which quicksort is not. Naturally external and parallelisable. It is the standard for linked lists (in-place with pointer manipulation; no random access needed) and for external sorts (sort chunks that fit in RAM, then merge them from disk streams). Also the base for Timsort (Python, Java) and for Go's `sort.Stable`.

## Quick Sort

Divide-and-conquer, but the interesting work is in the **partition** step, not the combine step. Choose a pivot element, partition the array so everything less than the pivot is left of it and everything greater is right of it, then recurse into each side.

The name refers to its typical speed on real data: with a good pivot, the partition is O(n) and the recursion depth is log n, giving n log n with the smallest constant factor of any comparison sort.

```
Input: [10, 80, 30, 90, 40, 50, 70], pivot = 70 (last element, Lomuto scheme)

Walk with two indices:
  i = fence (last known ≤ pivot), starts at lo - 1
  j = scanner, walks lo..hi-1

j=0, a[j]=10, 10 ≤ 70, i=0, swap a[0] with itself: [10, 80, 30, 90, 40, 50, 70]
j=1, a[j]=80, 80 > 70, skip
j=2, a[j]=30, 30 ≤ 70, i=1, swap a[1] with a[2]:   [10, 30, 80, 90, 40, 50, 70]
j=3, a[j]=90, 90 > 70, skip
j=4, a[j]=40, 40 ≤ 70, i=2, swap a[2] with a[4]:   [10, 30, 40, 90, 80, 50, 70]
j=5, a[j]=50, 50 ≤ 70, i=3, swap a[3] with a[5]:   [10, 30, 40, 50, 80, 90, 70]

Place pivot: swap a[i+1] with a[hi]:               [10, 30, 40, 50, 70, 90, 80]
Pivot 70 is now at its final index 4.

Recurse on [10, 30, 40, 50] and [90, 80].
```

```go
func QuickSort(a []int) {
    quickSortRec(a, 0, len(a)-1)
}

func quickSortRec(a []int, lo, hi int) {
    if lo >= hi {
        return
    }
    p := partition(a, lo, hi)
    quickSortRec(a, lo, p-1)
    quickSortRec(a, p+1, hi)
}

// Lomuto partition with median-of-three pivot selection.
func partition(a []int, lo, hi int) int {
    mid := lo + (hi-lo)/2
    // Sort a[lo], a[mid], a[hi] so median lands at a[hi] for use as pivot.
    if a[mid] < a[lo] {
        a[lo], a[mid] = a[mid], a[lo]
    }
    if a[hi] < a[lo] {
        a[lo], a[hi] = a[hi], a[lo]
    }
    if a[mid] < a[hi] {
        a[mid], a[hi] = a[hi], a[mid]
    }
    pivot := a[hi]
    i := lo - 1
    for j := lo; j < hi; j++ {
        if a[j] <= pivot {
            i++
            a[i], a[j] = a[j], a[i]
        }
    }
    a[i+1], a[hi] = a[hi], a[i+1]
    return i + 1
}
```

**Randomized pivot, as an alternative to median-of-three.** Instead of sampling three fixed positions, pick a uniformly random index in `[lo, hi]` and swap it into `a[hi]` before running the same Lomuto partition. This is the version most commonly asked for by name in interviews, since it needs no comparison logic of its own to select the pivot, just one random draw per partition call.

```go
func QuickSortRandomPivot(a []int) {
    quickSortRandomRec(a, 0, len(a)-1)
}

func quickSortRandomRec(a []int, lo, hi int) {
    if lo >= hi {
        return
    }
    p := partitionRandom(a, lo, hi)
    quickSortRandomRec(a, lo, p-1)
    quickSortRandomRec(a, p+1, hi)
}

// Lomuto partition with a uniformly random pivot.
func partitionRandom(a []int, lo, hi int) int {
    r := lo + rand.Intn(hi-lo+1)
    a[r], a[hi] = a[hi], a[r] // move the random pick to hi, then partition as usual
    pivot := a[hi]
    i := lo - 1
    for j := lo; j < hi; j++ {
        if a[j] <= pivot {
            i++
            a[i], a[j] = a[j], a[i]
        }
    }
    a[i+1], a[hi] = a[hi], a[i+1]
    return i + 1
}
```

Why this defeats adversarial input in a way median-of-three cannot: median-of-three always samples the *same three positions* (`lo`, `mid`, `hi`), so an adversary who knows the algorithm can still construct input that makes those three positions consistently bad. A random draw has no fixed positions to target — the adversary would need to predict the random number generator itself, which is why the guarantee is stated as an *expectation* (expected O(n log n) over the randomness) rather than a guarantee for every possible input, unlike median-of-three's deterministic-but-defeatable guard.

**Complexity.** Average: O(n log n) with a smaller constant than merge sort because it is in-place and cache-friendly (contiguous partition passes). Worst: O(n²) when the pivot is consistently the min or max, degenerating recursion into a linear chain. Space: O(log n) average call stack, O(n) worst case. Not stable (the partition swap jumps equal-key elements over each other). In-place.

**The pivot problem and how it is defused.**
- **Naive: always pick `a[hi]`.** Already-sorted or reverse-sorted input hits the O(n²) worst case immediately. Interviewers know this and will hand you sorted input to test it.
- **Median-of-three.** Sample `a[lo]`, `a[mid]`, `a[hi]` and use the median. Kills the sorted-input worst case; still adversarially defeatable.
- **Random pivot.** Pick a uniformly random index in `[lo, hi]`. Expected O(n log n) regardless of input distribution. Adversarial input needs to know the RNG seed to defeat it. Template above, right after median-of-three.
- **Introspection (introsort).** Track recursion depth; if it exceeds `2·log₂(n)`, bail out and finish with heap sort. Guarantees O(n log n) worst case while keeping quicksort's fast average path. This is what C++'s `std::sort` does.
- **Pattern-defeating quicksort (pdqsort).** Detects already-sorted, reverse-sorted, and equal-value patterns and switches strategies. This is what Go's `slices.Sort` uses.

**Three-way partitioning (Dutch National Flag) for arrays with many duplicates.** Split into three regions: `< pivot`, `== pivot`, `> pivot`. Elements equal to the pivot are placed in their final position in a single pass and are excluded from recursion. Turns O(n²) on all-equal input into O(n).

**Where it wins.** In-place, cache-friendly, smallest constant factor of any comparison sort on typical inputs. The default choice for sorting arrays of primitives in almost every industrial library.

## Heap Sort

Build a max-heap over the array in-place, then repeatedly extract the maximum by swapping the root to the end and sift-down-ing the new root over the shrinking heap. After n-1 extractions, the array is sorted ascending.

The name is literal: the sort is a byproduct of correctly using a heap as a priority queue.

```
Input: [3, 5, 1, 10, 2, 7]

Build max-heap by sifting down from last internal node (index n/2 - 1) upward:
  Start:     [3, 5, 1, 10, 2, 7]
  Sift(2):   swap 1 with max(children)=7 -> [3, 5, 7, 10, 2, 1]
  Sift(1):   swap 5 with max(children)=10 -> [3, 10, 7, 5, 2, 1]
  Sift(0):   swap 3 with 10, then 3 with 5 -> [10, 5, 7, 3, 2, 1]

Extract phase:
  Swap root(10) with last(1), heapify size 5: [7, 5, 1, 3, 2 | 10]
  Swap root(7)  with last(2), heapify size 4: [5, 3, 1, 2 | 7, 10]
  Swap root(5)  with last(2), heapify size 3: [3, 2, 1 | 5, 7, 10]
  Swap root(3)  with last(1), heapify size 2: [2, 1 | 3, 5, 7, 10]
  Swap root(2)  with last(1), heapify size 1: [1 | 2, 3, 5, 7, 10]
```

```go
func HeapSort(a []int) {
    n := len(a)
    // Build max-heap: last internal node is at n/2 - 1.
    for i := n/2 - 1; i >= 0; i-- {
        siftDown(a, i, n)
    }
    // Repeatedly move max to end, then restore heap property on the shrinking prefix.
    for end := n - 1; end > 0; end-- {
        a[0], a[end] = a[end], a[0]
        siftDown(a, 0, end)
    }
}

func siftDown(a []int, root, size int) {
    for {
        left := 2*root + 1
        if left >= size {
            return
        }
        child := left
        if right := left + 1; right < size && a[right] > a[left] {
            child = right
        }
        if a[root] >= a[child] {
            return
        }
        a[root], a[child] = a[child], a[root]
        root = child
    }
}
```

**Complexity.** Build-heap is O(n) (not O(n log n): the sift-down cost per level shrinks geometrically, and the sum converges). Each extraction is O(log n), and there are n of them, giving O(n log n). Total: O(n log n) in all cases. Space: O(1). Not stable (sift-down swaps equal-key elements over each other). In-place.

**Why the build-heap is O(n).** Half the nodes are leaves (zero work). A quarter are at height 1 (one swap each). An eighth at height 2 (two swaps each). The sum n/2·0 + n/4·1 + n/8·2 + ... converges to O(n). Contrast with inserting n items one at a time into an empty heap, which is O(n log n).

**Where it wins.** Guaranteed O(n log n) worst case with O(1) space. This is why introsort uses it as the fallback when quicksort's recursion goes too deep: it is the only comparison sort that is both worst-case optimal and in-place. Rarely the *primary* choice because its constant factor is worse than quicksort's (poor cache locality: parent-child index jumps are large) and it is not stable.

## Counting Sort

Non-comparison sort for integer keys in a bounded range `[0, k]`. Count the occurrences of each key, then use the prefix sum of counts to place each element directly at its final position.

The name is exact: you literally **count** how many times each key appears, and that count table is enough to reconstruct the sorted output.

```
Input: [4, 2, 2, 8, 3, 3, 1]

Step 1: count occurrences.
  index:  0  1  2  3  4  5  6  7  8
  count:  0  1  2  2  1  0  0  0  1

Step 2: convert to prefix sum. Now count[v] = number of elements ≤ v.
  index:  0  1  2  3  4  5  6  7  8
  prefix: 0  1  3  5  6  6  6  6  7

Step 3: walk input in reverse, place each element at count[v]-1, decrement.
  This right-to-left walk is what preserves stability.

  Result: [1, 2, 2, 3, 3, 4, 8]
```

```go
// Sorts non-negative ints in [0, max]. Stable. Returns a new slice.
func CountingSort(a []int) []int {
    if len(a) == 0 {
        return a
    }
    maxV := a[0]
    for _, v := range a {
        if v > maxV {
            maxV = v
        }
    }
    count := make([]int, maxV+1)
    for _, v := range a {
        count[v]++
    }
    // Prefix sum: count[v] becomes the ending index (exclusive) for key v.
    for i := 1; i <= maxV; i++ {
        count[i] += count[i-1]
    }
    out := make([]int, len(a))
    // Reverse walk keeps stability: equal keys keep original relative order.
    for i := len(a) - 1; i >= 0; i-- {
        count[a[i]]--
        out[count[a[i]]] = a[i]
    }
    return out
}
```

**Complexity.** Time: O(n + k) where k is the key range. Space: O(n + k). Stable. Not in-place.

**Where it wins and where it breaks.** Wins when k = O(n) (e.g. sorting exam scores from 0 to 100, sorting bytes, bucketing by age): faster than any O(n log n) sort because it dodges the comparison lower bound entirely. Breaks when k dominates n: sorting 100 integers each in `[0, 10^9]` would allocate a billion-entry count array to move 100 items. The rule of thumb is: use counting sort only when the key range is comparable to or smaller than n.

## Radix Sort

Non-comparison sort for keys that decompose into digits (or bytes, or fixed-width chunks). Sort by one digit at a time using a stable sub-sort (usually counting sort). LSD radix (least significant digit first) processes the rightmost digit first; MSD processes the leftmost.

The name comes from the mathematical term **radix** meaning "base of a number system" (base 10, base 2, base 256). Radix sort is sorting one radix-position at a time.

**Why stability of the sub-sort is load-bearing.** Sorting by the ones digit first, then the tens digit, then the hundreds must not disturb the ones-digit order among elements that share a tens digit. That preservation *is* the stability guarantee of the sub-sort. If you replaced counting sort here with an unstable sub-sort, LSD radix sort would return garbage.

```
Input: [170, 45, 75, 90, 802, 24, 2, 66]

Pass 1 (ones digit):
  170 (0), 45 (5), 75 (5), 90 (0), 802 (2), 24 (4), 2 (2), 66 (6)
  Stable sort by ones digit:
  [170, 90, 802, 2, 24, 45, 75, 66]

Pass 2 (tens digit):
  170 (7), 90 (9), 802 (0), 2 (0), 24 (2), 45 (4), 75 (7), 66 (6)
  Stable sort by tens digit:
  [802, 2, 24, 45, 66, 170, 75, 90]

Pass 3 (hundreds digit):
  802 (8), 2 (0), 24 (0), 45 (0), 66 (0), 170 (1), 75 (0), 90 (0)
  Stable sort by hundreds digit:
  [2, 24, 45, 66, 75, 90, 170, 802]
```

```go
// LSD radix sort, base 10, non-negative ints.
func RadixSort(a []int) {
    if len(a) == 0 {
        return
    }
    maxV := a[0]
    for _, v := range a {
        if v > maxV {
            maxV = v
        }
    }
    for exp := 1; maxV/exp > 0; exp *= 10 {
        countingSortByDigit(a, exp)
    }
}

func countingSortByDigit(a []int, exp int) {
    n := len(a)
    out := make([]int, n)
    count := make([]int, 10)
    for _, v := range a {
        count[(v/exp)%10]++
    }
    for i := 1; i < 10; i++ {
        count[i] += count[i-1]
    }
    for i := n - 1; i >= 0; i-- {
        d := (a[i] / exp) % 10
        count[d]--
        out[count[d]] = a[i]
    }
    copy(a, out)
}
```

**Complexity.** O(d · (n + b)) where d is the number of digits in the largest key and b is the base (usually 10 or 256). For fixed-width keys (32-bit ints, 64-bit ints, fixed-length strings), d is constant, so the effective complexity is O(n). Space: O(n + b). Stable when the sub-sort is stable.

**Where it wins.** Sorting large volumes of fixed-width keys: 32-bit or 64-bit integers, IP addresses, ASCII strings of bounded length. Databases use radix-like schemes for high-throughput integer indexing. For general-purpose interview problems on `[]int` in an unbounded range, quicksort or merge sort is usually the better default.

## Complexity Reference

| Algorithm | Best | Average | Worst | Space | Stable | In-Place | Adaptive |
|-----------|------|---------|-------|-------|--------|----------|----------|
| Bubble | O(n) | O(n²) | O(n²) | O(1) | Yes | Yes | Yes (with early exit) |
| Selection | O(n²) | O(n²) | O(n²) | O(1) | No | Yes | No |
| Insertion | O(n) | O(n²) | O(n²) | O(1) | Yes | Yes | Yes |
| Merge | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes | No | No |
| Quick | O(n log n) | O(n log n) | O(n²) | O(log n) | No | Yes | Partially (pdqsort) |
| Heap | O(n log n) | O(n log n) | O(n log n) | O(1) | No | Yes | No |
| Counting | O(n + k) | O(n + k) | O(n + k) | O(n + k) | Yes | No | No |
| Radix | O(d(n + b)) | O(d(n + b)) | O(d(n + b)) | O(n + b) | Yes | No | No |

Notation: n = element count, k = key range, d = digit count, b = radix base.

## Sorting In Go

Go's standard library ships two sorting APIs that cover 99% of real code. Know them.

**Generic API (`slices` package, Go 1.21+).** Prefer this in new code. `slices.Sort` sorts any slice whose element type is ordered (implements `cmp.Ordered`). `slices.SortFunc` takes a comparator returning `-1 / 0 / +1`. `slices.SortStableFunc` guarantees stability.

```go
import "slices"

// Ordered types: any int/float/string.
nums := []int{5, 3, 8, 1, 9, 2}
slices.Sort(nums)                       // ascending
slices.SortFunc(nums, func(a, b int) int { return b - a }) // descending

type Person struct{ Name string; Age int }
people := []Person{{"Bob", 30}, {"Alice", 25}, {"Bob", 25}}
slices.SortFunc(people, func(a, b Person) int {
    return a.Age - b.Age
})
// If you need Bob-30 to stay after Bob-25 when re-sorting later by name,
// use SortStableFunc.
slices.SortStableFunc(people, func(a, b Person) int {
    return strings.Compare(a.Name, b.Name)
})
```

**Legacy API (`sort` package).** Still fine, still supported, but generic-free.

```go
import "sort"

sort.Ints(nums)                  // sort []int
sort.Strings(strs)               // sort []string
sort.Float64s(floats)            // sort []float64

sort.Slice(people, func(i, j int) bool {
    return people[i].Age < people[j].Age
})
sort.SliceStable(people, func(i, j int) bool { ... }) // stable variant

// Binary search on a sorted slice.
idx, found := slices.BinarySearch(nums, 5)
```

**What algorithm does Go actually use?** `slices.Sort` and `sort.Sort` use **pdqsort** (pattern-defeating quicksort), which combines quicksort's average case with heap-sort's worst-case guarantee and detects several common patterns (already sorted, reverse-sorted, all equal) to short-circuit. `slices.SortStableFunc` and `sort.Stable` use a stable merge sort variant. The `<` in the comparator is called O(n log n) times, so keep it cheap.

**Comparator return convention.** The `slices.SortFunc` comparator returns an `int`: negative if `a < b`, zero if equal, positive if `a > b`. This is the same convention as `strings.Compare` and C's `qsort`. Do **not** compute `a - b` for floats or for ints that can overflow; use explicit branching instead:

```go
slices.SortFunc(nums, func(a, b int) int {
    if a < b { return -1 }
    if a > b { return 1 }
    return 0
})
```

## Recognizing Which Sort To Use

**Default choice for arbitrary comparable data.** Quicksort (specifically `slices.Sort`). It is the fastest in-place comparison sort on average and its pdqsort variant defuses the worst-case attacks.

**When stability is required.** Merge sort (`slices.SortStableFunc`). Whenever you are sorting on a secondary key, whenever equal-key elements carry hidden distinguishing information (e.g. sorting log lines by timestamp where equal timestamps must preserve arrival order).

**When worst-case latency matters more than average speed.** Merge sort or heap sort. Real-time systems, latency-SLA services, adversarial input.

**When the array is tiny (n ≤ 32ish) or already nearly sorted.** Insertion sort. This is what industrial sorts fall through to internally; you rarely call it explicitly.

**When keys are small integers.** Counting sort if k = O(n). Radix sort for fixed-width keys.

**When memory is tight.** Heap sort or in-place quicksort. Skip merge sort.

**When sorting a linked list.** Merge sort. Random access is expensive on lists; merging is not.

**When the data does not fit in RAM.** External merge sort: sort chunks that fit, spill sorted chunks to disk, merge the sorted streams.

## LeetCode Problems

### 1. Merge Sorted Array - [#88 (Easy)](https://leetcode.com/problems/merge-sorted-array/)

`nums1` has length `m + n`, with only the first `m` slots filled and the rest zeroed as placeholder space. Merge `nums2` (length `n`) into `nums1` in place so the result is one sorted array.

<details>
<summary>Brute Force</summary>

Copy the first `m` elements of `nums1` and all of `nums2` into a combined slice, then sort the combined slice with a general-purpose sort and copy it back into `nums1`.

Time: O((m+n) log(m+n)). Space: O(m+n) for the combined slice.

The waste: both halves are already individually sorted. A full general-purpose sort throws that structure away and re-derives the order from scratch, exactly the situation the merge step of merge sort is built to avoid.
</details>

<details>
<summary>Hint 1</summary>

Both arrays are already sorted. If you merge from the front, placing the smaller of the two current elements into `nums1[0]`, you will overwrite values in `nums1` that you have not read yet, since the writable region and the unread region of `nums1` overlap. Which direction avoids this?
</details>

<details>
<summary>Hint 2</summary>

Merge from the back instead. `nums1` has exactly `m + n` slots and its last `n` slots are unused padding, so writing to the end of `nums1` first can never overwrite an unread element of `nums1` (only already-consumed padding). Use three pointers: one at the end of the real data in `nums1`, one at the end of `nums2`, one at the end of the full array marking the next write position.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(m+n) | Space: O(1)
func merge(nums1 []int, m int, nums2 []int, n int) {
    i, j, k := m-1, n-1, m+n-1
    for j >= 0 {
        if i >= 0 && nums1[i] > nums2[j] {
            nums1[k] = nums1[i]
            i--
        } else {
            nums1[k] = nums2[j]
            j--
        }
        k--
    }
    // If nums1 is exhausted first, remaining nums2 elements are already
    // in their correct trailing positions once j hits -1; no copy needed.
}
```

This is the merge step of merge sort, done in place by exploiting the padding at the end of `nums1` and walking backward instead of forward.
</details>

### 2. Relative Sort Array - [#1122 (Easy)](https://leetcode.com/problems/relative-sort-array/)

Given `arr1` and `arr2`, where every element of `arr2` is distinct and also appears in `arr1`, sort `arr1` so its elements follow `arr2`'s relative order. Elements not in `arr2` go at the end, ascending.

<details>
<summary>Brute Force</summary>

For each value in `arr2` (in order), linearly scan `arr1` and pull out every matching occurrence into the result. Once `arr2` is exhausted, sort whatever is left in `arr1` and append it.

Time: O(n · m) where n = `len(arr1)`, m = `len(arr2)` (a full scan of `arr1` per value of `arr2`), plus O(k log k) for the leftover sort. Space: O(n).

The waste: this is exactly the situation counting sort is built for. The value range is small and bounded (`0 <= arr1[i] <= 1000` per the constraints), so instead of scanning `arr1` from scratch for every value in `arr2`, a single counting pass over `arr1` can answer "how many of each value exist" in one shot, and the rest becomes O(1) lookups.
</details>

<details>
<summary>Hint 1</summary>

The value range is small and bounded (`0 <= arr1[i] <= 1000` per the constraints), so a general-purpose comparison sort is overkill. What non-comparison sort takes advantage of a small, known key range to sort in linear time?
</details>

<details>
<summary>Hint 2</summary>

Run a single counting pass over `arr1` to record how many times each value appears. Then walk `arr2` in order and, for each value, emit it as many times as its count says. Whatever counts remain unconsumed afterward belong to values not in `arr2` - emit those last, in ascending order.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n + k) | Space: O(n + k), where k is the max value in arr1
func relativeSortArray(arr1 []int, arr2 []int) []int {
    maxV := 0
    for _, v := range arr1 {
        if v > maxV {
            maxV = v
        }
    }
    count := make([]int, maxV+1)
    for _, v := range arr1 {
        count[v]++
    }
    out := make([]int, 0, len(arr1))
    // Emit values in arr2's order first, using up their counts.
    for _, v := range arr2 {
        for count[v] > 0 {
            out = append(out, v)
            count[v]--
        }
    }
    // Whatever counts remain belongs to values not in arr2 - walk ascending.
    for v := 0; v <= maxV; v++ {
        for count[v] > 0 {
            out = append(out, v)
            count[v]--
        }
    }
    return out
}
```

This is Counting Sort applied directly: the count array from the Counting Sort section above, read out in two different orders (first `arr2`'s order, then ascending) instead of one.
</details>

### 3. H-Index - [#274 (Medium)](https://leetcode.com/problems/h-index/)

Given a researcher's citation counts, find the h-index: the largest `h` such that at least `h` papers have `h` or more citations each.

<details>
<summary>Brute Force</summary>

For every candidate `h` from `n` down to `0`, scan the whole citations array and count how many entries are `>= h`. The first `h` where that count is itself `>= h` is the answer.

Time: O(n²) (n candidates, each requiring a full O(n) scan). Space: O(1).

The waste: the candidate scan and the citation scan are both independent of order, but if the array is sorted first, checking every candidate collapses into a single pass, because sortedness lets you read off, for each position, exactly how many papers have at least that many citations without rescanning.
</details>

<details>
<summary>Hint 1</summary>

Sort the citations. Once sorted, what does the value at each index tell you about how many papers to its right have at least that many citations?
</details>

<details>
<summary>Hint 2</summary>

After sorting ascending, for index `i`, there are exactly `n - i` papers from `i` to the end (including `i` itself). If `citations[i] >= n - i`, then all `n - i` of those papers have at least `n - i` citations - a valid h-index candidate. Walk from the smallest index where this first holds; that gives the largest such `h`.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n log n) | Space: O(1) beyond the sort
func hIndex(citations []int) int {
    slices.Sort(citations)
    n := len(citations)
    for i, c := range citations {
        if c >= n-i {
            return n - i
        }
    }
    return 0
}
```

Sorting turns "check every candidate `h`" into a single linear scan, because sortedness guarantees the count of "papers with at least this many citations" only moves in one direction as `i` advances.
</details>

### 4. Kth Largest Element In An Array - [#215 (Medium)](https://leetcode.com/problems/kth-largest-element-in-an-array/)

Return the kth largest element. LeetCode's own follow-up asks: can you solve it without sorting?

<details>
<summary>Brute Force</summary>

Sort the array (ascending or descending) and read off the element at the appropriate index.

Time: O(n log n). Space: O(1) to O(n) depending on the sort used.

The waste: fully sorting determines the relative order of every element, but the problem only asks for one specific rank. Everything the sort learns about elements other than the kth largest is thrown away.
</details>

<details>
<summary>Hint 1</summary>

Quicksort's partition step places one element in its final sorted position and tells you exactly which index that is. If that index equals the target rank, you have found the answer without sorting the rest of the array.
</details>

<details>
<summary>Hint 2</summary>

Recurse into only one side of the partition (the side containing the target index), never both. That is what turns O(n log n) into O(n) average.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) average, O(n²) worst | Space: O(log n) average recursion
func findKthLargest(nums []int, k int) int {
    target := len(nums) - k // index of the kth largest in ascending sorted order
    lo, hi := 0, len(nums)-1
    for lo <= hi {
        p := partition(nums, lo, hi)
        if p == target {
            return nums[p]
        } else if p < target {
            lo = p + 1
        } else {
            hi = p - 1
        }
    }
    return -1
}

// Median-of-three partition (same as QuickSort above).
func partition(a []int, lo, hi int) int {
    mid := lo + (hi-lo)/2
    if a[mid] < a[lo] { a[lo], a[mid] = a[mid], a[lo] }
    if a[hi] < a[lo]  { a[lo], a[hi]  = a[hi], a[lo]  }
    if a[mid] < a[hi] { a[mid], a[hi] = a[hi], a[mid] }
    pivot := a[hi]
    i := lo - 1
    for j := lo; j < hi; j++ {
        if a[j] <= pivot {
            i++
            a[i], a[j] = a[j], a[i]
        }
    }
    a[i+1], a[hi] = a[hi], a[i+1]
    return i + 1
}
```

Quickselect: recurse into only the side that contains the target index. O(n) average. Use a randomized or median-of-three pivot for adversarial input.
</details>

<details>
<summary>Follow-Up: Solve It Without Sorting</summary>

The Quickselect solution above already answers this: it never fully sorts the array, it only partitions repeatedly until the target index is pinned down, discarding the side that cannot contain the answer.

A second, genuinely different way to avoid sorting is a bounded min-heap of size `k`: push every element, and whenever the heap exceeds size `k`, pop the smallest. After one pass, the heap's root is the kth largest.

```go
// Time: O(n log k) | Space: O(k)
type minHeap []int

func (h minHeap) Len() int            { return len(h) }
func (h minHeap) Less(i, j int) bool  { return h[i] < h[j] }
func (h minHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *minHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *minHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}

func findKthLargestHeap(nums []int, k int) int {
    h := &minHeap{}
    heap.Init(h)
    for _, n := range nums {
        heap.Push(h, n)
        if h.Len() > k {
            heap.Pop(h)
        }
    }
    return (*h)[0]
}
```

Trade-off between the two non-sorting approaches: Quickselect is O(n) average but O(n²) worst case and needs no auxiliary structure. The heap is O(n log k) in every case (no bad-input degradation) at the cost of O(k) extra space and a real, if smaller, log factor. Prefer the heap when `k` is small and worst-case latency matters; prefer Quickselect when average throughput matters more and `k` can be large.
</details>

### 5. Largest Number - [#179 (Medium)](https://leetcode.com/problems/largest-number/)

Given non-negative integers, arrange them to form the largest possible number. The insight is a custom comparator.

<details>
<summary>Brute Force</summary>

Generate every permutation of the input, concatenate each permutation's digits into a number, and keep the largest.

Time: O(n! · n) (n! permutations, O(n) to concatenate and compare each). Space: O(n) per permutation.

The waste: most pairwise orderings never need to be compared against each other at all. A single well-defined comparator between any two elements is enough to induce a correct global order, which is exactly what a comparison sort provides.
</details>

<details>
<summary>Hint 1</summary>

The natural numeric comparison is wrong: for `[3, 30]`, "330" beats "303", so 3 should sort before 30 even though 3 < 30 numerically.
</details>

<details>
<summary>Hint 2</summary>

Compare two candidates `a` and `b` by which concatenation produces the larger string: `a+b` vs `b+a`. If `a+b > b+a`, put `a` first.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n log n) | Space: O(n)
func largestNumber(nums []int) string {
    strs := make([]string, len(nums))
    for i, n := range nums {
        strs[i] = strconv.Itoa(n)
    }
    slices.SortFunc(strs, func(a, b string) int {
        return strings.Compare(b+a, a+b) // put a first if a+b > b+a
    })
    if strs[0] == "0" {
        return "0" // all zeros edge case
    }
    return strings.Join(strs, "")
}
```

The comparator defines a total order (transitivity follows from properties of string concatenation). O(n log n) sort with O(m) string compare per step, where m is the max digit count.
</details>

### 6. Sort An Array - [#912 (Medium)](https://leetcode.com/problems/sort-an-array/)

Implement your own sort. Solve it once with each of merge sort and quicksort to burn the templates in.

<details>
<summary>Brute Force</summary>

Bubble sort or selection sort: repeatedly find the next-smallest remaining element and place it, or repeatedly bubble the largest remaining element to the end.

Time: O(n²). Space: O(1).

`nums.length` up to `5·10^4` makes O(n²) too slow (2.5 billion operations), which is precisely the constraint that forces an O(n log n) approach and is the point of the exercise.
</details>

<details>
<summary>Hint 1</summary>

For merge sort, write the recursive split first, then the merge helper that consumes two sorted halves with two pointers.
</details>

<details>
<summary>Hint 2</summary>

For quicksort, use a randomized pivot or median-of-three. The constraint `-5·10^4 ≤ nums[i] ≤ 5·10^4` combined with adversarial sorted input means a naive `a[hi]` pivot will time out.
</details>

<details>
<summary>Solution (Go)</summary>

Both templates from the sections above (MergeSort and QuickSort with median-of-three pivot) pass. If you want the interview-safe answer, submit merge sort: predictable O(n log n) with no pivot to worry about.
</details>

### 7. Count Of Smaller Numbers After Self - [#315 (Hard)](https://leetcode.com/problems/count-of-smaller-numbers-after-self/)

For each element, count how many elements to its right are smaller. Modified merge sort solves it in O(n log n).

<details>
<summary>Brute Force</summary>

For each index `i`, scan every `j > i` and count how many satisfy `nums[j] < nums[i]`.

Time: O(n²). Space: O(1) beyond the output.

The waste: each pairwise comparison is redone from scratch for every `i`, even though the relative order between most pairs never actually changes as `i` advances.
</details>

<details>
<summary>Hint 1</summary>

Brute force is O(n²). To break that, notice that "count of smaller elements to the right" is exactly the number of inversions each index participates in as the left partner.
</details>

<details>
<summary>Hint 2</summary>

During the merge step of merge sort, when you take an element from the right half before exhausting the left half, every remaining element in the left half is a "greater on left, smaller on right" pair. Attribute the count to those left-half indices before merging them out.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n log n) | Space: O(n)
func countSmaller(nums []int) []int {
    n := len(nums)
    counts := make([]int, n)
    indices := make([]int, n)
    for i := range indices {
        indices[i] = i
    }
    var mergeSortCount func(idx []int) []int
    mergeSortCount = func(idx []int) []int {
        if len(idx) <= 1 {
            return idx
        }
        mid := len(idx) / 2
        left := mergeSortCount(idx[:mid])
        right := mergeSortCount(idx[mid:])
        merged := make([]int, 0, len(idx))
        i, j, rightCount := 0, 0, 0
        for i < len(left) && j < len(right) {
            if nums[left[i]] <= nums[right[j]] {
                counts[left[i]] += rightCount
                merged = append(merged, left[i])
                i++
            } else {
                rightCount++
                merged = append(merged, right[j])
                j++
            }
        }
        for ; i < len(left); i++ {
            counts[left[i]] += rightCount
            merged = append(merged, left[i])
        }
        merged = append(merged, right[j:]...)
        return merged
    }
    mergeSortCount(indices)
    return counts
}
```

Sort indices rather than values so the count array can be indexed by original position.
</details>

### 8. Wiggle Sort II - [#324 (Medium)](https://leetcode.com/problems/wiggle-sort-ii/)

Rearrange so that `nums[0] < nums[1] > nums[2] < nums[3]...`. LeetCode's own follow-up asks: can you do it in O(n) time and O(1) extra space?

<details>
<summary>Brute Force</summary>

Generate every permutation of the array and check each one for the wiggle property, keeping the first valid one found.

Time: O(n! · n). Space: O(n) per permutation.

The waste: the wiggle property only depends on relative order between adjacent elements, which a single sort plus a placement rule can satisfy directly, without ever searching the permutation space.
</details>

<details>
<summary>Hint 1</summary>

Sort the array. Split into a smaller half and a larger half. Interleave them so that odd indices come from the larger half and even indices from the smaller half.
</details>

<details>
<summary>Hint 2</summary>

Walk both halves in reverse when placing them. Doing so pushes equal-value elements apart (across the wiggle boundary) rather than adjacent, which matters when the input has duplicates like `[1,1,2,2,3,3]`.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n log n) | Space: O(n)
func wiggleSort(nums []int) {
    sorted := append([]int(nil), nums...)
    slices.Sort(sorted)
    n := len(nums)
    // Smaller half fills even indices, larger half fills odd indices,
    // both walked in reverse to separate duplicates. j starts at the
    // largest index of the small half ((n-1)/2), which is also the
    // median's rank in the sorted array - the same quantity the
    // follow-up solution passes to Quickselect.
    j, k := (n-1)/2, n-1
    for i := 0; i < n; i++ {
        if i%2 == 0 {
            nums[i] = sorted[j]
            j--
        } else {
            nums[i] = sorted[k]
            k--
        }
    }
}
```
</details>

<details>
<summary>Follow-Up (Partial): O(n) Average Time With O(n) Space</summary>

The base solution above is O(n log n) time, O(n) space. Dropping just the time factor (keeping O(n) space) is a clean, interview-realistic step.

The sort was only doing one thing we actually use: identifying the **median**. Once the median is known, the interleave only needs values grouped as `< median | == median | > median` — not fully sorted within each group. That weaker arrangement is cheaper to produce.

- Find the median in O(n) average using **Quickselect** (same template as #215 Kth Largest).
- Three-way partition a scratch copy of `nums` around the median in one O(n) pass (Dutch flag, same template as the brute force of Sort Colors).
- Interleave from the scratch copy back into `nums` using the same `j, k` read pattern as the base solution.

```go
// Time: O(n) average | Space: O(n)
func wiggleSortLinearTime(nums []int) {
    n := len(nums)
    tmp := append([]int(nil), nums...)
    median := quickSelect(tmp, (n-1)/2)

    // Three-way partition tmp around the median in one O(n) pass.
    // After this: tmp = [all < median][all == median][all > median].
    lo, mid, hi := 0, 0, n-1
    for mid <= hi {
        switch {
        case tmp[mid] < median:
            tmp[lo], tmp[mid] = tmp[mid], tmp[lo]
            lo++
            mid++
        case tmp[mid] > median:
            tmp[mid], tmp[hi] = tmp[hi], tmp[mid]
            hi--
        default:
            mid++
        }
    }

    // Same interleave as the base solution, reading from tmp instead of sorted.
    j, k := (n-1)/2, n-1
    for i := 0; i < n; i++ {
        if i%2 == 0 {
            nums[i] = tmp[j]
            j--
        } else {
            nums[i] = tmp[k]
            k--
        }
    }
}
```

Why this works: for the interleave to produce a valid wiggle, each valley slot only needs a value `≤ median` and each peak slot only needs a value `≥ median`. The three-way partition guarantees exactly that grouping without doing any within-group sorting, which is where the O(log n) factor was coming from in the base solution.

**This is the version most interviewers actually expect for the follow-up.** It reuses two patterns the candidate has likely already demonstrated (Quickselect, Dutch flag partition), and the connection "median + three-way partition is enough, full sort is overkill" is a reachable insight in a 45-minute window.
</details>

<details>
<summary>Follow-Up (Full): O(n) Average Time With O(1) Extra Space</summary>

Dropping space further to O(1) is dramatically harder and is where this problem's reputation as hard comes from. The O(n)-space solution above still needs a scratch copy because a direct three-way partition on `nums[0], nums[1], nums[2], ...` gives elements grouped around the median at **physical** positions 0, 1, 2, ..., which is sorted-descending-ish, not wiggled. What is needed is to place them at the **odd** indices first (largest values), then the **even** indices (smallest values) - the same interleave idea as the sort-based solution, but achieved through index arithmetic instead of a second array. Define a virtual index mapping `A(i) = nums[(1 + 2·i) % (n | 1)]`: as `i` walks `0, 1, 2, ...`, `A(i)` walks the physical array in the order odd, odd, odd, ..., even, even, even, ... (wrapping via `n | 1`, which is `n` rounded up to odd, so the mapping is a valid permutation of indices for both even and odd `n`). Running the standard Dutch-flag three-way partition through this virtual index, using the median as the pivot, produces a valid wiggle arrangement directly in `nums`, with no second array.

```go
// Time: O(n) average | Space: O(1)
func wiggleSortOptimal(nums []int) {
    n := len(nums)
    // Quickselect on a working copy: quickSelect mutates its input,
    // and nums itself must stay untouched until the partition pass below.
    tmp := append([]int(nil), nums...)
    median := quickSelect(tmp, (n-1)/2)

    // Virtual index: walks odd positions first, then even, wrapping via n|1
    // so the mapping is a valid permutation for both even and odd n.
    idx := func(i int) int {
        return (1 + 2*i) % (n | 1)
    }

    // Three-way partition on virtual indices: > median first, == median
    // middle, < median last. Same shape as Sort Colors' Dutch flag partition,
    // just addressed through idx() instead of directly.
    i, j, k := 0, 0, n-1
    for j <= k {
        switch {
        case nums[idx(j)] > median:
            nums[idx(i)], nums[idx(j)] = nums[idx(j)], nums[idx(i)]
            i++
            j++
        case nums[idx(j)] < median:
            nums[idx(j)], nums[idx(k)] = nums[idx(k)], nums[idx(j)]
            k--
        default:
            j++
        }
    }
}

func quickSelect(a []int, k int) int {
    lo, hi := 0, len(a)-1
    for {
        if lo == hi {
            return a[lo]
        }
        p := partition(a, lo, hi)
        if p == k {
            return a[p]
        } else if p < k {
            lo = p + 1
        } else {
            hi = p - 1
        }
    }
}
```

`quickSelect` is O(n) average (same argument as Kth Largest above); the virtual-index partition is a single O(n) pass. Total: O(n) average time, O(1) extra space (the `tmp` copy for Quickselect can be avoided too with an in-place median-finding variant, at the cost of more code; most interview answers stop at this version and call out the trade-off).

**Interview reality.** This full version is not a reasonable thing to derive from scratch under time pressure. The virtual-index permutation `(1 + 2i) % (n | 1)` is a known clever construction that routes partition pointers through the odd-then-even order needed for the wiggle; inventing it in a 45-minute window without prior exposure is unlikely. The honest move if asked this cold: present the O(n)-time + O(n)-space version above, then say "the follow-up to O(1) space uses an index-remapping trick on the three-way partition that I know exists but would need to work out carefully." Most interviewers accept that and either accept the O(n)-space answer or walk you through the trick themselves.

</details>

## References

- [`slices` package documentation](https://pkg.go.dev/slices) - canonical Go sorting API, generic.
- [`sort` package documentation](https://pkg.go.dev/sort) - legacy Go sorting API.
- [Go source for `slices.Sort` (pdqsort)](https://cs.opensource.google/go/go/+/refs/tags/go1.22.0:src/slices/sort.go) - read the actual implementation once.
- [Pattern-Defeating Quicksort paper (Peters, 2021)](https://arxiv.org/abs/2106.05123) - the algorithm behind Go's default sort.
- Cormen, Leiserson, Rivest, Stein. *Introduction to Algorithms* (CLRS), Chapters 2, 6, 7, 8 - canonical treatment of every algorithm in this file.
- Sedgewick, Wayne. *Algorithms, 4th Ed.*, Chapter 2 - clearest visual explanations of quicksort partitioning and heap sort.
- [`fundamentals/complexity-analysis.md`](../fundamentals/complexity-analysis.md) - decision-tree lower bound argument in detail.
- [`patterns/two-pointers.md`](../patterns/two-pointers.md) - the partition step of Dutch National Flag is a two-pointer technique.
- [`data-structures/heaps.md`](../data-structures/heaps.md) - heap operations used by heap sort.