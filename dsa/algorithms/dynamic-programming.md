---
Status: 🌳 Evergreen
Created: 2026-11-14
Last Updated: 2026-11-14
---

# Dynamic Programming

## Table Of Contents

1. [What It Is And Why It Exists](#what-it-is-and-why-it-exists)
2. [Why "Dynamic Programming"](#why-dynamic-programming)
3. [When DP Applies](#when-dp-applies)
4. [Top-Down Vs Bottom-Up](#top-down-vs-bottom-up)
5. [Designing The State](#designing-the-state)
6. [Space Optimisation](#space-optimisation)
7. [Reversing The Direction](#reversing-the-direction)
8. [Classical Problems](#classical-problems)
9. [DP On Trees](#dp-on-trees)
10. [Recognizing The Pattern](#recognizing-the-pattern)
11. [Go Template Code](#go-template-code)
12. [Where JavaScript Differs](#where-javascript-differs)
13. [LeetCode Problems](#leetcode-problems)
14. [References](#references)

## What It Is And Why It Exists

Dynamic programming (DP) turns an exponential recursion into a polynomial one by solving each distinct subproblem once and reusing the answer. Total work becomes `number of states × cost per transition`.

It is a paradigm, not an algorithm. [Fibonacci](https://leetcode.com/problems/fibonacci-number/) and [Edit Distance](https://leetcode.com/problems/edit-distance/) share no code, but they share the discipline: define the subproblems, express the answer as a recurrence over them, and evaluate the recurrence without repeating work.

### The Problem It Solves

Many optimisation and counting problems have a natural recursive definition: "the best answer for `n` is the best of a few choices, each of which leaves a smaller problem". Evaluated naively, the same smaller problems are solved again and again, and the running time explodes.

DP sits between two neighbours:

- **Divide and conquer** (merge sort) also splits into subproblems, but they never overlap, so there is nothing to reuse.
- **Greedy** commits to one local choice and never revisits. It works only when local optima compose into a global optimum. When they do not ([Coin Change](https://leetcode.com/problems/coin-change/) with arbitrary denominations), DP explores all choices but pays for each subproblem only once.

## Why "Dynamic Programming"

Richard Bellman coined the term at RAND in the early 1950s while working on multi-stage decision processes. In his autobiography he explains that the Secretary of Defense at the time, Charles Wilson, was hostile to the word "research", so Bellman picked a name nobody could object to.

- **Dynamic** means multi-stage, time-varying: a decision process that unfolds step by step. It has nothing to do with dynamic memory or dynamic dispatch.
- **Programming** means planning or tabulating, the same sense as "linear programming". It has nothing to do with writing code.

Read it as "multi-stage planning by filling a table".

## When DP Applies

Two properties must both hold.

**Overlapping subproblems.** The recursion revisits the same arguments. The Fibonacci recursion tree in [recursion-backtracking.md](recursion-backtracking.md#the-recursion-tree) shows `fib(2)` computed repeatedly; caching it collapses O(2^n) to O(n). If every call in the tree is distinct (merge sort halves), caching buys nothing.

**Optimal substructure.** DP stores one answer per subproblem, the best one, and discards the rest. That is only safe if the best answer to the whole can always be assembled from the best answers to its subproblems. If the whole ever needs a second-best piece, DP has already thrown it away.

Shortest path has this property. If the shortest route from A to C is A → B → C, then its A → B part must be the shortest route from A to B. If a shorter A → B route existed, swapping it in would give a shorter A → C route, a contradiction.

Longest simple path (no repeated vertices) does not. Take a triangle:

```
    A
   / \
  B---C
```

- The longest simple path from A to C is A → B → C.
- Its piece A → B is not the longest path from A to B. That is A → C → B.

So the best whole uses a non-best piece. Gluing the best pieces fails too. The longest A → B path is A → C → B, and the longest B → C path is B → A → C. Joined, they give A → C → B → A → C, which repeats vertices and is invalid.

The two halves are not independent: vertices used by one half are off-limits to the other. To make DP work anyway, the state must remember which vertices are already used, a subset of `n` vertices, giving `2^n` states (bitmask DP, see [Classical Problems](#classical-problems)). No polynomial algorithm is known; the problem is NP-hard.

The practical test when reading a problem:

| Question | What It Checks | [Coin Change](https://leetcode.com/problems/coin-change/) |
|---|---|---|
| 1. Can I describe it as a sequence of decisions? | A recurrence exists | "Which coin do I use last?" |
| 2. Does the naive recursion hit the same arguments repeatedly? | Overlapping subproblems | `f(11)` reaches `f(9)` through many coin orders |
| 3. Is the answer a `min`, `max`, `sum` or `count` over sub-answers? | Optimal substructure | `f(a) = 1 + min(f(a - c))` over coins `c` |

Yes to all three means DP. Question 2 decides whether caching helps; question 3 decides whether caching is correct. Merge sort fails question 2 (plain divide and conquer). Longest simple path fails question 3.

## Top-Down Vs Bottom-Up

Both are DP. They evaluate the same recurrence in opposite directions.

**Top-down (memoisation)** starts from the original problem, recurses, and caches each result. It is the naive recursion plus three lines: check the cache, store in the cache, return. The mechanics and the origin of the word are covered in [recursion-backtracking.md](recursion-backtracking.md#memoisation-recursion-with-a-cache). It computes only the states actually reached, which wins when the reachable state space is sparse.

**Bottom-up (tabulation)** enumerates states in dependency order and fills a table, so every cell's inputs are ready when it is computed. Same recurrence body, different control flow:

```go
// Time: O(n) | Space: O(1)
func fib(n int) int {
    if n < 2 {
        return n
    }
    a, b := 0, 1
    for i := 2; i <= n; i++ {
        a, b = b, a+b
    }
    return b
}
```

No recursion means no stack growth and no overflow risk on deep inputs. A contiguous slice also beats a `map` cache on access cost. The catch is that tabulation computes every state, needed or not.

| Situation | Prefer |
|---|---|
| Deriving a solution under time pressure | Top-down |
| Sparse reachable states | Top-down |
| Deep recursion would risk stack growth | Bottom-up |
| You want to shrink memory with a rolling window | Bottom-up |

## Designing The State

A **state** is a snapshot of everything about your progress so far that still matters for the rest of the problem. Anything that does not affect the future can be forgotten. That forgetting is what makes DP fast: many different histories collapse into one state, and that state is solved once.

Choosing the state is the hardest step in any DP problem. Once the state is right, the recurrence usually falls out in one line.

### The Method

1. **Name the decision.** What does the problem make you choose, one step at a time? Rob or skip this house. Which coin to use last. Match these two characters, or drop one.
2. **Ask what the future needs to know.** After making some decisions, what information do the remaining decisions depend on? That information is the state.
3. **Write the recurrence.** Express the answer for a state using the answers for states one decision away.
4. **Set the base cases.** Find the states whose answers are obvious: zero items, empty string, amount 0.
5. **Test sufficiency.** Two different histories that reach the same state must have the same best future. If they don't, the state is missing information.

### What The State Looks Like In Common Problems

| Problem | Decision At Each Step | What The Future Needs To Know | State |
|---|---|---|---|
| [Climbing Stairs](https://leetcode.com/problems/climbing-stairs/) | 1 step or 2 | Which step you are on | `dp[step]` |
| [Coin Change](https://leetcode.com/problems/coin-change/) | Which coin to use | Amount still to make | `dp[amount]` |
| 0-1 Knapsack | Take or skip this item | Which items are left, capacity left | `dp[item][capacity]` |
| [Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence/) | Match or drop a character | Position in each string | `dp[i][j]` |
| [Burst Balloons](https://leetcode.com/problems/burst-balloons/) | Which balloon goes last | Which range is still unsolved | `dp[left][right]` |
| [House Robber III](https://leetcode.com/problems/house-robber-iii/) | Rob or skip this node | Which node, and whether it was robbed | pair returned per node |
| [Travelling Salesman](#bitmask-dp-travelling-salesman) | Which city next | Cities visited, current city | `dp[visited][city]` |

### Worked Example: House Robber

[House Robber](https://leetcode.com/problems/house-robber/): houses in a row hold money, and you cannot rob two neighbours.

**Steps 1 and 2.** The decision at house `i` is rob or skip. Whether you are *allowed* to rob it depends on one fact about the past: was house `i-1` robbed? So a natural state is (house index, was this house robbed).

**Step 3.** Keep two values per house:

- `robbed[i]` = best loot from houses `0..i` when house `i` **is** robbed. The neighbour must have been skipped: `robbed[i] = skipped[i-1] + nums[i]`.
- `skipped[i]` = best loot from houses `0..i` when house `i` is **not** robbed. The neighbour can be either: `skipped[i] = max(robbed[i-1], skipped[i-1])`.

Trace for `nums = [2, 7, 9, 3, 1]`:

| House i | nums[i] | robbed[i] | skipped[i] |
|---|---|---|---|
| 0 | 2 | 2 | 0 |
| 1 | 7 | 0 + 7 = 7 | max(2, 0) = 2 |
| 2 | 9 | 2 + 9 = 11 | max(7, 2) = 7 |
| 3 | 3 | 7 + 3 = 10 | max(11, 7) = 11 |
| 4 | 1 | 11 + 1 = 12 | max(10, 11) = 11 |

The answer is `max(12, 11) = 12`: rob houses 0, 2 and 4.

```go
// Time: O(n) | Space: O(n)
func robWithFlag(nums []int) int {
    n := len(nums)
    // robbed[i]  = best loot from houses 0..i when house i IS robbed
    // skipped[i] = best loot from houses 0..i when house i is NOT robbed
    robbed := make([]int, n)
    skipped := make([]int, n)
    robbed[0] = nums[0]
    for i := 1; i < n; i++ {
        robbed[i] = skipped[i-1] + nums[i]          // neighbour must have been skipped
        skipped[i] = max(robbed[i-1], skipped[i-1]) // neighbour can be either
    }
    return max(robbed[n-1], skipped[n-1])
}
```

**A tighter state.** Look at what `skipped[i]` actually is: `max(robbed[i-1], skipped[i-1])`, which is simply "the best answer for houses `0..i-1`". So both values can be expressed with a single number, `best[i]` = the best loot from the first `i` houses, whatever happened to the last one:

`best[i] = max(best[i-1], best[i-2] + nums[i-1])`

- `best[i-1]`: skip house `i-1`.
- `best[i-2] + nums[i-1]`: rob it, so jump back past its neighbour.

The rob-or-skip flag did not disappear. It moved inside the `max`. This is a common move: once a decision is written as the two arms of a `max` or `min`, it no longer needs its own dimension.

Same trace, one number per step:

| i (houses considered) | best[i] |
|---|---|
| 0 | 0 |
| 1 | 2 |
| 2 | max(2, 0 + 7) = 7 |
| 3 | max(7, 2 + 9) = 11 |
| 4 | max(11, 7 + 3) = 11 |
| 5 | max(11, 11 + 1) = 12 |

```go
// Time: O(n) | Space: O(n)
func robTable(nums []int) int {
    n := len(nums)
    dp := make([]int, n+1) // dp[i] = best loot from the first i houses
    dp[1] = nums[0]
    for i := 2; i <= n; i++ {
        dp[i] = max(dp[i-1], dp[i-2]+nums[i-1])
    }
    return dp[n]
}
```

Both versions are correct. The second has fewer moving parts, and [Space Optimisation](#space-optimisation) shrinks it further.

### When The State Is Missing Information

[Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence/): find the longest strictly increasing subsequence of `nums`.

**First attempt:** `dp[i]` = length of the longest increasing subsequence within the first `i` elements.

Take `nums = [1, 5, 2, 3]`. After the first three elements, `dp[3] = 2`, but two different subsequences give that length: `[1, 5]` and `[1, 2]`. Now the next element, 3, arrives:

- it **cannot** extend `[1, 5]`, because 5 > 3
- it **can** extend `[1, 2]`, giving `[1, 2, 3]`

The stored value 2 does not say which subsequence produced it, so the recurrence cannot decide whether 3 extends it. This is step 5 failing: two histories reach the same state but have different futures.

**Fix: add what the future needs.** To extend a subsequence, you must know its last value. So anchor the state to it: `dp[i]` = length of the longest increasing subsequence that **ends exactly at index `i`**. Now the last value is always `nums[i]`, and any later, larger element can extend it.

| i | nums[i] | Earlier Elements Smaller Than It | dp[i] |
|---|---|---|---|
| 0 | 1 | none | 1 |
| 1 | 5 | 1 (dp=1) | 2 |
| 2 | 2 | 1 (dp=1) | 2 |
| 3 | 3 | 1 (dp=1), 2 (dp=2) | 3 |

The answer is the maximum over all `i`: 3.

### When The State Has Information It Never Uses

The opposite mistake is to add a dimension the recurrence never reads.

[Climbing Stairs](https://leetcode.com/problems/climbing-stairs/) with state `(step, moves taken so far)` gives correct answers. But the number of ways to reach the top from step 7 is the same whether you got there in 4 moves or 6, so the second dimension is never consulted. It only multiplies the work.

The test: if removing a dimension never changes any answer, remove it.

## Space Optimisation

A DP table is scaffolding. The answer is usually one cell, yet the table holds every intermediate result. If each cell only ever reads a few nearby cells, the older cells become useless once you have moved past them, and you can stop storing them.

### Step 1: Draw What Each Cell Reads

Before shrinking anything, write down which cells the recurrence reads. That tells you how much history must stay alive.

| Recurrence Reads | Keep | Example |
|---|---|---|
| The last `k` cells of a 1D table | `k` variables | [Climbing Stairs](https://leetcode.com/problems/climbing-stairs/), [House Robber](https://leetcode.com/problems/house-robber/) |
| The row above, and the cell to the left | one row | [Unique Paths](https://leetcode.com/problems/unique-paths/), [Minimum Path Sum](https://leetcode.com/problems/minimum-path-sum/) |
| The row above, the left cell, and the diagonal | one row plus one saved value | [Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence/), [Edit Distance](https://leetcode.com/problems/edit-distance/) |
| The previous row, at a shifted column | one row, with the loop direction chosen carefully | 0-1 and unbounded knapsack |

### Last Few Cells: Variables Instead Of An Array

[Climbing Stairs](https://leetcode.com/problems/climbing-stairs/) fills `dp[i] = dp[i-1] + dp[i-2]`:

| i | 1 | 2 | 3 | 4 | 5 |
|---|---|---|---|---|---|
| dp[i] | 1 | 2 | 3 | 5 | 8 |

When computing `dp[5]`, only `dp[4]` and `dp[3]` matter; `dp[1]` and `dp[2]` will never be read again. So keep two variables, `a` (two steps back) and `b` (one step back), and slide them forward:

| After Step | a | b |
|---|---|---|
| start (i = 2) | 1 | 2 |
| i = 3 | 2 | 3 |
| i = 4 | 3 | 5 |
| i = 5 | 5 | 8 |

```go
a, b = b, a+b
```

Go evaluates the whole right-hand side before assigning, so `a+b` uses the old `a`. In a language without tuple assignment, you need a temporary variable: `next := a + b; a = b; b = next`.

The same idea turns `robTable` from the previous section into two variables. That is the House Robber solution in the LeetCode section.

### Previous Row: One Row Instead Of A Grid

[Unique Paths](https://leetcode.com/problems/unique-paths/) on a 3 × 3 grid. The full table, where each cell is "above + left":

| | col 0 | col 1 | col 2 |
|---|---|---|---|
| row 0 | 1 | 1 | 1 |
| row 1 | 1 | 2 | 3 |
| row 2 | 1 | 3 | 6 |

Row 2 reads only row 1, and row 1 reads only row 0. So keep one array and overwrite it left to right. At the moment you update `dp[j]`:

- `dp[j]` still holds the **old** value, which is the cell **above**.
- `dp[j-1]` was already updated in this pass, so it holds the cell to the **left**.

So `dp[j] += dp[j-1]` is exactly "above + left":

| Moment | dp[0] | dp[1] | dp[2] |
|---|---|---|---|
| row 0 | 1 | 1 | 1 |
| row 1, after j=1: 1 + 1 | 1 | 2 | 1 |
| row 1, after j=2: 1 + 2 | 1 | 2 | 3 |
| row 2, after j=1: 2 + 1 | 1 | 3 | 3 |
| row 2, after j=2: 3 + 3 | 1 | 3 | 6 |

Memory drops from `m × n` cells to `n`.

### Diagonal Too: One Row Plus A Saved Value

[Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence/) reads three cells: above `(i-1, j)`, left `(i, j-1)`, and diagonal `(i-1, j-1)`.

With one row, the diagonal is a problem. The diagonal for column `j` lived in `dp[j-1]`, but `dp[j-1]` was just overwritten with the current row's value. The fix is to save the old value of each cell in a variable just before overwriting it. That saved value is exactly the diagonal for the next column.

```go
// Time: O(m·n) | Space: O(n)
func lcsOneRow(text1, text2 string) int {
    n := len(text2)
    dp := make([]int, n+1) // dp[j] holds row i-1 until overwritten in this pass
    for i := 1; i <= len(text1); i++ {
        diag := 0 // dp[i-1][j-1], saved before dp[j-1] is overwritten
        for j := 1; j <= n; j++ {
            above := dp[j] // dp[i-1][j]
            if text1[i-1] == text2[j-1] {
                dp[j] = diag + 1
            } else {
                dp[j] = max(above, dp[j-1]) // dp[j-1] is already row i: the left cell
            }
            diag = above // becomes the diagonal for column j+1
        }
    }
    return dp[n]
}
```

### Shifted Column: Loop Direction Matters

Knapsack reads the previous row at column `w - weight`, not just the cell above. With one row, whether that cell still holds the previous row's value depends on which direction you loop. [Classical Problems](#classical-problems) traces both directions: downwards keeps each item to one use, and upwards allows repeats.

### When You Cannot Shrink

- **The look-back distance is not fixed.** [Coin Change](https://leetcode.com/problems/coin-change/) reads `dp[amount - coin]`, and a coin worth 1,000 reaches 1,000 cells back. The whole 1D array has to stay.
- **Any shorter range can be read.** Interval DP ([Burst Balloons](https://leetcode.com/problems/burst-balloons/)) reads arbitrary sub-ranges, so no row can be thrown away.

### What You Give Up

A shrunk table keeps the final *number* but loses the path that produced it. To print the actual common subsequence, or list which houses were robbed, you walk backwards through the full table. If the problem asks for the choices themselves, keep the full table.

Space optimisation works only bottom-up. Recursion visits states in an order you don't control, so there is no "older row" to throw away. Write and test the full-table version first, then shrink it; rolling arrays are where overwrite-before-read bugs hide.

## Reversing The Direction

A state must be sufficient: any two ways of arriving at it must have the same best continuation, so DP can keep one answer and discard the history. Sometimes the natural forward state is not sufficient. Two arrivals look different, and which one is better depends on what comes later. DP keeps only one, so it may keep the wrong one.

The fix is to redefine the state so the future stops mattering:

- "What I've achieved so far" becomes "what I need from here on" ([Dungeon Game](https://leetcode.com/problems/dungeon-game/)).
- "What happens first" becomes "what happens last" ([Burst Balloons](https://leetcode.com/problems/burst-balloons/)).
- "Answer for this input" becomes "input coverable with this budget" ([Super Egg Drop](https://leetcode.com/problems/super-egg-drop/)).

### Prefix To Suffix: Dungeon Game

Minimum starting health, where health must stay at least 1. Going forward, two paths reach the same cell X:

| Path To X | Cells Passed | Start Health Needed | Health At X |
|---|---|---|---|
| P1 | -5, then +10 | 6 | start + 5 |
| P2 | -1, then -1 | 3 | start - 2 |

Which is better depends on the next cell:

- **Next cell is 0:** P1 needs 6, P2 needs 3. P2 wins.
- **Next cell is -20:** P1 needs `start + 5 - 20 ≥ 1`, so 16. P2 needs `start - 2 - 20 ≥ 1`, so 23. P1 wins.

The forward DP must choose a winner at X before seeing the future, so it cannot be correct.

Backwards, define `need(X)` as the health required on entering X to survive from X to the end. It depends only on cells after X, and every path arriving at X needs exactly that amount:

`need(X) = max(1, min(need(right), need(down)) - value(X))`

### First To Last: Burst Balloons

Take `[3, 1, 5]`, padded with 1 on each end.

**First burst.** If 1 bursts first (earning `3 × 1 × 5`), 3 and 5 become neighbours. The payout for 3 now depends on whether 5 is still standing, so the left side depends on the right side's order. The halves are not independent.

**Last burst.** If 1 bursts last, it stands until the end, so:

- 3 always has neighbours (left wall, 1).
- 5 always has neighbours (1, right wall).

Balloon 1 is a fixed wall that splits the range into two independent subproblems. When it finally bursts, its neighbours are exactly the range boundaries.

### Answer To Budget: Super Egg Drop

The direct question, "fewest moves for `k` eggs and `n` floors", must try every first-drop floor for each state: O(n) work per state.

Invert it: `g(k, m)` is the number of floors fully decidable with `k` eggs and `m` moves. One drop splits the work:

- **Breaks:** `k-1` eggs and `m-1` moves cover `g(k-1, m-1)` floors below.
- **Survives:** `k` eggs and `m-1` moves cover `g(k, m-1)` floors above.
- **The dropped floor** itself is decided too.

`g(k, m) = g(k-1, m-1) + g(k, m-1) + 1`, one addition and no search.

| Moves m | Floors g(2, m) |
|---|---|
| 1 | 1 |
| 2 | 3 |
| 3 | 6 |
| 4 | 10 |

Check `g(2, 3) = 6`: one egg with two moves covers 2 floors, so drop first from floor 3. If it breaks, test floors 1 and 2 with the last egg. If it survives, two eggs and two moves cover floors 4, 5 and 6. For 6 floors, the answer is 3 moves.

### How To Spot It

You need a reversal when the forward recurrence leaves you thinking "I can't pick the best way into this state until I know what's ahead". At that point, look for the reframing that makes the future irrelevant to the state:

- a suffix requirement instead of a prefix accumulation
- the last event instead of the first
- the resource budget instead of the answer

## Classical Problems

These three are not LeetCode problems, but they are the templates many LeetCode problems are built on. Each one is explained in the same order: the problem, why brute force is slow, the state, the recurrence, a traced example, then code.

### 0-1 Knapsack

**The problem.** You have a bag that holds at most `W` kilograms and a set of items, each with a weight and a value. Each item can be taken at most once (that is the "0-1": take 0 or 1 copy). Maximise the total value without exceeding `W`.

Example, with `W = 4`:

| Item | Weight | Value |
|---|---|---|
| 1 | 1 | 15 |
| 2 | 3 | 20 |
| 3 | 4 | 30 |

The best choice is items 1 and 2: weight 4, value 35. Taking item 3 alone also fills the bag, but is worth only 30.

**Brute force.** Every item is either in or out, so there are `2^n` subsets. Checking all of them is O(2^n · n).

**The decision.** Look at items one at a time and decide skip or take. That is all the problem ever asks you to decide.

**The state.** After deciding about the first `i` items, the only things that matter for the remaining items are:

- which items are left (everything after `i`)
- how much capacity is left

So `dp[i][w]` = the best value using only the first `i` items with a bag of capacity `w`.

**The recurrence.** For item `i` with weight `wt` and value `val`:

- **Skip it:** the answer is whatever the first `i-1` items could do with the same capacity: `dp[i-1][w]`.
- **Take it** (only if `wt ≤ w`): you gain `val`, and the first `i-1` items must fit in what is left: `dp[i-1][w-wt] + val`.

`dp[i][w] = max(dp[i-1][w], dp[i-1][w-wt] + val)`

Both branches read row `i-1`, never row `i`. That is what enforces "at most once": after taking item `i`, you only look at earlier items.

**Traced table.** Rows are "items considered so far", columns are capacity 0 to 4. Row 0 is all zeros (no items, no value).

| Items Considered | w=0 | w=1 | w=2 | w=3 | w=4 |
|---|---|---|---|---|---|
| none | 0 | 0 | 0 | 0 | 0 |
| item 1 (wt 1, val 15) | 0 | 15 | 15 | 15 | 15 |
| + item 2 (wt 3, val 20) | 0 | 15 | 15 | 20 | 35 |
| + item 3 (wt 4, val 30) | 0 | 15 | 15 | 20 | 35 |

Two cells worth checking by hand:

- **Row "+ item 2", w=4:** skip gives 15 (the cell above). Take gives `20 + dp[item 1 row][4-3] = 20 + 15 = 35`. The max is 35.
- **Row "+ item 3", w=4:** skip gives 35 (the cell above). Take gives `30 + dp[previous row][0] = 30 + 0 = 30`. The max stays 35.

The answer is the bottom-right cell, 35.

**Code, full table.**

```go
// Time: O(n·W) | Space: O(n·W)
func knapsack01Table(weights, values []int, W int) int {
    n := len(weights)
    dp := make([][]int, n+1) // dp[i][w] = best value using the first i items with capacity w
    for i := range dp {
        dp[i] = make([]int, W+1)
    }
    for i := 1; i <= n; i++ {
        wt, val := weights[i-1], values[i-1]
        for w := 0; w <= W; w++ {
            dp[i][w] = dp[i-1][w] // skip item i
            if wt <= w {
                dp[i][w] = max(dp[i][w], dp[i-1][w-wt]+val) // take item i
            }
        }
    }
    return dp[n][W]
}
```

**Shrinking to one row.** Each row reads only the row above it, so one array can be overwritten in place. The catch is the take branch, which reads `dp[w-wt]` from the previous row. If you loop `w` upwards, `dp[w-wt]` may already hold the current row's value.

Trace it with item 1 (weight 1, value 15), starting from an all-zero array:

- **Looping upwards (wrong):** `dp[1] = 0 + 15 = 15`. Then `dp[2] = dp[1] + 15 = 30`, but `dp[1]` was just updated to include item 1, so item 1 has now been taken twice.
- **Looping downwards (right):** `dp[4] = dp[3] + 15 = 15`, `dp[3] = dp[2] + 15 = 15`, and so on. Every read hits a cell not yet touched in this pass, so it still holds the previous row's value.

```go
// Time: O(n·W) | Space: O(W)
func knapsack01(weights, values []int, W int) int {
    dp := make([]int, W+1)
    for i := range weights {
        for w := W; w >= weights[i]; w-- { // downwards: dp[w-wt] still holds the previous row
            dp[w] = max(dp[w], dp[w-weights[i]]+values[i])
        }
    }
    return dp[W]
}
```

**Why O(n·W) is not "polynomial".** The running time grows with the *value* of `W`, not the length of the input. Each extra digit in `W` multiplies the work by 10, so the time is exponential in the number of digits needed to write `W`. This is called pseudo-polynomial. It is fast when `W` is small (say up to 10^5) and useless when `W` is 10^18.

**LeetCode problems built on it:** [Partition Equal Subset Sum](https://leetcode.com/problems/partition-equal-subset-sum/) (can some items fill the bag exactly to half the total?) and [Target Sum](https://leetcode.com/problems/target-sum/) (how many ways can items fill the bag exactly?).

### Unbounded Knapsack

**The problem.** Same bag, same items, but now each item can be taken any number of times. With the same example and `W = 4`, the best choice is four copies of item 1: value 60.

**The only change.** After taking item `i`, you may take it again, so the take branch stays in row `i` instead of dropping to `i-1`:

`dp[i][w] = max(dp[i-1][w], dp[i][w-wt] + val)`

In the one-row version, this is exactly the "upwards" loop that was a bug for 0-1. There, reading an already-updated `dp[w-wt]` meant "item used twice". Here, that is the intended behaviour.

```go
// Time: O(n·W) | Space: O(W)
func knapsackUnbounded(weights, values []int, W int) int {
    dp := make([]int, W+1)
    for i := range weights {
        for w := weights[i]; w <= W; w++ { // upwards: dp[w-wt] may already include item i
            dp[w] = max(dp[w], dp[w-weights[i]]+values[i])
        }
    }
    return dp[W]
}
```

Remember it as: **downwards = each item once, upwards = unlimited copies.**

**LeetCode problems built on it:** [Coin Change](https://leetcode.com/problems/coin-change/) (coins are items with unlimited copies; minimise the count instead of maximising value) and [Coin Change II](https://leetcode.com/problems/coin-change-ii/) (count the ways instead).

### Matrix Chain Multiplication

**Background.** Multiplying a `p × q` matrix by a `q × r` matrix takes `p · q · r` scalar multiplications and produces a `p × r` matrix. Matrix multiplication is associative, so `(AB)C` and `A(BC)` give the same result, but they can cost very different amounts.

**Example.** `A` is 10 × 30, `B` is 30 × 5, `C` is 5 × 60.

| Order | First Step | Second Step | Total |
|---|---|---|---|
| `(AB)C` | `AB`: 10·30·5 = 1,500, gives 10 × 5 | `(AB)C`: 10·5·60 = 3,000 | 4,500 |
| `A(BC)` | `BC`: 30·5·60 = 9,000, gives 30 × 60 | `A(BC)`: 10·30·60 = 18,000 | 27,000 |

Same answer, six times the work. **The problem:** given a chain of matrices, find the bracketing with the fewest scalar multiplications.

**Brute force.** The number of ways to bracket `n` matrices is the Catalan number, which grows roughly like `4^n`. Trying them all is exponential.

**The decision.** Every bracketing has a *last* multiplication: it splits the chain into a left part and a right part, each fully computed first. Choose where that split happens.

**The state.** Store the dimensions in an array `p`, where matrix `i` is `p[i-1] × p[i]`. For the example, `p = [10, 30, 5, 60]`. Then `dp[i][j]` = the cheapest cost to compute matrices `i` through `j`.

**The recurrence.** Try every split point `k` between `i` and `j-1`:

- the left part (matrices `i..k`) costs `dp[i][k]` and produces a `p[i-1] × p[k]` matrix
- the right part (matrices `k+1..j`) costs `dp[k+1][j]` and produces a `p[k] × p[j]` matrix
- multiplying the two results costs `p[i-1] · p[k] · p[j]`

`dp[i][j] = min over k of dp[i][k] + dp[k+1][j] + p[i-1]·p[k]·p[j]`

Base case: `dp[i][i] = 0`, since a single matrix needs no multiplication.

**Fill order.** `dp[i][j]` needs shorter ranges, so fill all ranges of length 1, then length 2, then length 3, and so on. In the table, that means one diagonal at a time, moving away from the main diagonal:

```
         j=1    j=2    j=3
i=1       0    1500   4500
i=2              0    9000
i=3                     0
```

- **Length 1:** the diagonal of zeros.
- **Length 2:** `dp[1][2] = 10·30·5 = 1500`, `dp[2][3] = 30·5·60 = 9000`.
- **Length 3:** `dp[1][3]` tries both splits:
  - `k=1`, meaning `A(BC)`: `0 + 9000 + 10·30·60 = 27000`
  - `k=2`, meaning `(AB)C`: `1500 + 0 + 10·5·60 = 4500`
  - The minimum is 4500.

**Code.**

```go
// Time: O(n³) | Space: O(n²)
func matrixChainOrder(p []int) int {
    n := len(p) - 1 // number of matrices
    dp := make([][]int, n+1)
    for i := range dp {
        dp[i] = make([]int, n+1)
    }
    for length := 2; length <= n; length++ {
        for i := 1; i+length-1 <= n; i++ {
            j := i + length - 1
            dp[i][j] = math.MaxInt
            for k := i; k < j; k++ { // k = where the last multiplication splits the chain
                dp[i][j] = min(dp[i][j], dp[i][k]+dp[k+1][j]+p[i-1]*p[k]*p[j])
            }
        }
    }
    return dp[1][n]
}
```

**Why O(n³).** There are O(n²) ranges `(i, j)`, and each tries up to `n` split points. That third loop over the split point is the signature of every interval DP.

**LeetCode problem built on it:** [Burst Balloons](https://leetcode.com/problems/burst-balloons/) has the same shape: a range, a split point, and a cost that depends on the boundaries.

### Bitmask DP: Travelling Salesman

**The problem.** Visit every one of `n` cities exactly once, starting and ending at city 0, with the smallest total distance.

**Brute force.** Try every order of the other `n-1` cities: `(n-1)!` tours. For 15 cities that is about 87 billion.

**Why a normal state is not enough.** To decide where to go next, you must know which cities are already visited, not just how many. Two paths ending at the same city with different visited sets have different futures. So the state must include the visited set itself.

**Storing a set as a number (a bitmask).** Use bit `i` of an integer to mean "city `i` is visited". With 4 cities:

| Visited Set | Bits (City 3 2 1 0) | Integer |
|---|---|---|
| {0} | 0001 | 1 |
| {0, 2} | 0101 | 5 |
| {0, 1, 2, 3} | 1111 | 15 |

The name comes from hardware: a mask is a pattern of bits used to pick out ("mask") certain positions. The operations you need are one-liners:

| Operation | Go |
|---|---|
| Is city `i` in `mask`? | `mask&(1<<i) != 0` |
| Add city `i` | `mask \| 1<<i` |
| Remove city `i` | `mask &^ (1<<i)` |

**The state.** `dp[mask][i]` = the shortest path that starts at city 0, visits exactly the cities in `mask`, and is currently at city `i`.

**The transition.** From state `(mask, i)`, move to any unvisited city `j`. That reaches state `(mask with j added, j)` at cost `dp[mask][i] + dist[i][j]`. Keep the minimum for each target state.

**The answer.** Once every city is visited (`mask` = all ones), add the trip back to city 0 and take the best over all final cities.

**Code.**

```go
// Time: O(2^n · n²) | Space: O(2^n · n)
func tsp(dist [][]int) int {
    n := len(dist)
    full := 1<<n - 1
    const inf = math.MaxInt32
    dp := make([][]int, 1<<n) // dp[mask][i] = cheapest path from city 0 visiting exactly mask, ending at i
    for m := range dp {
        dp[m] = make([]int, n)
        for i := range dp[m] {
            dp[m][i] = inf
        }
    }
    dp[1][0] = 0 // only city 0 visited, standing at city 0
    for mask := 1; mask <= full; mask++ {
        if mask&1 == 0 {
            continue // every path starts at city 0
        }
        for i := 0; i < n; i++ {
            if mask&(1<<i) == 0 || dp[mask][i] == inf {
                continue
            }
            for j := 0; j < n; j++ {
                if mask&(1<<j) != 0 {
                    continue // already visited
                }
                next := mask | 1<<j
                dp[next][j] = min(dp[next][j], dp[mask][i]+dist[i][j])
            }
        }
    }
    best := inf
    for i := 1; i < n; i++ {
        best = min(best, dp[full][i]+dist[i][0]) // close the tour
    }
    return best
}
```

Looping `mask` upwards is a valid fill order. Adding a city always makes the integer larger, so every state is final before anything reads it.

**Why it only works for small `n`.** There are `2^n · n` states, and each tries `n` next cities. For `n = 20`, that is about 400 million steps, which is near the practical limit. Still, `2^n · n²` beats `(n-1)!` by a huge margin: for 15 cities, about 7 million versus 87 billion. This algorithm is due to Held and Karp (1962).

**LeetCode problem built on it:** LeetCode has no problem named Travelling Salesman. [Find the Shortest Superstring](https://leetcode.com/problems/find-the-shortest-superstring/) is the same algorithm in disguise: words are the cities, and the overlap between consecutive words plays the role of distance.

## DP On Trees

Every DP so far has filled an array or a grid, where the fill order came from the indices: left to right, row by row, or by length. A tree has no indices. The tree itself is the order.

### What Changes On A Tree

**The subproblem is a subtree.** "The best answer for the subtree rooted at this node" plays the role that `dp[i]` played for arrays.

**Children are independent.** A node's left and right subtrees share no nodes, so each can be solved on its own and the results combined at the parent. Optimal substructure comes for free from the tree's shape.

**The fill order is post-order DFS.** A node needs its children's answers before it can compute its own, so recurse into the children first and combine on the way back up. Each node is processed once, so the whole DP is O(n).

**There is no table.** Each node's answer is the return value of the DFS call for that node, handed straight to its parent. The recursion stack does the job that the `dp` array did.

### Steps For A Tree DP

1. **Subproblem:** the subtree rooted at a node.
2. **Ask what the parent needs to know** about this subtree to make its own decision. That is the return value.
3. **If the parent's choice depends on the child's own choice**, a single number is not enough. Return one value per situation the node can be in, usually as a small tuple.
4. **Combine** the children's returned values at the node.
5. **Read the answer at the root.** Sometimes the root needs special handling, because it has no parent.

### Worked Example: House Robber III

[House Robber III](https://leetcode.com/problems/house-robber-iii/): houses form a binary tree, and you cannot rob a node and its direct child.

```
        A(3)
       /    \
     B(2)   C(3)
        \      \
        D(3)   E(1)
```

Letters name the nodes; the numbers in brackets are the money.

**Step 2: what does the parent need?** A parent deciding whether to rob itself must know whether its child was robbed, because robbing both is not allowed. "The best loot in my subtree" alone does not say that.

**Step 3: return one value per situation.** Each node returns a pair:

- `with` = best loot in this subtree **if this node is robbed**. Its children must not be robbed: `node.Val + left.without + right.without`.
- `without` = best loot in this subtree **if this node is not robbed**. Each child picks whichever is better for it: `max(left.with, left.without) + max(right.with, right.without)`.

A missing child returns `(0, 0)`.

**Step 4: trace bottom-up.** Children are processed before their parent:

| Node | with | without |
|---|---|---|
| D (leaf, 3) | 3 | 0 |
| B (2, child D) | 2 + 0 = 2 | max(3, 0) = 3 |
| E (leaf, 1) | 1 | 0 |
| C (3, child E) | 3 + 0 = 3 | max(1, 0) = 1 |
| A (3, children B and C) | 3 + 3 + 1 = 7 | max(2, 3) + max(3, 1) = 6 |

**Step 5: answer at the root.** `max(7, 6) = 7`: rob A, D and E.

**Why the pair matters.** A natural first attempt returns one number: `best(node) = max(node.Val + best(grandchildren), best(children))`. That is correct, but every grandchild gets reached twice, once from its parent and once from its grandparent, and the repeats multiply down the tree. That is overlapping subproblems again. Returning the pair means each node is visited exactly once.

### When The Parent Must Finish The Job: Binary Tree Cameras

[Binary Tree Cameras](https://leetcode.com/problems/binary-tree-cameras/): a camera on a node watches its parent, itself and its children. Find the fewest cameras that watch every node.

The new difficulty is that a node can be watched by its parent, and the parent has not been decided yet when the node is processed. So a node is allowed to stay unwatched, as long as its parent promises to fix it. That gives three situations:

| Situation | Meaning | How To Get It |
|---|---|---|
| uncovered | everything below is watched, this node is not | both children watched, neither has a camera |
| covered | everything watched, no camera here | at least one child has a camera |
| camera | everything watched, camera here | children can be in any situation |

A missing child counts as watched (there is nothing to watch), but it cannot hold a camera.

Trace on this tree:

```
      R
      |
      M
     / \
    P   Q
```

R is the root, M its only child, and P and Q are leaves under M.

| Node | uncovered | covered | camera |
|---|---|---|---|
| P (leaf) | 0: nothing below, P waits for its parent | impossible: P has no child to watch it | 1 |
| Q (leaf) | 0 | impossible | 1 |
| M (children P, Q) | impossible: a leaf can only be watched by its own camera or its parent's | 2: a camera on P watches M, but Q still needs its own camera | 1 + 0 + 0 = 1: P and Q may stay uncovered, M watches them |
| R (child M) | 2: M covered, no camera on M | 1: camera on M | 1 + 1 = 2: camera here, plus M's cheapest |

**Answer at the root.** The root has no parent, so "uncovered" is not allowed there. The answer is `min(covered, camera) = min(1, 2) = 1`: one camera on M watches R, P and Q.

### Return Value Versus Global Answer

Sometimes the answer is not "the best for the root's subtree" but something that bends through a middle node, like the longest path in a tree. Then the DFS returns one thing to the parent (the best path going down) and updates a global variable with the other (the best path bending at this node). [tree-dfs.md](../patterns/tree-dfs.md#the-two-answer-split) covers that pattern. Here, the returned values are the answer itself.

## Recognizing The Pattern

Recognition happens in two steps. First decide whether the problem is DP at all. Then work out what the state looks like, which mostly follows from the shape of the input.

### Is It A DP Problem?

The wording gives the first hint. DP problems ask for a single number or a yes/no about the best way to make a series of choices:

- "number of ways to..."
- "minimum cost", "maximum profit", "fewest steps"
- "longest" or "shortest" subsequence, path or partition
- "is it possible to..."

That alone is not enough. Greedy problems use the same words. The deciding question is whether **a choice now changes which choices are good later**. Robbing a house rules out its neighbour. Spending capacity on one item leaves less for the others. When choices interact like this, you cannot commit to one locally and move on, so you need to compare alternatives, and DP does that without re-solving shared subproblems.

Two kinds of problem look similar but are not DP:

- **"Return all the ..."** (all subsets, all paths). The output itself is exponential, so there is nothing to compress. That is backtracking.
- **A local choice is provably safe** (always take the earliest-ending meeting). That is greedy, which is simpler and faster when it applies.

The constraints are the second hint, because they tell you how slow a solution may be. A common rule of thumb is that judges allow around 10^8 simple operations:

| Input Size | Affordable Time | Usual DP Shape |
|---|---|---|
| n ≤ 20 | O(2^n · n²) | a bitmask over a small set |
| n ≤ 500 | O(n³) | ranges with a split point |
| n ≤ 5,000 | O(n²) | two indices, or each element looking at all earlier ones |
| n ≤ 10^5 | O(n) or O(n log n) | one index looking back a few steps |

### What Does The State Look Like?

The state usually mirrors the input. Find the input below that matches your problem, and the state comes with it.

**One sequence, or one number, processed left to right.** The state is a position, `dp[i]`: the best answer (or count) for everything up to `i`. Usually `dp[i]` looks back one or two steps, as in [Climbing Stairs](https://leetcode.com/problems/climbing-stairs/) and [House Robber](https://leetcode.com/problems/house-robber/). If any earlier element can come right before the current one, it looks at all of them, as in [Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence/).

**A grid you cross by moving only right or down.** The state is a cell, `dp[row][col]`, and a cell can only be entered from above or from the left. See [Unique Paths](https://leetcode.com/problems/unique-paths/) and [Minimum Path Sum](https://leetcode.com/problems/minimum-path-sum/).

**Items to pick, and a total to hit or stay under.** The state is the remaining budget, `dp[capacity]`, with the items as the outer loop. This is knapsack, and the one question that matters is whether an item can be reused. Once each: [Partition Equal Subset Sum](https://leetcode.com/problems/partition-equal-subset-sum/). Unlimited: [Coin Change](https://leetcode.com/problems/coin-change/).

**Two strings to compare or convert.** The state is a pair of prefix lengths, `dp[i][j]`, and each step compares the last characters of the two prefixes. See [Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence/) and [Edit Distance](https://leetcode.com/problems/edit-distance/).

**A range that gets split or shrinks from the ends.** The state is the range itself, `dp[i][j]`, filled from short ranges to long ones. Look for words like merge, burst, or palindrome. See [Burst Balloons](https://leetcode.com/problems/burst-balloons/) and [Longest Palindromic Subsequence](https://leetcode.com/problems/longest-palindromic-subsequence/).

**A tree.** Each node computes its answer from its children's answers and hands it to its parent. When a node's choice restricts its neighbours, return one value per situation the node can be in. See [DP On Trees](#dp-on-trees) and [House Robber III](https://leetcode.com/problems/house-robber-iii/).

**A small set where each item is used exactly once.** The state must remember which items are used, so it stores them as bits, `dp[mask]`. The tell-tale sign is `n ≤ 20`. See [Find the Shortest Superstring](https://leetcode.com/problems/find-the-shortest-superstring/).

### Seeing Through A Disguise

Harder problems rarely look like any of these shapes at first. The work is translating the problem until one appears. Two examples from this file:

**[Target Sum](https://leetcode.com/problems/target-sum/).** "Put `+` or `-` before each number and count the ways to reach the target." On the surface, this is about signs, not items and a budget. But the numbers given `+` form one group with sum `P`, and the rest form a group with sum `N`. Since `P - N = target` and `P + N = total`, it follows that `P = (total + target) / 2`. The question becomes: how many subsets sum to `P`? That is items with a budget, each used once: 0-1 knapsack.

**[Find the Shortest Superstring](https://leetcode.com/problems/find-the-shortest-superstring/).** "Return the shortest string containing every word." On the surface, this is a string problem. But the final length is the total length of all words minus the overlaps between neighbours. So the real question is which order of words gives the most total overlap. Choosing an order where each word is used once, with at most 12 words, is a small set: bitmask DP.

In both cases, the useful move was the same. Instead of asking what the problem talks about, ask what is actually being chosen, and what the remaining choices depend on.

## Go Template Code

Top-down, when deriving from the recursion:

```go
func solveTopDown(n int) int {
    memo := make([]int, n+1)
    seen := make([]bool, n+1)
    var f func(int) int
    f = func(i int) int {
        if i <= 1 { // base case
            return i
        }
        if seen[i] {
            return memo[i]
        }
        seen[i] = true
        memo[i] = f(i-1) + f(i-2) // the recurrence
        return memo[i]
    }
    return f(n)
}
```

A separate `seen` slice avoids the bug where a legitimately computed `0` is mistaken for "not computed". Use a `map` instead of slices only when states are sparse or not small integers.

Bottom-up over two indices:

```go
func solveBottomUp(a, b string) int {
    m, n := len(a), len(b)
    dp := make([][]int, m+1) // dp[i][j] covers prefixes a[:i], b[:j]
    for i := range dp {
        dp[i] = make([]int, n+1)
    }
    // base cases: row 0 and column 0 (empty prefixes)
    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if a[i-1] == b[j-1] { // string index is always one behind the prefix length
                dp[i][j] = dp[i-1][j-1] + 1
            } else {
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
            }
        }
    }
    return dp[m][n]
}
```

Indexing by prefix length (`dp[0]` is the empty prefix) keeps base cases as a plain row and column instead of special-cased negative indices.

## Where JavaScript Differs

- Build tables with `Array.from({ length: m + 1 }, () => Array(n + 1).fill(0))`. `Array(m + 1).fill(Array(n + 1).fill(0))` shares one row object across every row, so writing one cell writes the whole column.
- `Infinity` is a safe sentinel: `Infinity + 1 === Infinity`, so `min` comparisons need no guard. In Go, adding to `math.MaxInt` wraps to a negative number, which is why the Go solutions use `math.MaxInt32` with an explicit check.
- Top-down recursion in JS has a much smaller default stack than Go's growable goroutine stacks, so deep memoised recursion overflows sooner. Prefer bottom-up for large `n`.

## LeetCode Problems

### 1. Fibonacci Number - [#509 (Easy)](https://leetcode.com/problems/fibonacci-number/)

Return `F(n)`, where `F(0) = 0`, `F(1) = 1` and `F(n) = F(n-1) + F(n-2)`.

[recursion-backtracking.md](recursion-backtracking.md) uses this recurrence to show memoisation; here the point is bottom-up evaluation in O(1) space.

<details>
<summary>Brute Force</summary>

Evaluate the recurrence directly with recursion. Each call makes two more, and values like `F(2)` are recomputed many times.

Time: O(2^n). Space: O(n) recursion depth.
</details>

<details>
<summary>Hint 1</summary>

Compute the values from the bottom up, starting at `F(0)` and `F(1)`, so each one is computed exactly once.
</details>

<details>
<summary>Hint 2</summary>

Each value needs only the two before it. Two variables are enough.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func fib(n int) int {
    if n < 2 {
        return n
    }
    a, b := 0, 1 // F(i-2), F(i-1)
    for i := 2; i <= n; i++ {
        a, b = b, a+b
    }
    return b
}
```
</details>

### 2. Climbing Stairs - [#70 (Easy)](https://leetcode.com/problems/climbing-stairs/)

You climb a staircase of `n` steps, one or two steps at a time. Count the distinct ways to reach the top.

<details>
<summary>Brute Force</summary>

Recurse on the last move: `ways(n) = ways(n-1) + ways(n-2)`, with no caching. Each call spawns two more, and the same `ways(k)` is recomputed many times.

Time: O(2^n). Space: O(n) recursion depth.
</details>

<details>
<summary>Hint 1</summary>

Think only about the final move. It was either a single step from `n-1` or a double step from `n-2`.
</details>

<details>
<summary>Hint 2</summary>

The count for `n` needs only the counts for the two steps below it. Two variables are enough. One way to stand on step 1, two ways to stand on step 2.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func climbStairs(n int) int {
    if n <= 2 {
        return n
    }
    a, b := 1, 2
    for i := 3; i <= n; i++ {
        a, b = b, a+b
    }
    return b
}
```

The recurrence is Fibonacci; only the base cases differ, because this counts paths rather than sequence terms.
</details>

### 3. Min Cost Climbing Stairs - [#746 (Easy)](https://leetcode.com/problems/min-cost-climbing-stairs/)

`cost[i]` is the price of using stair `i`: once you pay it, you can climb one or two stairs from there. You may start on stair 0 or stair 1. Return the minimum cost to reach the top, the position just past the last stair.

<details>
<summary>Brute Force</summary>

Recursively try both moves from every stair and take the cheaper total. Both branches are explored at every level with no reuse.

Time: O(2^n). Space: O(n) recursion depth.
</details>

<details>
<summary>Hint 1</summary>

Let the state be the cheapest total cost to stand on stair `i`, including paying for it. You got there from stair `i-1` or stair `i-2`, so add `cost[i]` to the cheaper of those two.
</details>

<details>
<summary>Hint 2</summary>

Standing on stair 0 costs `cost[0]` and standing on stair 1 costs `cost[1]`, because you can start on either. The top is one or two steps from the last two stairs, so the answer is the cheaper of those two totals.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func minCostClimbingStairs(cost []int) int {
    a, b := cost[0], cost[1] // cheapest total to stand on stair i-2 and i-1
    for i := 2; i < len(cost); i++ {
        a, b = b, cost[i]+min(a, b)
    }
    return min(a, b) // the top is one or two steps from the last two stairs
}
```
</details>

### 4. House Robber - [#198 (Medium)](https://leetcode.com/problems/house-robber/)

Houses in a row hold `nums[i]` money. Robbing two adjacent houses trips the alarm. Return the maximum you can rob.

<details>
<summary>Brute Force</summary>

At each house, branch on rob or skip, and forbid robbing a house right after a robbed one. Every valid subset is explored.

Time: O(2^n). Space: O(n) recursion depth.
</details>

<details>
<summary>Hint 1</summary>

If you rob house `i`, the house before it was untouched. If you skip it, the best answer is whatever was best one house earlier.
</details>

<details>
<summary>Hint 2</summary>

Best up to `i` is the larger of "best up to `i-1`" and "best up to `i-2` plus this house". Keep two running values.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func rob(nums []int) int {
    prev2, prev1 := 0, 0 // best loot up to i-2 and i-1
    for _, x := range nums {
        prev2, prev1 = prev1, max(prev1, prev2+x)
    }
    return prev1
}
```
</details>

### 5. House Robber II - [#213 (Medium)](https://leetcode.com/problems/house-robber-ii/)

Same as [House Robber](https://leetcode.com/problems/house-robber/), but the houses form a circle, so the first and last are adjacent.

<details>
<summary>Brute Force</summary>

Enumerate every subset of houses, reject any with two neighbours (including the first and last pair), and keep the best sum.

Time: O(2^n · n). Space: O(n).
</details>

<details>
<summary>Hint 1</summary>

The circle adds exactly one conflict: the first and last houses. Any valid plan leaves at least one of them untouched.
</details>

<details>
<summary>Hint 2</summary>

Solve the straight-line version twice, once without the first house and once without the last, and take the larger. A single house needs its own case, because both slices would be empty.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1)
func robCircular(nums []int) int {
    n := len(nums)
    if n == 1 {
        return nums[0]
    }
    return max(rob(nums[:n-1]), rob(nums[1:])) // rob from House Robber
}
```
</details>

### 6. Coin Change - [#322 (Medium)](https://leetcode.com/problems/coin-change/)

Given coin denominations and an `amount`, return the fewest coins that make the amount, or `-1` if impossible. Each coin may be used any number of times.

<details>
<summary>Brute Force</summary>

Recursively try every coin as the next one and recurse on the remaining amount, keeping the minimum count. With `k` coins and smallest coin `c`, the recursion is up to `amount / c` deep with `k` branches per level.

Time: O(k^(amount/c)). Space: O(amount/c) recursion depth.
</details>

<details>
<summary>Hint 1</summary>

Greedy (largest coin first) fails: with coins `[1, 3, 4]` and amount 6, greedy picks `4+1+1` but `3+3` is better. Consider every coin as the last one used.
</details>

<details>
<summary>Hint 2</summary>

Fill answers for every amount from 1 upward. A sentinel value larger than any real answer marks "impossible". Check for the sentinel before adding 1, or the addition overflows.
</details>

<details>
<summary>Solution 1: Amount Outer (Go)</summary>

For each amount, try every coin as the last one used. This reads exactly like the recurrence `dp[a] = 1 + min over coins c of dp[a-c]`.

```go
// Time: O(amount · len(coins)) | Space: O(amount)
func coinChange(coins []int, amount int) int {
    dp := make([]int, amount+1) // dp[a] = fewest coins summing to a
    for a := 1; a <= amount; a++ {
        dp[a] = math.MaxInt
        for _, c := range coins {
            if c <= a && dp[a-c] != math.MaxInt { // check before adding 1: MaxInt+1 overflows
                dp[a] = min(dp[a], dp[a-c]+1)
            }
        }
    }
    if dp[amount] == math.MaxInt {
        return -1
    }
    return dp[amount]
}
```
</details>

<details>
<summary>Solution 2: Coins Outer (Go)</summary>

Process one coin at a time. After the pass for a coin, `dp[v]` is the fewest coins making `v` using only the coins seen so far. The inner loop runs upwards, so `dp[v-coin]` may already include this coin, which allows reuse. This is the unbounded knapsack loop from [Classical Problems](#classical-problems), with `min(..., +1)` in place of `max(..., +value)`.

```go
// Time: O(amount · len(coins)) | Space: O(amount)
func coinChange(coins []int, amount int) int {
    dp := make([]int, amount+1) // dp[v] = fewest coins summing to v
    for v := 1; v <= amount; v++ {
        dp[v] = math.MaxInt
    }
    for _, coin := range coins {
        for v := coin; v <= amount; v++ { // upwards: each coin can be reused
            if dp[v-coin] != math.MaxInt { // check before adding 1: MaxInt+1 overflows
                dp[v] = min(dp[v], dp[v-coin]+1)
            }
        }
    }
    if dp[amount] == math.MaxInt {
        return -1
    }
    return dp[amount]
}
```
</details>

**Which loop goes outside does not matter here**, because the answer is a minimum: the best combination wins whichever order coins are tried in. Order only matters when counting. In [Coin Change II](https://leetcode.com/problems/coin-change-ii/), amounts-outer would count `1+2` and `2+1` as two different ways, so coins must go outside there. Solution 2 uses the same loop shape as Coin Change II and the unbounded knapsack; only the combine step changes.

In both versions, the look-back `v-coin` depends on the coin, so the array cannot shrink to a fixed number of variables.

### 7. Coin Change II - [#518 (Medium)](https://leetcode.com/problems/coin-change-ii/)

Return the number of combinations of coins that make up `amount`. Coins are unlimited, and different orderings of the same coins count once.

<details>
<summary>Brute Force</summary>

Recurse with an index into `coins`: at each step either use the current coin again or move to the next coin. Fixing the coin order this way counts each multiset once, but subproblems repeat.

Time: exponential, O(2^(amount/c + k)) in the worst case. Space: O(amount/c + k) recursion depth.
</details>

<details>
<summary>Hint 1</summary>

There is exactly one way to make 0: take nothing. With amounts in the outer loop and coins inside, `1+2` and `2+1` both get counted. You need each multiset once.
</details>

<details>
<summary>Hint 2</summary>

Put coins in the outer loop. While processing a coin, the table already holds every combination of earlier coins, so each combination is built in one fixed coin order.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(amount · len(coins)) | Space: O(amount)
func change(amount int, coins []int) int {
    dp := make([]int, amount+1) // dp[a] = combinations summing to a
    dp[0] = 1
    for _, c := range coins { // coins outer: combinations, not permutations
        for a := c; a <= amount; a++ { // forwards: coin c may repeat
            dp[a] += dp[a-c]
        }
    }
    return dp[amount]
}
```
</details>

### 8. Unique Paths - [#62 (Medium)](https://leetcode.com/problems/unique-paths/)

A robot moves only right or down on an `m × n` grid. Count paths from the top-left to the bottom-right.

Also in [math-for-dsa.md](../fundamentals/math-for-dsa.md) as a binomial coefficient; here it is the base case for grid DP, which generalises to grids with costs or obstacles where no closed form exists.

<details>
<summary>Brute Force</summary>

Recurse from each cell into its right and down neighbours and count arrivals. There are C(m+n-2, m-1) paths and the recursion walks each one.

Time: O(2^(m+n)). Space: O(m+n) recursion depth.
</details>

<details>
<summary>Hint 1</summary>

Every path into a cell arrives from above or from the left. The count for a cell is the sum of those two counts.
</details>

<details>
<summary>Hint 2</summary>

The first row and first column each have exactly one path. Processing row by row, a single row array is enough, because the old value at `j` is "above" and the new value at `j-1` is "left".
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(m·n) | Space: O(n)
func uniquePaths(m, n int) int {
    dp := make([]int, n)
    for j := range dp {
        dp[j] = 1
    }
    for i := 1; i < m; i++ {
        for j := 1; j < n; j++ {
            dp[j] += dp[j-1] // above (old dp[j]) + left (new dp[j-1])
        }
    }
    return dp[n-1]
}
```
</details>

### 9. Minimum Path Sum - [#64 (Medium)](https://leetcode.com/problems/minimum-path-sum/)

Each grid cell has a non-negative cost. Moving only right or down, find the path from the top-left to the bottom-right with the smallest total cost.

<details>
<summary>Brute Force</summary>

Enumerate every right/down path and sum its cells. There are C(m+n-2, m-1) paths, each of length m+n-1.

Time: O(C(m+n, m) · (m+n)). Space: O(m+n).
</details>

<details>
<summary>Hint 1</summary>

The cheapest way into a cell comes through whichever of its two predecessors is cheaper.
</details>

<details>
<summary>Hint 2</summary>

The first row and column have only one predecessor each, so they are running sums.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(m·n) | Space: O(n)
func minPathSum(grid [][]int) int {
    n := len(grid[0])
    dp := make([]int, n)
    dp[0] = grid[0][0]
    for j := 1; j < n; j++ {
        dp[j] = dp[j-1] + grid[0][j]
    }
    for i := 1; i < len(grid); i++ {
        dp[0] += grid[i][0]
        for j := 1; j < n; j++ {
            dp[j] = min(dp[j], dp[j-1]) + grid[i][j]
        }
    }
    return dp[n-1]
}
```
</details>

### 10. Dungeon Game - [#174 (Hard)](https://leetcode.com/problems/dungeon-game/)

A knight starts at the top-left of a grid and must reach the bottom-right, moving only right or down. Each cell adds or subtracts health, and health must stay at least 1 at all times, including in the first cell. Return the minimum starting health.

<details>
<summary>Brute Force</summary>

For every path, track the running health and record the lowest point; the path needs `1 - lowest` starting health (at least 1). Take the minimum over all paths.

Time: O(C(m+n, m) · (m+n)). Space: O(m+n).
</details>

<details>
<summary>Hint 1</summary>

Going forward, two paths can arrive at a cell with the same health but very different worst dips, and you cannot tell which is better without seeing what comes next. The forward state is not enough.
</details>

<details>
<summary>Hint 2</summary>

Work backwards. For each cell, compute the minimum health needed on entering it to survive the rest of the journey. That depends only on the cells to the right and below.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(m·n) | Space: O(n)
func calculateMinimumHP(dungeon [][]int) int {
    r, c := len(dungeon), len(dungeon[0])
    dp := make([]int, c) // dp[j] = health needed on entering column j of the current row
    // princess cell: leave it with at least 1 health
    dp[c-1] = max(1, 1-dungeon[r-1][c-1])
    // bottom row: the only move is right
    for j := c - 2; j >= 0; j-- {
        dp[j] = max(1, dp[j+1]-dungeon[r-1][j])
    }
    for i := r - 2; i >= 0; i-- {
        // right column: the only move is down (dp[c-1] still holds the row below)
        dp[c-1] = max(1, dp[c-1]-dungeon[i][c-1])
        for j := c - 2; j >= 0; j-- {
            // dp[j] = cell below (not yet overwritten), dp[j+1] = cell to the right
            dp[j] = max(1, min(dp[j], dp[j+1])-dungeon[i][j])
        }
    }
    return dp[0]
}
```

`max(1, ...)` handles potions: a big potion can make the requirement negative, but health on entry must still be at least 1.

This is [Minimum Path Sum](https://leetcode.com/problems/minimum-path-sum/) mirrored: the edge row and column are filled first, then each cell combines its two neighbours, only here the neighbours are below and to the right.
</details>

### 11. Partition Equal Subset Sum - [#416 (Medium)](https://leetcode.com/problems/partition-equal-subset-sum/)

Given positive integers, decide whether they can be split into two subsets with equal sums.

<details>
<summary>Brute Force</summary>

Assign each number to one side or the other and check whether the sides match.

Time: O(2^n). Space: O(n) recursion depth.
</details>

<details>
<summary>Hint 1</summary>

If the total is odd, the answer is no. Otherwise the question is whether some subset sums to exactly half the total.
</details>

<details>
<summary>Hint 2</summary>

Track which sums are reachable using a boolean per sum. Each number may be used once, so update sums from high to low.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n · sum) | Space: O(sum)
func canPartition(nums []int) bool {
    total := 0
    for _, x := range nums {
        total += x
    }
    if total%2 != 0 {
        return false
    }
    target := total / 2
    dp := make([]bool, target+1) // dp[w] = some subset sums to w
    dp[0] = true
    for _, x := range nums {
        for w := target; w >= x; w-- { // backwards: 0-1, each number once
            dp[w] = dp[w] || dp[w-x]
        }
    }
    return dp[target]
}
```
</details>

### 12. Target Sum - [#494 (Medium)](https://leetcode.com/problems/target-sum/)

Put `+` or `-` before each number in `nums`. Count the sign assignments whose result equals `target`.

<details>
<summary>Brute Force</summary>

Try both signs for every number and count assignments that hit the target.

Time: O(2^n). Space: O(n) recursion depth.
</details>

<details>
<summary>Hint 1</summary>

Let `P` be the sum of the positive group and `N` the sum of the negative group. You know `P - N = target` and `P + N = total`. Solve for `P`.
</details>

<details>
<summary>Hint 2</summary>

`P = (total + target) / 2`. Now count subsets summing to `P`. If `total + target` is odd, or `|target| > total`, there are none.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n · s) | Space: O(s), where s = (total + target) / 2
func findTargetSumWays(nums []int, target int) int {
    total := 0
    for _, x := range nums {
        total += x
    }
    if total < target || total < -target || (total+target)%2 != 0 {
        return 0
    }
    s := (total + target) / 2
    dp := make([]int, s+1) // dp[w] = subsets summing to w
    dp[0] = 1
    for _, x := range nums {
        for w := s; w >= x; w-- {
            dp[w] += dp[w-x]
        }
    }
    return dp[s]
}
```

Zeros in `nums` are handled for free: `dp[w] += dp[w-0]` doubles every count, which matches the two signs a zero can take.
</details>

### 13. Longest Common Subsequence - [#1143 (Medium)](https://leetcode.com/problems/longest-common-subsequence/)

Return the length of the longest subsequence common to two strings. A subsequence keeps relative order but need not be contiguous.

<details>
<summary>Brute Force</summary>

Generate every subsequence of the shorter string and check whether each is a subsequence of the other.

Time: O(2^m · n). Space: O(m).
</details>

<details>
<summary>Hint 1</summary>

Compare the last characters of two prefixes. If they match, they can end the common subsequence together.
</details>

<details>
<summary>Hint 2</summary>

If they do not match, at least one of them is not in the answer. Drop the last character of either prefix and take the better result.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(m·n) | Space: O(m·n)
func longestCommonSubsequence(text1, text2 string) int {
    m, n := len(text1), len(text2)
    dp := make([][]int, m+1) // dp[i][j] = LCS of text1[:i], text2[:j]
    for i := range dp {
        dp[i] = make([]int, n+1)
    }
    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if text1[i-1] == text2[j-1] {
                dp[i][j] = dp[i-1][j-1] + 1
            } else {
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
            }
        }
    }
    return dp[m][n]
}
```

To recover the subsequence itself, walk back from `dp[m][n]`: on a match take the character and move diagonally, otherwise move to whichever neighbour holds the same value.
</details>

### 14. Edit Distance - [#72 (Medium)](https://leetcode.com/problems/edit-distance/)

Return the minimum number of single-character inserts, deletes and replaces that turn `word1` into `word2`. Also known as Levenshtein distance, used in spell checkers and diff tools.

<details>
<summary>Brute Force</summary>

On a mismatch, try all three operations recursively and keep the cheapest. Each mismatch branches three ways.

Time: O(3^(m+n)). Space: O(m+n) recursion depth.
</details>

<details>
<summary>Hint 1</summary>

Turning an empty prefix into a prefix of length `k` takes exactly `k` operations. That gives the first row and column.
</details>

<details>
<summary>Hint 2</summary>

On a match, no operation is needed for the last characters. On a mismatch, replace shrinks both prefixes, delete shrinks the first, insert shrinks the second. Take the cheapest plus one.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(m·n) | Space: O(m·n)
func minDistance(word1, word2 string) int {
    m, n := len(word1), len(word2)
    dp := make([][]int, m+1) // dp[i][j] = edits to turn word1[:i] into word2[:j]
    for i := range dp {
        dp[i] = make([]int, n+1)
        dp[i][0] = i
    }
    for j := 0; j <= n; j++ {
        dp[0][j] = j
    }
    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if word1[i-1] == word2[j-1] {
                dp[i][j] = dp[i-1][j-1]
            } else {
                // replace, delete, insert
                dp[i][j] = 1 + min(dp[i-1][j-1], dp[i-1][j], dp[i][j-1])
            }
        }
    }
    return dp[m][n]
}
```
</details>

### 15. Longest Increasing Subsequence - [#300 (Medium)](https://leetcode.com/problems/longest-increasing-subsequence/)

Return the length of the longest strictly increasing subsequence.

<details>
<summary>Brute Force</summary>

Generate every subsequence and check whether it is strictly increasing.

Time: O(2^n · n). Space: O(n).
</details>

<details>
<summary>Hint 1</summary>

Define the state as the longest increasing subsequence that ends exactly at index `i`. Without "ends at", there is no anchor for the recurrence.
</details>

<details>
<summary>Hint 2</summary>

To end at `i`, extend the best subsequence ending at any earlier `j` with a smaller value. The answer is the maximum over all `i`, not the value at the last index.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n²) | Space: O(n)
func lengthOfLIS(nums []int) int {
    dp := make([]int, len(nums)) // dp[i] = LIS ending at i
    best := 0
    for i := range nums {
        dp[i] = 1
        for j := 0; j < i; j++ {
            if nums[j] < nums[i] {
                dp[i] = max(dp[i], dp[j]+1)
            }
        }
        best = max(best, dp[i])
    }
    return best
}
```
</details>

#### Follow-Up

*Can you come up with an algorithm that runs in O(n log(n)) time complexity?*

Keep `tails`, where `tails[k]` is the smallest possible last value of any increasing subsequence of length `k+1` seen so far. `tails` is always sorted, so for each new value, binary search for the first entry not smaller than it. Replace that entry, or append if the value is larger than all of them. A smaller tail never hurts, because it leaves more room for future values to extend.

The name "patience sorting" comes from a card game: deal cards onto piles, each card on the leftmost pile whose top is not smaller, and start a new pile when none fits. The number of piles equals the LIS length, and `tails` holds the pile tops.

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n log n) | Space: O(n)
func lengthOfLISFast(nums []int) int {
    var tails []int
    for _, x := range nums {
        lo, hi := 0, len(tails)
        for lo < hi { // first index with tails[i] >= x
            mid := lo + (hi-lo)/2
            if tails[mid] < x {
                lo = mid + 1
            } else {
                hi = mid
            }
        }
        if lo == len(tails) {
            tails = append(tails, x)
        } else {
            tails[lo] = x
        }
    }
    return len(tails)
}
```

Only the length of `tails` is meaningful. Its contents are not necessarily a valid subsequence of the input; recovering the actual LIS needs parent pointers.
</details>

### 16. Longest Palindromic Subsequence - [#516 (Medium)](https://leetcode.com/problems/longest-palindromic-subsequence/)

Return the length of the longest subsequence of `s` that reads the same forwards and backwards.

<details>
<summary>Brute Force</summary>

Generate every subsequence and check whether it is a palindrome.

Time: O(2^n · n). Space: O(n).
</details>

<details>
<summary>Hint 1</summary>

Look at the two ends of a range. If they match, both can wrap around the best palindrome strictly inside.
</details>

<details>
<summary>Hint 2</summary>

If they do not match, drop one end and keep the better result. Fill so that shorter ranges are ready first: start index descending, end index ascending.
</details>

<details>
<summary>Solution 1: Start Index Descending (Go)</summary>

Rows are filled from the bottom up and each row left to right. When computing `dp[i][j]`, row `i+1` is already complete and `dp[i][j-1]` was just filled, so every shorter range it needs is ready.

```go
// Time: O(n²) | Space: O(n²)
func longestPalindromeSubseq(s string) int {
    n := len(s)
    dp := make([][]int, n) // dp[i][j] = LPS inside s[i..j]
    for i := range dp {
        dp[i] = make([]int, n)
    }
    for i := n - 1; i >= 0; i-- {
        dp[i][i] = 1
        for j := i + 1; j < n; j++ {
            if s[i] == s[j] {
                dp[i][j] = dp[i+1][j-1] + 2
            } else {
                dp[i][j] = max(dp[i+1][j], dp[i][j-1])
            }
        }
    }
    return dp[0][n-1]
}
```
</details>

<details>
<summary>Solution 2: By Length (Go)</summary>

Fill all ranges of length 1, then length 2, and so on: one diagonal at a time, the same order as [Matrix Chain Multiplication](#matrix-chain-multiplication). The fill order says directly that shorter ranges come first.

```go
// Time: O(n²) | Space: O(n²)
func longestPalindromeSubseq(s string) int {
    n := len(s)
    dp := make([][]int, n) // dp[i][j] = LPS inside s[i..j]
    for i := range dp {
        dp[i] = make([]int, n)
    }
    for i := 0; i < n; i++ {
        dp[i][i] = 1 // length 1: a single character
    }
    for l := 2; l <= n; l++ {
        for i := 0; i+l-1 < n; i++ {
            j := i + l - 1
            if s[i] == s[j] {
                // for l == 2, dp[i+1][j-1] is an empty range: 0 is its true answer
                dp[i][j] = 2 + dp[i+1][j-1]
            } else {
                dp[i][j] = max(dp[i][j-1], dp[i+1][j])
            }
        }
    }
    return dp[0][n-1]
}
```
</details>

**Why no special case for length 2.** When two adjacent characters match, both solutions read `dp[i+1][i]`, the range from `i+1` to `i`. That range is empty, and the longest palindromic subsequence of an empty string has length 0. Go's zero value for `int` is also 0, so the cell already holds the right answer.

This is equivalent to the LCS of `s` and its reverse, but the interval form makes the palindrome structure explicit.

### 17. Palindromic Substrings - [#647 (Medium)](https://leetcode.com/problems/palindromic-substrings/)

Count the substrings of `s` that are palindromes. Substrings are contiguous.

<details>
<summary>Brute Force</summary>

Check each of the O(n²) substrings with a two-pointer scan.

Time: O(n³). Space: O(1).
</details>

<details>
<summary>Hint 1</summary>

A substring is a palindrome when its two ends match and the part strictly inside is also a palindrome. Ranges of length 1 or 2 need only the end check.
</details>

<details>
<summary>Hint 2</summary>

Store the answer for each range so the inside check is a lookup, not a scan. Fill shorter ranges first.
</details>

<details>
<summary>Solution 1: Start Index Descending (Go)</summary>

```go
// Time: O(n²) | Space: O(n²)
func countSubstrings(s string) int {
    n := len(s)
    pal := make([][]bool, n) // pal[i][j] = s[i..j] is a palindrome
    for i := range pal {
        pal[i] = make([]bool, n)
    }
    count := 0
    for i := n - 1; i >= 0; i-- {
        for j := i; j < n; j++ {
            if s[i] == s[j] && (j-i < 2 || pal[i+1][j-1]) {
                pal[i][j] = true
                count++
            }
        }
    }
    return count
}
```
</details>

<details>
<summary>Solution 2: By Length (Go)</summary>

```go
// Time: O(n²) | Space: O(n²)
func countSubstrings(s string) int {
    n := len(s)
    pal := make([][]bool, n) // pal[i][j] = s[i..j] is a palindrome
    for i := range pal {
        pal[i] = make([]bool, n)
    }
    count := 0
    for l := 1; l <= n; l++ {
        for i := 0; i+l-1 < n; i++ {
            j := i + l - 1
            if s[i] == s[j] && (l <= 2 || pal[i+1][j-1]) {
                pal[i][j] = true
                count++
            }
        }
    }
    return count
}
```
</details>

**Why this one needs a guard.** For a length-2 range, `pal[i+1][j-1]` is again an empty range. An empty string is a palindrome, so its true value is `true`, but Go's zero value for `bool` is `false`. Without the `j-i < 2` (or `l <= 2`) guard, every matching pair like `"aa"` would be rejected. Compare with [Longest Palindromic Subsequence](https://leetcode.com/problems/longest-palindromic-subsequence/), where the zero value happens to be correct:

| Problem | Empty Range's True Value | Go Zero Value | Guard Needed? |
|---|---|---|---|
| Longest Palindromic Subsequence | 0 | 0 | No |
| Palindromic Substrings | `true` | `false` | Yes |

The general check: if the zero value of the cell type is the correct answer for an empty range, the loop can start right after the base case. If it is not, add a guard or prefill.

Expanding around each of the `2n - 1` centres has the same O(n²) time with O(1) space. The table version is shown because the next problem reuses it.

### 18. Palindrome Partitioning II - [#132 (Hard)](https://leetcode.com/problems/palindrome-partitioning-ii/)

Return the minimum number of cuts needed so that every piece of `s` is a palindrome.

Related to [Palindrome Partitioning (#131)](https://leetcode.com/problems/palindrome-partitioning/) in [recursion-backtracking.md](recursion-backtracking.md), which enumerates every partition; here only the minimum is needed, so DP replaces enumeration.

<details>
<summary>Brute Force</summary>

Try every subset of the `n-1` cut positions and check that every piece is a palindrome.

Time: O(2^n · n). Space: O(n).
</details>

<details>
<summary>Hint 1</summary>

First precompute which substrings are palindromes, so each check is O(1).
</details>

<details>
<summary>Hint 2</summary>

For the prefix ending at `i`: if it is a palindrome, it needs zero cuts. Otherwise choose the last piece `s[j..i]` (a palindrome) and add one cut to the answer for the prefix ending at `j-1`.
</details>

<details>
<summary>Solution 1: Start Index Descending (Go)</summary>

```go
// Time: O(n²) | Space: O(n²)
func minCut(s string) int {
    n := len(s)
    pal := make([][]bool, n)
    for i := range pal {
        pal[i] = make([]bool, n)
    }
    for i := n - 1; i >= 0; i-- {
        for j := i; j < n; j++ {
            pal[i][j] = s[i] == s[j] && (j-i < 2 || pal[i+1][j-1])
        }
    }
    cuts := make([]int, n) // cuts[i] = min cuts for s[0..i]
    for i := 0; i < n; i++ {
        if pal[0][i] {
            continue // whole prefix is a palindrome: 0 cuts
        }
        cuts[i] = i // worst case: cut between every character
        for j := 1; j <= i; j++ {
            if pal[j][i] {
                cuts[i] = min(cuts[i], cuts[j-1]+1)
            }
        }
    }
    return cuts[n-1]
}
```
</details>

<details>
<summary>Solution 2: By Length (Go)</summary>

Only the palindrome table changes; the cuts loop is identical.

```go
// Time: O(n²) | Space: O(n²)
func minCut(s string) int {
    n := len(s)
    pal := make([][]bool, n)
    for i := range pal {
        pal[i] = make([]bool, n)
    }
    for l := 1; l <= n; l++ {
        for i := 0; i+l-1 < n; i++ {
            j := i + l - 1
            pal[i][j] = s[i] == s[j] && (l <= 2 || pal[i+1][j-1])
        }
    }
    cuts := make([]int, n) // cuts[i] = min cuts for s[0..i]
    for i := 0; i < n; i++ {
        if pal[0][i] {
            continue // whole prefix is a palindrome: 0 cuts
        }
        cuts[i] = i // worst case: cut between every character
        for j := 1; j <= i; j++ {
            if pal[j][i] {
                cuts[i] = min(cuts[i], cuts[j-1]+1)
            }
        }
    }
    return cuts[n-1]
}
```
</details>

### 19. Burst Balloons - [#312 (Hard)](https://leetcode.com/problems/burst-balloons/)

Bursting balloon `i` earns `nums[left] × nums[i] × nums[right]`, where `left` and `right` are its current neighbours. Missing neighbours count as 1. Return the maximum coins from bursting all balloons.

<details>
<summary>Brute Force</summary>

Try every burst order, simulating the shrinking array.

Time: O(n! · n). Space: O(n).
</details>

<details>
<summary>Hint 1</summary>

If you pick which balloon bursts first in a range, its neighbours keep changing as other balloons go, so the two sides are not independent subproblems.
</details>

<details>
<summary>Hint 2</summary>

Pick which balloon bursts last in a range instead. At that moment, its neighbours are exactly the range boundaries. Pad the array with 1 on both ends so boundaries always exist.
</details>

<details>
<summary>Solution 1: Exclusive Walls (Go)</summary>

Here `i` and `j` are not balloons you burst. They are the walls on either side, and `dp[i][j]` covers everything strictly between them. That is why the length starts at 2 (two walls, one balloon between) and `k` runs from `i+1` to `j-1`.

```go
// Time: O(n³) | Space: O(n²)
func maxCoins(nums []int) int {
    b := make([]int, 0, len(nums)+2)
    b = append(b, 1)
    b = append(b, nums...)
    b = append(b, 1)
    n := len(b)
    dp := make([][]int, n) // dp[i][j] = best coins bursting everything strictly between i and j
    for i := range dp {
        dp[i] = make([]int, n)
    }
    for length := 2; length < n; length++ {
        for i := 0; i+length < n; i++ {
            j := i + length
            for k := i + 1; k < j; k++ { // k bursts last, neighbours are i and j
                dp[i][j] = max(dp[i][j], dp[i][k]+dp[k][j]+b[i]*b[k]*b[j])
            }
        }
    }
    return dp[0][n-1]
}
```
</details>

<details>
<summary>Solution 2: Inclusive Range (Go)</summary>

Here `i` and `j` are the first and last balloons in the range, and both get burst. This is the same convention as the by-length solutions to the palindrome problems.

```go
// Time: O(n³) | Space: O(n²)
func maxCoins(nums []int) int {
    n := len(nums)
    b := make([]int, n+2) // b[1..n] = balloons, b[0] = b[n+1] = 1 (imaginary walls)
    b[0], b[n+1] = 1, 1
    copy(b[1:], nums)
    dp := make([][]int, n+2) // dp[i][j] = best coins from bursting balloons i..j (inclusive)
    for i := range dp {
        dp[i] = make([]int, n+2)
    }
    for l := 1; l <= n; l++ {
        for i := 1; i+l-1 <= n; i++ {
            j := i + l - 1
            for k := i; k <= j; k++ { // k = the last balloon to burst in i..j
                // left part i..k-1 and right part k+1..j are already gone,
                // so when k bursts, its neighbours are b[i-1] and b[j+1]
                dp[i][j] = max(dp[i][j], dp[i][k-1]+dp[k+1][j]+b[i-1]*b[k]*b[j+1])
            }
        }
    }
    return dp[1][n]
}
```

How to read it:

- `k` is the last balloon to burst in `i..j`. The left part `i..k-1` and the right part `k+1..j` burst first. They are independent subproblems, because `k` stands between them as a wall.
- When `k` finally bursts, everything else in the range is gone, so its neighbours are whatever sits just outside the range: `b[i-1]` and `b[j+1]`.
- If `k == i`, the left part `dp[i][i-1]` is an empty range and holds 0. As in [Longest Palindromic Subsequence](https://leetcode.com/problems/longest-palindromic-subsequence/), the zero value is the correct answer, so no special case is needed. The same holds on the right when `k == j`.

**Trace for `[3, 1, 5]`** (padded: `1, 3, 1, 5, 1`, so the balloons are at indices 1 to 3).

The smaller ranges come first. Each one uses its own outside neighbours, not the array's ends:

| Range | Balloons | Outside Neighbours | Best | Order |
|---|---|---|---|---|
| `dp[1][1]` | 3 | 1, 1 | 1·3·1 = 3 | |
| `dp[2][2]` | 1 | 3, 5 | 3·1·5 = 15 | |
| `dp[3][3]` | 5 | 1, 1 | 1·5·1 = 5 | |
| `dp[1][2]` | 3, 1 | 1, 5 | 30 | burst 1 (3·1·5 = 15), then 3 (1·3·5 = 15) |
| `dp[2][3]` | 1, 5 | 3, 1 | 30 | burst 1 (3·1·5 = 15), then 5 (3·5·1 = 15) |

Then the whole array, `dp[1][3]`, trying each balloon as the last one:

| Last To Burst (k) | Left Part | Right Part | Final Burst | Total |
|---|---|---|---|---|
| 3 (index 1) | 0 | `dp[2][3]` = 30 | 1·3·1 = 3 | 33 |
| 1 (index 2) | `dp[1][1]` = 3 | `dp[3][3]` = 5 | 1·1·1 = 1 | 9 |
| 5 (index 3) | `dp[1][2]` = 30 | 0 | 1·5·1 = 5 | 35 |

The answer is 35: burst 1, then 3, then 5.
</details>

Both solutions have the same shape as [Matrix Chain Multiplication](#matrix-chain-multiplication): a range, a split point, and a cost that depends on the boundaries.

### 20. Super Egg Drop - [#887 (Hard)](https://leetcode.com/problems/super-egg-drop/)

With `k` identical eggs and `n` floors, some critical floor `f` (`0 ≤ f ≤ n`) exists: eggs dropped above it break, at or below it survive. Return the minimum number of drops that guarantees finding `f`.

<details>
<summary>Brute Force</summary>

Let `f(k, n)` be the answer. Try every drop floor `x`. If the egg breaks, the answer is below with one egg fewer: `f(k-1, x-1)`. If it survives, the answer is above: `f(k, n-x)`. The worst case is the larger branch; choose `x` to minimise it.

`f(k, n) = 1 + min over x of max(f(k-1, x-1), f(k, n-x))`

Without memoisation this is exponential. Even memoised, there are `k·n` states with O(n) work each, so O(k·n²), about 10^10 operations at the limits (`k ≤ 100`, `n ≤ 10^4`).
</details>

<details>
<summary>Hint 1</summary>

The direct question "how many moves for `n` floors?" is expensive. Turn it around: with `k` eggs and `m` moves, how many floors can you fully decide?
</details>

<details>
<summary>Hint 2</summary>

The first drop splits the building into floors below (decided with one egg fewer and one move fewer), floors above (same eggs, one move fewer), and the floor you dropped from. Increase `m` until the count reaches `n`.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(k · m), m = the answer (at most n) | Space: O(k)
func superEggDrop(k, n int) int {
    floors := make([]int, k+1) // floors[e] = floors decidable with e eggs and the current moves
    moves := 0
    for floors[k] < n {
        moves++
        for e := k; e >= 1; e-- { // descending: floors[e-1] still holds the previous move count
            floors[e] = floors[e] + floors[e-1] + 1 // survives + breaks + this floor
        }
    }
    return moves
}
```

The answer grows very slowly. With 2 eggs and 10,000 floors it is 141 moves; with 3 eggs it is 40.
</details>

### 21. House Robber III - [#337 (Medium)](https://leetcode.com/problems/house-robber-iii/)

Houses form a binary tree. Robbing two directly connected houses trips the alarm. Return the maximum you can rob.

The approach is walked through step by step, with a trace, in [DP On Trees](#dp-on-trees).

<details>
<summary>Brute Force</summary>

For each node, compare "rob it, then recurse into the grandchildren" with "skip it, then recurse into the children", without caching. Each node is evaluated once from its parent and once from its grandparent, so the call count grows like Fibonacci in depth.

Time: O(φ^n) on a path-shaped tree, where φ ≈ 1.618. Space: O(h) recursion depth.
</details>

<details>
<summary>Hint 1</summary>

The array version decided each house from the previous one. On a tree, the decision at a node couples to both children, so a single number per node is not enough.
</details>

<details>
<summary>Hint 2</summary>

Return two numbers from every subtree: the best if its root is robbed, and the best if it is not. A parent can then combine its children's pairs in O(1).
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(h)
func robTree(root *TreeNode) int {
    var dfs func(*TreeNode) (int, int)
    dfs = func(node *TreeNode) (withNode, withoutNode int) {
        if node == nil {
            return 0, 0
        }
        lWith, lWithout := dfs(node.Left)
        rWith, rWithout := dfs(node.Right)
        withNode = node.Val + lWithout + rWithout // robbed: children must be skipped
        withoutNode = max(lWith, lWithout) + max(rWith, rWithout)
        return withNode, withoutNode
    }
    return max(dfs(root))
}
```

The tuple return removes the repeated grandchild calls, so every node is visited once.
</details>

### 22. Binary Tree Cameras - [#968 (Hard)](https://leetcode.com/problems/binary-tree-cameras/)

A camera on a node monitors its parent, itself and its children. Return the minimum number of cameras needed to monitor every node.

The approach is walked through step by step, with a trace, in [DP On Trees](#dp-on-trees).

<details>
<summary>Brute Force</summary>

Try every subset of nodes as camera positions and check full coverage.

Time: O(2^n · n). Space: O(n).
</details>

<details>
<summary>Hint 1</summary>

A node's coverage can come from itself, a child, or its parent. The parent is not known yet when processing bottom-up, so a node may be left uncovered on purpose for its parent to fix.
</details>

<details>
<summary>Hint 2</summary>

Return three costs per subtree: node uncovered (but everything below covered), node covered without a camera on it, and camera on the node. A missing child is covered for free but cannot hold a camera.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(h)
func minCameraCover(root *TreeNode) int {
    const inf = 1 << 29 // far above any real answer (≤ 1000 nodes), and inf+inf still fits in 32 bits
    var dfs func(*TreeNode) (uncovered, covered, camera int)
    dfs = func(node *TreeNode) (int, int, int) {
        if node == nil {
            return 0, 0, inf // nil is covered, but cannot host a camera
        }
        lU, lC, lCam := dfs(node.Left)
        rU, rC, rCam := dfs(node.Right)
        uncovered := lC + rC // children covered by their own subtrees, nothing watches node
        covered := min(lCam+min(rC, rCam), rCam+min(lC, lCam)) // at least one child has a camera
        camera := 1 + min(lU, lC, lCam) + min(rU, rC, rCam)    // children may be anything
        return uncovered, covered, camera
    }
    _, covered, camera := dfs(root)
    return min(covered, camera) // the root has no parent to cover it
}
```

The "uncovered" state is what makes this DP on trees rather than plain DFS: the subtree hands an unresolved obligation up to its parent.
</details>

### 23. Find The Shortest Superstring - [#943 (Hard)](https://leetcode.com/problems/find-the-shortest-superstring/)

Given up to 12 strings, return the shortest string that contains every one of them as a substring. No string is a substring of another. Any shortest answer is accepted.

This is the Travelling Salesman problem in disguise: the "cities" are the words, and the "distance" from one word to the next is how much it adds to the string.

<details>
<summary>Brute Force</summary>

Try every order of the words. For each order, join consecutive words with their largest overlap (the longest suffix of one that is a prefix of the next), and keep the shortest result.

Time: O(n! · n · L), where `L` is the word length. Space: O(n · L).
</details>

<details>
<summary>Hint 1</summary>

Joining `"catg"` then `"atgcatc"` gives `"catgcatc"`: the overlap `"atg"` is written once. The final length is the total length of all words minus the total overlap between neighbours. So maximise the sum of overlaps over an order of the words.
</details>

<details>
<summary>Hint 2</summary>

The overlap gained by the next word depends only on the previous word, not on the earlier ones. So the state is (which words are used, which word is last). With at most 12 words, the set fits in a bitmask. Store a parent pointer for each state so you can rebuild the string at the end.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(2^n · n² + n² · L) | Space: O(2^n · n)
func shortestSuperstring(words []string) string {
    n := len(words)
    // overlap[i][j] = longest suffix of words[i] that is a prefix of words[j]
    overlap := make([][]int, n)
    for i := range overlap {
        overlap[i] = make([]int, n)
        for j := range overlap[i] {
            if i == j {
                continue
            }
            for k := min(len(words[i]), len(words[j])); k > 0; k-- {
                if strings.HasSuffix(words[i], words[j][:k]) {
                    overlap[i][j] = k
                    break
                }
            }
        }
    }
    full := 1<<n - 1
    // dp[mask][i] = max total overlap using exactly the words in mask, ending with word i
    dp := make([][]int, 1<<n)
    parent := make([][]int, 1<<n)
    for m := range dp {
        dp[m] = make([]int, n)
        parent[m] = make([]int, n)
        for i := range dp[m] {
            dp[m][i] = -1 // unreachable
            parent[m][i] = -1
        }
    }
    for i := 0; i < n; i++ {
        dp[1<<i][i] = 0 // any word can come first
    }
    for mask := 1; mask <= full; mask++ {
        for i := 0; i < n; i++ {
            if dp[mask][i] < 0 {
                continue
            }
            for j := 0; j < n; j++ {
                if mask&(1<<j) != 0 {
                    continue
                }
                next := mask | 1<<j
                if v := dp[mask][i] + overlap[i][j]; v > dp[next][j] {
                    dp[next][j] = v
                    parent[next][j] = i
                }
            }
        }
    }
    last := 0
    for i := 1; i < n; i++ {
        if dp[full][i] > dp[full][last] {
            last = i
        }
    }
    // walk the parent pointers back to recover the order
    order := make([]int, 0, n)
    for mask, i := full, last; i != -1; {
        order = append(order, i)
        prev := parent[mask][i]
        mask &^= 1 << i
        i = prev
    }
    var sb strings.Builder
    sb.WriteString(words[order[n-1]])
    for k := n - 2; k >= 0; k-- {
        prev, cur := order[k+1], order[k]
        sb.WriteString(words[cur][overlap[prev][cur]:])
    }
    return sb.String()
}
```

Compared with the Travelling Salesman code in [Classical Problems](#classical-problems), three things differ:

- any word can come first, so there is no fixed starting city
- the goal is the maximum total overlap, not the minimum total distance
- the problem asks for the string itself, so parent pointers are kept to rebuild it

</details>

## References

- Richard Bellman (1957), *Dynamic Programming*, Princeton University Press. The original treatment.
- Richard Bellman (1984), *Eye of the Hurricane: An Autobiography*, World Scientific. Source of the story behind the name.
- Cormen, Leiserson, Rivest, Stein, *Introduction to Algorithms*, 3rd ed., chapter 15. Rod cutting, matrix chain multiplication, LCS, and the elements of DP.
- Vladimir Levenshtein (1966), "Binary codes capable of correcting deletions, insertions, and reversals", *Soviet Physics Doklady* 10(8). Origin of edit distance.
- Michael Held, Richard Karp (1962), "A Dynamic Programming Approach to Sequencing Problems", *Journal of the SIAM* 10(1). Bitmask DP for Travelling Salesman.
- David Aldous, Persi Diaconis (1999), "Longest Increasing Subsequences: From Patience Sorting to the Baik-Deift-Johansson Theorem", *Bulletin of the AMS* 36(4). Survey of patience sorting and LIS.
- [recursion-backtracking.md](recursion-backtracking.md): recursion trees and memoisation, the starting point for top-down DP.
- [binary-search.md](binary-search.md): lower-bound search used by the O(n log n) LIS.
- [../patterns/tree-dfs.md](../patterns/tree-dfs.md): post-order DFS and the return-value pattern used by DP on trees.
- [../fundamentals/math-for-dsa.md](../fundamentals/math-for-dsa.md): [Unique Paths](https://leetcode.com/problems/unique-paths/) as a binomial coefficient.