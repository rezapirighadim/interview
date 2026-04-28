# Recursion & Backtracking

## What is it?

**Recursion** is a function that calls itself to solve a smaller version of the same problem. Every recursive function needs a **base case** (when to stop) and a **recursive case** (how to reduce the problem).

**Backtracking** is recursion with a "choose → explore → un-choose" pattern. You build a solution incrementally, and when you reach a dead end (a constraint is violated), you **backtrack** — undo the last choice and try the next option.

Think of it as navigating a maze: you go down a path, and if it leads nowhere, you step back and try a different turn.

## When to use it?

- Finding **all combinations or permutations** of elements
- **Subset** generation
- **Constraint satisfaction** problems (Sudoku, N-Queens)
- **Word search** on a grid
- Parsing / generating structured outputs (e.g. valid parentheses)
- Phrases like: "generate all", "find all possible", "return all valid"

## Template

```python
def backtrack(state, choices, result):
    if is_complete(state):
        result.append(list(state))  # make a copy!
        return

    for choice in choices:
        if is_valid(state, choice):
            state.append(choice)           # choose
            backtrack(state, choices, result)  # explore
            state.pop()                    # un-choose (backtrack)
```

---

## Problem 1 — Subsets (LeetCode #78)

**Given** a set of unique integers, return all possible subsets.

### Solution

```python
def subsets(nums: list[int]) -> list[list[int]]:
    result = []

    def backtrack(start, current):
        result.append(list(current))  # every state is a valid subset

        for i in range(start, len(nums)):
            current.append(nums[i])
            backtrack(i + 1, current)
            current.pop()

    backtrack(0, [])
    return result
```

### Explanation

- We add the current state to the result **before** making any more choices — the empty list is also a subset.
- `start` prevents us from revisiting earlier elements (avoids duplicate subsets).
- For `[1, 2, 3]`: root → [], [1], [1,2], [1,2,3], [1,3], [2], [2,3], [3].
- **Time:** O(2ⁿ · n) | **Space:** O(n) recursion depth

---

## Problem 2 — Permutations (LeetCode #46)

**Given** a list of distinct integers, return all permutations.

### Solution

```python
def permute(nums: list[int]) -> list[list[int]]:
    result = []

    def backtrack(current, remaining):
        if not remaining:
            result.append(list(current))
            return

        for i in range(len(remaining)):
            current.append(remaining[i])
            backtrack(current, remaining[:i] + remaining[i+1:])
            current.pop()

    backtrack([], nums)
    return result
```

### Explanation

- At each step, choose any element from `remaining` and add it to `current`.
- After the recursive call, remove it (backtrack) and try the next option.
- Base case: no remaining elements → we have a complete permutation.
- **Time:** O(n! · n) | **Space:** O(n)

---

## Problem 3 — Combination Sum (LeetCode #39)

**Given** candidates and a target, return all combinations of candidates that sum to target. Each number can be used multiple times.

### Solution

```python
def combinationSum(candidates: list[int], target: int) -> list[list[int]]:
    result = []
    candidates.sort()

    def backtrack(start, current, remaining):
        if remaining == 0:
            result.append(list(current))
            return

        for i in range(start, len(candidates)):
            if candidates[i] > remaining:
                break  # pruning: sorted, so all further candidates are too big

            current.append(candidates[i])
            backtrack(i, current, remaining - candidates[i])  # i not i+1: reuse allowed
            current.pop()

    backtrack(0, [], target)
    return result
```

### Explanation

- We pass `i` (not `i + 1`) to the recursive call because elements can be reused.
- **Pruning:** sort first, then `break` when a candidate exceeds the remaining target — this avoids exploring dead-end paths.
- **Time:** O(n^(t/m)) where t=target, m=min(candidates) | **Space:** O(t/m)

---

## Problem 4 — Word Search (LeetCode #79)

**Given** a 2D grid and a word, return `True` if the word exists in the grid (connecting adjacent cells, no reuse).

### Solution

```python
def exist(board: list[list[str]], word: str) -> bool:
    rows, cols = len(board), len(board[0])

    def backtrack(r, c, idx):
        if idx == len(word):
            return True
        if r < 0 or r >= rows or c < 0 or c >= cols:
            return False
        if board[r][c] != word[idx]:
            return False

        temp = board[r][c]
        board[r][c] = '#'  # mark as visited

        found = (
            backtrack(r+1, c, idx+1) or
            backtrack(r-1, c, idx+1) or
            backtrack(r, c+1, idx+1) or
            backtrack(r, c-1, idx+1)
        )

        board[r][c] = temp  # restore (backtrack)
        return found

    for r in range(rows):
        for c in range(cols):
            if backtrack(r, c, 0):
                return True

    return False
```

### Explanation

- We modify the board in-place (replace with `#`) to mark visited cells — this avoids a separate `visited` set.
- Restoring the cell after recursion is the backtracking step.
- We return `True` as soon as we find one valid path (short-circuit with `or`).
- **Time:** O(rows·cols·4^len(word)) | **Space:** O(len(word)) recursion depth

---

## Problem 5 — Generate Parentheses (LeetCode #22)

**Given** `n`, generate all valid combinations of `n` pairs of parentheses.

### Solution

```python
def generateParenthesis(n: int) -> list[str]:
    result = []

    def backtrack(current, open_count, close_count):
        if len(current) == 2 * n:
            result.append(current)
            return

        if open_count < n:
            backtrack(current + '(', open_count + 1, close_count)

        if close_count < open_count:
            backtrack(current + ')', open_count, close_count + 1)

    backtrack('', 0, 0)
    return result
```

### Explanation

- Only add `(` if we haven't used all `n` opening brackets.
- Only add `)` if there are more open brackets than close brackets (keeps string valid).
- These constraints replace explicit backtracking — invalid states are never entered.
- **Time:** O(4ⁿ / √n) (Catalan number) | **Space:** O(n)

---

## Key Takeaways

| Concept | Detail |
|---|---|
| Always copy before saving | `result.append(list(current))` not `result.append(current)` |
| Pruning | Sort + break early to skip dead-end branches |
| start index | Prevents revisiting → controls duplicates |
| i vs i+1 | `i` = reuse allowed, `i+1` = each element once |
| Mark visited | In-place modification + restore is cleaner than a separate set |

**The core mental model:** draw the recursion tree. Each level is one choice. Backtracking prunes branches of that tree early.
