---
Status: 🌳 Evergreen
Created: 2026-11-07
Last Updated: 2026-11-07
---

# Recursion And Backtracking

## Table Of Contents

1. [What It Is And Why It Exists](#what-it-is-and-why-it-exists)
2. [Why "Recursion" And Why "Backtracking"](#why-recursion-and-why-backtracking)
3. [The Call Stack: How Recursion Actually Runs](#the-call-stack-how-recursion-actually-runs)
4. [Base Case And Recursive Case](#base-case-and-recursive-case)
5. [The Recursion Tree](#the-recursion-tree)
6. [Recursion Vs Iteration](#recursion-vs-iteration)
7. [Tail Recursion And Why Go Does Not Optimize It](#tail-recursion-and-why-go-does-not-optimize-it)
8. [Memoisation: Recursion With A Cache](#memoisation-recursion-with-a-cache)
9. [Backtracking: Choose, Explore, Unchoose](#backtracking-choose-explore-unchoose)
10. [Pruning: The Real Payoff Of Backtracking](#pruning-the-real-payoff-of-backtracking)
11. [Classical Problems](#classical-problems)
12. [Recognizing The Pattern](#recognizing-the-pattern)
13. [Go Template Code](#go-template-code)
14. [Where JavaScript Differs](#where-javascript-differs)
15. [LeetCode Problems](#leetcode-problems)
16. [References](#references)

## What It Is And Why It Exists

Recursion is a way of expressing a computation as a smaller version of itself, plus a base case that stops the shrinking. A function calls itself on a strictly simpler input until the input is trivial enough to answer directly. The call chain then unwinds, combining partial answers on the way back up.

Backtracking is a specific *use* of recursion for search problems. You build a candidate solution one decision at a time. At each step you make a choice, recurse to explore its consequences, then undo the choice before trying the next one. When a partial candidate cannot possibly lead to a valid solution, you abandon that branch immediately (this is called pruning). Backtracking is essentially DFS over an implicit tree of decisions, with the ability to give up early.

### The Problem It Solves

Two distinct problems, sharing the same mechanism:

1. **Structurally self-similar problems.** Many problems have the shape "the answer for size `n` is built from the answer for size `n-1` (or two halves of size `n/2`)". Trees, sorted-array bisection, divide-and-conquer sorts, and expression parsing all fit. Writing the recursive definition directly matches the structure of the problem, so the code is short and correct-by-construction. An iterative version usually needs an explicit stack that reproduces exactly what the call stack would have done for free.

2. **Combinatorial search over choices.** Problems like "list every subset", "place N queens so none attack", "solve this Sudoku", "find a path that spells this word" cannot be reduced to a closed-form loop. The state space is a tree of decisions. You must actually walk it. Backtracking is the canonical way to walk it while pruning branches that cannot succeed.

The naive alternative to (2) is "generate all candidates, then filter". For subsets of an n-element set that is 2^n candidates; for permutations, n!; for Sudoku, 9^81. Enumeration without pruning is not tractable for even moderate inputs. Backtracking works because at every level of the tree you can often eliminate whole subtrees with a single cheap check, cutting the effective branching factor far below the worst case.

## Why "Recursion" And Why "Backtracking"

**Recursion** comes from Latin *recurrere*, "to run back". The name reflects the mechanism literally: control runs back into the same function. In mathematics the word describes definitions that refer to themselves (`fact(n) = n * fact(n-1)`); in programming it describes functions that call themselves. The two meanings match because a self-referential definition and a self-calling function compute the same thing.

**Backtracking** is even more literal. When a branch of the search fails, control physically tracks back up the call stack to the last unresolved choice point and tries the next alternative. The word was coined by D. H. Lehmer in the 1950s while working on combinatorial enumeration. It captures the essential move: you *back* up when stuck, then take a different *track*.

**Memoisation** comes from *memo*, "a note". You leave yourself a note of a subproblem's answer so you never solve it twice. It was coined by Donald Michie in 1968. Spelled with an "s" in his original paper; the American "memoization" is common in code but the British spelling is the historically correct one. It is not the same word as "memorization" (which is what humans do to remember things), though the two overlap in meaning.

## The Call Stack: How Recursion Actually Runs

Every function call, recursive or not, pushes a **stack frame** onto the call stack. A frame holds the function's parameters, its local variables, the return address (where to resume in the caller after this call returns), and space for the return value. When the function returns, its frame is popped.

Recursion is unremarkable to the machine: it is just a lot of frames from the same function stacked on top of each other. Each frame has its own copies of the parameters and locals, so `factorial(5)` and `factorial(4)` do not interfere with each other even though they run the same code.

### Frame Growth And Stack Overflow

Deep recursion runs out of stack. In most languages the stack is a fixed small size (Linux default is 8MB per thread, JVM defaults are lower, browsers cap JS at a few tens of thousands of frames). Overflowing gives you the classic `stack overflow` crash.

Go is unusual here. **Goroutine stacks start small (2KB) and grow dynamically**, copied to a larger allocation when they run out. The practical limit is `runtime.SetMaxStack`, default 1GB on 64-bit systems. So Go tolerates deep recursion far better than most languages, but it is not free: each stack copy is O(current stack size), and very deep recursion still costs cache misses and GC pressure. Iterative solutions remain preferred when the depth is unbounded (parsing user input, walking a user-supplied tree).

## Base Case And Recursive Case

Every recursive function has two mandatory parts:

- **Base case.** The input is small enough that you can answer without recursing. Returns directly.
- **Recursive case.** The input is not small enough; delegate to a strictly simpler input.

"Strictly simpler" is the correctness contract. If the recursive call is not smaller by some measure that eventually hits the base case, the recursion never terminates. The measure is called the **variant**. For `factorial(n)` it is `n`; for tree traversal it is the height of the subtree; for backtracking it is usually "positions remaining to fill".

```go
func factorial(n int) int {
    if n <= 1 { // base case
        return 1
    }
    return n * factorial(n-1) // recursive case, strictly smaller
}
```

The three habitual bugs, all failures of the variant contract:

1. **Missing base case.** The function recurses forever. Stack overflow.
2. **Base case never reached.** The variant does not shrink on every call (or shrinks the wrong way, like calling `f(n)` from inside `f(n)`).
3. **Base case is wrong.** Recursion terminates but returns the wrong answer at the bottom, contaminating everything above it.

The debugging discipline: when a recursion breaks, first assert that the recursive call really is on a smaller input, then verify the base case in isolation with a hand-computed expected value.

## The Recursion Tree

The **recursion tree** is a drawing of every call that happens during execution. Each node is one invocation; its children are the calls it makes. Reading it tells you the running time and the memory profile at a glance.

Trace `fib(4)` with the naive definition `fib(n) = fib(n-1) + fib(n-2)`:

```
                    fib(4)
                   /      \
              fib(3)        fib(2)
             /     \       /     \
         fib(2)  fib(1) fib(1)  fib(0)
         /    \
     fib(1)  fib(0)
```

Two facts jump out:

- The **height** of the tree is `n` (each edge decreases the argument by at least 1). Height equals maximum recursion depth, so this bounds the stack space at O(n).
- The **total number of nodes** is the running time. For naive Fibonacci it is roughly `phi^n` where `phi ~ 1.618`. Exponential.

The tree also exposes the repeated work: `fib(2)` appears twice, `fib(1)` three times, `fib(0)` twice. This is exactly the observation that motivates memoisation.

For divide-and-conquer, the tree has a different shape. Merge sort on `n` elements:

```
                    n
                  /   \
               n/2     n/2
              /  \    /   \
            n/4 n/4 n/4  n/4
```

Height is `log2(n)`. Each level does O(n) total work (splitting, merging). Total work is `O(n log n)`. This drawing is where the `n log n` claim comes from mechanically; no formula memorisation needed.

## Recursion Vs Iteration

Every recursion can be converted to iteration by managing an explicit stack. Every iteration can be written recursively. They are equivalent in computational power. The choice is about clarity, performance, and language ergonomics.

Use recursion when:

- The problem is inherently self-similar (trees, divide-and-conquer, backtracking search).
- The recursive version reads like the problem statement and the iterative version needs an explicit `[]Frame` stack that recreates the call stack manually.
- Depth is bounded by the input in a controllable way (a balanced tree is O(log n) deep; a linked-list-like tree is O(n) deep and needs care).

Use iteration when:

- Depth could be adversarially large (parsing untrusted input, walking arbitrary graphs).
- The recursion is tail-recursive but you are in a language that does not eliminate tail calls (Go, Java, Python). Iteration avoids the frame pileup entirely.
- Every microsecond matters and the function is small enough that call overhead dominates.

## Tail Recursion And Why Go Does Not Optimize It

A call is in **tail position** if it is the last thing a function does before returning: the caller has no more work waiting. In principle a compiler can transform such a call into a jump, reusing the current stack frame instead of pushing a new one. This is **tail call optimization (TCO)**. With TCO, tail recursion is exactly as cheap as iteration and cannot overflow the stack.

```go
// tail-recursive shape
func factTail(n, acc int) int {
    if n <= 1 {
        return acc
    }
    return factTail(n-1, n*acc) // last action is the call itself
}
```

The Go compiler does not perform TCO. Two reasons the Go team have stated publicly:

1. **Debugging cost.** TCO drops frames from the stack, which mangles stack traces. Panics and profilers would show a lie: the caller that "really" ran is gone from the trace. The Go team values debuggability of production code over the performance win.
2. **Growing stacks make it less necessary.** Because goroutine stacks grow, tail recursion does not overflow the way it does on a fixed stack. The main motivation for TCO in Scheme or ML is that it's the only way to write loops, since those languages lean heavily on recursion. Go has `for`; if you need iteration semantics, use iteration.

The practical rule in Go: do not rely on tail recursion. If a recursion would need to be deep and its recursive call is in tail position, rewrite it as a `for` loop. If you cannot rewrite it (the recursive call is not in tail position, or the algorithm is genuinely tree-shaped), keep it recursive and trust the growable stack, but be aware that extremely deep recursion still costs stack copies.

## Memoisation: Recursion With A Cache

The naive `fib(n)` computes the same subproblems exponentially many times. Memoisation is the fix: keep a cache of subproblem answers, and on entry check whether the answer is already known.

```go
func fibMemo(n int) int {
    memo := make(map[int]int)
    var f func(k int) int
    f = func(k int) int {
        if k < 2 {
            return k
        }
        if v, ok := memo[k]; ok {
            return v
        }
        v := f(k-1) + f(k-2)
        memo[k] = v
        return v
    }
    return f(n)
}
```

The recursion tree collapses. Each `fib(k)` is now computed at most once. The tree becomes a DAG. Complexity drops from `O(phi^n)` time to `O(n)` time, at the cost of `O(n)` space for the cache.

Memoisation works whenever the recursive function's return value depends only on its arguments. That is, the function must be **pure over its arguments**. If it depends on hidden mutable state, caching by argument is wrong.

Memoisation is also the bridge to **dynamic programming**. Top-down DP is exactly "recursion plus memoisation". Bottom-up DP is the same computation done iteratively by filling a table in dependency order. Both are covered in the DP notes; the point here is that recognizing overlapping subproblems in a recursion tree is the first move that turns an exponential recursion into a polynomial DP.

> [!TIP]
> The signal that memoisation applies: draw the recursion tree, look for the *same call* appearing more than once. If it does, you have overlapping subproblems and memoisation gives an immediate speedup. If every node in the tree is a distinct call (typical of divide-and-conquer, where the two halves never share subproblems), memoisation buys nothing.

## Backtracking: Choose, Explore, Unchoose

Backtracking is a template. Every problem that fits looks identical at the top level:

```
backtrack(partial):
    if partial is a complete solution:
        record it
        return
    for each valid next choice c:
        add c to partial          // choose
        backtrack(partial)        // explore
        remove c from partial     // unchoose
```

Three moves, always in that order. Each move maps directly to a concept:

- **Choose.** Commit to one specific decision. The set of choices at this step is problem-defined: for permutations it's "any unused element"; for N-Queens it's "any safe column in this row"; for Sudoku it's "any digit that fits this cell".
- **Explore.** Recurse. This trusts that the subproblem left after your choice will be solved correctly by the same procedure.
- **Unchoose.** Undo the choice, restoring `partial` to the state it had on entry. This is the crucial move that lets the *next* iteration of the loop start from a clean slate.

The unchoose step is the "backtrack" in backtracking. It is what turns a one-shot recursion into a systematic tree walk.

### Why The Order Matters

Choose-explore-unchoose is not a stylistic preference. The correctness proof rests on it:

- If you *choose then explore* without unchoosing, `partial` accumulates all choices across all branches. The second iteration of the loop starts from a polluted state.
- If you *choose then unchoose* without exploring, you never actually go deeper. Nothing is searched.
- If you *explore before choosing*, you are recursing on a state that does not reflect the current decision. The recursion tree is wrong.

The single trickiest bug is a mismatch between choose and unchoose. If you added one element and forgot to remove it, the frame above sees dirty state. In Go this is where slice aliasing bites hard: `partial = append(partial, x)` might or might not mutate the caller's slice depending on capacity. The safe pattern is explicit index-based push and pop, or defensive copies at the record step.

### The Record Step

At the base case ("partial is a complete solution"), you must **copy** `partial` before appending it to the result set. `partial` is a shared buffer that will be mutated by later iterations. If you store the reference, every entry in `result` ends up pointing to the same slice, and after the recursion finishes they all show the final state.

```go
cp := make([]int, len(current))
copy(cp, current)
result = append(result, cp)
```

Every backtracking solution in this file follows that pattern. Forgetting the copy is one of the two most common bugs in a live coding round (the other is a botched unchoose).

## Pruning: The Real Payoff Of Backtracking

Pruning is what makes backtracking practical. Without it, backtracking is just a fancy way to enumerate the whole tree.

There are two flavors:

- **Constraint pruning (feasibility).** Skip a choice that would violate the problem's constraints. In N-Queens, skip a column that shares a diagonal with an already-placed queen. In Sudoku, skip a digit that already appears in the row, column, or 3x3 box. The check is O(1) with the right auxiliary state (a `used[]` array, a set of blocked diagonals), and it prunes an entire subtree of failed candidates before any deeper work happens.
- **Bound pruning (optimality).** Only for optimization problems. Skip a subtree whose best possible outcome cannot beat the best answer found so far. This is the core idea of branch-and-bound, used in the Traveling Salesman search and in min-max game trees (alpha-beta pruning).

The right pruning decisions can turn a `9^81` search space (Sudoku) into something that solves in milliseconds on a laptop. The wrong pruning is worse than none: an incorrect prune throws out valid solutions silently. The rule is to prune only on *provable* infeasibility. If you are not certain a subtree contains no solution, do not prune it.

The interviewer's follow-up question, whenever they ask a backtracking problem, is almost always "how would you speed this up?" The answer is almost always "add a pruning check here, using this auxiliary structure to keep the check O(1)".

## Classical Problems

### Tower Of Hanoi

Three pegs, `n` disks on the source peg, largest at the bottom. Move them all to the destination peg. Constraint: only one disk at a time, and a larger disk can never sit on a smaller one.

The recursive insight: to move `n` disks from A to C using B as spare, you must first move the top `n-1` disks out of the way (to B), then move the bottom disk (A to C), then move the `n-1` disks back on top (B to C). Two recursive calls plus one direct move.

```go
func hanoi(n int, src, dst, aux string) [][2]string {
    var moves [][2]string
    var solve func(k int, from, to, spare string)
    solve = func(k int, from, to, spare string) {
        if k == 0 {
            return
        }
        solve(k-1, from, spare, to)
        moves = append(moves, [2]string{from, to})
        solve(k-1, spare, to, from)
    }
    solve(n, src, dst, aux)
    return moves
}
```

**Move count: exactly `2^n - 1`.** Let `T(n)` be the number of moves the algorithm makes. The body does `T(n-1)` moves to shift the top stack aside, one move for the bottom disk, then `T(n-1)` moves to stack the rest back. So `T(n) = 2 * T(n-1) + 1` with `T(0) = 0`. Unroll: `T(1)=1, T(2)=3, T(3)=7, T(4)=15` — always `2^n - 1`. That gives time O(2^n). Space is O(n), the maximum call-stack depth.

**Why that's optimal.** No strategy can do better. At some point the largest disk must move from A to C. The only legal destination for it is an empty peg (any disk sitting on a peg is smaller than it, so the rule "larger cannot sit on smaller" blocks it). The only way to make a peg empty is to have every smaller disk piled on the one remaining peg, which must be B. So before the big disk can move, the top `n-1` disks are all on B — a state that takes at least `T(n-1)` moves to reach. One move for the big disk. Then at least `T(n-1)` more moves to put the smaller stack on top of it on C. Any legal play therefore satisfies `T(n) >= 2 * T(n-1) + 1`, matching the upper bound. The algorithm hits the floor.

**Pure recursion, not backtracking.** Backtracking means trying a choice, and if it leads nowhere, undoing it to try another. Hanoi never has alternatives: at every level the three sub-steps are forced ("move n-1 off, move the bottom, move n-1 back"). No branching, no unchoose step. The recursion is a deterministic decomposition, like factorial, not a search.

**The legend, for scale.** The folk framing: monks in a temple moving 64 golden disks one per second; when they finish the world ends. The math: `2^64 - 1 ~ 1.84 * 10^19` moves. Seconds in a year `~ 3.156 * 10^7`. Dividing gives `~ 5.85 * 10^11` years, roughly 42 times the current age of the universe (~13.8 billion years). It is a memorable illustration of exponential growth, nothing more.

### Subsets - [#78 (Medium)](https://leetcode.com/problems/subsets/)

Given an array of distinct integers, return all `2^n` subsets. The decision tree has depth `n`: at each level, decide whether to include `nums[i]`.

```go
func subsets(nums []int) [][]int {
    var result [][]int
    var current []int
    var backtrack func(i int)
    backtrack = func(i int) {
        if i == len(nums) {
            cp := make([]int, len(current))
            copy(cp, current)
            result = append(result, cp)
            return
        }
        backtrack(i + 1)                               // skip nums[i]
        current = append(current, nums[i])             // choose
        backtrack(i + 1)                               // explore
        current = current[:len(current)-1]             // unchoose
    }
    backtrack(0)
    return result
}
```

**Time complexity: `O(n * 2^n)`.** The decision tree is a binary tree of depth `n` — at each level we choose "skip" or "include" `nums[i]`. A binary tree of depth `n` has exactly `2^n` leaves, so there are `2^n` subsets. Writing a subset of average length `n/2` costs `O(n)` at the record step (allocate + copy). Multiply: `2^n * n` total. Internal tree work is also `O(2^n)` (each node does O(1) push/pop around its two recursive calls), which is dominated by the leaf-copy term, so the total stays `O(n * 2^n)`. This is output-sensitive — you cannot list every subset faster than the time it takes to write them all down.

**Space complexity: `O(n)`.** Only the recursion depth and the shared `current` buffer count as auxiliary space. The tree is `n` deep, so the call stack holds up to `n` frames at any moment. The `current` buffer never exceeds length `n`. The `result` slice holding every subset is the output, which convention excludes from space analysis.

### Permutations - [#46 (Medium)](https://leetcode.com/problems/permutations/)

Return every ordering of `n` distinct integers. There are `n!` outputs.

Two common templates. The in-place swap version is minimal:

```go
func permute(nums []int) [][]int {
    var result [][]int
    var backtrack func(start int)
    backtrack = func(start int) {
        if start == len(nums) {
            cp := make([]int, len(nums))
            copy(cp, nums)
            result = append(result, cp)
            return
        }
        for i := start; i < len(nums); i++ {
            nums[start], nums[i] = nums[i], nums[start]
            backtrack(start + 1)
            nums[start], nums[i] = nums[i], nums[start]
        }
    }
    backtrack(0)
    return result
}
```

The swap at position `start` picks which element sits there, recursion arranges the rest, then the unchoose-swap restores the array.

**Time complexity: `O(n * n!)`.** The decision tree has branching factor `n - start` at depth `start`: at the root, `n` choices for position 0; at depth 1, `n-1` remaining; and so on. Multiplied across depths: `n * (n-1) * ... * 1 = n!` leaves, each a distinct permutation. The copy at the record step costs `O(n)` per leaf. Multiply: `n * n!`. Internal nodes contribute `O(n!)` work, dominated by the leaf-copy term. Same output-sensitive floor: writing down `n!` arrays of length `n` cannot be done in fewer than `n * n!` steps.

**Space complexity: `O(n)`.** The recursion goes `n` levels deep (one per filled position). The in-place swap means no auxiliary buffer other than the input array itself — that is one of the reasons to prefer the swap template over `used[]`, which adds an `O(n)` boolean array alongside.

The `used[]` template (a boolean array marking which elements are already in the current prefix) is functionally equivalent and easier to extend to permutations-with-duplicates.

### Combinations - [#77 (Medium)](https://leetcode.com/problems/combinations/)

Choose `k` numbers from `1..n`, order doesn't matter. `C(n, k)` outputs. The decision tree is different from subsets: at each level, pick a starting index and iterate forward from there, so you never revisit smaller numbers (which would produce out-of-order duplicates of the same combination).

```go
func combine(n, k int) [][]int {
    var result [][]int
    var current []int
    var backtrack func(start int)
    backtrack = func(start int) {
        if len(current) == k {
            cp := make([]int, k)
            copy(cp, current)
            result = append(result, cp)
            return
        }
        need := k - len(current)
        for i := start; i <= n-need+1; i++ { // pruning
            current = append(current, i)
            backtrack(i + 1)
            current = current[:len(current)-1]
        }
    }
    backtrack(1)
    return result
}
```

The pruning is the `n - need + 1` upper bound on the loop. If we need `need` more elements and `n - i + 1` remaining numbers is less than `need`, no completion is possible from this start; skip it. Without the prune the loop is `i <= n` and much of the tree explores dead branches.

**Time complexity: `O(k * C(n, k))`.** The problem produces exactly `C(n, k) = n! / (k! * (n-k)!)` combinations, which is the lower bound: you cannot list them faster than you can write them. Factor breakdown:
- **`C(n, k)` = how many outputs.** Thanks to the `start` index enforcing non-decreasing picks and the `n - need + 1` prune, the tree has exactly `C(n, k)` leaves, each a valid combination. No wasted exploration.
- **`k` = per-output work.** At each leaf we copy `k` ints into a new slice (base case) — that is `O(k)` per record.
- **Internal nodes.** A tree with `L` leaves at depth `d` has at most `L * d` total nodes. Here `L = C(n, k)` and `d = k`, so internal work is also bounded by `O(k * C(n, k))`. The two terms are the same order; neither dominates.

Without the prune, the tree would wander into dead branches that can never reach length `k`, so complexity would be worse (`O(k * 2^n)` in the unpruned shape). The `i <= n - need + 1` prune is what makes the complexity tight to the output count.

**Space complexity: `O(k)`.** The recursion goes exactly `k` levels deep (one per element picked), and `current` holds at most `k` values. No auxiliary structures beyond that.

### N-Queens - [#51 (Hard)](https://leetcode.com/problems/n-queens/)

Place N queens on an `NxN` board so that no two attack each other. Queens attack along rows, columns, and both diagonals.

The key insight to keep this from being O(N^N) is that each row must contain exactly one queen. So the decision at row `r` is only "which column?", reducing the branching factor from `N^N` to `N!`. Pruning drops it further.

Three auxiliary boolean arrays make the constraint check O(1) per placement:

- `cols[c]` for occupied columns.
- `diag1[r+c]` for occupied `/`-diagonals (indexed by `r + c`, which is constant along that direction).
- `diag2[r-c+n]` for occupied `\`-diagonals (indexed by `r - c`, shifted by `+n` to keep it non-negative).

```go
func solveNQueens(n int) [][]string {
    var result [][]string
    cols := make([]bool, n)
    diag1 := make([]bool, 2*n)
    diag2 := make([]bool, 2*n)
    board := make([]int, n)

    var place func(r int)
    place = func(r int) {
        if r == n {
            rows := make([]string, n)
            for i := 0; i < n; i++ {
                row := make([]byte, n)
                for j := range row {
                    row[j] = '.'
                }
                row[board[i]] = 'Q'
                rows[i] = string(row)
            }
            result = append(result, rows)
            return
        }
        for c := 0; c < n; c++ {
            if cols[c] || diag1[r+c] || diag2[r-c+n] {
                continue
            }
            cols[c], diag1[r+c], diag2[r-c+n] = true, true, true
            board[r] = c
            place(r + 1)
            cols[c], diag1[r+c], diag2[r-c+n] = false, false, false
        }
    }
    place(0)
    return result
}
```

Diagonal indexing derivation: on a `/`-diagonal, every cell has the same `r + c`. On a `\`-diagonal, every cell has the same `r - c`. The `+n` shift is only needed because `r - c` can be negative and array indices cannot. Any offset that maps `r - c` in `[-(n-1), n-1]` to a non-negative index would work; `+n` gives `[1, 2n-1]` and keeps the arithmetic obvious.

**Time complexity.** The upper bound without pruning is `O(N!)`: at row 0 there are `N` column choices, at row 1 there are `N - 1` (excluding the column used above), and so on — giving `N * (N-1) * (N-2) * ... = N!` leaves. With the diagonal constraints added, most of those branches get pruned before reaching the bottom, so the real tree is strictly smaller than `N!`. No closed form exists for the pruned count — the number of valid solutions grows roughly like `N! / c^N` empirically, but it's an open mathematical question. Interview-level statement: "worst case `O(N!)`, with constraint pruning making the practical runtime much lower." For scale: N=8 has 92 solutions (microseconds), N=15 has 2,279,184 (still tractable in seconds).

**Space complexity: `O(N)`.** Recursion is `N` deep (one call per row). Auxiliary state is three boolean arrays of sizes `N`, `2N`, `2N` for columns and the two diagonal directions — all `O(N)`. The `board` int slice is also `O(N)`. Total auxiliary: `O(N)`.

#### Follow-Up: N-Queens II - [#52 (Hard)](https://leetcode.com/problems/n-queens-ii/)

The official follow-up asks for the *count* of distinct solutions rather than the configurations. Nothing about the search changes; only the recording step does. Instead of materialising each board into a slice of strings and appending it to a result list, just increment a counter at the base case. Everything else stays identical: same per-row loop, same three-way constraint check, same choose/unchoose on `cols`, `diag1`, `diag2`.

```go
func totalNQueens(n int) int {
    count := 0
    cols := make([]bool, n)
    diag1 := make([]bool, 2*n)
    diag2 := make([]bool, 2*n)
    var place func(r int)
    place = func(r int) {
        if r == n {
            count++
            return
        }
        for c := 0; c < n; c++ {
            if cols[c] || diag1[r+c] || diag2[r-c+n] {
                continue
            }
            cols[c], diag1[r+c], diag2[r-c+n] = true, true, true
            place(r + 1)
            cols[c], diag1[r+c], diag2[r-c+n] = false, false, false
        }
    }
    place(0)
    return count
}
```

The win is memory. The original `solveNQueens` allocates N strings per solution plus the outer slice holding every one; for `n=14` with 365,596 solutions, that is roughly 5M strings held in memory just to be counted. The follow-up drops that to a single `int`. Time is unchanged, since the pruned tree walked is the same tree. This is the common pattern for "all vs count" follow-ups: keep the search, swap the aggregator.

### Sudoku Solver - [#37 (Hard)](https://leetcode.com/problems/sudoku-solver/)

Fill a 9x9 grid so every row, column, and 3x3 box contains 1..9 exactly once. Some cells are pre-filled.

Naive backtracking: for each empty cell in order, try each digit 1-9 that does not violate any constraint. On the first violation-free assignment, recurse; if the recursion cannot complete, undo and try the next digit.

Constraint tracking again uses O(1) checks. `rows[r][d]`, `cols[c][d]`, `boxes[r/3][c/3][d]` each mark whether digit `d` is already placed in that unit.

```go
func solveSudoku(board [][]byte) {
    var rows, cols [9][10]bool
    var boxes [3][3][10]bool
    for r := 0; r < 9; r++ {
        for c := 0; c < 9; c++ {
            if board[r][c] != '.' {
                d := int(board[r][c] - '0')
                rows[r][d] = true
                cols[c][d] = true
                boxes[r/3][c/3][d] = true
            }
        }
    }
    var solve func(pos int) bool
    solve = func(pos int) bool {
        if pos == 81 {
            return true
        }
        r, c := pos/9, pos%9
        if board[r][c] != '.' {
            return solve(pos + 1)
        }
        for d := 1; d <= 9; d++ {
            if rows[r][d] || cols[c][d] || boxes[r/3][c/3][d] {
                continue
            }
            board[r][c] = byte('0' + d)
            rows[r][d], cols[c][d], boxes[r/3][c/3][d] = true, true, true
            if solve(pos + 1) {
                return true
            }
            board[r][c] = '.'
            rows[r][d], cols[c][d], boxes[r/3][c/3][d] = false, false, false
        }
        return false
    }
    solve(0)
}
```

Notice `solve` returns `bool`. A Sudoku puzzle in the standard formulation has exactly one solution, so as soon as the recursion finds one it can return `true` all the way up and stop searching. This is a small but common variation: return-as-soon-as-found versus record-all-and-continue. Which one to use is a problem-specific decision.

**Time complexity: `O(9^m)` worst case**, where `m` is the number of empty cells (at most 81). The reasoning: at each empty cell we try up to 9 digits, and the recursion tree has one level per empty cell. A tree with branching factor 9 and depth `m` has `9^m` leaves. Constraint pruning (skipping digits that already appear in the row, column, or box) drops the effective branching factor well below 9 — in a typical puzzle most cells have only one or two legal candidates by the time the recursion reaches them, so real boards solve in microseconds despite the exponential upper bound.

**Space complexity: `O(m)`** for recursion depth (bounded by 81, so this is really `O(1)` since the board size is fixed), plus `O(1)` auxiliary for the `rows`, `cols`, `boxes` arrays — all fixed 9x10 or 3x3x10 shapes. The input board is modified in place, so no auxiliary copy of it is kept.

An interviewer's follow-up: how would you speed this up further? The standard answer is **most-constrained-variable heuristic**: instead of filling cells in position order, pick the empty cell with the fewest legal candidates and fill that one next. This dramatically prunes the tree because the highest-constraint cells fail fastest.

## Recognizing The Pattern

You are in recursion-or-backtracking territory when the problem says any of:

- "Return all possible ..." (subsets, permutations, combinations, partitions, arrangements, paths).
- "Find any valid ..." (Sudoku solution, N-Queens placement, path spelling a word).
- "Count the number of ways to ..." where each way is a sequence of decisions.
- "Generate all ..." over a structure defined recursively (all balanced parenthesizations, all binary trees of size n).

You are specifically in *backtracking* territory (as opposed to plain recursion) when the problem is a search over choices and you can incrementally build a candidate solution one decision at a time.

You are *not* in backtracking territory when:

- The problem asks for the *count* of valid outcomes and each subproblem's count depends only on its state, not on the path taken. That is DP.
- The problem asks for the *optimal* value of a valid outcome and the optimal-substructure property holds. That is DP (or greedy).
- The problem is a shortest path in a graph. That is BFS or Dijkstra.

The line between backtracking and DP is thin. If a backtracking search shows overlapping subproblems in its recursion tree, memoising the state converts it to DP.

## Go Template Code

The canonical backtracking skeleton, ready to adapt:

```go
func solveXxx(input Input) []Output {
    var result []Output
    var current State // mutable buffer holding the in-progress candidate

    var backtrack func(step int)
    backtrack = func(step int) {
        if isComplete(current, step) {
            result = append(result, copyOf(current))
            return
        }
        for _, choice := range validChoices(current, step, input) {
            apply(&current, choice)   // choose
            backtrack(step + 1)       // explore
            undo(&current, choice)    // unchoose
        }
    }

    backtrack(0)
    return result
}
```

Substitutions to make per problem:

| Slot | Subsets | Permutations | N-Queens |
|---|---|---|---|
| `State` | `[]int` prefix | full array being permuted | `[]int` col-per-row |
| `isComplete` | index == len(input) | start == len(input) | row == n |
| `validChoices` | {skip, include nums[i]} | unused elements | safe columns |
| `apply` | append | swap | mark col/diag |
| `undo` | pop | swap back | unmark |
| `copyOf` | `copy` into new slice | `copy` into new slice | render as strings |

The skeleton itself does not change. What changes is the definition of the state and the choice set.

### Closure Vs Passing Parameters

Two Go idioms for backtracking: closure over the outer variables (the `var backtrack func(...); backtrack = func(...) {...}` shape used above), or pass everything as parameters. Closure is more concise and avoids threading `result`, `current`, and `input` through every call. The self-reference dance (declare, then assign) is required because Go does not let a `func` literal reference itself by name.

The alternative, passing state as parameters, is more explicit and easier to unit-test but requires more typing. In interviews the closure form is standard; in production, either is fine.

## Where JavaScript Differs

**Stack depth.** JS engines cap the stack at a fixed size (V8 defaults to ~15,000 frames). Node lets you raise it with `--stack-size=...`. Deep recursion that Go handles by growing goroutine stacks will overflow in JS.

**Tail calls.** ES6 mandated TCO in strict mode, but only Safari ever implemented it. V8 removed its (never-shipped) implementation citing the same debuggability concerns as Go. Do not rely on TCO in production JS either.

**Closures over mutable state.** The pattern `let current = []; backtrack(); current.push(x); ... current.pop()` works identically to Go, but `Array.push`/`pop` mutate in place with no capacity surprises. In Go, `append` may or may not mutate the underlying array depending on capacity; this can cause aliasing bugs across recursive calls. In JS the pain shifts elsewhere: `result.push(current)` stores a reference, so you must `result.push([...current])` to copy. Same bug, different keystroke.

## LeetCode Problems

### 1. Combination Sum - [#39 (Medium)](https://leetcode.com/problems/combination-sum/)

Given distinct positive integers `candidates` and a `target`, return every combination that sums to `target`. Each candidate may be used unlimited times.

<details>
<summary>Brute Force</summary>

Enumerate every multiset of candidates whose length is at most `target / min(candidates)`, then filter to those that sum to `target`. Since ordering does not matter, this over-enumerates permutations of the same multiset.

Time: O(N^(T/M)) with a large permutation constant. Space: O(T/M) recursion.

The waste: the same multiset is produced in every ordering, then deduplicated at the end. Enforcing a non-decreasing pick order collapses all those permutations to one, and sorting once lets an early `break` cut branches that already exceed the target.
</details>

<details>
<summary>Hint 1</summary>

The decision at each step is "which candidate do I add next?". To avoid producing the same combination in different orders (like `[2,2,3]` and `[3,2,2]`), enforce that each recursive call can only pick from a non-decreasing index of `candidates`.
</details>

<details>
<summary>Hint 2</summary>

Because reuse is allowed, the recursive call after picking `candidates[i]` passes `i` (not `i+1`) as the new start. Sort the candidates once so you can prune with an early `break` when `candidates[i] > remaining`.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(N^(T/M)) where N=len(candidates), T=target, M=min candidate | Space: O(T/M) recursion
func combinationSum(candidates []int, target int) [][]int {
    sort.Ints(candidates)
    var result [][]int
    var current []int
    var backtrack func(start, remaining int)
    backtrack = func(start, remaining int) {
        if remaining == 0 {
            cp := make([]int, len(current))
            copy(cp, current)
            result = append(result, cp)
            return
        }
        for i := start; i < len(candidates); i++ {
            if candidates[i] > remaining {
                break
            }
            current = append(current, candidates[i])
            backtrack(i, remaining-candidates[i])
            current = current[:len(current)-1]
        }
    }
    backtrack(0, target)
    return result
}
```
</details>

<details>
<summary>Complexity</summary>

**Time: `O(N^(T/M))`** where `N = len(candidates)`, `T = target`, `M = min(candidates)`.
- **Depth = `T/M`.** The deepest branch keeps picking the smallest candidate every step, so it bottoms out after `T/M` picks (any more and `remaining` goes negative). That bounds the recursion depth.
- **Branching factor = `N`.** Each node iterates up to `N` candidates.
- A tree with branching factor `N` and depth `T/M` has at most `N^(T/M)` nodes. Each node does O(1) work — one comparison, one push/pop, one subtraction.
- The sort-plus-prune cuts most of that in practice, but there is no cleaner closed form for the pruned size.

**Space: `O(T/M)`** for the recursion stack. Each frame holds a constant amount of state. The `current` buffer grows by one entry per frame, bounded by `T/M`.
</details>

### 2. Generate Parentheses - [#22 (Medium)](https://leetcode.com/problems/generate-parentheses/)

Given `n`, return all combinations of `n` well-formed pairs of parentheses.

<details>
<summary>Brute Force</summary>

Generate every binary string of length `2n` over `{'(', ')'}`, then filter by a validity check that scans left to right maintaining a balance counter.

Time: O(2^(2n) * n). Space: O(n) recursion.

The waste: the filter step throws away most strings. Building the tree so that only valid prefixes are ever extended (open count bounded by `n`, close count bounded by open) prunes the invalid branches at construction time. The output count drops to the Catalan number and each output is built directly.
</details>

<details>
<summary>Hint 1</summary>

Every well-formed string of length `2n` is a sequence of decisions "open" or "close" at each position. Two constraints keep the string valid: never open more than `n` times, and never close more than are currently open.
</details>

<details>
<summary>Hint 2</summary>

Track two counters: `open` (opens used so far) and `close` (closes used so far). Recurse with an `(` only if `open < n`; recurse with a `)` only if `close < open`. When length reaches `2n`, record and return.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(4^n / sqrt(n)) (Catalan number) | Space: O(n) recursion depth
func generateParenthesis(n int) []string {
    var result []string
    var build func(cur []byte, open, close int)
    build = func(cur []byte, open, close int) {
        if len(cur) == 2*n {
            result = append(result, string(cur))
            return
        }
        if open < n {
            build(append(cur, '('), open+1, close)
        }
        if close < open {
            build(append(cur, ')'), open, close+1)
        }
    }
    build(make([]byte, 0, 2*n), 0, 0)
    return result
}
```
</details>

<details>
<summary>Complexity</summary>

**Time: `O(4^n / sqrt(n))`** — the asymptotic growth of the nth Catalan number.
- Every leaf of the recursion tree is a valid output. The constraints (`open < n`, `close < open`) prune invalid prefixes at construction, so no branch is ever wasted.
- The number of valid n-pair strings is the nth Catalan number `C(n) = binomial(2n, n) / (n+1)`. By Stirling, `binomial(2n, n) ~ 4^n / sqrt(pi * n)`, so `C(n) ~ 4^n / n^(3/2)`.
- Each leaf copies a string of length `2n`, costing `O(n)` per record. Multiplying: `C(n) * n ~ 4^n / sqrt(n)`. That is the stated bound.
- Contrast with brute force: generating every binary string of length `2n` and filtering is `O(4^n * n)`. Pruning saves a factor of `n` by never building invalid strings.

**Space: `O(n)`** for recursion depth. Max depth is `2n`, and each frame holds constant state plus a slice reference. The `cur` buffer never exceeds `2n` bytes.
</details>

### 3. Word Search - [#79 (Medium)](https://leetcode.com/problems/word-search/)

Given a 2D board of characters and a word, return true if the word can be spelled by moving to horizontally or vertically adjacent cells without reusing a cell.

<details>
<summary>Brute Force</summary>

Enumerate every simple path of length `L = len(word)` on the grid, compare each to `word`.

Time: O(m*n * 4^L). Space: O(L) for the path.

The waste: paths that mismatch the word at their second character are still fully generated to length `L` before being rejected. Cutting the walk the instant `board[r][c] != word[i]` avoids all of that. Complexity looks the same on paper but the pruned version returns as soon as a mismatch happens, so real workloads are much faster.
</details>

<details>
<summary>Hint 1</summary>

Try starting the search from every cell whose value matches the first character of the word. From each candidate start, do DFS through the neighbors as long as they keep matching the next character.
</details>

<details>
<summary>Hint 2</summary>

To enforce "no cell reused within one search path", temporarily overwrite the current cell with a sentinel like `'#'` before recursing, then restore it after. This costs zero extra space compared to a visited set.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(m*n * 4^L) where L is word length | Space: O(L) recursion
func exist(board [][]byte, word string) bool {
    rows, cols := len(board), len(board[0])
    var dfs func(r, c, i int) bool
    dfs = func(r, c, i int) bool {
        if i == len(word) {
            return true
        }
        if r < 0 || r >= rows || c < 0 || c >= cols || board[r][c] != word[i] {
            return false
        }
        saved := board[r][c]
        board[r][c] = '#'
        found := dfs(r+1, c, i+1) || dfs(r-1, c, i+1) ||
            dfs(r, c+1, i+1) || dfs(r, c-1, i+1)
        board[r][c] = saved
        return found
    }
    for r := 0; r < rows; r++ {
        for c := 0; c < cols; c++ {
            if dfs(r, c, 0) {
                return true
            }
        }
    }
    return false
}
```
</details>

<details>
<summary>Complexity</summary>

Let `m * n` be the grid dimensions and `L = len(word)`.

**Time: `O(m * n * 4^L)`.**
- **`m * n`.** The outer loop starts a DFS from every cell.
- **`4^L`.** Each DFS call branches in up to 4 directions (up, down, left, right), and the recursion depth is `L` (one call per character of `word`). A 4-way tree of depth `L` has `4^L` leaves. In practice the "no cell reuse" sentinel drops one of those 4 branches (we never go back the way we came), so the effective branching is 3 and the tree is closer to `3^L`. The textbook statement is `4^L` as the clean upper bound.
- Early termination (`!= word[i]` returns immediately) prunes most paths long before they reach depth `L`. On a sparse board the real runtime is far below `4^L`.

**Space: `O(L)`** for recursion depth. The board is modified in place (sentinel `'#'`), so no visited set is kept on the side. Each frame holds constant state.
</details>

<details>
<summary>Follow-Up: Search Pruning For A Larger Board</summary>

The official follow-up asks: "Could you use search pruning to make your solution faster with a larger `board`?"

Two cheap wins before the DFS even starts, and one direction-of-search optimization:

**1. Frequency fail-fast.** Count every character on the board once. Count every character in `word`. If any character in `word` needs more copies than the board holds, return `false` immediately — no DFS at all. On a large sparse board where the target word contains a rare letter not on the grid, this skips the entire search.

**2. Search from the rarer endpoint.** Compare the board frequency of `word[0]` against `word[len-1]`. If the last letter is rarer on the board, reverse the word and search for the reversed string. Fewer candidate starting cells means a smaller outer loop and a smaller tree — DFS only begins from cells matching the first character of whichever end is rarer. Correctness holds because a valid path for the original word, traversed backward, is a valid path for the reversed word.

**3. Prune the outer loop.** The baseline calls `dfs(r, c, 0)` at every cell. Guarding with `board[r][c] == w[0]` skips cells that cannot start a match. Negligible on small boards; meaningful on large ones where the first character is rare.

```go
// Time: O(m*n * 4^L) worst case, same as baseline, but typical runtime is far lower
// Space: O(L) recursion + O(1) auxiliary for frequency maps (bounded by alphabet size)
func existPruned(board [][]byte, word string) bool {
    rows, cols := len(board), len(board[0])

    freq := make(map[byte]int)
    for r := 0; r < rows; r++ {
        for c := 0; c < cols; c++ {
            freq[board[r][c]]++
        }
    }
    wordFreq := make(map[byte]int)
    for i := 0; i < len(word); i++ {
        wordFreq[word[i]]++
    }
    for ch, need := range wordFreq {
        if freq[ch] < need {
            return false // fail-fast: board lacks enough of this char
        }
    }

    w := []byte(word)
    if freq[w[len(w)-1]] < freq[w[0]] {
        for l, r := 0, len(w)-1; l < r; l, r = l+1, r-1 {
            w[l], w[r] = w[r], w[l]
        }
    }

    var dfs func(r, c, i int) bool
    dfs = func(r, c, i int) bool {
        if i == len(w) {
            return true
        }
        if r < 0 || r >= rows || c < 0 || c >= cols || board[r][c] != w[i] {
            return false
        }
        saved := board[r][c]
        board[r][c] = '#'
        found := dfs(r+1, c, i+1) || dfs(r-1, c, i+1) ||
            dfs(r, c+1, i+1) || dfs(r, c-1, i+1)
        board[r][c] = saved
        return found
    }
    for r := 0; r < rows; r++ {
        for c := 0; c < cols; c++ {
            if board[r][c] == w[0] && dfs(r, c, 0) {
                return true
            }
        }
    }
    return false
}
```

Worst-case big-O is unchanged — an adversarial grid that satisfies every prune check still forces full exploration. The win is in the common case: on a 200x200 board where the target word contains a rare character, the fail-fast alone can return in microseconds instead of seconds.

</details>

### 4. Letter Combinations Of A Phone Number - [#17 (Medium)](https://leetcode.com/problems/letter-combinations-of-a-phone-number/)

Given a string of digits 2-9, return all possible letter combinations, using the classic phone-keypad mapping.

<details>
<summary>Brute Force</summary>

Iterative Cartesian product: start with `[""]`, and for each digit rebuild the working list by concatenating every prefix with every letter for that digit.

Time: O(4^n * n). Space: O(4^n * n) held in the working list at every step.

The waste: the iterative version materialises every partial-length list before moving on, holding intermediate lists in memory. Recursion streams outputs one at a time using only a single buffer of length `n`, so the auxiliary space drops to O(n). Time is identical since the output count is fixed.
</details>

<details>
<summary>Hint 1</summary>

At each digit, the choice set is the 3 or 4 letters mapped to that digit. Depth of the recursion is the number of digits. Total outputs are the product of choices per level.
</details>

<details>
<summary>Hint 2</summary>

Keep a `map[byte]string` from digit to its letters. The backtracking function takes the current digit index and a byte-slice buffer; at the base case, convert the buffer to a string and record it.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(4^n * n) where n=len(digits) | Space: O(n) recursion + output buffer
func letterCombinations(digits string) []string {
    if len(digits) == 0 {
        return nil
    }
    pad := map[byte]string{
        '2': "abc", '3': "def", '4': "ghi", '5': "jkl",
        '6': "mno", '7': "pqrs", '8': "tuv", '9': "wxyz",
    }
    var result []string
    var backtrack func(i int, cur []byte)
    backtrack = func(i int, cur []byte) {
        if i == len(digits) {
            result = append(result, string(cur))
            return
        }
        for _, ch := range pad[digits[i]] {
            backtrack(i+1, append(cur, byte(ch)))
        }
    }
    backtrack(0, make([]byte, 0, len(digits)))
    return result
}
```
</details>

<details>
<summary>Complexity</summary>

Let `n = len(digits)`.

**Time: `O(4^n * n)`.**
- **`4^n`.** Each digit maps to 3 or 4 letters (7 and 9 give 4). The recursion tree has branching factor at most 4 and depth `n`, giving at most `4^n` leaves. Every leaf is a valid output (no pruning needed — every letter-choice is legal), so the number of outputs is also `O(4^n)`.
- **`n`.** At each leaf, we convert the byte buffer to a string, allocating and copying `n` bytes.
- Internal nodes contribute `O(4^n)`, dominated by the leaf cost.

**Space: `O(n)`** for recursion depth. The `cur` buffer holds at most `n` bytes. The digit-to-letters map is `O(1)` (fixed 8 entries). The result slice is output, not auxiliary.
</details>

### 5. Subsets II - [#90 (Medium)](https://leetcode.com/problems/subsets-ii/)

Like Subsets, but the input may contain duplicates and the output must not contain duplicate subsets.

<details>
<summary>Brute Force</summary>

Run the plain Subsets backtracking to produce all `2^n` outputs including duplicates, then insert each into a set (keyed on the sorted subset serialised as a string) to deduplicate.

Time: O(n * 2^n) generation plus O(n * 2^n) hashing. Space: O(n * 2^n) for the set.

The waste: duplicates are created and then thrown away. Sorting once and skipping equal values at the same recursion level prevents duplicates from being generated in the first place, so the set is unnecessary and space drops to O(n) auxiliary.
</details>

<details>
<summary>Hint 1</summary>

Reuse the plain Subsets template exactly: at each index, binary choice — include `nums[i]`, or skip it. The base case is still `i == len(nums)`. The only extension needed is how the "skip" branch handles duplicates.
</details>

<details>
<summary>Hint 2</summary>

Sort the input so duplicate values are adjacent. When you choose to skip `nums[i]`, also skip every *other* copy of that same value in one jump — because picking a later copy of the same value would produce a multiset identical to picking this one. So "include this value" and "skip this value entirely" are the only two non-redundant branches.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n * 2^n) | Space: O(n) recursion
func subsetsWithDup(nums []int) [][]int {
    sort.Ints(nums)
    var result [][]int
    var current []int
    var backtrack func(i int)
    backtrack = func(i int) {
        if i == len(nums) {
            cp := make([]int, len(current))
            copy(cp, current)
            result = append(result, cp)
            return
        }
        // Choice 1: include nums[i]
        current = append(current, nums[i])
        backtrack(i + 1)
        current = current[:len(current)-1]

        // Choice 2: skip nums[i] AND every other copy of nums[i].
        // Picking a later copy would be equivalent to picking this one,
        // so the only non-redundant alternative is to skip the whole group.
        j := i + 1
        for j < len(nums) && nums[j] == nums[i] {
            j++
        }
        backtrack(j)
    }
    backtrack(0)
    return result
}
```

This mirrors the plain Subsets template (`backtrack`, base case `i == len(nums)`, binary include/skip, choose-explore-unchoose on the include branch). The duplicate-handling is now a property of the *skip* branch — one extra `for` loop that advances past the duplicate group — rather than a `continue` nested inside a loop. The record step only fires at the base case, same as plain Subsets, instead of at every recursion entry.
</details>

<details>
<summary>Complexity</summary>

Let `n = len(nums)`.

**Time: `O(n * 2^n)`.**
- The worst case is when all values are distinct — the duplicate-skip never fires and the tree is identical to plain Subsets: `2^n` subsets.
- Each subset is of average length `n/2`, copied at the record step for `O(n)` per output.
- Multiply: `n * 2^n`.
- When duplicates dominate, the tree is strictly smaller because `i > start && nums[i] == nums[i-1]` prunes same-level repeats, so real runtime can be much less. `O(n * 2^n)` is the ceiling.
- The initial `sort.Ints(nums)` costs `O(n log n)`, dominated by the tree traversal.

**Space: `O(n)`** for recursion depth plus the `current` buffer. The sort is in-place.
</details>

### 6. Palindrome Partitioning - [#131 (Medium)](https://leetcode.com/problems/palindrome-partitioning/)

Given a string `s`, return all ways to split it so every piece is a palindrome.

<details>
<summary>Brute Force</summary>

There are `2^(n-1)` ways to choose cut points in a string of length `n` (each gap is cut or not cut). Generate all of them, then for each cut-set check that every resulting piece is a palindrome.

Time: O(2^(n-1) * n) with a per-cut-set validity scan. Space: O(n) for a cut-set representation.

The waste: cut-sets whose first piece is already non-palindromic are still fully generated before being rejected. Building the split left to right and only recursing when the next piece is a palindrome prunes those branches at construction time.
</details>

<details>
<summary>Hint 1</summary>

At each step, choose the length of the next piece: try every substring starting at the current position that is itself a palindrome. Recurse from the end of that piece.
</details>

<details>
<summary>Hint 2</summary>

An in-place palindrome check with two pointers is O(length) and keeps memory allocation off the hot path. If you want to make it faster, precompute `isPal[i][j]` in O(n^2) once so each check is O(1).
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n * 2^n) worst case | Space: O(n) recursion
func partition(s string) [][]string {
    var result [][]string
    var current []string
    isPal := func(l, r int) bool {
        for l < r {
            if s[l] != s[r] {
                return false
            }
            l++
            r--
        }
        return true
    }
    var backtrack func(start int)
    backtrack = func(start int) {
        if start == len(s) {
            cp := make([]string, len(current))
            copy(cp, current)
            result = append(result, cp)
            return
        }
        for end := start; end < len(s); end++ {
            if isPal(start, end) {
                current = append(current, s[start:end+1])
                backtrack(end + 1)
                current = current[:len(current)-1]
            }
        }
    }
    backtrack(0)
    return result
}
```
</details>

<details>
<summary>Complexity</summary>

Let `n = len(s)`.

**Time: `O((n + n^2) * 2^n) = O(n^2 * 2^n)` worst case.** Two independent sources of work per node, added together, then multiplied by the tree size:

- **`n` per leaf = recording cost.** Between every pair of adjacent characters there is a "cut or don't cut" decision — that is `n - 1` binary decisions, giving up to `2^(n-1)` partitions. In a string where every substring is a palindrome (e.g. `"aaaa"`), all `2^(n-1)` survive. Each partition has at most `n` pieces; copying them into the result slice costs `O(n)`. Total recording: `O(n * 2^n)`.
- **`n^2` per node = palindrome-check cost.** At a node with `start = s`, the loop iterates `end` from `s` to `n-1` and calls `isPal(s, end)` on each candidate, costing `O(end - s + 1)` per check. Summed at that single node: `1 + 2 + ... + (n - s) = O((n - s)^2)`, worst case `O(n^2)`. Tree has up to `2^n` nodes. Total check work: `O(n^2 * 2^n)`.
- **Combined:** `O((n + n^2) * 2^n)`. Big-O collapses to `O(n^2 * 2^n)` since `n^2` dominates `n`. The additive form is useful for seeing where each factor comes from.
- **Textbook `O(n * 2^n)` reconciled.** Many editorials cite the tighter `O(n * 2^n)` bound. Two ways to justify it: (1) precompute an `n x n` palindrome table in `O(n^2)` upfront with a simple DP (`isPal[i][j] = (s[i] == s[j]) && isPal[i+1][j-1]`), turning every in-recursion check into `O(1)` and collapsing the per-node cost from `O(n^2)` to `O(n)`; (2) argue that on real inputs most `isPal` calls return false on the first character mismatch, making the cost output-dominated in practice. The code in this file is the no-precompute version, so the strict bound is `O(n^2 * 2^n)`.

**Space: `O(n)`** for recursion depth and the `current` buffer of pieces. No palindrome table is cached in this version; the in-place `isPal` uses `O(1)` extra. Adding the precompute would push auxiliary space to `O(n^2)`.
</details>

## References

- [LeetCode: Backtracking Tag](https://leetcode.com/tag/backtracking/)
- [NeetCode: Backtracking Roadmap](https://neetcode.io/roadmap)
- Cormen, Leiserson, Rivest, Stein - *Introduction to Algorithms*, 3rd ed., §15 (Dynamic Programming) for the memoisation-to-DP transition and §35 (Approximation Algorithms) for branch-and-bound
- [Go Team: FAQ on Tail Call Optimization](https://go.dev/doc/faq)
- Donald Michie (1968), "Memo Functions and Machine Learning", *Nature* 218 (the origin of the word memoisation)
- Knuth - *The Art of Computer Programming*, Vol. 4A, §7.2.2 for the definitive treatment of backtracking as combinatorial search