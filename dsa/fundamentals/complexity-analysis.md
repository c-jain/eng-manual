---
Status: 🌳 Evergreen
Created: 2026-06-12
Last Updated: 2026-06-12
---

# Complexity Analysis

## Table of Contents

1. [What It Is and Why It Exists](#what-it-is-and-why-it-exists)
2. [Why "Asymptotic"](#why-asymptotic)
3. [The Notation Family](#the-notation-family)
4. [Calculating Time Complexity](#calculating-time-complexity)
5. [Recurrence Relations and the Master Theorem](#recurrence-relations-and-the-master-theorem)
6. [Amortized Analysis](#amortized-analysis)
7. [Space Complexity](#space-complexity)
8. [Complexity Class Reference](#complexity-class-reference)
9. [How to Remember This](#how-to-remember-this)
10. [Interview Cheat Sheet](#interview-cheat-sheet)
11. [References](#references)

---

## What It Is and Why It Exists

Complexity analysis is a framework for measuring how an algorithm's resource requirements — time and space — **grow relative to input size**. The key word is *grow*. Not "how fast it runs on my laptop," but "how does its behaviour change as inputs scale."

### The Problem It Solves

Before this framework, comparing algorithms meant benchmarking: run both, see which is faster. This broke down because results depended on the machine, the input, compiler flags, system load — everything except the algorithm itself.

The solution: abstract the machine into "operations". Express operation count as a function of input size `n`. Then strip constants (they're just faster hardware) and focus only on the dominant term (how the algorithm scales).

### Why It Is Called "Complexity Analysis"

- **Complexity** — from Latin *complexus*, "entwined, interwoven." In CS it refers to *how resource-intensive a computation is*. Formally rooted in **computational complexity theory**, developed in the 1960s–70s (Hartmanis and Stearns formalised time/space complexity in 1965; Cook and Karp developed NP-completeness in the early 1970s).
- **Analysis** — from Greek *analyein*, "to unloose, to separate." Breaking something into components to understand it.

Together: *systematically studying how resource requirements are structured and how they grow*.

### What Problems Does It Introduce?

Complexity analysis is an abstraction. Every abstraction leaks:

| Limitation | What It Means in Practice |
|---|---|
| Constants are hidden | O(n log n) with constant 50,000 may be slower than O(n²) with constant 1 for practical n |
| Cache behaviour ignored | Poor cache locality can make an O(n) algorithm slower than O(n log n) |
| Worst case ≠ typical case | Quicksort is O(n²) worst case but O(n log n) average — worst-case analysis alone misleads |
| Amortized ≠ per-operation | Amortized O(1) can have occasional O(n) spikes — unacceptable in real-time systems |

---

## Why "Asymptotic"

"Asymptotic" refers to behaviour *as n approaches infinity* — borrowed from the mathematical concept of an **asymptote** (a line a curve approaches but never reaches).

We care about asymptotic behaviour because the dominant term completely overwhelms everything else at large n:

```
f(n) = 3n² + 2n + 1

n = 10:       300 + 20 + 1        = 321        (lower terms still visible)
n = 10,000:   300,000,000 + 20,000 + 1         (lower terms are noise)
n → ∞:        n² is all that matters
```

This is why we drop constants (machine-dependent) and lower-order terms (they vanish asymptotically). At scale, only the growth rate of the dominant term determines algorithm behaviour.

---

## The Notation Family

These notations come from **Bachmann–Landau notation**, developed in 19th-century mathematics and later adapted into CS to describe algorithm growth.

### Big-O (Upper Bound)

> f(n) = O(g(n)) means f grows **at most as fast as** g, within a constant factor.

Formal definition: ∃ c > 0 and n₀ such that for all n ≥ n₀: **f(n) ≤ c · g(n)**

This is a worst-case upper bound — the ceiling. "My algorithm does *at most* this many operations."

In practice, O is used colloquially to mean the tight bound (Θ). Know the distinction if an interviewer probes.

### Big-Omega (Lower Bound)

> f(n) = Ω(g(n)) means f grows **at least as fast as** g, within a constant factor.

Formal definition: ∃ c > 0 and n₀ such that for all n ≥ n₀: **f(n) ≥ c · g(n)**

Most useful for proving impossibility: any comparison sort is Ω(n log n), meaning you cannot sort by comparisons faster than n log n, regardless of how clever the algorithm is.

### Big-Theta (Tight Bound)

> f(n) = Θ(g(n)) means f grows **exactly as fast as** g, within constant factors.

Formal: f(n) = O(g(n)) **and** f(n) = Ω(g(n)) simultaneously.

The most precise characterisation. Merge sort is Θ(n log n) — no better and no worse, definitively.

### Little-o and Little-ω (Strict Bounds)

Strict versions where the bound is never tight (the ratio f/g → 0). Mostly used in theoretical CS; rarely needed in engineering interviews.

### Summary

```
Notation   Meaning                 Intuition
──────────────────────────────────────────────
O(g)       at most g               f ≤ g (ceiling)
Ω(g)       at least g              f ≥ g (floor)
Θ(g)       exactly g               f = g (tight)
o(g)       strictly less than g    f < g (strict)
ω(g)       strictly more than g    f > g (strict)
```

---

## Calculating Time Complexity

### The Four Core Rules

**Rule 1 — Drop Constants**
O(5n) = O(n). A constant factor is equivalent to a proportionally faster machine — meaningless for growth rate.

**Rule 2 — Drop Lower-Order Terms**
O(n² + n + 1) = O(n²). At large n, lower-order terms become negligible relative to the dominant term.

**Rule 3 — Sequential Steps Add**
Two separate blocks of O(n) each → O(n) + O(n) = O(n). Different input sizes → O(n) + O(m) = O(n + m).

**Rule 4 — Nested Steps Multiply**
A loop inside a loop → O(n) × O(n) = O(n²).

### Pattern Recognition

| Structure | Complexity | Reasoning |
|---|---|---|
| Fixed operations regardless of n | O(1) | No growth |
| Halving or doubling the input each step | O(log n) | log₂ halvings to reach 1 |
| Single pass through n elements | O(n) | Linear growth |
| Sort the input | O(n log n) | Best achievable for comparison sort |
| Nested loop over n × n | O(n²) | Quadratic growth |
| All subsets of n elements | O(2ⁿ) | 2 choices per element |
| All permutations of n elements | O(n!) | n × (n-1) × ... × 1 |

### Go Examples

```go
// O(1) — index access is constant regardless of slice size
func getFirst(s []int) int {
    return s[0]
}

// O(log n) — halves search space each iteration
func binarySearch(s []int, target int) int {
    lo, hi := 0, len(s)-1
    for lo <= hi {
        mid := lo + (hi-lo)/2
        switch {
        case s[mid] == target:
            return mid
        case s[mid] < target:
            lo = mid + 1
        default:
            hi = mid - 1
        }
    }
    return -1
}

// O(n) — single pass through the slice
func linearSearch(s []int, target int) int {
    for i, v := range s {
        if v == target {
            return i
        }
    }
    return -1
}

// O(n²) — inner loop runs up to n times for each outer iteration
func bubbleSort(s []int) {
    n := len(s)
    for i := 0; i < n; i++ {
        for j := 0; j < n-i-1; j++ {
            if s[j] > s[j+1] {
                s[j], s[j+1] = s[j+1], s[j]
            }
        }
    }
}

// O(2ⁿ) — two recursive calls at every level; tree doubles in size
func fibNaive(n int) int {
    if n <= 1 {
        return n
    }
    return fibNaive(n-1) + fibNaive(n-2)
}
```

### A Note on Go Maps

```go
m := map[string]int{"key": 1}
v := m["key"] // O(1) average, O(n) worst case (all keys hash to same bucket)
```

Always qualify: "O(1) *average* case." The O(n) worst case exists and is worth mentioning if asked.

---

## Recurrence Relations and the Master Theorem

Recursive algorithms express their complexity through **recurrence relations** — equations defining T(n) in terms of T at smaller inputs.

### Common Recurrences

| Recurrence | Example | Complexity |
|---|---|---|
| T(n) = T(n/2) + O(1) | Binary search | O(log n) |
| T(n) = T(n−1) + O(1) | Tail recursion | O(n) |
| T(n) = 2T(n/2) + O(1) | — | O(n) |
| T(n) = 2T(n/2) + O(n) | Merge sort | O(n log n) |
| T(n) = T(n−1) + O(n) | Recursive selection sort | O(n²) |
| T(n) = T(n−1) + T(n−2) | Naive Fibonacci | O(2ⁿ) |

### The Master Theorem

Applies to recurrences of the form **T(n) = aT(n/b) + f(n)** where a ≥ 1, b > 1.

- **a** = number of subproblems per recursive call
- **b** = factor by which input size reduces per call
- **f(n)** = work done outside the recursive calls (the "root cost")

Compute the **critical exponent**: **c\* = log\_b(a)**

This represents the total "leaf weight" of the recursion tree — how heavily the recursive branching fans out relative to the work at each level.

```
T(n) = aT(n/b) + f(n)
               │
        c* = log_b(a)
               │
   ┌───────────┼───────────┐
   │           │           │
f < n^c*   f ≈ n^c*   f > n^c*
(leaves     (equal      (root
dominate)    work)     dominates)
   │           │           │
Θ(n^c*)  Θ(n^c* · log n)  Θ(f(n))
[Case 1]    [Case 2]     [Case 3]
```

**Mnemonic**: "The heavy side wins. A tie multiplies by log n."

**Verification — merge sort**: a=2, b=2 → c\* = log₂(2) = 1. f(n) = n = n¹. Equal weight → Case 2 → **Θ(n log n)**. ✓

**Limitation**: Master Theorem requires equal-sized subproblems. It does not apply to T(n) = T(n/3) + T(2n/3) + O(n). In those cases, draw the recursion tree and sum costs level by level.

---

## Amortized Analysis

Some algorithms have operations that are **occasionally expensive but cheap over a sequence**. Worst-case per-operation analysis misleads here — we need to analyse a *sequence* of n operations together.

**Amortized cost = total cost of n operations / n**

### The Canonical Example: Go Slice `append`

When capacity is available: O(1) write. When full: Go doubles capacity and copies all elements — O(current len). But this doubling is rare.

```
Op:      1    2    3    4    5    6    7    8    9    10
Cap:     1    2    4    4    8    8    8    8    16   16
Cost:    1    2    3    1    5    1    1    1    9     1
         ↑    ↑    ↑         ↑                   ↑
    (capacity grows at each marked step; all others are O(1) writes)

Total cost across 10 ops = 1+2+3+1+5+1+1+1+9+1 = 25
Amortized cost per op    = 25 / 10 = 2.5  →  O(1)
```

Across n appends, total copy work = 1+2+4+...+n/2+n ≈ 2n. Amortized cost = 2n/n = **O(1) per append**.

### Three Formal Methods

| Method | How It Works | When to Reach For It |
|---|---|---|
| Aggregate | Total cost / n operations | Simplest; all ops share the same structure |
| Accounting | Assign "credits" to cheap ops; expensive ops spend saved credits | When cheap ops pre-pay for future expensive ones |
| Potential | Define a potential Φ representing stored work; amortized cost = actual + ΔΦ | Most general; used in rigorous proofs |

The aggregate method and the intuition behind it are sufficient for most engineering interviews.

### Amortized vs Average Case

These are different guarantees:

- **Average case** — assumes a probability distribution over inputs; one call *might* be cheap on average
- **Amortized** — a deterministic guarantee over a *sequence* of calls; no randomness required

---

## Space Complexity

### Auxiliary vs Total Space

- **Total space**: input space + auxiliary space
- **Auxiliary space**: extra memory beyond the input — what most problems ask about

### Recursion Uses Stack Space

The most commonly missed point in interviews:

```go
// O(1) auxiliary space — single accumulator variable
func sumIterative(n int) int {
    sum := 0
    for i := 1; i <= n; i++ {
        sum += i
    }
    return sum
}

// O(n) auxiliary space — n stack frames live simultaneously
func sumRecursive(n int) int {
    if n == 0 {
        return 0
    }
    return n + sumRecursive(n-1)
}
```

Both compute the same result. The recursive version silently uses O(n) stack space. Tail-call optimisation (which Go does **not** perform) would eliminate this — in Go, iterative is always safer for space.

### String Concatenation in Go

```go
// O(n²) time and space — new string allocation on every iteration
s := ""
for i := 0; i < n; i++ {
    s += strconv.Itoa(i) // creates and discards a new string each time
}

// O(n) time and space — use strings.Builder
var b strings.Builder
for i := 0; i < n; i++ {
    b.WriteString(strconv.Itoa(i))
}
result := b.String()
```

---

## Complexity Class Reference

| Complexity | Name | Algorithm Example | n=10 | n=100 | n=1,000 |
|---|---|---|---|---|---|
| O(1) | Constant | Array index, map lookup (avg) | 1 | 1 | 1 |
| O(log n) | Logarithmic | Binary search | 3 | 7 | 10 |
| O(n) | Linear | Linear scan, sum | 10 | 100 | 1,000 |
| O(n log n) | Linearithmic | Merge sort, heap sort | 33 | 664 | 9,966 |
| O(n²) | Quadratic | Bubble sort, naive string match | 100 | 10,000 | 10⁶ |
| O(n³) | Cubic | Naive matrix multiply | 1,000 | 10⁶ | 10⁹ |
| O(2ⁿ) | Exponential | Naive Fibonacci, all subsets | 1,024 | ~10³⁰ | ∞ |
| O(n!) | Factorial | All permutations | 3.6M | ∞ | ∞ |

**Practical threshold**: for competitive programming, O(n²) is usually fine up to n ≈ 10⁴, O(n log n) up to n ≈ 10⁶, O(n) up to n ≈ 10⁸.

---

## How to Remember This

### Notation: Ceiling, Floor, Both

O = ceiling (at most). Ω = floor (at least). Θ = both (exact).

Mnemonic: **"O is the cap, Ω is the obligatory minimum, Θ is the True answer."**

### Master Theorem: "Heavy Side Wins — Ties Add Log"

Who carries more weight — the recursive leaves (n^c\*) or the root work (f(n))? The heavier one dominates. When they're equal, add a log n factor (one per level of the recursion tree).

### Growth Order: Left Is Always Better

```
O(1) < O(log n) < O(n) < O(n log n) < O(n²) < O(2ⁿ) < O(n!)
```

Mnemonic phrase (first letters): **"Constant Logs Never Notably Overload Eager Fans"**
(Constant, Log, n, n log n, n², 2ⁿ, n! [Fans])

### Amortized: Mortgage Analogy

Each cheap operation makes a small overpayment that funds the next expensive one. Like a mortgage — small fixed monthly cost, no surprise bills.

---

## Interview Cheat Sheet

### Signal Phrases to Use

- "The time complexity is O(...) because [the nested loops / recursion depth / divide-and-conquer structure]."
- "The space complexity is O(...) auxiliary — the call stack goes [n / log n] frames deep."
- "Hash map lookup is O(1) *average case*, with O(n) worst case if all keys collide."
- "This is amortized O(1) per operation, over a sequence of n operations."
- "Applying Master Theorem: a=[x], b=[y], c\*=log\_b(a)=[z]. f(n)=[...] falls into Case [1/2/3], giving T(n) = Θ(...)."

### Red Flags to Avoid

| Mistake | Correct Approach |
|---|---|
| "HashMap lookup is O(1)" without qualification | Add "average case" — worst case is O(n) |
| Ignoring call stack space for recursive functions | State O(n) auxiliary for O(n)-deep recursion |
| Treating amortized and average case as the same | Amortized = deterministic sequence guarantee; average = probabilistic |
| Saying O(n²) for nested loops without verifying inner loop bounds | Check carefully — may be O(n × m) or even O(n) if inner loop is bounded |
| Confusing O with Θ when precision is asked for | In practice this is fine; if probed, clarify that you mean tight bound |

### Common Interviewer Probes

| Probe | What They're Looking For |
|---|---|
| "Can you improve the space complexity?" | Iterative approach to eliminate call stack; memoisation for recursion |
| "What's the worst case for your hash map operation?" | O(n) due to hash collisions in the same bucket |
| "Why is merge sort O(n log n) and not O(n²)?" | log n levels in the recursion tree × O(n) merge work per level |
| "What's the amortized cost of appending to your slice?" | O(1) — doubling distributes the copy cost across all prior appends |
| "Is this O(n log n) or Θ(n log n)?" | Prove both: show the O upper bound and the Ω lower bound |
| "What happens at n=10 vs n=10⁶?" | Demonstrate feel for how the growth rate translates to real operational cost |

---

## References

- Cormen, Leiserson, Rivest, Stein — *Introduction to Algorithms* (CLRS), Ch. 3 (Growth of Functions) and Ch. 4 (Divide-and-Conquer)
- Knuth — *The Art of Computer Programming*, Vol. 1, §1.2.11 (Asymptotic Representations)
- Sedgewick & Wayne — *Algorithms* (4th ed.), §1.4 (Analysis of Algorithms)
- [MIT 6.006 — Introduction to Algorithms, Lecture 1](https://ocw.mit.edu/courses/6-006-introduction-to-algorithms-fall-2011/)
- [Big-O Cheat Sheet](https://www.bigocheatsheet.com/)