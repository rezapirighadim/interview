# Trees & BFS / DFS

## What is it?

A **tree** is a hierarchical data structure with a root node and children. Binary trees are the most common in LeetCode — each node has at most a left and a right child.

Two fundamental traversal strategies:

- **DFS (Depth-First Search)** — goes deep into one branch before backtracking. Implemented with recursion (or an explicit stack). Variants: **preorder** (root → left → right), **inorder** (left → root → right), **postorder** (left → right → root).
- **BFS (Breadth-First Search)** — processes nodes level by level. Implemented with a queue.

```
       1          DFS inorder:  4 2 5 1 6 3 7
      / \         DFS preorder: 1 2 4 5 3 6 7
     2   3        BFS:          1 2 3 4 5 6 7
    / \ / \
   4  5 6  7
```

## When to use DFS?

- Problems about **paths** (root-to-leaf, path sum, max path)
- **Height / depth** calculations
- **Symmetry / mirror** checks
- Finding **ancestors** or **subtrees**

## When to use BFS?

- **Level-order** traversal
- Finding the **shortest path** in an unweighted structure
- Processing nodes **level by level**
- **Right/left side view**

## Templates

```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

# DFS (recursive)
def dfs(node):
    if not node:
        return        # base case

    dfs(node.left)   # inorder: process left first
    process(node)    # visit node
    dfs(node.right)  # then right

# BFS (iterative)
from collections import deque

def bfs(root):
    if not root:
        return
    queue = deque([root])

    while queue:
        node = queue.popleft()
        process(node)

        if node.left:
            queue.append(node.left)
        if node.right:
            queue.append(node.right)
```

---

## Problem 1 — Maximum Depth of Binary Tree (LeetCode #104)

**Given** a binary tree, return its maximum depth.

### Solution

```python
def maxDepth(root: TreeNode) -> int:
    if not root:
        return 0

    return 1 + max(maxDepth(root.left), maxDepth(root.right))
```

### Explanation

- Base case: `None` node has depth 0.
- The depth of a node is 1 (itself) plus the maximum depth of its subtrees.
- This is a **postorder** DFS: we compute children first, then combine.
- **Time:** O(n) | **Space:** O(h) where h = tree height

---

## Problem 2 — Invert Binary Tree (LeetCode #226)

**Given** a binary tree, invert it (mirror it).

### Solution

```python
def invertTree(root: TreeNode) -> TreeNode:
    if not root:
        return None

    root.left, root.right = invertTree(root.right), invertTree(root.left)
    return root
```

### Explanation

- Recursively invert both subtrees, then swap them.
- The swap happens after the children return — postorder pattern.
- **Time:** O(n) | **Space:** O(h)

---

## Problem 3 — Binary Tree Level Order Traversal (LeetCode #102)

**Given** a binary tree, return its nodes' values level by level.

### Solution

```python
from collections import deque

def levelOrder(root: TreeNode) -> list[list[int]]:
    if not root:
        return []

    result = []
    queue = deque([root])

    while queue:
        level_size = len(queue)  # snapshot of current level's node count
        level = []

        for _ in range(level_size):
            node = queue.popleft()
            level.append(node.val)

            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

        result.append(level)

    return result
```

### Explanation

- At the start of each iteration, `len(queue)` tells us exactly how many nodes are on the current level.
- We process exactly that many nodes, adding their children for the next level.
- **Time:** O(n) | **Space:** O(n)

---

## Problem 4 — Validate Binary Search Tree (LeetCode #98)

**Given** a binary tree, determine if it is a valid BST.

### Solution

```python
def isValidBST(root: TreeNode) -> bool:
    def validate(node, min_val, max_val):
        if not node:
            return True

        if node.val <= min_val or node.val >= max_val:
            return False

        return (validate(node.left, min_val, node.val) and
                validate(node.right, node.val, max_val))

    return validate(root, float('-inf'), float('inf'))
```

### Explanation

- Every node must satisfy a **range constraint**: it must be greater than all ancestors to its left and less than all ancestors to its right.
- We pass the valid range `(min_val, max_val)` down the tree.
- Going left narrows the upper bound (`max_val = node.val`); going right narrows the lower bound (`min_val = node.val`).
- **Time:** O(n) | **Space:** O(h)

---

## Problem 5 — Lowest Common Ancestor (LeetCode #236)

**Given** a binary tree and two nodes `p` and `q`, find their lowest common ancestor.

### Solution

```python
def lowestCommonAncestor(root: TreeNode, p: TreeNode, q: TreeNode) -> TreeNode:
    if not root or root == p or root == q:
        return root

    left = lowestCommonAncestor(root.left, p, q)
    right = lowestCommonAncestor(root.right, p, q)

    if left and right:
        return root   # p is in one subtree, q in the other → current node is LCA

    return left or right  # both are in the same subtree
```

### Explanation

- If we find `p` or `q`, return it immediately.
- After searching both subtrees: if both return non-null, the current node is the LCA.
- If only one side is non-null, both nodes are in that side — return what we found.
- **Time:** O(n) | **Space:** O(h)

---

## Key Takeaways

| Need to... | Use |
|---|---|
| Calculate height / depth | DFS postorder |
| Check path conditions | DFS with state passed down |
| Process level by level | BFS with queue |
| Find shortest path | BFS |
| Validate BST | DFS with min/max bounds |
| Find LCA | DFS returning node or None |

**The most important habit:** always handle the `None` base case first. Almost every tree problem starts with `if not node: return ...`.
