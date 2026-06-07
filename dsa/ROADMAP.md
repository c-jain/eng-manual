# DSA Interview Preparation Roadmap

## Overview

This roadmap covers Data Structures, Algorithms, and Problem-Solving Patterns for software engineering interviews. It is a lifetime reference — structured to understand deeply once, and revise from forever.

**Primary language:** Go. JavaScript added only where the idiom differs meaningfully.
**Schedule:** ~9 hours/week (1h weekdays, 2h weekends).
**Total estimate:** ~11 weeks for full coverage. Phases 1–4 alone cover ~85% of interview problems.

---

## What This Covers (And Why Three Separate Layers)

Interviews test three distinct things, and conflating them is the most common preparation mistake:

- **Data Structures** (`data-structures/`) — how data is organised; what operations cost what; how it is implemented in Go.
- **Algorithms** (`algorithms/`) — specific named techniques for processing data: Dijkstra, KMP, Merge Sort.
- **Patterns** (`patterns/`) — how to frame an interview problem into a known solution shape: sliding window, two pointers, union-find.
- **Fundamentals** (`fundamentals/`) — the reasoning layer: complexity analysis and the mathematical tools that underpin everything.

---

## Directory Structure

```
dsa/
├── ROADMAP.md                              ← this file
│
├── fundamentals/
│   ├── complexity-analysis.md              Big O, time, space, amortised, recurrences
│   └── math-for-dsa.md                    Modular arithmetic, GCD/LCM, primes, combinatorics, logarithms
│
├── data-structures/
│   ├── arrays.md                           Internals, 2D arrays, in-place operations
│   ├── strings.md                          String internals in Go, builder, immutability
│   ├── linked-lists.md                     Singly, doubly, circular; dummy head technique
│   ├── stacks-queues.md                    Stack, Queue, Deque, Circular Queue in Go
│   ├── hash-maps.md                        Hash maps, hash sets, Go map internals, collision handling
│   ├── trees.md                            Binary Tree, BST, AVL (concept), Segment Tree (intro), Fenwick Tree (intro)
│   ├── heaps.md                            Min/max heap, heapify, container/heap in Go
│   ├── graphs.md                           Adjacency list/matrix, directed/undirected, weighted/unweighted
│   └── tries.md                            Trie node structure, insert/search/delete, XOR Trie
│
├── algorithms/
│   ├── sorting.md                          Bubble, Selection, Insertion, Merge, Quick, Heap, Counting, Radix
│   ├── binary-search.md                    Classic, on answer space, rotated arrays, 2D matrix, lower/upper bound
│   ├── recursion-backtracking.md           Call stack, base cases, recursion tree, backtracking template, pruning
│   ├── graph-traversal.md                  BFS, DFS on graphs; connected components; iterative vs recursive
│   ├── graph-shortest-path.md              Dijkstra, Bellman-Ford, Floyd-Warshall
│   ├── graph-mst.md                        Kruskal's (with DSU), Prim's (with heap)
│   ├── graph-advanced.md                   Bridges, Articulation Points, SCC — Kosaraju's and Tarjan's [optional]
│   ├── dynamic-programming.md              1D DP, Grid DP, 0-1 Knapsack, Unbounded Knapsack, LCS, LIS, Edit Distance, Palindromic DP, Interval DP, DP on Trees
│   ├── greedy.md                           Greedy vs DP, exchange argument, Activity Selection, Jump Game, Huffman
│   ├── string-algorithms.md                KMP, Z-algorithm, Rabin-Karp, string hashing, Manacher's
│   └── bit-manipulation.md                 AND/OR/XOR/shifts, set/clear/toggle, Brian Kernighan, XOR tricks
│
└── patterns/
    ├── two-pointers.md                     Opposite-end and same-direction techniques
    ├── sliding-window.md                   Fixed-size and variable-size windows
    ├── prefix-sum.md                       Prefix sum, difference array, 2D prefix sum
    ├── fast-slow-pointers.md               Floyd's cycle detection, middle of list, nth from end
    ├── in-place-reversal.md                Reversing linked lists and subarrays
    ├── monotonic-stack.md                  Next greater element, histogram area, span problems
    ├── intervals.md                        Merge, Insert, Non-overlapping, Meeting Rooms, Sweep Line
    ├── top-k-heap.md                       Top-K elements, K closest points, sliding window median, Two Heaps
    ├── union-find.md                       DSU with path compression and union by rank
    ├── topological-sort.md                 Kahn's (BFS), DFS-based; course scheduling; cycle detection in DAG
    ├── tree-dfs.md                         Pre/in/post-order, path sum, LCA, diameter, serialisation
    └── tree-bfs.md                         Level-order, zigzag, right-side view, cousins
```

Total: 35 files across 4 directories.

---

## Session Format

Each topic is covered in one dedicated session, following this structure:

1. **Teach** — what it is, why it exists, why it is named that, what problems it brings; internals and diagrams.
2. **Template** — canonical Go implementation with line-by-line annotations. JavaScript added where the idiom differs meaningfully.
3. **Problems** — 5–10 curated problems, all free on LeetCode unless an alternative link is explicitly noted. Each problem has:
   - Platform link with difficulty label
   - Two collapsible hints
   - Collapsible Go solution with explanation
4. **File** — production-ready markdown, ready to push to `eng-manual`.

### Hint Discipline

Every problem in these notes has two hints and a solution, all behind `<details>` tags. The intended order:

```
Attempt (≥15 min) → Hint 1 → Attempt again (≥10 min) → Hint 2 → Solution
```

Never open the solution before you have a working approach written out — even a wrong one.

### Problem Format

```markdown
**Two Sum** — [LeetCode #1](https://leetcode.com/problems/two-sum/) · Easy · Free

<details>
<summary>Hint 1</summary>
For each element, what are you actually looking for?
If you need `target - nums[i]`, what data structure gives O(1) lookup?
</details>

<details>
<summary>Hint 2</summary>
Use a hash map. As you iterate, check if `target - nums[i]` already exists
in the map. If yes, return the indices. If no, store `nums[i] → i` and continue.
</details>

<details>
<summary>Solution (Go)</summary>

```go
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

**Why this works:** We turn the "find complement" problem into an O(1) lookup
by storing what we have seen. One pass — O(n) time, O(n) space.
</details>
```

### Classical Problems

Classical problems (N-Queens, Tower of Hanoi, Matrix Chain Multiplication, Egg Drop, Word Break, Edit Distance) are included as curated problems inside the relevant topic file — they are not separate files. They live where they conceptually belong:

- N-Queens, Tower of Hanoi, Sudoku Solver → `algorithms/recursion-backtracking.md`
- Matrix Chain Multiplication, Egg Drop, Word Break → `algorithms/dynamic-programming.md` (Interval DP section)
- Fibonacci, Coin Change → `algorithms/dynamic-programming.md` (1D DP section)

What is deliberately excluded: Strassen's matrix multiplication, FFT, network flow. These are academic or competitive programming topics and do not appear in product company interviews.

---

## Phase 1 — Foundations (Weeks 1–2, ~18h)

*Mental framework and array-based patterns. Everything else builds on this.*

🔄 = reinforcement (Striver's context exists, focus on Go templates and patterns) · 🆕 = new topic

### Fundamentals

| File | Topic |
|------|-------|
| `fundamentals/complexity-analysis.md` | Big O, Ω, Θ; time complexity; space complexity; amortised analysis; solving recurrences (master theorem) |
| `fundamentals/math-for-dsa.md` | Modular arithmetic; GCD/LCM (Euclidean algorithm); prime sieve; combinatorics basics; logarithm intuition |

### Arrays & Strings

| File | Topic | Type |
|------|-------|------|
| `data-structures/arrays.md` | Array internals; static vs dynamic arrays; 2D arrays; cache locality; in-place operations | DS 🔄 |
| `patterns/two-pointers.md` | Opposite-end two pointers (sorted array); same-direction two pointers; partition problems | Pattern 🔄 |
| `patterns/sliding-window.md` | Fixed-size window; variable-size window (shrink on violation); when to use which | Pattern 🔄 |
| `patterns/prefix-sum.md` | Prefix sum array; range sum queries; difference array; 2D prefix sum | Pattern 🔄 |

### Hashing

| File | Topic | Type |
|------|-------|------|
| `data-structures/hash-maps.md` | Hash function; collision resolution (chaining, open addressing); load factor; Go map internals; frequency counting patterns | DS 🔄 |

### Sorting

| File | Topic | Type |
|------|-------|------|
| `algorithms/sorting.md` | Bubble, Selection, Insertion (O(n²)); Merge Sort (stable, O(n log n)); Quick Sort (in-place, pivot strategies); Heap Sort; Counting Sort; Radix Sort; stability; when to use which | Algo 🔄 |

---

## Phase 2 — Search & Linear Structures (Weeks 3–4, ~18h)

*Binary search is deeper than it looks. Linked lists are well-known but confusing without explicit pattern names.*

### Binary Search

| File | Topic | Type |
|------|-------|------|
| `algorithms/binary-search.md` | Classic binary search; lower bound / upper bound; search on rotated sorted array; binary search on answer space; search in 2D matrix; left ≤ right vs left < right template comparison | Algo 🔄 |

### Linked Lists

| File | Topic | Type |
|------|-------|------|
| `data-structures/linked-lists.md` | Singly linked list; doubly linked list; circular linked list; dummy head node technique; Go struct-based implementation | DS 🔄 |
| `patterns/fast-slow-pointers.md` | Floyd's cycle detection; finding cycle entry point; middle of list; Kth from end; Happy Number | Pattern 🔄 |
| `patterns/in-place-reversal.md` | Reversing a full linked list; reversing a sublist; reversing K-groups; in-place subarray reversal | Pattern 🔄 |

### Stacks & Queues

| File | Topic | Type |
|------|-------|------|
| `data-structures/stacks-queues.md` | Stack (LIFO); Queue (FIFO); Deque; Circular Queue; LRU Cache structure; Go slice-based and linked-list-based implementations | DS 🔄 |
| `patterns/monotonic-stack.md` | Monotonic increasing vs decreasing stack; next greater element; previous smaller element; largest rectangle in histogram; daily temperatures | Pattern 🔄 |

### Bit Manipulation

| File | Topic | Type |
|------|-------|------|
| `algorithms/bit-manipulation.md` | AND / OR / XOR / NOT / left shift / right shift; set, clear, toggle a bit; check power of 2; Brian Kernighan's algorithm; XOR to find unique element; bitmask for subsets | Algo 🔄 |

---

## Phase 3 — Trees & Heaps (Weeks 5–6, ~18h)

*The most interview-dense topic cluster. Tree DFS and BFS problems appear in nearly every onsite round.*

### Recursion & Backtracking

| File | Topic | Type |
|------|-------|------|
| `algorithms/recursion-backtracking.md` | Call stack mechanics; recursion tree visualisation; memoisation introduction; backtracking template (choose → explore → unchoose); pruning; subsets, permutations, combinations, N-Queens, Sudoku Solver, Tower of Hanoi | Algo 🔄 |

### Trees

| File | Topic | Type |
|------|-------|------|
| `data-structures/trees.md` | Binary tree; BST (insert/search/delete); balanced BST concept (AVL, Red-Black — not implementation); Segment Tree (range queries — intro); Fenwick Tree / BIT (prefix sums — intro) | DS 🔄 |
| `patterns/tree-dfs.md` | Preorder, inorder, postorder traversal; path sum problems; lowest common ancestor (LCA); diameter of binary tree; binary tree serialisation/deserialisation | Pattern 🔄 |
| `patterns/tree-bfs.md` | Level-order traversal; zigzag traversal; right-side view; average of levels; cousins in binary tree; connect next right pointers | Pattern 🔄 |

### Heaps & Priority Queues

| File | Topic | Type |
|------|-------|------|
| `data-structures/heaps.md` | Binary heap structure; min-heap vs max-heap; heapify (sift-up, sift-down); heap sort; Go's `container/heap` interface with a concrete implementation | DS 🔄 |
| `patterns/top-k-heap.md` | Top-K largest/smallest elements; K closest points to origin; sliding window median; Find Median from Data Stream (Two Heaps) | Pattern 🔄 |

### Intervals

| File | Topic | Type |
|------|-------|------|
| `patterns/intervals.md` | Merge Intervals; Insert Interval; Non-overlapping Intervals; Meeting Rooms I/II; Sweep Line algorithm | Pattern 🔄 |

---

## Phase 4 — Graphs (Weeks 7–8, ~18h)

*Entirely new territory from Striver's. The richest single topic cluster for senior-level interviews.*

### Graph Fundamentals & Traversal

| File | Topic | Type |
|------|-------|------|
| `data-structures/graphs.md` | Graph properties (directed/undirected, weighted/unweighted, cyclic/acyclic, bipartite); adjacency list vs adjacency matrix; Go representation; when each representation wins | DS 🆕 |
| `algorithms/graph-traversal.md` | BFS (shortest path in unweighted graphs); DFS (component counting, flood fill, cycle detection); iterative DFS with explicit stack; common mistakes | Algo 🆕 |
| `patterns/union-find.md` | Disjoint Set Union (DSU); find with path compression; union by rank; cycle detection in undirected graphs; number of connected components | Pattern 🆕 |
| `patterns/topological-sort.md` | Kahn's Algorithm (BFS with in-degree); DFS-based topological sort; detecting cycles in a DAG; course schedule problems; alien dictionary | Pattern 🆕 |

### Shortest Path Algorithms

| File | Topic | Type |
|------|-------|------|
| `algorithms/graph-shortest-path.md` | Dijkstra (non-negative weights, heap-based); Bellman-Ford (negative weights, negative cycle detection); Floyd-Warshall (all-pairs shortest path); when to use which | Algo 🆕 |

### Minimum Spanning Tree

| File | Topic | Type |
|------|-------|------|
| `algorithms/graph-mst.md` | MST definition and properties; Kruskal's algorithm (sort edges + DSU); Prim's algorithm (greedy + heap); comparison and use cases | Algo 🆕 |

### Advanced Graph (Optional — Targeted for FAANG / Senior Roles)

| File | Topic | Type |
|------|-------|------|
| `algorithms/graph-advanced.md` | Bridge edges; articulation points; Strongly Connected Components (SCC) — Kosaraju's algorithm; Tarjan's algorithm; Eulerian path | Algo 🆕 |

> Skip `graph-advanced.md` if time is short. Bridges and SCC appear rarely in standard product company interviews.

---

## Phase 5 — Advanced Techniques (Weeks 9–10, ~18h)

*DP is the most feared topic and the most rewarding to master. Tries and string algorithms complete the picture.*

### Dynamic Programming

| File | Topic | Type |
|------|-------|------|
| `algorithms/dynamic-programming.md` | Identifying DP problems; memoisation (top-down) vs tabulation (bottom-up); state design; 1D DP (Fibonacci, Climbing Stairs, House Robber, Coin Change); Grid/2D DP (Unique Paths, Minimum Path Sum, Dungeon Game); 0-1 Knapsack and variants; Unbounded Knapsack; LCS; LIS; Edit Distance; Palindromic Subsequence/Partitioning; Interval DP (Matrix Chain Multiplication, Burst Balloons); Egg Drop; DP on Trees | Algo 🆕 |

> DP is the largest single topic. Budget 2–3 sessions for it. The file is structured with H2 sections per sub-pattern so you can tackle one sub-pattern per session.

### Tries

| File | Topic | Type |
|------|-------|------|
| `data-structures/tries.md` | Trie node structure; insert, search, startsWith; delete; prefix-based problems; XOR Trie (maximum XOR of two numbers); word search variants | DS 🆕 |

### String Algorithms

| File | Topic | Type |
|------|-------|------|
| `data-structures/strings.md` | String internals in Go (immutability, `strings.Builder`); rune vs byte; common string operations and their complexities | DS 🔄 |
| `algorithms/string-algorithms.md` | KMP algorithm (failure function, pattern matching); Z-algorithm; Rabin-Karp (rolling hash); string hashing; Manacher's algorithm (longest palindromic substring) | Algo 🆕 |

### Greedy

| File | Topic | Type |
|------|-------|------|
| `algorithms/greedy.md` | Greedy paradigm; how to prove greedy correctness (exchange argument); Greedy vs DP decision; Activity Selection; Interval Scheduling; Jump Game I/II; Task Scheduler; Huffman encoding concept | Algo 🔄 |

---

## Phase 6 — Integration & Mock Practice (Week 11+)

- Solve 2–3 mixed-topic problems per day without looking at the pattern first
- Target: Medium problems solved in ≤20 min, Hard in ≤35 min
- After each problem: identify the pattern, compare your approach to your notes template
- Identify weak areas → re-read notes → solve 2–3 problems from that pattern again
- Run at least 2 full mock interviews before real interviews

**Mock interview platforms:** [Pramp](https://www.pramp.com/), [interviewing.io](https://interviewing.io/), LeetCode Weekly Contest

---

## Priority If Time Is Short

Complete phases in order. Stop where time runs out — earlier phases give the highest interview coverage per hour spent.

| Phases Completed | Approximate Coverage | What You Can Handle |
|-----------------|---------------------|---------------------|
| 1–2 | ~55% | Most array, string, linked list, stack, binary search problems |
| 1–3 | ~70% | + Trees, heaps, intervals, backtracking |
| 1–4 | ~85% | + Full graph problems (shortest path, MST, topological sort) |
| 1–5 | ~95% | + DP, tries, string pattern matching, greedy |

---

## Cross-Reference: Striver's A2Z vs This Roadmap

| Striver's Step | This Roadmap |
|----------------|-------------|
| Step 1: Basics (Patterns, Maths, Recursion) | `fundamentals/`, `algorithms/recursion-backtracking.md` |
| Step 2: Sorting | `algorithms/sorting.md` |
| Step 3: Arrays | `data-structures/arrays.md`, `patterns/two-pointers.md`, `patterns/sliding-window.md`, `patterns/prefix-sum.md` |
| Step 4: Binary Search | `algorithms/binary-search.md` |
| Step 5: Strings | `data-structures/strings.md`, `algorithms/string-algorithms.md` |
| Step 6: Linked Lists | `data-structures/linked-lists.md`, `patterns/fast-slow-pointers.md`, `patterns/in-place-reversal.md` |
| Step 7: Recursion | `algorithms/recursion-backtracking.md` |
| Step 8: Bit Manipulation | `algorithms/bit-manipulation.md` |
| Step 9: Stacks & Queues | `data-structures/stacks-queues.md`, `patterns/monotonic-stack.md` |
| Step 10: Sliding Window | `patterns/sliding-window.md` |
| Step 11: Heaps | `data-structures/heaps.md`, `patterns/top-k-heap.md` |
| Step 12: Greedy | `algorithms/greedy.md` |
| Step 13: Binary Trees | `data-structures/trees.md`, `patterns/tree-dfs.md`, `patterns/tree-bfs.md` |
| Step 14: BST | `data-structures/trees.md` (BST section) |
| Step 15: Graphs | `data-structures/graphs.md`, `algorithms/graph-traversal.md`, `patterns/union-find.md`, `patterns/topological-sort.md`, `algorithms/graph-shortest-path.md`, `algorithms/graph-mst.md` |
| Step 16: DP | `algorithms/dynamic-programming.md` |
| Step 17: Tries | `data-structures/tries.md` |
| Step 18: Strings (advanced) | `algorithms/string-algorithms.md` |

---

## References

- [Striver's A2Z DSA Sheet](https://takeuforward.org/strivers-a2z-dsa-course/strivers-a2z-dsa-course-sheet-2/) — comprehensive topic-by-topic problem list; use as supplementary problems per topic
- [NeetCode 150](https://neetcode.io/practice) — curated pattern-based problem list; NeetCode's YouTube has excellent Go-compatible walkthroughs
- [NeetCode Roadmap](https://neetcode.io/roadmap) — visual pattern dependency map
- [Grokking the Coding Interview](https://www.designgurus.io/course/grokking-the-coding-interview) — the original pattern-based framing reference
- [LeetCode](https://leetcode.com) — primary problem platform
- [AlgoMonster](https://algo.monster/problems/stats) — pattern frequency data from real interview reports
- [CP-Algorithms](https://cp-algorithms.com/) — deep algorithm explanations with mathematical proofs
- [Go `container/heap` docs](https://pkg.go.dev/container/heap) — Go's heap interface; read before the heaps session
- [Visualgo](https://visualgo.net/) — interactive DS/algorithm visualiser; useful during learning phases