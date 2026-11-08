---
Status: 🌳 Evergreen
Created: 2026-11-00
Last Updated: 2026-11-09
---

# Trees

## Table of Contents

1. [What It Is and Why It Exists](#what-it-is-and-why-it-exists)
2. [Terminology](#terminology)
3. [Tree Shapes](#tree-shapes)
4. [Representations](#representations)
5. [Traversals](#traversals)
6. [Binary Search Tree](#binary-search-tree)
7. [Balanced BSTs](#balanced-bsts)
8. [When Not to Use a BST](#when-not-to-use-a-bst)
9. [Go Template Code](#go-template-code)
10. [Where JavaScript Differs](#where-javascript-differs)
11. [LeetCode Problems](#leetcode-problems)
12. [References](#references)

## What It Is and Why It Exists

A tree is a **connected, acyclic graph with one designated root**, where every node except the root has exactly one parent. That single-parent rule is what separates a tree from a general graph, and every useful tree property falls out of it: exactly one path between any two nodes, `n - 1` edges for `n` nodes, and a recursive shape where every child is itself the root of a smaller tree.

Trees exist because a lot of data is hierarchical, not linear. Arrays and linked lists offer at most one "next" element. A filesystem, an org chart, a parsed expression, or an HTML document has many children under each parent. A tree models that directly, and the recursive shape turns most operations into divide and conquer: handle the root, recurse into the children. Add an ordering rule (BST) and that recursion discards half the remaining data per step, giving `O(log n)` search, insert, and delete with ordered iteration for free.

The name is the picture drawn upside down: a single root at the top branching repeatedly until it ends in leaves. The rest of the vocabulary is borrowed from botany (root, branch, leaf) and genealogy (parent, child, sibling, ancestor).

### The Problems It Introduces

- **Height decides everything.** Every BST operation is `O(h)`. A balanced tree has `h = O(log n)`; inserting sorted keys into a plain BST produces a chain with `h = n - 1`, which is a linked list with extra steps.
- **Balancing is not free.** Keeping `h = O(log n)` needs rotations (AVL, Red-Black), which add real implementation complexity.
- **Deletion is the hard operation.** Removing a node with two children needs a successor swap, and the pointer surgery is a classic interview bug.
- **Cache-unfriendly.** Each node is a separate heap allocation reached through a pointer. Walking a tree costs far more cache misses than walking an array of the same size, which is why on-disk indexes use wide B-trees instead of binary trees.
- **Recursion depth.** Recursive code on a degenerate tree goes `n` frames deep. Go grows goroutine stacks dynamically (up to 1 GB on 64-bit), so it rarely crashes, but other runtimes cap far lower. Iterative versions avoid the issue.

## Terminology

```
            (10)        depth 0
           /    \
         (5)    (15)    depth 1
         / \       \
       (3) (7)    (20)  depth 2   (3, 7, 20 are leaves)
```

| Term | Meaning |
|---|---|
| **Root** | The only node with no parent. |
| **Leaf** | A node with no children. |
| **Internal node** | A node with at least one child. |
| **Ancestor / Descendant** | `A` is an ancestor of `B` if `A` lies on the path from the root to `B`. |
| **Subtree** | A node together with all of its descendants. |
| **Depth of a node** | Edges from the root down to the node. Root depth is 0. |
| **Height of a node** | Edges on the longest path from the node down to a leaf. Leaf height is 0. |
| **Height of the tree** | Height of the root. In the tree above, 2. |

Depth is measured downward from the root, height upward from the leaves. Some sources (and many LeetCode problems) count nodes instead of edges, so a single-node tree has "depth 1". Check which convention a problem uses before comparing numbers.

The bound worth memorising: a binary tree with `n` nodes has height between `⌊log₂ n⌋` (perfectly packed) and `n - 1` (a chain).

## Tree Shapes

```
Full                       Complete                   Perfect
(every node has 0 or 2)    (all levels full except    (all levels full)
                            the last, filled
                            left to right)

     (1)                        (1)                        (1)
    /   \                      /   \                      /   \
  (2)   (3)                  (2)   (3)                  (2)   (3)
  / \                        / \   /                    / \   / \
(4) (5)                    (4) (5)(6)                 (4)(5)(6)(7)
```

- **Full**: no node has exactly one child.
- **Complete**: no gaps when read level by level. This is the heap shape, and it is what makes array storage possible.
- **Perfect**: full and complete with all leaves at one depth. Height `h` means exactly `2^(h+1) - 1` nodes.
- **Balanced**: height is `O(log n)`. Each balanced BST defines it precisely (AVL: subtree heights differ by at most 1 at every node).
- **Degenerate**: every node has at most one child. A linked list in disguise.

## Representations

**Linked** (the default for general trees and BSTs, and what every LeetCode tree problem uses):

```go
type TreeNode struct {
    Val         int
    Left, Right *TreeNode
}
```

**Array-backed** (complete trees only):

```
        (A)              index:  0  1  2  3  4  5  6
       /   \             value: [A, B, C, D, E, F, G]
     (B)   (C)
     / \   / \           for the node at index i:
   (D) (E)(F)(G)           left   = 2i + 1
                           right  = 2i + 2
                           parent = (i - 1) / 2
```

A complete tree has no gaps in level order, so positions map to indices with pure arithmetic and no pointers. Heaps and segment trees rely on this. It fails for arbitrary shapes: a chain of height `h` would need `2^(h+1) - 1` slots to hold `h + 1` nodes.

## Traversals

A traversal visits every node exactly once. The three depth-first orders are named by **when the root is visited relative to its subtrees**: pre (before), in (between), post (after). Level order is breadth-first.

```
       (1)
      /   \
    (2)   (3)          Preorder:   1 2 4 5 3 6   root, left, right
   /  \     \          Inorder:    4 2 5 1 3 6   left, root, right
 (4)  (5)   (6)        Postorder:  4 5 2 6 3 1   left, right, root
                       Level:      1 2 3 4 5 6   top to bottom, left to right
```

| Order | Natural fit |
|---|---|
| Preorder | Copying or serialising a tree: a parent must exist before its children can be attached. |
| Inorder | BSTs. Inorder of a BST is sorted, which is the whole point of the BST invariant. |
| Postorder | Anything that needs results from both children first: height, subtree sums, freeing nodes, balance checks. |
| Level order | Anything defined by depth: per-level aggregates, shortest root-to-node distance. |

### Making Traversals Iterative

Recursion uses the call stack implicitly; the iterative version makes that stack an explicit `[]*TreeNode`.

- **Preorder**: pop a node, visit it, push right then left (so left pops first).
- **Inorder**: push the entire left spine, pop and visit, then move to the popped node's right child and repeat.
- **Postorder**: run preorder with the push order swapped (left then right), which produces root, right, left. Reverse it to get left, right, root.

Preorder stack trace on the tree above:

```
pop 1, visit -> push 3, 2   stack [3 2]
pop 2, visit -> push 5, 4   stack [3 5 4]
pop 4, visit                stack [3 5]
pop 5, visit                stack [3]
pop 3, visit -> push 6      stack [6]
pop 6, visit                stack []
visited: 1 2 4 5 3 6
```

**Morris traversal** does inorder in `O(1)` extra space by temporarily pointing each node's inorder predecessor back at it, then removing that link on the second visit. It is worth naming when an interviewer asks for constant space; the explicit-stack version is what you would actually write.

## Binary Search Tree

A BST is a binary tree with one invariant:

> For every node `x`, every value in `x`'s left subtree is less than `x.Val`, and every value in its right subtree is greater.

The invariant applies to the **whole subtree**, not only the direct children. That detail is the source of the most common BST bug.

One comparison at a node rules out an entire subtree, which is binary search on a sorted array turned into a structure that also supports cheap inserts. Hence the name. And because left < root < right holds recursively, inorder traversal emits values in sorted order.

### Search and Insert

Compare, go left or right, repeat. Search returns when it finds the value or falls off the tree. Insert falls off the tree at exactly the position where the value belongs and attaches a new leaf there. Insert 4:

```
Before                      After
     (5)                         (5)
    /   \                       /   \
  (3)   (8)                   (3)   (8)
        /                       \   /
      (7)                      (4)(7)

4 < 5 go left; 4 > 3 go right; empty slot, attach.
```

Plain inserts never restructure existing nodes, so **insertion order decides the shape**. Inserting `1, 2, 3, 4, 5` in that order gives:

```
(1)
  \
  (2)
    \
    (3)
      \
      (4)
        \
        (5)
```

Height `n - 1`, every operation `O(n)`. Sorted or nearly sorted input (timestamps, auto-increment IDs) is common in practice, which is why production code never uses an unbalanced BST.

### Delete

Three cases, by how many children the target has.

**Scenario A: no children.** Remove it; the parent's pointer becomes `nil`.

**Scenario B: one child.** Splice it out; the parent points to the target's only child.

```
Delete 3 (one child, 4):

Before          After
     (5)             (5)
    /   \           /   \
  (3)   (8)       (4)   (8)
    \
    (4)
```

**Scenario C: two children.** There are two subtrees but only one parent slot to reconnect. Instead, copy the **inorder successor's** value (smallest value in the right subtree: go right once, then left until you can't) into the target, then delete the successor from the right subtree. The successor has no left child, so that second delete is always Scenario A or B.

```
Delete 5 (two children). Successor = 6.

Before                After
     (5)                   (6)        <- target now holds the successor's value
    /   \                 /   \
  (3)   (8)             (3)   (8)
        / \                   / \
      (6) (9)               (7) (9)   <- old successor node removed;
        \                                its right child 7 moved up
        (7)
```

The inorder predecessor (largest in the left subtree) works symmetrically. The full code is in the Delete Node in a BST problem below.

## Balanced BSTs

Balanced BSTs fix the degenerate-shape problem by restoring `O(log n)` height after each insert or delete using **rotations**: local rewrites that change shape without breaking the BST invariant.

```
Right rotation at C:

Before          After
     (C)             (B)
     /              /   \
   (B)            (A)   (C)
   /
 (A)

Inorder before: A B C.  Inorder after: A B C.  Height: 2 -> 1.
```

| | AVL | Red-Black |
|---|---|---|
| Invariant | Subtree heights differ by at most 1 at every node | Nodes colored red/black; no red node has a red child; every root-to-nil path has the same number of black nodes |
| Max height | about `1.44 log₂ n` | `2 log₂(n + 1)` |
| Trade-off | Shorter tree, faster lookups, more rotations on write | Taller, but fewer rotations per update |
| Name | Initials of Adelson-Velsky and Landis (1962) | The two node colors, from Guibas and Sedgewick (1978) |
| Seen in | Read-heavy in-memory indexes | Java's `TreeMap` (documented as Red-Black), the major C++ `std::map` implementations, the Linux kernel's `rbtree` |

Interviews expect the why, not the implementation: an unbalanced BST degrades to `O(n)`, rotations restore height locally, AVL trades write cost for lookup speed, Red-Black the reverse.

**Go has no balanced BST in its standard library.** `container/heap` is a priority queue and `container/list` is a doubly linked list; neither is a search tree. For an ordered map, either keep a sorted slice with `sort.Search` (good when reads dominate) or use `github.com/google/btree` (the B-tree etcd uses for its in-memory index).

## When Not to Use a BST

| Need | Better Choice | Why |
|---|---|---|
| Point lookup only | Hash map | `O(1)` average, no ordering cost |
| Sorted data, rarely modified | Sorted slice + binary search | Contiguous, cache-friendly, no pointers |
| Repeated min or max only | Heap | No total ordering maintained |
| On-disk index | B-tree | High fanout means far fewer disk reads per lookup |
| Range sums with point updates | Fenwick or segment tree | Built for exactly this |

## Go Template Code

```go
type TreeNode struct {
    Val         int
    Left, Right *TreeNode
}

// Recursive DFS. Change where the append sits to switch orders.
func preorder(root *TreeNode, out *[]int) {
    if root == nil {
        return
    }
    *out = append(*out, root.Val)
    preorder(root.Left, out)
    preorder(root.Right, out)
}

func inorder(root *TreeNode, out *[]int) {
    if root == nil {
        return
    }
    inorder(root.Left, out)
    *out = append(*out, root.Val)
    inorder(root.Right, out)
}

func postorder(root *TreeNode, out *[]int) {
    if root == nil {
        return
    }
    postorder(root.Left, out)
    postorder(root.Right, out)
    *out = append(*out, root.Val)
}

// Iterative preorder: push right before left so left is popped first.
func preorderIter(root *TreeNode) []int {
    if root == nil {
        return nil
    }
    out := []int{}
    stack := []*TreeNode{root}
    for len(stack) > 0 {
        n := stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        out = append(out, n.Val)
        if n.Right != nil {
            stack = append(stack, n.Right)
        }
        if n.Left != nil {
            stack = append(stack, n.Left)
        }
    }
    return out
}

// Iterative inorder: push the left spine, pop and visit, move right.
func inorderIter(root *TreeNode) []int {
    out := []int{}
    stack := []*TreeNode{}
    curr := root
    for curr != nil || len(stack) > 0 {
        for curr != nil {
            stack = append(stack, curr)
            curr = curr.Left
        }
        curr = stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        out = append(out, curr.Val)
        curr = curr.Right
    }
    return out
}

// Iterative postorder: build root-right-left, then reverse.
func postorderIter(root *TreeNode) []int {
    if root == nil {
        return nil
    }
    out := []int{}
    stack := []*TreeNode{root}
    for len(stack) > 0 {
        n := stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        out = append(out, n.Val)
        if n.Left != nil {
            stack = append(stack, n.Left)
        }
        if n.Right != nil {
            stack = append(stack, n.Right)
        }
    }
    for i, j := 0, len(out)-1; i < j; i, j = i+1, j-1 {
        out[i], out[j] = out[j], out[i]
    }
    return out
}

// Level order: process the queue one level (size snapshot) at a time.
func levelOrder(root *TreeNode) [][]int {
    if root == nil {
        return nil
    }
    out := [][]int{}
    queue := []*TreeNode{root}
    for len(queue) > 0 {
        size := len(queue)
        level := make([]int, 0, size)
        for _, n := range queue[:size] {
            level = append(level, n.Val)
            if n.Left != nil {
                queue = append(queue, n.Left)
            }
            if n.Right != nil {
                queue = append(queue, n.Right)
            }
        }
        queue = queue[size:]
        out = append(out, level)
    }
    return out
}

// BST search: iterative, O(h) time, O(1) space.
func bstSearch(root *TreeNode, val int) *TreeNode {
    for root != nil && root.Val != val {
        if val < root.Val {
            root = root.Left
        } else {
            root = root.Right
        }
    }
    return root
}

// BST insert: returns the subtree root, so callers write
// root = bstInsert(root, v). Duplicates are ignored.
func bstInsert(root *TreeNode, val int) *TreeNode {
    if root == nil {
        return &TreeNode{Val: val}
    }
    if val < root.Val {
        root.Left = bstInsert(root.Left, val)
    } else if val > root.Val {
        root.Right = bstInsert(root.Right, val)
    }
    return root
}
```

**Return the subtree root from every mutating function.** `root.Left = bstInsert(root.Left, v)` handles the "subtree was empty" and "subtree root was deleted" cases without parent pointers or pointer-to-pointer tricks. A reviewer will flag mutating helpers that return nothing, because they silently fail when the root itself changes.

## Where JavaScript Differs

- **Stack depth is a hard limit.** V8 throws `RangeError: Maximum call stack size exceeded` after roughly ten thousand frames (the exact number depends on frame size). A recursive traversal of a degenerate tree with `10^5` nodes fails in JS where Go would just grow the stack. Use the iterative versions.
- **No ordered map.** `Map` iterates in insertion order, not key order. There is no built-in `TreeMap` equivalent.
- **`null` children.** LeetCode's JS node uses `null` for missing children; check `node === null` (or just `!node`), not `undefined`.

Algorithms are identical.

## LeetCode Problems

Traversal-pattern problems (path sums, general LCA, diameter, serialisation, level-order variants) belong to `patterns/tree-dfs.md` and `patterns/tree-bfs.md`. The problems here exercise the structure itself: height and balance, traversal properties, and the BST invariant.

### 1. Balanced Binary Tree — [LeetCode #110](https://leetcode.com/problems/balanced-binary-tree/) · Easy

**Problem**

Given a binary tree, determine whether it is height-balanced: at every node, the heights of the left and right subtrees differ by at most one. The tree has 0 to 5000 nodes.

**Brute Force**

At every node, compute the height of each subtree from scratch and compare. Height costs `O(subtree size)`, and it is recomputed at every ancestor, so a degenerate tree costs `O(n²)` time, `O(h)` stack space.

<details>
<summary>Hint 1</summary>
The brute force recomputes the same subtree heights repeatedly. If one pass returned each subtree's height to its parent, every node could compare its children's heights at no extra cost. Which traversal order makes both children's results available before the parent is processed?
</details>

<details>
<summary>Hint 2</summary>
Compute heights bottom-up. If a subtree is already unbalanced, there is no need to keep computing: return a sentinel like `-1` and let it propagate to the root.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(h)
func isBalanced(root *TreeNode) bool {
    return checkHeight(root) != -1
}

// Returns the subtree height (nil = 0), or -1 if any node inside is unbalanced.
func checkHeight(n *TreeNode) int {
    if n == nil {
        return 0
    }
    l := checkHeight(n.Left)
    if l == -1 {
        return -1
    }
    r := checkHeight(n.Right)
    if r == -1 {
        return -1
    }
    if l-r > 1 || r-l > 1 {
        return -1
    }
    return 1 + max(l, r)
}
```
</details>

### 2. Construct Binary Tree from Preorder and Inorder Traversal — [LeetCode #105](https://leetcode.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal/) · Medium

**Problem**

Given the `preorder` and `inorder` traversals of a binary tree with unique values, rebuild and return the tree. Up to 3000 nodes.

**Brute Force**

Take `preorder[0]` as the root, scan `inorder` linearly to find it, split both arrays into left and right parts, and recurse on slices. The linear scan at each level makes a degenerate tree `O(n²)` time.

<details>
<summary>Hint 1</summary>
Preorder always lists a subtree's root first. Inorder lists everything in a node's left subtree before it and everything in its right subtree after it. Once you know the root, what does its position in inorder tell you?
</details>

<details>
<summary>Hint 2</summary>
Pre-build a map from value to inorder index so the split point is found in `O(1)`. Then consume preorder with a single moving index: building the left subtree fully before the right subtree consumes preorder in exactly the order it was written.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(n)
func buildTree(preorder []int, inorder []int) *TreeNode {
    idx := make(map[int]int, len(inorder))
    for i, v := range inorder {
        idx[v] = i
    }
    next := 0 // next unused root in preorder
    var build func(lo, hi int) *TreeNode // builds from inorder[lo..hi]
    build = func(lo, hi int) *TreeNode {
        if lo > hi {
            return nil
        }
        val := preorder[next]
        next++
        mid := idx[val]
        root := &TreeNode{Val: val}
        root.Left = build(lo, mid-1)
        root.Right = build(mid+1, hi)
        return root
    }
    return build(0, len(inorder)-1)
}
```

**Why this works:** Neither traversal alone determines the tree (many trees share the same preorder), but together they do, as long as values are unique. With duplicates, the root's position in inorder becomes ambiguous.
</details>

### 3. Validate Binary Search Tree — [LeetCode #98](https://leetcode.com/problems/validate-binary-search-tree/) · Medium

**Problem**

Determine whether a binary tree is a valid BST: every value in a node's left subtree is strictly less than the node, every value in its right subtree is strictly greater, and both subtrees are BSTs. Values span the full `int32` range.

**Brute Force**

At each node, find the maximum of the left subtree and the minimum of the right subtree, and compare against the node. Those scans repeat at every ancestor: `O(n²)` time on a degenerate tree.

Checking only `left.Val < node.Val < right.Val` is **not** a slower correct approach; it is wrong:

```
     (5)
    /
  (3)
    \
    (6)    6 > 3 passes the local check, but 6 sits in 5's left subtree.
```

<details>
<summary>Hint 1</summary>
Every ancestor constrains a node, not just its parent. Going left from a node tightens the upper limit for everything below; going right tightens the lower limit. What if each node received the range it is allowed to fall in?
</details>

<details>
<summary>Hint 2</summary>
Start the root with an open range wider than any valid value. Pass `(low, node.Val)` to the left child and `(node.Val, high)` to the right. A node outside its range, or equal to a bound, fails.
</details>

<details>
<summary>Solution (Go)</summary>

```go
import "math"

// Time: O(n) | Space: O(h)
func isValidBST(root *TreeNode) bool {
    return validate(root, math.MinInt64, math.MaxInt64)
}

func validate(n *TreeNode, low, high int) bool {
    if n == nil {
        return true
    }
    if n.Val <= low || n.Val >= high {
        return false
    }
    return validate(n.Left, low, n.Val) && validate(n.Right, n.Val, high)
}
```

**Why this works:** Using `int64` limits as the initial bounds keeps nodes holding `math.MinInt32` or `math.MaxInt32` valid (on a 64-bit platform, where Go's `int` is 64 bits). Starting at `math.MinInt32` would wrongly reject a node equal to it. An equivalent check: inorder traversal must be strictly increasing.
</details>

### 4. Delete Node in a BST — [LeetCode #450](https://leetcode.com/problems/delete-node-in-a-bst/) · Medium

**Problem**

Given the root of a BST and a `key`, delete the node with that value if it exists and return the (possibly new) root. Values are unique; up to `10^4` nodes.

**Brute Force**

Collect all values by inorder traversal, drop `key`, and rebuild a BST from the remaining sorted values. Always `O(n)` time and `O(n)` space, regardless of where the key lives, and every node is reallocated.

<details>
<summary>Hint 1</summary>
Locating the key is an ordinary BST search. The difficulty is what replaces it. How many children can the target have, and which of those cases can be resolved by a single pointer change?
</details>

<details>
<summary>Hint 2</summary>
Zero or one child: the parent can point straight at the remaining child (or `nil`). Two children: the smallest value in the right subtree is larger than everything on the left and smaller than everything else on the right, so it can take the target's place. Copy it in, then delete it from the right subtree.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(h) | Space: O(h)
func deleteNode(root *TreeNode, key int) *TreeNode {
    if root == nil {
        return nil // key not present
    }
    switch {
    case key < root.Val:
        root.Left = deleteNode(root.Left, key)
    case key > root.Val:
        root.Right = deleteNode(root.Right, key)
    default:
        if root.Left == nil {
            return root.Right
        }
        if root.Right == nil {
            return root.Left
        }
        succ := root.Right
        for succ.Left != nil {
            succ = succ.Left
        }
        root.Val = succ.Val
        root.Right = deleteNode(root.Right, succ.Val)
    }
    return root
}
```
</details>

**Follow-Up: Could you solve it with time complexity O(height of tree)?**

The solution above already is. The search descends one path (`O(h)`). In the two-children case, finding the successor continues down the right subtree, and deleting it walks that same right-subtree path again, so the total is at most about `2h` steps, still `O(h)`. The rebuild in Brute Force is the `O(n)` approach this rules out.

### 5. Lowest Common Ancestor of a Binary Search Tree — [LeetCode #235](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-search-tree/) · Medium

**Problem**

Given a BST and two nodes `p` and `q` that exist in it, return their lowest common ancestor: the deepest node that has both as descendants (a node counts as its own descendant). Values are unique.

**Brute Force**

Ignore the ordering and use the general binary-tree LCA: a postorder search that reports which targets each subtree contains, `O(n)` time. Or record the root-to-`p` and root-to-`q` paths and take the last node they share, `O(h)` time but `O(h)` extra space. Using the BST ordering drops the space to `O(1)`. (The general-tree LCA is covered in `patterns/tree-dfs.md`; this problem is here because the BST invariant changes the technique.)

<details>
<summary>Hint 1</summary>
Starting at the root, compare both target values with the current node. If both are smaller, where must their common ancestor be? If both are larger?
</details>

<details>
<summary>Hint 2</summary>
The first node where the two targets stop going the same direction (one on each side, or one of them equals the node) is the answer. Walk down a single path; no recursion needed.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(h) | Space: O(1)
func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
    for root != nil {
        switch {
        case p.Val < root.Val && q.Val < root.Val:
            root = root.Left
        case p.Val > root.Val && q.Val > root.Val:
            root = root.Right
        default:
            return root // split point
        }
    }
    return nil
}
```

**Why this works:** At the split node, `p` and `q` are in different subtrees (or one is the node itself), so any deeper node lies in only one of those subtrees and cannot contain both.
</details>

### 6. Kth Smallest Element in a BST — [LeetCode #230](https://leetcode.com/problems/kth-smallest-element-in-a-bst/) · Medium

**Problem**

Given a BST and an integer `k` (1-indexed), return the `k`th smallest value. `1 ≤ k ≤ n ≤ 10^4`.

**Brute Force**

Run a full inorder traversal into a slice and return `values[k-1]`. `O(n)` time and `O(n)` space even when `k` is 1.

<details>
<summary>Hint 1</summary>
The values come out sorted if you visit left subtree, node, right subtree. Do you need the whole sorted sequence, or only its first `k` items?
</details>

<details>
<summary>Hint 2</summary>
Use the explicit-stack version of that traversal so you can stop early. Decrement `k` each time a node is visited; the node that brings it to zero is the answer.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(h + k) | Space: O(h)
func kthSmallest(root *TreeNode, k int) int {
    stack := []*TreeNode{}
    curr := root
    for curr != nil || len(stack) > 0 {
        for curr != nil {
            stack = append(stack, curr)
            curr = curr.Left
        }
        curr = stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        k--
        if k == 0 {
            return curr.Val
        }
        curr = curr.Right
    }
    return -1 // unreachable for valid k
}
```
</details>

**Follow-Up: If the BST is modified often (insert and delete) and you need the kth smallest frequently, how would you optimize?**

Augment each node with the **size of its subtree**. Then the `k`th smallest is found by one root-to-leaf walk: if `k ≤ size(left)`, go left; if `k == size(left) + 1`, it is this node; otherwise subtract `size(left) + 1` and go right. Insert and delete update sizes on the way back up their path. Every operation becomes `O(h)`, independent of `k`. On a balanced BST that is `O(log n)`; a Red-Black tree augmented this way is called an **order-statistic tree** (CLRS Ch. 14).

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(h) per insert, delete, and query | Space: O(n)
type sizeNode struct {
    val, size   int // size = number of nodes in this subtree
    left, right *sizeNode
}

func size(n *sizeNode) int {
    if n == nil {
        return 0
    }
    return n.size
}

func insertSized(n *sizeNode, v int) *sizeNode {
    if n == nil {
        return &sizeNode{val: v, size: 1}
    }
    if v < n.val {
        n.left = insertSized(n.left, v)
    } else if v > n.val {
        n.right = insertSized(n.right, v)
    }
    n.size = 1 + size(n.left) + size(n.right)
    return n
}

func deleteSized(n *sizeNode, v int) *sizeNode {
    if n == nil {
        return nil
    }
    switch {
    case v < n.val:
        n.left = deleteSized(n.left, v)
    case v > n.val:
        n.right = deleteSized(n.right, v)
    default:
        if n.left == nil {
            return n.right
        }
        if n.right == nil {
            return n.left
        }
        succ := n.right
        for succ.left != nil {
            succ = succ.left
        }
        n.val = succ.val
        n.right = deleteSized(n.right, succ.val)
    }
    n.size = 1 + size(n.left) + size(n.right)
    return n
}

func kthSmallestSized(n *sizeNode, k int) int {
    for n != nil {
        leftSize := size(n.left)
        switch {
        case k <= leftSize:
            n = n.left
        case k == leftSize+1:
            return n.val
        default:
            k -= leftSize + 1
            n = n.right
        }
    }
    return -1
}
```
</details>

### 7. Binary Search Tree Iterator — [LeetCode #173](https://leetcode.com/problems/binary-search-tree-iterator/) · Medium

**Problem**

Implement `BSTIterator` over a BST's inorder sequence: `Constructor(root)`, `Next()` returns the next smallest value, `HasNext()` reports whether one remains. `Next()` is only called when a value remains.

**Brute Force**

Flatten the whole tree into a sorted slice in the constructor and keep an index. `Next` and `HasNext` are `O(1)`, but memory is `O(n)` and the full traversal runs up front even if the caller only wants a few values.

<details>
<summary>Hint 1</summary>
An inorder traversal written with an explicit stack can be paused between visits. What does the stack hold at the moment the next value is about to be visited?
</details>

<details>
<summary>Hint 2</summary>
Keep only the current chain of left descendants on the stack. When a node is popped and returned, push the left chain of its right child. The stack never holds more than one root-to-leaf path.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Next: O(1) amortized, O(h) worst | HasNext: O(1) | Space: O(h)
type BSTIterator struct {
    stack []*TreeNode
}

func Constructor(root *TreeNode) BSTIterator {
    it := BSTIterator{}
    it.pushLeft(root)
    return it
}

func (it *BSTIterator) pushLeft(n *TreeNode) {
    for n != nil {
        it.stack = append(it.stack, n)
        n = n.Left
    }
}

func (it *BSTIterator) Next() int {
    n := it.stack[len(it.stack)-1]
    it.stack = it.stack[:len(it.stack)-1]
    it.pushLeft(n.Right)
    return n.Val
}

func (it *BSTIterator) HasNext() bool {
    return len(it.stack) > 0
}
```
</details>

**Follow-Up: Could you implement `Next()` and `HasNext()` to run in average O(1) time and use O(h) memory?**

The solution above does. Memory: the stack holds at most one root-to-leaf chain, so `O(h)`. Time: a single `Next` can push up to `h` nodes, but across a full iteration every node is pushed once and popped once, so `n` calls do `O(n)` total work, averaging `O(1)` per call. `HasNext` is always `O(1)`.

### 8. Convert Sorted Array to Binary Search Tree — [LeetCode #108](https://leetcode.com/problems/convert-sorted-array-to-binary-search-tree/) · Easy

**Problem**

Given an integer array sorted in strictly increasing order, return a height-balanced BST containing its values.

**Brute Force**

Insert the values one by one in array order with a plain BST insert. Each insert lands to the right of the previous one, producing a chain: `O(n²)` time to build, height `n - 1`, and not balanced. It is a valid BST but fails the requirement.

<details>
<summary>Hint 1</summary>
For the tree to be balanced at the root, about half the values must go to each side. Which element of a sorted array has that property?
</details>

<details>
<summary>Hint 2</summary>
Make the middle element the root, then build the left subtree from the left half and the right subtree from the right half the same way.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(log n)
func sortedArrayToBST(nums []int) *TreeNode {
    if len(nums) == 0 {
        return nil
    }
    mid := len(nums) / 2
    return &TreeNode{
        Val:   nums[mid],
        Left:  sortedArrayToBST(nums[:mid]),
        Right: sortedArrayToBST(nums[mid+1:]),
    }
}
```

**Why this works:** Sub-slicing shares the original backing array, so recursion copies nothing and space is just the `O(log n)` stack. The same idea rebalances any BST: inorder into a slice, then build from the middle.
</details>

## References

- Cormen, Leiserson, Rivest, Stein, *Introduction to Algorithms* (3rd ed.), Ch. 12 (Binary Search Trees), Ch. 13 (Red-Black Trees), Ch. 14 (Augmenting Data Structures: order-statistic trees)
- Sedgewick and Wayne, *Algorithms* (4th ed.), §3.2 (Binary Search Trees), §3.3 (Balanced Search Trees)
- [Java `TreeMap`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/TreeMap.html) — documented as a Red-Black tree based `NavigableMap`
- [Linux kernel: Red-black Trees](https://docs.kernel.org/core-api/rbtree.html) — where the kernel uses them and why
- [google/btree](https://github.com/google/btree) — in-memory B-tree for Go
- Internal: `dsa/algorithms/binary-search.md` — the halving idea a BST turns into a structure
- Internal: `dsa/data-structures/stacks-queues.md` — the explicit stack and queue behind iterative and level-order traversals
- Internal: `dsa/data-structures/linked-lists.md` — what a degenerate tree collapses into
- Internal: `dsa/patterns/tree-dfs.md` — DFS problem patterns built on these traversals
- Internal: `dsa/patterns/tree-bfs.md` — level-order problem patterns
- Internal: `dsa/data-structures/segment-tree.md` — range queries over an array using a tree of segments
- Internal: `dsa/data-structures/fenwick-tree.md` — the lighter range-sum structure built on binary index arithmetic