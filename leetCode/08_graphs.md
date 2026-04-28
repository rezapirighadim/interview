# Graphs

## What is it?

A **graph** is a collection of **nodes (vertices)** connected by **edges**. Unlike trees, graphs can have cycles, disconnected components, and edges can be directed or undirected.

Common graph representations:
- **Adjacency list** — `{node: [neighbors]}` — most common in LeetCode
- **Adjacency matrix** — `matrix[i][j] = 1` if edge exists
- **Edge list** — `[(u, v), ...]` — common in input format

## Core algorithms

| Algorithm | Purpose | Data structure |
|---|---|---|
| BFS | Shortest path (unweighted), level order | Queue |
| DFS | Connected components, cycle detection, path exploration | Stack / recursion |
| Union-Find | Connected components, cycle detection | Array with path compression |
| Topological sort | Ordering of dependencies (DAG) | BFS (Kahn's) or DFS |

## Templates

```python
from collections import deque, defaultdict

# Build adjacency list from edge list
graph = defaultdict(list)
for u, v in edges:
    graph[u].append(v)
    graph[v].append(u)  # omit for directed graph

# BFS
def bfs(start):
    visited = set([start])
    queue = deque([start])
    while queue:
        node = queue.popleft()
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)

# DFS (iterative)
def dfs(start):
    visited = set()
    stack = [start]
    while stack:
        node = stack.pop()
        if node in visited:
            continue
        visited.add(node)
        for neighbor in graph[node]:
            stack.append(neighbor)
```

---

## Problem 1 — Number of Islands (LeetCode #200)

**Given** a 2D grid of `'1'` (land) and `'0'` (water), count the number of islands.

### Solution

```python
def numIslands(grid: list[list[str]]) -> int:
    if not grid:
        return 0

    rows, cols = len(grid), len(grid[0])
    count = 0

    def dfs(r, c):
        if r < 0 or r >= rows or c < 0 or c >= cols or grid[r][c] != '1':
            return
        grid[r][c] = '0'  # mark visited by sinking the land
        dfs(r+1, c)
        dfs(r-1, c)
        dfs(r, c+1)
        dfs(r, c-1)

    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == '1':
                count += 1
                dfs(r, c)  # sink the entire island

    return count
```

### Explanation

- Every time we find an unvisited land cell, we increment the count and DFS to sink the whole island.
- Sinking (setting to `'0'`) is an in-place visited marker — no extra space needed.
- **Time:** O(rows·cols) | **Space:** O(rows·cols) worst case recursion

---

## Problem 2 — Clone Graph (LeetCode #133)

**Given** a reference to a node in a connected undirected graph, return a deep copy.

### Solution

```python
from collections import deque

def cloneGraph(node):
    if not node:
        return None

    clones = {node: Node(node.val)}  # original → clone map
    queue = deque([node])

    while queue:
        curr = queue.popleft()
        for neighbor in curr.neighbors:
            if neighbor not in clones:
                clones[neighbor] = Node(neighbor.val)
                queue.append(neighbor)
            clones[curr].neighbors.append(clones[neighbor])

    return clones[node]
```

### Explanation

- Use a hash map to track which nodes have already been cloned.
- BFS ensures we visit every node exactly once.
- For each original node's neighbor, we link the corresponding cloned nodes.
- **Time:** O(V + E) | **Space:** O(V)

---

## Problem 3 — Course Schedule (LeetCode #207)

**Given** `numCourses` and a list of `[a, b]` prerequisites (must take `b` before `a`), return `True` if you can finish all courses (no cycle).

### Solution

```python
from collections import deque

def canFinish(numCourses: int, prerequisites: list[list[int]]) -> bool:
    graph = defaultdict(list)
    in_degree = [0] * numCourses

    for course, pre in prerequisites:
        graph[pre].append(course)
        in_degree[course] += 1

    # Kahn's algorithm: start with courses that have no prerequisites
    queue = deque(i for i in range(numCourses) if in_degree[i] == 0)
    taken = 0

    while queue:
        course = queue.popleft()
        taken += 1
        for next_course in graph[course]:
            in_degree[next_course] -= 1
            if in_degree[next_course] == 0:
                queue.append(next_course)

    return taken == numCourses  # if we took all courses, no cycle exists
```

### Explanation

- This is **topological sort** (Kahn's BFS algorithm).
- Nodes with `in_degree == 0` have no dependencies — safe to take.
- After taking a course, reduce in-degree of its dependents. If any reaches 0, enqueue it.
- If a cycle exists, some nodes will never reach in-degree 0, so `taken < numCourses`.
- **Time:** O(V + E) | **Space:** O(V + E)

---

## Problem 4 — Word Ladder (LeetCode #127)

**Given** `beginWord`, `endWord`, and a word list, find the shortest transformation sequence where each step changes one letter.

### Solution

```python
from collections import deque

def ladderLength(beginWord: str, endWord: str, wordList: list[str]) -> int:
    word_set = set(wordList)
    if endWord not in word_set:
        return 0

    queue = deque([(beginWord, 1)])
    visited = {beginWord}

    while queue:
        word, steps = queue.popleft()

        for i in range(len(word)):
            for c in 'abcdefghijklmnopqrstuvwxyz':
                new_word = word[:i] + c + word[i+1:]

                if new_word == endWord:
                    return steps + 1

                if new_word in word_set and new_word not in visited:
                    visited.add(new_word)
                    queue.append((new_word, steps + 1))

    return 0
```

### Explanation

- BFS guarantees the **shortest** path.
- We generate all one-letter variations of the current word and check if they are in the word list.
- Using a `visited` set prevents revisiting words.
- **Time:** O(M² · N) where M=word length, N=word list size | **Space:** O(M² · N)

---

## Problem 5 — Union-Find: Number of Connected Components (LeetCode #323)

**Given** `n` nodes and edges, count the number of connected components.

### Solution

```python
def countComponents(n: int, edges: list[list[int]]) -> int:
    parent = list(range(n))
    rank = [0] * n

    def find(x):
        if parent[x] != x:
            parent[x] = find(parent[x])  # path compression
        return parent[x]

    def union(x, y):
        px, py = find(x), find(y)
        if px == py:
            return 0  # already connected
        if rank[px] < rank[py]:
            px, py = py, px
        parent[py] = px
        if rank[px] == rank[py]:
            rank[px] += 1
        return 1  # merged two components

    components = n
    for u, v in edges:
        components -= union(u, v)

    return components
```

### Explanation

- Each node starts as its own component (`parent[i] = i`).
- `find` with **path compression** flattens the tree, making future finds faster.
- `union` with **rank** merges smaller trees under larger ones.
- Every successful `union` reduces the component count by 1.
- **Time:** O(α(n)) per operation — effectively O(1) | **Space:** O(n)

---

## Key Takeaways

| Problem type | Algorithm | Key detail |
|---|---|---|
| Connected components | DFS/BFS or Union-Find | Visit all, count starts |
| Shortest path (unweighted) | BFS | Queue, level-by-level |
| Cycle detection | DFS with colors / Kahn's | Back edge → cycle |
| Dependency ordering | Topological sort | in-degree → queue |
| Dynamic connectivity | Union-Find | Path compression + rank |

**BFS = shortest path. DFS = exhaustive exploration.** When the problem says "minimum steps" or "minimum path", reach for BFS first.
