---
Status: 🌳 Evergreen
Created: 2026-11-10
Last Updated: 2026-11-10
---

# Tree BFS

## Table Of Contents

1. [What It Is And Why It Exists](#what-it-is-and-why-it-exists)
2. [Why "BFS" And Why A Queue](#why-bfs-and-why-a-queue)
3. [The Level-Boundary Trick](#the-level-boundary-trick)
4. [When BFS Beats DFS](#when-bfs-beats-dfs)
5. [The `Next`-Pointer Trick For O(1) Space](#the-next-pointer-trick-for-o1-space)
6. [Complexity Reference](#complexity-reference)
7. [Recognizing The Pattern](#recognizing-the-pattern)
8. [Go Template Code](#go-template-code)
9. [Where JavaScript Differs](#where-javascript-differs)
10. [LeetCode Problems](#leetcode-problems)
11. [References](#references)

## What It Is And Why It Exists

Tree BFS visits nodes in order of increasing depth: the root first, then every node at depth 1, then every node at depth 2, and so on. It exists because a whole class of tree questions depends on *what level a node is on*, not on its position within a subtree. "Right-side view," "average per level," "minimum depth," "cousins," "zigzag by level" - none of these can be answered cleanly by preorder, inorder, or postorder, because DFS interleaves depths. It plunges to the bottom of one subtree before touching the next.

The brute-force alternative for level-dependent questions is DFS-with-depth-tagging: recurse with a `depth` argument, push each visited node into `buckets[depth]`, then post-process the buckets. It works, and it's O(n). But you carry the whole tree's worth of buckets even for questions that only need the last level (right-side view) or the current level's aggregate (average per level). BFS answers those in one pass without a per-level bucket outliving the level it belongs to, and it lets you *return early* the moment you find what you need - see minimum depth below.

### The Problem It Solves

Any traversal that needs "process level k completely before touching level k+1" - either because the answer is per-level, or because a shallow answer lets you stop early - is what BFS is for. DFS reaches the deepest node before the second child of the root; BFS reaches every depth-1 node before any depth-2 node. That ordering is the pattern's whole reason for existing.

## Why "BFS" And Why A Queue

**BFS = breadth-first search**: expand "broadly" across the current frontier before going any deeper. Contrast with DFS ("depth-first"), which commits to one branch all the way down. The name is a direct description of the visit order, not a metaphor.

The queue is what makes the order work. Here is the claim, stated precisely: at every point in the traversal, the FIFO queue contains nodes from at most two consecutive levels, with all nodes of the shallower level in front of all nodes of the deeper level.

Proof by induction on steps:

```
Base:  queue = [root]  -- one node, one level. Holds.

Step:  Dequeue the front node u. By the invariant, u is on the
       shallower of the two levels present (call it L).

       We enqueue u's children at the back. u's children are on
       level L+1. If the queue already held some level-L+1 nodes,
       the new ones go behind them -- still all level-L+1 at the
       back. If the queue held no level-L+1 nodes yet, the new
       children are the only level-L+1 entries.

       Either way: all remaining level-L nodes are still in front
       of all level-L+1 nodes. Invariant holds.
```

The invariant is what guarantees the visit order matches level order. If we replaced the queue with a stack (LIFO), the last-in-first-out discipline would send us down the newly-added children immediately, which is exactly DFS. FIFO is load-bearing.

## The Level-Boundary Trick

The invariant above says the queue holds at most two levels at once, but it does not by itself tell you where one level ends and the next begins. That's where the level-boundary trick comes in: **snapshot `size := len(queue)` at the top of each outer iteration**, then process exactly `size` items before the next snapshot.

```
q = [A]              size=1  -- process 1 node (A), enqueue B, C
q = [B, C]           size=2  -- process 2 nodes (B, C), enqueue D, E, F
q = [D, E, F]        size=3  -- process 3 nodes, enqueue G
q = [G]              size=1  -- process 1 node
q = []               done
```

Why the snapshot works: at the moment you take `size = len(queue)`, the queue holds exactly the nodes of the next level (by the invariant, they're all bunched at the front, followed by nothing yet). While you process those `size` nodes, you'll enqueue their children - those children go behind the current level, so `len(queue)` grows, but you only consume `size` items, so you stop exactly at the boundary. The children stay in the queue for the next outer iteration.

This is why every BFS-per-level template you'll see has the shape:

```go
for len(q) > 0 {
    size := len(q)          // snapshot the level boundary
    for i := 0; i < size; i++ {
        // process q[i], enqueue children
    }
    q = q[size:]            // advance past the finished level
}
```

Without the snapshot, you'd process `len(q)` in the inner loop, which grows as you enqueue children - you'd blur two levels together and get the DFS-with-depth-tagging shape at best, or an infinite loop at worst.

## When BFS Beats DFS

Both traversals visit every node in O(n). BFS wins when the answer depends on depth in a way that lets you either terminate early or discard past levels.

- **Minimum depth of a binary tree.** DFS has to fully explore every branch to know which one was shortest. BFS returns the moment it dequeues the first leaf - that leaf sits on the shallowest level containing any leaf. On a left-skewed tree with a small right subtree, DFS wastes work walking the long left branch; BFS stops as soon as the small subtree's leaf is reached.
- **Shortest path in an unweighted graph or tree.** Same reason: the first time BFS reaches the target, it's via a shortest path. Every subsequent path is at least as long.
- **Level-only questions where past levels can be forgotten.** Right-side view keeps one node per level, then throws the level away. Average per level appends one float per level. Neither needs the whole tree's history in memory.

DFS wins for subtree-shaped answers - path sum, LCA, diameter, subtree aggregates - because those need "finish both children before combining," which is postorder, not level order.

## The `Next`-Pointer Trick For O(1) Space

For problems that populate `Next` pointers between siblings on the same level (LC 116, 117), you can BFS *without a queue at all* by reusing the `Next` pointers you've already built on level `L` as your "queue" for level `L+1`. The trick reduces space from O(width) to O(1).

The intuition: once every node on level `L` has its `Next` correctly set, you can walk level `L` by following those pointers, and at each node you have direct access to its children. Wire up the children's `Next` pointers as you walk, and by the time you reach the end of level `L`, level `L+1` is fully linked - ready to walk the same way.

For a perfect binary tree (LC 116), wiring is simple: at each parent `p`, set `p.Left.Next = p.Right`, and if `p.Next` exists, set `p.Right.Next = p.Next.Left` (the "bridge" across sibling parents).

```
Level L, already linked:      p1 -> p2 -> p3 -> nil
                              /\    /\    /\
                             a  b  c  d  e  f

Wire level L+1 by walking p1, p2, p3 in order:
  at p1: a.Next = b;    b.Next = p1.Next.Left = c
  at p2: c.Next = d;    d.Next = p2.Next.Left = e
  at p3: e.Next = f;    f.Next = nil (p3.Next is nil)

Level L+1, now linked:  a -> b -> c -> d -> e -> f -> nil
```

For a general binary tree (LC 117) the wiring is messier because children can be missing, so the "bridge" pointer isn't always `parent.Next.Left`. The clean fix is a **dummy head**: walk level `L` via `Next`, and for each child encountered append it to a linked chain rooted at a throwaway dummy node. When level `L` is exhausted, `dummy.Next` is the leftmost node of level `L+1`, and every `Next` on that level is already set.

## Complexity Reference

| Operation | Time | Space | Note |
|---|---|---|---|
| Level-order traversal (queue) | O(n) | O(w) | `w` = max width of the tree; for a perfect tree, `w ≈ n/2` |
| Zigzag / right-side view / averages | O(n) | O(w) | Same shape; per-level scratch is O(w) at most |
| Minimum depth (BFS with early return) | O(n) worst, O(1) best | O(w) | Best case: root is a leaf |
| Populate `Next` pointers via queue | O(n) | O(w) | Straightforward BFS |
| Populate `Next` pointers via `Next`-walk | O(n) | O(1) | The trick above; standard "follow-up" ask |

Worst-case width `w` is O(n) for a balanced tree at its widest level, so BFS is *not* asymptotically cheaper in space than a DFS recursion stack - a balanced tree costs O(log n) on the DFS side and O(n) on the BFS side. Choose BFS for correctness of the ordering, not for space savings.

## Recognizing The Pattern

Reach for tree BFS when the problem talks about:

- **Levels, depths, or rows.** "Level order," "kth level," "average per level," "right-side view," "bottom-up level order."
- **Shortest / minimum distance in an unweighted structure.** "Minimum depth," "shortest path from root to a target," "closest leaf."
- **Level-adjacent relationships.** "Next right node," "cousins" (same depth, different parents), "widest level."
- **Zigzag or alternating-direction visits.** Same BFS skeleton with a direction flag; no fancy stack tricks needed.

Reach for DFS instead when the answer folds up from children: path sums, LCA, diameter, subtree size or sum, serialization/deserialization, tree equality.

## Go Template Code

The standard per-level template. Every problem in this file starts from this skeleton and adds one or two lines inside the inner loop.

```go
type TreeNode struct {
    Val         int
    Left, Right *TreeNode
}

func bfsPerLevel(root *TreeNode) {
    if root == nil {
        return
    }
    q := []*TreeNode{root}
    for len(q) > 0 {
        size := len(q)
        for i := 0; i < size; i++ {
            node := q[i]
            _ = node // process node here
            if node.Left != nil {
                q = append(q, node.Left)
            }
            if node.Right != nil {
                q = append(q, node.Right)
            }
        }
        q = q[size:]
    }
}
```

> [!NOTE]
> `q = q[size:]` reslices the head forward; the backing array grows for the life of the traversal and is freed at the end. Fine for interview code and for `n ≤ 10^5`. Under production load with very large trees, prefer `container/list` or a ring-buffer queue so consumed memory is actually reclaimed as the traversal advances.

### Direction-Flipping Template (Zigzag)

Same skeleton, but write into a fixed-size level slice at either the head or the tail based on a flag:

```go
level := make([]int, size)
for i := 0; i < size; i++ {
    node := q[i]
    if leftToRight {
        level[i] = node.Val
    } else {
        level[size-1-i] = node.Val
    }
    // ... enqueue children as normal
}
leftToRight = !leftToRight
```

No stack-swap needed. The queue stays FIFO left-to-right at all times; only the *write position* into the output slice flips.

## Where JavaScript Differs

Go slices used as queues are cheap to `append` at the tail, and `q[size:]` reslices in O(1). JavaScript has no direct equivalent: `Array.prototype.shift()` is O(n) because it reindexes the array. Two common workarounds:

- Use a two-pointer index into the same array: keep a `head` counter and read `q[head++]` instead of shifting. Same idea as the Go `q = q[size:]` trick.
- Use a proper deque library, or a linked-list-backed queue, when performance actually matters.

Everything else - the level-boundary snapshot, the invariant, the templates - carries over unchanged.

## LeetCode Problems

### 1. Binary Tree Level Order Traversal - [#102 (Medium)](https://leetcode.com/problems/binary-tree-level-order-traversal/)

Return the values of a binary tree's nodes as a list of lists, one list per level.

<details>
<summary>Brute Force</summary>

DFS with a `depth` argument. On each recursive call, append the node's value to `buckets[depth]` (extending the slice if needed), then recurse into left and right with `depth+1`. Time O(n), space O(n) for the recursion stack plus the output.

The waste: you carry every level's bucket in memory at once, and the output has to be constructed by depth-tagged appends rather than being built level by level. It works, but for level-only questions BFS gives the same answer in one pass without indexing into a growing 2D structure.
</details>

<details>
<summary>Hint 1</summary>

The answer wants nodes grouped by depth. If you had a way to process every node at depth `k` before touching any node at depth `k+1`, one pass would build the output directly. What data structure enforces "process the oldest queued item first"?
</details>

<details>
<summary>Hint 2</summary>

FIFO queue. But you also need to know when one level ends and the next begins. At the top of each outer iteration, snapshot `size = len(queue)` - that's exactly the count of nodes on the current level. Process that many items, enqueuing children as you go; anything you enqueue lands after the boundary.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(w) where w is the max level width
func levelOrder(root *TreeNode) [][]int {
    if root == nil {
        return [][]int{}
    }
    res := [][]int{}
    q := []*TreeNode{root}
    for len(q) > 0 {
        size := len(q)
        level := make([]int, 0, size)
        for i := 0; i < size; i++ {
            node := q[i]
            level = append(level, node.Val)
            if node.Left != nil {
                q = append(q, node.Left)
            }
            if node.Right != nil {
                q = append(q, node.Right)
            }
        }
        q = q[size:]
        res = append(res, level)
    }
    return res
}
```
</details>

### 2. Binary Tree Zigzag Level Order Traversal - [#103 (Medium)](https://leetcode.com/problems/binary-tree-zigzag-level-order-traversal/)

Same as level order, but alternate direction per level: level 0 left-to-right, level 1 right-to-left, level 2 left-to-right, and so on.

<details>
<summary>Brute Force</summary>

Do a plain level-order traversal, then reverse every odd-indexed level as a post-processing step. Time O(n), space O(n). Correct, but wasteful: you build each level in one direction, then walk it again to flip it.
</details>

<details>
<summary>Hint 1</summary>

The queue must stay FIFO for the level-boundary invariant to hold - swapping to a stack would break the ordering guarantee entirely. But the *output slice* for each level doesn't have to be written left-to-right just because the queue is read left-to-right.
</details>

<details>
<summary>Hint 2</summary>

Pre-allocate the level's output slice with `make([]int, size)` and write into it by index. On left-to-right levels, write at position `i`. On right-to-left levels, write at position `size - 1 - i`. Flip the direction flag between outer iterations. No reversal step, no stack swap.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(w)
func zigzagLevelOrder(root *TreeNode) [][]int {
    if root == nil {
        return [][]int{}
    }
    res := [][]int{}
    q := []*TreeNode{root}
    leftToRight := true
    for len(q) > 0 {
        size := len(q)
        level := make([]int, size)
        for i := 0; i < size; i++ {
            node := q[i]
            if leftToRight {
                level[i] = node.Val
            } else {
                level[size-1-i] = node.Val
            }
            if node.Left != nil {
                q = append(q, node.Left)
            }
            if node.Right != nil {
                q = append(q, node.Right)
            }
        }
        q = q[size:]
        res = append(res, level)
        leftToRight = !leftToRight
    }
    return res
}
```
</details>

### 3. Binary Tree Right Side View - [#199 (Medium)](https://leetcode.com/problems/binary-tree-right-side-view/)

Imagine standing to the right of a tree. Return the values of the nodes visible from that side, top to bottom.

<details>
<summary>Brute Force</summary>

DFS with depth-tagging: for every node, if `depth == len(res)` (i.e., no node has been recorded for this depth yet), append it. Visit right subtree before left so the first node seen at each depth is the rightmost. Time O(n), space O(h) for the recursion stack.

Not really wasteful - it's a valid O(n) solution - but it hinges on a subtle "visit right first" ordering that isn't obviously correct on first glance. BFS makes the intent explicit: for each level, the last node processed is the rightmost visible one.
</details>

<details>
<summary>Hint 1</summary>

"Rightmost node visible on level `k`" is the last node processed by a left-to-right BFS on that level. You don't need to keep the whole level, just the last value.
</details>

<details>
<summary>Hint 2</summary>

Inside the inner loop, check whether the current index equals `size - 1`. If so, that node is the rightmost on this level - append its value to the result. No extra bookkeeping.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(w)
func rightSideView(root *TreeNode) []int {
    res := []int{}
    if root == nil {
        return res
    }
    q := []*TreeNode{root}
    for len(q) > 0 {
        size := len(q)
        for i := 0; i < size; i++ {
            node := q[i]
            if i == size-1 {
                res = append(res, node.Val)
            }
            if node.Left != nil {
                q = append(q, node.Left)
            }
            if node.Right != nil {
                q = append(q, node.Right)
            }
        }
        q = q[size:]
    }
    return res
}
```

The rightmost visible node on a level is not necessarily on the right subtree of the root: a deep-left branch can be the rightmost node on its level if the right subtree ended earlier. BFS-with-`i == size-1` handles that case with no extra code, because "last processed at this level" is a purely positional check.
</details>

### 4. Average Of Levels In Binary Tree - [#637 (Easy)](https://leetcode.com/problems/average-of-levels-in-binary-tree/)

Return the average value of every level of the tree as a `[]float64`.

<details>
<summary>Brute Force</summary>

DFS with depth-tagging: build a `sums []int64` and `counts []int` per depth, then divide element-wise at the end. Time O(n), space O(h). Works fine, but again forces you to hold every level's aggregate at once instead of emitting one float per level as you go.
</details>

<details>
<summary>Hint 1</summary>

You already know how many nodes are on the current level - that's the `size` snapshot. Sum their values in the inner loop and divide by `size`. One float appended to the result per level, no per-level state kept afterward.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(w)
func averageOfLevels(root *TreeNode) []float64 {
    res := []float64{}
    if root == nil {
        return res
    }
    q := []*TreeNode{root}
    for len(q) > 0 {
        size := len(q)
        sum := 0
        for i := 0; i < size; i++ {
            node := q[i]
            sum += node.Val
            if node.Left != nil {
                q = append(q, node.Left)
            }
            if node.Right != nil {
                q = append(q, node.Right)
            }
        }
        q = q[size:]
        res = append(res, float64(sum)/float64(size))
    }
    return res
}
```

> [!WARNING]
> LeetCode's constraints allow node values up to `2^31 - 1` and up to `10^4` nodes per level. A single-level sum can exceed `int32` range but stays well inside `int64`. Go's `int` is 32 or 64 bits depending on the platform (spec) but is 64-bit on every architecture LeetCode's grader runs, so `sum int` is safe here. For portable code, use `int64` explicitly; in another language where `int` is 32-bit, definitely use a 64-bit accumulator.
</details>

### 5. Cousins In Binary Tree - [#993 (Easy)](https://leetcode.com/problems/cousins-in-binary-tree/)

Two nodes are cousins if they are at the same depth but have different parents. Given a tree and two distinct values `x` and `y`, return whether the nodes with those values are cousins.

<details>
<summary>Brute Force</summary>

Two DFS passes: one to find the depth and parent of `x`, another for `y`. Compare. Time O(n), space O(h). Simple and correct; the drawback is two full traversals for a question that can be answered in one BFS pass with early termination.
</details>

<details>
<summary>Hint 1</summary>

"Same depth" is a per-level condition - the BFS boundary trick puts you exactly at the right level. "Different parents" is a per-parent condition - you can spot it while enqueuing children.
</details>

<details>
<summary>Hint 2</summary>

While processing a level, watch for whether `x` and `y` show up. Two rules to close the case: if both are found in the same level, they're at the same depth (candidate cousins); if you also see that they share a parent (a node whose two children are exactly `x` and `y`), reject. If only one of the two appears in a level, the other is at a different depth - reject.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) worst case, often less due to early return | Space: O(w)
func isCousins(root *TreeNode, x, y int) bool {
    if root == nil {
        return false
    }
    q := []*TreeNode{root}
    for len(q) > 0 {
        size := len(q)
        foundX, foundY := false, false
        for i := 0; i < size; i++ {
            node := q[i]
            // Sibling check: if x and y are children of the same parent, not cousins.
            if node.Left != nil && node.Right != nil {
                a, b := node.Left.Val, node.Right.Val
                if (a == x && b == y) || (a == y && b == x) {
                    return false
                }
            }
            if node.Val == x {
                foundX = true
            }
            if node.Val == y {
                foundY = true
            }
            if node.Left != nil {
                q = append(q, node.Left)
            }
            if node.Right != nil {
                q = append(q, node.Right)
            }
        }
        q = q[size:]
        if foundX && foundY {
            return true
        }
        if foundX || foundY {
            return false
        }
    }
    return false
}
```

The sibling check is *inside* the enqueue loop, on the parent, because that's where the parent-child relationship is directly visible without needing to store `parent` pointers alongside every queued node.
</details>

### 6. Populating Next Right Pointers In Each Node - [#116 (Medium)](https://leetcode.com/problems/populating-next-right-pointers-in-each-node/)

You're given a **perfect** binary tree (every parent has two children, every leaf at the same depth) where each node has a `Next` pointer initially set to `nil`. Populate `Next` so that it points to the next node on the same level, or `nil` if it's the last node on that level.

<details>
<summary>Brute Force</summary>

Plain BFS. For each level, walk the queue and set `q[i].Next = q[i+1]` for all but the last node. Time O(n), space O(w) - up to n/2 in a perfect tree at the widest level.

Works, uses the standard template. The follow-up asks for O(1) auxiliary space, which forces the `Next`-walk trick.
</details>

<details>
<summary>Hint 1</summary>

Once level `L`'s `Next` pointers are set, walking level `L` requires zero extra storage - start from the leftmost node and follow `Next` until you hit `nil`. At each node on level `L` you have direct access to both children on level `L+1`. What has to happen at each parent so that after one pass across level `L`, level `L+1` is fully linked?
</details>

<details>
<summary>Hint 2</summary>

At each parent `p`: (1) `p.Left.Next = p.Right` handles the intra-parent link. (2) If `p.Next != nil`, then `p.Right.Next = p.Next.Left` bridges from `p`'s right child to the next parent's left child. In a perfect tree, `p.Next.Left` is always non-nil when `p.Next` is non-nil, so no null check is needed. After the pass, drop down to `leftmost.Left` and repeat.
</details>

<details>
<summary>Follow-Up: O(1) Extra Space</summary>

The problem's own follow-up: solve it without recursion and without an explicit queue. The `Next`-walk trick answers this directly. The recursion stack isn't counted against the space bound in the official problem statement's typical framing, but the cleanest presentation is the iterative walk below - it's also faster in practice.

```go
// Time: O(n) | Space: O(1) auxiliary (ignoring the Next pointers themselves)
type Node struct {
    Val               int
    Left, Right, Next *Node
}

func connect(root *Node) *Node {
    if root == nil {
        return nil
    }
    leftmost := root
    for leftmost.Left != nil { // stop when we've linked the last non-leaf level
        cur := leftmost
        for cur != nil {
            cur.Left.Next = cur.Right
            if cur.Next != nil {
                cur.Right.Next = cur.Next.Left
            }
            cur = cur.Next
        }
        leftmost = leftmost.Left
    }
    return root
}
```

The outer loop drops down one level per iteration; the inner loop links level `L+1` using level `L`'s already-set `Next` chain. Because the tree is perfect, `cur.Left` and `cur.Right` are never nil until you hit a leaf, and the outer loop's `leftmost.Left != nil` guard stops exactly at the leaves.
</details>

### 7. Populating Next Right Pointers In Each Node II - [#117 (Medium)](https://leetcode.com/problems/populating-next-right-pointers-in-each-node-ii/)

Same as LC 116 but the tree is *arbitrary* - nodes may have one child, no children, or two.

<details>
<summary>Brute Force</summary>

Plain BFS with a queue, same shape as LC 116's brute force. Time O(n), space O(w). Correct, no missing-child issues to think about, but doesn't meet the O(1) follow-up.
</details>

<details>
<summary>Hint 1</summary>

The LC 116 trick breaks because `p.Next.Left` may be nil - the bridge target might be `p.Next.Right`, or `p.Next.Next.Left`, or further along. Rather than search for the right bridge at every parent, decouple "walk level `L`" from "figure out the next level's leftmost node." What if you build the next level's `Next` chain into a throwaway list as you walk?
</details>

<details>
<summary>Hint 2</summary>

Use a dummy head node for level `L+1`. Walk level `L` via `Next`; at each node, if it has a left child, append that child to the dummy's chain (advance a `tail` pointer). Same for the right child. When the walk of level `L` ends, `dummy.Next` is the leftmost node of level `L+1`, and every `Next` on that level is already correctly set as a byproduct of appending in order. Drop down and repeat.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1) auxiliary
func connect(root *Node) *Node {
    if root == nil {
        return nil
    }
    cur := root
    for cur != nil {
        dummy := &Node{}
        tail := dummy
        for n := cur; n != nil; n = n.Next {
            if n.Left != nil {
                tail.Next = n.Left
                tail = tail.Next
            }
            if n.Right != nil {
                tail.Next = n.Right
                tail = tail.Next
            }
        }
        cur = dummy.Next
    }
    return root
}
```

The dummy node exists only to avoid a special case for "first child appended at this level." When the walk finishes, `dummy` is discarded; only `dummy.Next` matters. Because we only ever append existing children in left-then-right order, the resulting `Next` chain is guaranteed to be the correct left-to-right ordering of level `L+1`.
</details>

### 8. Minimum Depth Of Binary Tree - [#111 (Easy)](https://leetcode.com/problems/minimum-depth-of-binary-tree/)

Return the length of the shortest root-to-leaf path.

<details>
<summary>Brute Force</summary>

DFS from the root, tracking depth, returning `1 + min(minDepth(left), minDepth(right))` - with the standard trap: if a child is nil, you must treat that side as "no path" (don't take its zero), otherwise a root with only a left subtree returns 1 instead of the actual left subtree's depth.

Time O(n), space O(h). Correct but doesn't exploit the shape of the answer: DFS visits every node even when a shallow leaf exists near the root.
</details>

<details>
<summary>Hint 1</summary>

BFS visits nodes in order of depth. The very first leaf it dequeues sits on the shallowest level that contains any leaf. Everything deeper is irrelevant.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) worst; often much less due to early return | Space: O(w)
func minDepth(root *TreeNode) int {
    if root == nil {
        return 0
    }
    q := []*TreeNode{root}
    depth := 1
    for len(q) > 0 {
        size := len(q)
        for i := 0; i < size; i++ {
            node := q[i]
            if node.Left == nil && node.Right == nil {
                return depth
            }
            if node.Left != nil {
                q = append(q, node.Left)
            }
            if node.Right != nil {
                q = append(q, node.Right)
            }
        }
        q = q[size:]
        depth++
    }
    return depth
}
```

The "empty child = no path" trap that ambushes the DFS solution disappears entirely here: BFS only asks "is this a leaf?" (both children nil), and returns immediately when it finds one. On a tree with one deep branch and one shallow leaf, BFS returns after touching only the shallow branch's ancestors, while DFS walks the deep branch to completion before comparing.
</details>

## References

- [LeetCode: Breadth-First Search Tag](https://leetcode.com/tag/breadth-first-search/)
- [NeetCode: Trees roadmap](https://neetcode.io/roadmap)
- Cormen, Leiserson, Rivest, Stein - *Introduction to Algorithms* (4th ed.), §20.2 for BFS correctness (shortest-path property in unweighted graphs, which specialises directly to trees)
- Sedgewick & Wayne - *Algorithms* (4th ed.), §4.1 (Undirected Graphs) for the queue-vs-stack contrast between BFS and DFS