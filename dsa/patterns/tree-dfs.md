---
Status: 🌳 Evergreen
Created: 2026-11-09
Last Updated: 2026-11-09
---

# Tree DFS

## Table Of Contents

1. [What It Is And Why It Exists](#what-it-is-and-why-it-exists)
2. [Why "DFS"](#why-dfs)
3. [Binary Tree Refresher](#binary-tree-refresher)
4. [The Three Traversal Orders](#the-three-traversal-orders)
5. [Recursion Vs Iteration](#recursion-vs-iteration)
6. [The Return-Value Pattern](#the-return-value-pattern)
7. [The Two-Answer Split](#the-two-answer-split)
8. [The Top-Down Accumulator](#the-top-down-accumulator)
9. [Complexity Reference](#complexity-reference)
10. [Recognizing The Pattern](#recognizing-the-pattern)
11. [Go Template Code](#go-template-code)
12. [Where JavaScript Differs](#where-javascript-differs)
13. [LeetCode Problems](#leetcode-problems)
14. [References](#references)

## What It Is And Why It Exists

Depth-first search on a tree commits to one branch, descends as far as it can, then backtracks to try the next branch. On a binary tree that reduces to a two-line rule: visit the current node, recurse into the left child, recurse into the right child. The order in which the three actions fire (before children, between them, or after) gives the three named traversals.

The pattern exists because most interesting questions about a tree are questions about its subtrees. "What is the height?" is "1 plus the max height of my children". "Is this a valid BST?" is "my value fits my ancestors' interval, and both my subtrees are valid on tighter intervals". "What is the diameter?" is "the deepest path on my left plus the deepest on my right, or the best answer from either subtree". Every one of those decomposes cleanly along parent-child lines. DFS is the traversal that walks the tree in exactly that decomposition order: children first when you need aggregates, parent first when you need to push context down.

Contrast with BFS: BFS walks level by level and is the right tool when the answer is layer-shaped ("shallowest leaf", "rightmost node per row", "cousins"). DFS is the right tool when the answer is path-shaped or subtree-shaped, which covers the majority of tree problems in senior interviews.

### The Problem It Solves

Any computation on a tree that can be expressed as a recurrence over subtrees. If you can write `answer(node) = f(answer(node.Left), answer(node.Right), node.Val)`, DFS is the traversal that computes it in a single O(n) pass, and the call stack does the bookkeeping for free.

## Why "DFS"

Depth-first search. The name comes from graph theory (Tarjan formalised its properties in 1972, though the idea is older). "Depth-first" contrasts with "breadth-first": at every branching point, depth-first commits to descending, breadth-first commits to widening. On a tree there is no cycle detection to worry about, so DFS collapses to plain recursion. On a graph the same walk with a `visited` set is what powers cycle detection, topological sort, and strongly connected components. The tree version is the graph algorithm with training wheels off.

## Binary Tree Refresher

Enough vocabulary to read the rest of the file without ambiguity. The full data-structure treatment lives in `data-structures/trees.md`.

- **Node**: a value plus pointers to a left child and a right child, either of which may be nil.
- **Root**: the one node with no parent.
- **Leaf**: a node with two nil children.
- **Depth of a node**: the number of edges from the root to that node. Root has depth 0.
- **Height of a node**: the number of edges on the longest downward path from that node to any leaf. A leaf has height 0. The height of the tree is the height of the root.
- **Path**: a sequence of nodes connected by parent-child edges. A "root-to-leaf" path starts at the root and ends at a leaf.
- **Subtree rooted at n**: n together with everything reachable by going downward from n.

Depth counts edges from the top down, height counts edges from the bottom up; the two coincide only for a perfectly balanced tree. LeetCode's "maximum depth" problem, confusingly, defines depth in *node* count (root has depth 1); this file follows the LeetCode convention only inside that problem's solution, and the edge-count convention everywhere else.

```
Sample Tree Used Throughout This File

        1
       / \
      2   3
     / \
    4   5

Depth(1)=0, Depth(2)=Depth(3)=1, Depth(4)=Depth(5)=2
Height(4)=Height(5)=Height(3)=0
Height(2)=1
Height(1)=2 (tree height)
```

## The Three Traversal Orders

Preorder, inorder, and postorder differ only in *when* the current node is emitted relative to its two recursive calls.

| Order | Rule | Best For |
|---|---|---|
| Preorder | node, left, right | Push context down; serialise; clone a tree |
| Inorder | left, node, right | BST-sorted output; kth smallest in BST |
| Postorder | left, right, node | Subtree aggregates; anything with a `return` from children |

On the sample tree the three orders produce:

```
Sample Tree

        1
       / \
      2   3
     / \
    4   5

Preorder :  1  2  4  5  3
Inorder  :  4  2  5  1  3
Postorder:  4  5  2  3  1
```

**Preorder** is "parent first". Use it when the answer at a node depends on information flowing *down* from the root (running path sum, current depth, the interval `(lo, hi)` a BST value must sit inside). Serialisation prefers preorder because emitting the root first lets the deserializer allocate the current node before recursing, which keeps the code short.

**Inorder** is meaningful specifically for binary *search* trees, where "left then node then right" produces values in sorted order. Outside BSTs it is rarely the right choice.

**Postorder** is "children first". Use it whenever the answer at a node depends on the answers of its subtrees. Height, subtree sum, subtree size, diameter, LCA, and max path sum are all postorder. In practice, most interview tree problems are postorder problems in disguise.

> [!TIP]
> If you ever stall on a tree problem, ask "does the answer at a node depend on its ancestors, or on its descendants?" Ancestors means preorder with an accumulator argument. Descendants means postorder with a return value. This one question resolves the ordering for roughly 90% of interview problems.

## Recursion Vs Iteration

Recursion is the natural expression. The three-line traversal reads like the definition: visit, recurse left, recurse right. Reordering the visit line changes the traversal name and nothing else.

Iteration with an explicit stack is worth learning for four reasons.

1. **Stack overflow risk in some languages.** V8's default stack takes roughly 10^4 recursive frames before it panics; a skewed tree of 10^5 nodes will crash. Go's goroutine stack starts small (typically 8 KB) but grows on demand, so this concern is much smaller in Go than in JavaScript or Python.
2. **Pause / resume.** An iterator that yields the next inorder value on demand (LC 173 "BST Iterator") is naturally stateful; the explicit stack *is* the state.
3. **Interviewer test.** The recursive answer to LC 94 is trivial; the follow-up asks for the iterative version to test whether the candidate understands what the call stack was doing.
4. **Custom traversal state.** When each frame needs more than "which node am I at" (for example, a "have I seen the right child yet?" flag for iterative postorder), the explicit stack makes that state visible instead of hidden in call frames.

The mechanics for each order:

**Iterative preorder**: push root; loop popping a node, emit it, push right, push left. Right is pushed first so left is popped first.

**Iterative inorder**: from the current pointer, walk left as far as possible, pushing each step. When you cannot go left, pop, emit, and switch the current pointer to the popped node's right child. Repeat.

**Iterative postorder**: two clean approaches. The one used in the template code walks root-right-left order onto an output slice, then reverses. This is a modified preorder that happens to produce the postorder sequence when read backwards, which is cheap to prove: preorder of a mirrored tree, read right-to-left, is the postorder of the original. The alternative is a single stack with a "last visited" pointer that decides whether to descend into the right subtree or emit the current node; it is faithful to the traversal but easy to get wrong under interview pressure.

```
Scenario: Iterative Preorder Of The Sample Tree

initial stack = [1]

pop 1, emit 1         push right=3, push left=2       stack=[3,2]
pop 2, emit 2         push right=5, push left=4       stack=[3,5,4]
pop 4, emit 4         no children                     stack=[3,5]
pop 5, emit 5         no children                     stack=[3]
pop 3, emit 3         no children                     stack=[]

output: 1 2 4 5 3     matches recursive preorder
```

## The Return-Value Pattern

The single most useful DFS shape for interviews. Each recursive call returns *some fact about its subtree*, and the parent combines the two returned values with its own node data.

Formula:

```
answer(node) = combine( answer(node.Left), answer(node.Right), node.Val )
answer(nil)  = base case
```

Concrete instances:

| Question | Base Case | Combine |
|---|---|---|
| Height (edges) | `-1` for nil, or `0` for nil if counting nodes | `1 + max(L, R)` |
| Subtree sum | `0` | `node.Val + L + R` |
| Node count | `0` | `1 + L + R` |
| Is balanced? | `(true, 0)` | `(both balanced AND \|hL - hR\| <= 1, 1 + max)` |
| Is valid BST? | `true` on nil | `lo < node.Val < hi AND left AND right` (uses top-down interval, not this pattern) |

The pattern is postorder because the combine step needs both children's answers before it can execute. In code it appears as: `l := f(node.Left); r := f(node.Right); return combine(l, r, node.Val)`.

## The Two-Answer Split

A refinement of the return-value pattern, and the highest-signal senior-SDE tree pattern. It appears whenever the quantity a node can *pass up to its parent* is different from the quantity that is *the answer*. Diameter and LC 124 (max path sum) are the canonical examples.

The tension: a path that goes through node n, using both its left and right subtrees, cannot be extended further up through n's parent (using n twice would revisit a node). Yet such a path is a perfectly valid candidate for the global answer. So each node has two quantities:

- **Gain**: the best one-sided downward extension. This is what the parent can use.
- **Through**: the best two-sided path with this node as the turning point. This is a candidate answer, and it dies here.

The template:

```go
best := <sentinel>
var gain func(*TreeNode) int
gain = func(n *TreeNode) int {
    if n == nil { return 0 }
    l := gain(n.Left)
    r := gain(n.Right)
    // clamp to zero if negative-contributions can be discarded (LC 124)
    // for diameter (edge counting on unweighted tree) no clamp is needed
    through := combine(l, r, n.Val)
    if through > best { best = through }
    return oneSided(l, r, n.Val)  // extendable upward
}
gain(root)
return best
```

Two rules of thumb sit inside this shape.

**Rule 1: The return value is "the best I can offer my parent".** For diameter that is the deeper of the two child depths plus one edge. For max path sum that is `n.Val + max(l, r)`, because the parent will attach n's value to whichever side is better.

**Rule 2: The tracked answer is "the best I can produce entirely below or through myself, that my parent cannot extend".** For diameter that is `l + r` (edges through this node). For max path sum that is `n.Val + l + r`.

The reason this pattern is worth naming: candidates who understand it write LC 124 in six lines. Candidates who do not get tangled trying to make one recursive function do both jobs and end up returning tuples or writing wrappers.

## The Top-Down Accumulator

The mirror image of the return-value pattern. Used when the answer at a node depends on information *from the ancestors*, not from the subtrees.

The shape: the recursive function takes an extra argument that carries the state accumulated from root to current. Path sum (LC 112) carries the remaining target. Sum root-to-leaf (LC 129) carries the number formed by the path so far. Valid BST (LC 98) carries the open interval `(lo, hi)` the current value must sit inside.

```go
var walk func(n *TreeNode, carry T) bool  // or accumulate to a shared result
walk = func(n *TreeNode, carry T) bool {
    if n == nil { return false }
    carry = update(carry, n.Val)
    if isLeaf(n) { return check(carry) }
    return walk(n.Left, carry) || walk(n.Right, carry)
}
```

The carry travels *down* through the argument. There is no return combine step, because the answer is a property of a path, not of a subtree. Preorder in shape (visit before descending), because the update to the carry happens before the children are visited.

## Complexity Reference

| Traversal | Time | Recursive Space | Iterative Space |
|---|---|---|---|
| Preorder / Inorder / Postorder | O(n) | O(h) call stack | O(h) explicit stack |
| Return-value pattern | O(n) | O(h) | O(h) |
| Two-answer split | O(n) | O(h) | rarely written iteratively |
| Top-down accumulator | O(n) | O(h) | O(h) |
| Morris inorder (constant space) | O(n) amortised | O(1) | not applicable |

`n` is the node count, `h` is the tree height. For a balanced tree `h = O(log n)`; for a fully skewed tree `h = O(n)`, and the "space is O(h)" line becomes O(n) worst case. Morris traversal reaches O(1) space by temporarily rewiring right-child pointers of predecessor nodes; it is worth knowing exists, and worth *not* writing under time pressure unless specifically asked.

## Recognizing The Pattern

- "Compute X for the whole tree, where X decomposes over subtrees" → postorder return-value.
- "Compute the best X over any *path* in the tree, and paths can turn at internal nodes" → two-answer split.
- "Verify / count / accumulate along root-to-leaf paths" → top-down accumulator.
- "Kth smallest / kth largest in a BST" → inorder (with an early-exit counter).
- "Serialise / clone / round-trip a tree" → preorder with a null marker.
- "Find the lowest ancestor of two nodes" → bubble-up postorder (return the first non-nil hit; when both children return non-nil, you are the LCA).
- Anything answered by "shallowest", "level by level", or "next node to the right" → not DFS, use BFS instead.

## Go Template Code

```go
package treedfs

import (
	"strconv"
	"strings"
)

// TreeNode is the standard LeetCode binary tree node.
type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

// ===== Recursive Traversals =====

// Preorder returns node values in root -> left -> right order.
// Time: O(n) | Space: O(h) call stack.
func Preorder(root *TreeNode) []int {
	var out []int
	var walk func(*TreeNode)
	walk = func(n *TreeNode) {
		if n == nil {
			return
		}
		out = append(out, n.Val)
		walk(n.Left)
		walk(n.Right)
	}
	walk(root)
	return out
}

// Inorder returns node values in left -> root -> right order.
// For a BST this yields sorted output.
// Time: O(n) | Space: O(h).
func Inorder(root *TreeNode) []int {
	var out []int
	var walk func(*TreeNode)
	walk = func(n *TreeNode) {
		if n == nil {
			return
		}
		walk(n.Left)
		out = append(out, n.Val)
		walk(n.Right)
	}
	walk(root)
	return out
}

// Postorder returns node values in left -> right -> root order.
// The natural fit for subtree aggregates.
// Time: O(n) | Space: O(h).
func Postorder(root *TreeNode) []int {
	var out []int
	var walk func(*TreeNode)
	walk = func(n *TreeNode) {
		if n == nil {
			return
		}
		walk(n.Left)
		walk(n.Right)
		out = append(out, n.Val)
	}
	walk(root)
	return out
}

// ===== Iterative Traversals =====

// PreorderIter walks preorder with an explicit stack.
// Push right first so left is popped next.
// Time: O(n) | Space: O(h).
func PreorderIter(root *TreeNode) []int {
	var out []int
	if root == nil {
		return out
	}
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

// InorderIter walks inorder with an explicit stack: descend left,
// visit, descend right of the visited node.
// Time: O(n) | Space: O(h).
func InorderIter(root *TreeNode) []int {
	var out []int
	stack := []*TreeNode{}
	cur := root
	for cur != nil || len(stack) > 0 {
		for cur != nil {
			stack = append(stack, cur)
			cur = cur.Left
		}
		cur = stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		out = append(out, cur.Val)
		cur = cur.Right
	}
	return out
}

// PostorderIter walks postorder using the "reverse of a modified
// preorder" trick: root -> right -> left, then reverse.
// Time: O(n) | Space: O(h).
func PostorderIter(root *TreeNode) []int {
	var out []int
	if root == nil {
		return out
	}
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

// ===== Return-Value Template (Post-Order Aggregate) =====

// MaxDepth is the canonical post-order aggregate.
// Time: O(n) | Space: O(h).
func MaxDepth(root *TreeNode) int {
	if root == nil {
		return 0
	}
	l, r := MaxDepth(root.Left), MaxDepth(root.Right)
	if l > r {
		return l + 1
	}
	return r + 1
}

// ===== Two-Answer Split (Diameter) =====

// Diameter returns the length in edges of the longest path
// between any two nodes.
// Time: O(n) | Space: O(h).
func Diameter(root *TreeNode) int {
	best := 0
	var depth func(*TreeNode) int
	depth = func(n *TreeNode) int {
		if n == nil {
			return 0
		}
		l := depth(n.Left)
		r := depth(n.Right)
		if l+r > best {
			best = l + r
		}
		if l > r {
			return l + 1
		}
		return r + 1
	}
	depth(root)
	return best
}

// ===== Top-Down Accumulator (Root-To-Leaf) =====

// HasPathSum returns whether some root-to-leaf path has values
// summing exactly to target. The accumulator carries what remains.
// Time: O(n) | Space: O(h).
func HasPathSum(root *TreeNode, target int) bool {
	if root == nil {
		return false
	}
	if root.Left == nil && root.Right == nil {
		return target == root.Val
	}
	return HasPathSum(root.Left, target-root.Val) ||
		HasPathSum(root.Right, target-root.Val)
}
```

Two Go-specific notes.

**Closure capture is the idiomatic way to share state across recursive frames.** The `var walk func(*TreeNode)` declaration followed by `walk = func...` is the standard way to write a recursive closure in Go; the two-step form is required because the closure body references `walk` by name, and the name is not in scope inside the anonymous function until the declaration completes. A single-expression assignment `walk := func(n *TreeNode) { walk(...) }` fails to compile.

**Slice-backed stacks are fine, but be aware of aliasing.** `stack = stack[:len(stack)-1]` shortens the slice header without shrinking the backing array. This is correct for stack semantics; the popped element remains in the backing array but is unreachable through the header. If the popped element was the last pointer to a large subtree and you want it garbage-collected during a long-running traversal, explicitly `stack[len(stack)-1] = nil` before shortening. For interview code this is overkill.

## Where JavaScript Differs

Same recursion, two idiom differences.

```javascript
// Recursion via a named function expression or an outer const:
function maxDepth(root) {
  if (!root) return 0;
  return 1 + Math.max(maxDepth(root.left), maxDepth(root.right));
}

// TypeScript needs an explicit nullable node type:
type TreeNode = { val: number; left: TreeNode | null; right: TreeNode | null };
```

The stack-overflow risk is real. V8 tolerates roughly 10^4 recursive frames before it panics with `RangeError: Maximum call stack size exceeded`. LeetCode tree problems commonly cap `n` at 10^4 or 10^5; the latter can crash a naive recursive JS solution on a fully skewed tree. Go's stack grows on demand and does not have this failure mode at LeetCode scale. When porting a solution from Go to JS, keep the iterative variant in your back pocket for the skewed-tree edge case.

## LeetCode Problems

### 1. Maximum Depth Of Binary Tree — [#104 (Easy)](https://leetcode.com/problems/maximum-depth-of-binary-tree/)

Given the root of a binary tree, return its maximum depth: the number of nodes along the longest root-to-leaf path. (LeetCode counts nodes here, so a single-node tree has depth 1.)

<details>
<summary>Brute Force</summary>

Enumerate every root-to-leaf path by DFS, tracking a running length, and return the maximum length seen.

Time: O(n). Space: O(h).

The waste: enumerating paths is not wasted work here (this problem is O(n) either way), but it obscures the recurrence. The point of the "return-value pattern" solution is that height decomposes cleanly as `1 + max(hL, hR)`, so the answer at each node is a two-line combine step; there is no reason to think about paths at all.
</details>

<details>
<summary>Hint 1</summary>

If you already knew the maximum depth of each subtree of the root, how would you get the answer for the whole tree in constant time?
</details>

<details>
<summary>Hint 2</summary>

Write a recursive function that returns the maximum depth of the subtree rooted at its argument. Base case: a nil node has depth 0. Recursive case: `1 + max(f(left), f(right))`.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(h)
func maxDepth(root *TreeNode) int {
	if root == nil {
		return 0
	}
	l, r := maxDepth(root.Left), maxDepth(root.Right)
	if l > r {
		return l + 1
	}
	return r + 1
}
```

The two-line body *is* the recurrence `height(n) = 1 + max(height(L), height(R))` with `height(nil) = 0`. Every node contributes one addition and one comparison, so total work is O(n). Space is the call stack, O(h).
</details>

### 2. Path Sum — [#112 (Easy)](https://leetcode.com/problems/path-sum/)

Given the root of a binary tree and an integer `targetSum`, return true if some root-to-leaf path has values summing exactly to `targetSum`.

<details>
<summary>Brute Force</summary>

Enumerate all root-to-leaf paths, sum each, and check.

Time: O(n) in path count (each node appears on exactly the paths that pass through it, and total work is O(n) since paths partition edges from the root's perspective). Space: O(h).

The waste: materialising each path as a list is unnecessary. The recursive decomposition only needs the running remainder, which is one integer per stack frame.
</details>

<details>
<summary>Hint 1</summary>

Think of the target as a debt. As you descend, each node repays part of that debt with its own value. What must be true at a leaf for the debt to be settled exactly?
</details>

<details>
<summary>Hint 2</summary>

Recurse with a parameter `remaining` that starts at `targetSum` and drops by `n.Val` on each step. At a leaf, the path is valid iff `remaining == n.Val`. Internal nodes return true if either subtree can complete the path.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(h)
func hasPathSum(root *TreeNode, targetSum int) bool {
	if root == nil {
		return false
	}
	if root.Left == nil && root.Right == nil {
		return targetSum == root.Val
	}
	return hasPathSum(root.Left, targetSum-root.Val) ||
		hasPathSum(root.Right, targetSum-root.Val)
}
```

The `root == nil` guard handles a nil entry into the tree, but critically *not* the "leaf child" of a real leaf. A single-child node like `root = [1, 2]` (root has left child 2 only) has `root.Right == nil`, but `root` is not a leaf. Returning true from `hasPathSum(root.Right, ...)` would incorrectly claim a valid path ending at the missing right child. The leaf check `Left == nil && Right == nil` handles this by only accepting the target when both children are absent; the outer `||` then relies on the nil branch returning false, which it does.
</details>

### 3. Diameter Of Binary Tree — [#543 (Easy)](https://leetcode.com/problems/diameter-of-binary-tree/)

Given the root of a binary tree, return the length of the diameter: the longest path between any two nodes, measured in edges. The path need not pass through the root.

<details>
<summary>Brute Force</summary>

For every node, compute the depth of its left and right subtrees, sum them, and take the maximum across all nodes.

Time: O(n^2). Every node runs a depth calculation over its subtree, and depth calculations overlap across parent and child nodes so total work is `sum of subtree sizes` = O(n^2) in the worst case. Space: O(h).

The waste: the depth of each subtree is recomputed from scratch at every ancestor. The two-answer split fixes this by making the recursive function return the depth once (a value the parent can extend by one edge) while simultaneously tracking `l + r` as a candidate answer.
</details>

<details>
<summary>Hint 1</summary>

The longest path in the tree turns at some node. If you fix that turning point, the path length is "deepest downward reach on the left" plus "deepest downward reach on the right". How would you compute both quantities for every possible turning point in one traversal?
</details>

<details>
<summary>Hint 2</summary>

A single recursive function is not enough to return both "the depth I can offer my parent" and "the best diameter seen so far", because the parent uses only the former. Return the depth from the function, and track the best diameter in a variable closed over by the recursion. At each node, update the best with `l + r` (the diameter through this node in edges) before returning `1 + max(l, r)` to the caller.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(h)
func diameterOfBinaryTree(root *TreeNode) int {
	best := 0
	var depth func(*TreeNode) int
	depth = func(n *TreeNode) int {
		if n == nil {
			return 0
		}
		l := depth(n.Left)
		r := depth(n.Right)
		if l+r > best {
			best = l + r
		}
		if l > r {
			return l + 1
		}
		return r + 1
	}
	depth(root)
	return best
}
```

The returned value counts edges from the current node down to the deepest leaf. `l + r` is the number of edges on the path that turns at the current node, which is the diameter candidate. `best` is closed over rather than passed and returned; both work, but the closure keeps the function signature clean.
</details>

### 4. Binary Tree Inorder Traversal — [#94 (Easy)](https://leetcode.com/problems/binary-tree-inorder-traversal/)

Given the root of a binary tree, return the inorder traversal of its node values.

<details>
<summary>Brute Force</summary>

The recursive three-liner is already O(n). There is no lower-complexity solution.

Time: O(n). Space: O(h).

The interesting axis is the follow-up: replace the recursion with an explicit stack, which is what interviewers usually ask.
</details>

<details>
<summary>Hint 1</summary>

Inorder means the sequence "everything in the left subtree, then this node, then everything in the right subtree". Write a three-line recursive function that does exactly that.
</details>

<details>
<summary>Hint 2</summary>

For the iterative version: maintain a stack and a current pointer. Walk left as far as possible, pushing every node you pass. When you cannot go further left, pop, emit, and set the current pointer to that popped node's right child. Continue until both the stack and the pointer are empty.
</details>

<details>
<summary>Solution (Go, Recursive)</summary>

```go
// Time: O(n) | Space: O(h)
func inorderTraversal(root *TreeNode) []int {
	var out []int
	var walk func(*TreeNode)
	walk = func(n *TreeNode) {
		if n == nil {
			return
		}
		walk(n.Left)
		out = append(out, n.Val)
		walk(n.Right)
	}
	walk(root)
	return out
}
```
</details>

<details>
<summary>Follow-Up: Iterative Solution (Go)</summary>

LeetCode's stated follow-up: "Recursive solution is trivial, could you do it iteratively?"

```go
// Time: O(n) | Space: O(h)
func inorderTraversal(root *TreeNode) []int {
	var out []int
	stack := []*TreeNode{}
	cur := root
	for cur != nil || len(stack) > 0 {
		for cur != nil {
			stack = append(stack, cur)
			cur = cur.Left
		}
		cur = stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		out = append(out, cur.Val)
		cur = cur.Right
	}
	return out
}
```

The inner `for cur != nil` loop is the "descend left as far as possible" phase, and each push corresponds to a "not yet visited, keep for later" bookmark. When descent stalls, the top of the stack is the leftmost unvisited node, so it is emitted and the traversal pivots to that node's right subtree. Each node is pushed once and popped once, giving O(n) time and O(h) space for the stack.
</details>

### 5. Lowest Common Ancestor Of A Binary Tree — [#236 (Medium)](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree/)

Given a binary tree and two nodes `p` and `q` guaranteed to exist in the tree, return their lowest common ancestor: the deepest node that is an ancestor of both.

<details>
<summary>Brute Force</summary>

Find the root-to-p path and the root-to-q path as two lists. Walk both from the start; the last common node is the LCA.

Time: O(n) to find each path, O(min(depth(p), depth(q))) to walk. Total O(n). Space: O(h) for the two path lists.

The waste: two full traversals and two allocated path lists, when a single postorder pass can decide the answer on the way back up.
</details>

<details>
<summary>Hint 1</summary>

Imagine a recursive function that returns the "first among p, q, or an already-found LCA that shows up in this subtree, or nil if none". What does it return at a node whose left subtree finds p and whose right subtree finds q?
</details>

<details>
<summary>Hint 2</summary>

Base cases: a nil node returns nil; a node equal to p or q returns itself. Recursive case: recurse into both children. If both return non-nil, the current node is the LCA. If only one is non-nil, propagate that one upward, which either carries "we found p (or q) somewhere below and the other is still elsewhere" or "we already found the LCA below". The single-value return is enough to distinguish these cases because the LCA, once found, is the deepest non-nil return the recursion produces.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(h)
func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
	if root == nil || root == p || root == q {
		return root
	}
	l := lowestCommonAncestor(root.Left, p, q)
	r := lowestCommonAncestor(root.Right, p, q)
	if l != nil && r != nil {
		return root
	}
	if l != nil {
		return l
	}
	return r
}
```

The subtlety worth staring at: what if `l` is p, and q sits somewhere in the *right* subtree of p (so q is a descendant of p, and p is itself the LCA)? The recursion into `root.Left` returns p as soon as it sees it, without descending further to find q. That looks like a bug, because we never see q. But the answer is still correct: p is the LCA in that case, and p is what got returned. The two-non-nil branch was never needed. This works because once `p` or `q` is found, the LCA is either that node itself (if the other is a descendant) or a proper ancestor (in which case the other branch will independently find the other node and the two-non-nil branch fires higher up).
</details>

### 6. Binary Tree Maximum Path Sum — [#124 (Hard)](https://leetcode.com/problems/binary-tree-maximum-path-sum/)

Given a binary tree, return the maximum sum of any path where a path is a sequence of nodes connected by parent-child edges and includes at least one node. Values may be negative.

<details>
<summary>Brute Force</summary>

For every node, treat it as the turning point and compute "best downward extension from left" plus "best downward extension from right" plus `node.Val`, allowing either side to be skipped if it hurts. Return the max.

Time: O(n^2) if each downward computation walks its subtree from scratch. Space: O(h).

The waste: each subtree's best downward extension is recomputed at every ancestor. The two-answer split from the diameter problem reappears, this time with a twist: negative subtree contributions should be clamped to zero, because a path is free to end at the current node instead of extending into a value-losing subtree.
</details>

<details>
<summary>Hint 1</summary>

For any candidate optimal path, there is a unique highest node on it (its "turning point"). Fix that node; the path is "left downward extension ending at that node, then the node, then right downward extension". Either side may be empty. How do you scan every possible turning point in a single traversal?
</details>

<details>
<summary>Hint 2</summary>

Recursive function returns the best *one-sided* extension from a node downward, clamped to zero if that side would contribute negatively (a path is free to skip a value-losing branch). Track a separate best-so-far for the *two-sided* candidate `n.Val + l + r` at each node. The one-sided return is what the parent will use; the two-sided value is a candidate answer that can never be extended upward, so it is compared against best-so-far immediately.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(h)
func maxPathSum(root *TreeNode) int {
	const NEG_INF = -1 << 31
	best := NEG_INF
	var gain func(*TreeNode) int
	gain = func(n *TreeNode) int {
		if n == nil {
			return 0
		}
		l := gain(n.Left)
		if l < 0 {
			l = 0
		}
		r := gain(n.Right)
		if r < 0 {
			r = 0
		}
		through := n.Val + l + r
		if through > best {
			best = through
		}
		if l > r {
			return n.Val + l
		}
		return n.Val + r
	}
	gain(root)
	return best
}
```

The clamp `if l < 0 { l = 0 }` is what distinguishes this from diameter. On diameter every edge counts positive, so no clamp is needed. Here, negative values mean "we would rather not use this subtree at all", and a zero clamp encodes exactly that choice: the path terminates at the current node on that side. The initial `best` must be `-inf` (not zero) so that a tree of all-negative values still returns the least-negative single node rather than an invalid empty-path zero.
</details>

### 7. Serialize And Deserialize Binary Tree — [#297 (Hard)](https://leetcode.com/problems/serialize-and-deserialize-binary-tree/)

Design an algorithm to convert a binary tree to a string and reconstruct the original tree from that string.

<details>
<summary>Brute Force</summary>

Store the tree as a list of (parent-index, is-left, value) tuples produced by BFS, and reconstruct by walking that list.

Time: O(n) either way. Space: O(n).

The waste: bookkeeping parent indices is unnecessary. Preorder DFS with an explicit null marker is enough to make the structure recoverable, and the deserialiser is a mirror of the serialiser with a shared read pointer.
</details>

<details>
<summary>Hint 1</summary>

Preorder writes the root before its children. If you also write a marker for missing children, can you reconstruct the tree by reading tokens left-to-right and recursively building?
</details>

<details>
<summary>Hint 2</summary>

Serialise: preorder, emitting `n.Val,` for real nodes and a sentinel like `N,` for nil. Deserialise: split by commas, walk left-to-right with a shared index; a sentinel returns nil, a value allocates a node and recursively builds its left then its right child.
</details>

<details>
<summary>Solution (Go)</summary>

```go
type Codec struct{}

// Time: O(n) | Space: O(n)
func (Codec) serialize(root *TreeNode) string {
	var b strings.Builder
	var walk func(*TreeNode)
	walk = func(n *TreeNode) {
		if n == nil {
			b.WriteString("N,")
			return
		}
		b.WriteString(strconv.Itoa(n.Val))
		b.WriteByte(',')
		walk(n.Left)
		walk(n.Right)
	}
	walk(root)
	return b.String()
}

// Time: O(n) | Space: O(n)
func (Codec) deserialize(data string) *TreeNode {
	tokens := strings.Split(strings.TrimSuffix(data, ","), ",")
	i := 0
	var build func() *TreeNode
	build = func() *TreeNode {
		tok := tokens[i]
		i++
		if tok == "N" {
			return nil
		}
		v, _ := strconv.Atoi(tok)
		n := &TreeNode{Val: v}
		n.Left = build()
		n.Right = build()
		return n
	}
	return build()
}
```

Two design choices worth flagging.

`strings.Builder` avoids the O(n^2) blowup that repeated string concatenation would cause, since Go strings are immutable and `s += t` allocates fresh each time. The builder amortises to O(n) total by growing its backing buffer geometrically.

The shared read pointer `i` across recursive `build` calls is the deserialisation counterpart to preorder emission during serialisation. Because preorder emits parent-then-left-subtree-then-right-subtree and `build` reads in exactly that order, the recursion naturally consumes the correct number of tokens for each subtree before returning to its parent. This is why postorder or inorder alone cannot serialise an arbitrary binary tree without extra structural information: only preorder puts the "how to allocate this node" decision at the right point in the stream.

An inorder-only serialisation loses the ability to distinguish shapes: for an arbitrary binary tree (not a BST), two different structures can produce the same inorder sequence, so the round-trip is not injective. Preorder plus null markers *is* injective; postorder plus null markers works too, but the deserialiser has to read the stream in reverse.
</details>

## References

- [LeetCode: Tree Tag](https://leetcode.com/tag/tree/)
- [LeetCode: Depth-First Search Tag](https://leetcode.com/tag/depth-first-search/)
- [NeetCode: Trees Roadmap](https://neetcode.io/roadmap)
- Cormen, Leiserson, Rivest, Stein, *Introduction to Algorithms* (4th ed.), Ch. 12 (Binary Search Trees) and §20.3 (Depth-First Search on graphs; the tree case is the same recursion without the visited set).
- Tarjan, Robert (1972). "Depth-first search and linear graph algorithms". SIAM Journal on Computing 1 (2): 146-160. The paper that formalised DFS as a general-purpose algorithmic tool.