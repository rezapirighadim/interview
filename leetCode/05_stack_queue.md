# Stack & Queue

## What is it?

**Stack** — Last In, First Out (LIFO). The last element pushed is the first one popped. Think of a stack of plates.

**Queue** — First In, First Out (FIFO). The first element enqueued is the first dequeued. Think of a line at a cashier.

Both are simple abstractions, but they power elegant solutions for problems involving **matching**, **ordering**, **traversal**, and **monotonic** relationships.

## When to use a Stack?

- **Matching brackets** / parentheses
- **Undo/redo** operations
- **DFS** traversal (iteratively)
- **Monotonic stack** — finding the next greater/smaller element
- Any problem where you need to **remember history** and process it in reverse order

## When to use a Queue?

- **BFS** traversal (level-order)
- **Sliding window maximum** (with `deque`)
- Task scheduling / order processing

## Core Python tools

```python
# Stack — use a plain list
stack = []
stack.append(x)   # push
stack.pop()       # pop from top
stack[-1]         # peek

# Queue — use collections.deque for O(1) popleft
from collections import deque
queue = deque()
queue.append(x)    # enqueue
queue.popleft()    # dequeue
queue[0]           # peek front

# Monotonic deque (for sliding window max)
dq = deque()  # stores indices
```

---

## Problem 1 — Valid Parentheses (LeetCode #20)

**Given** a string of brackets, return `True` if they are valid (properly opened and closed).

### Solution

```python
def isValid(s: str) -> bool:
    stack = []
    matching = {')': '(', '}': '{', ']': '['}

    for char in s:
        if char in '({[':
            stack.append(char)
        else:
            if not stack or stack[-1] != matching[char]:
                return False
            stack.pop()

    return len(stack) == 0
```

### Explanation

- Push every opening bracket onto the stack.
- For every closing bracket, check if the top of the stack is its matching opener. If not → invalid.
- At the end, the stack must be empty (no unmatched openers left).
- **Time:** O(n) | **Space:** O(n)

---

## Problem 2 — Daily Temperatures (LeetCode #739)

**Given** daily temperatures, return an array where each element is the number of days until a warmer temperature.

### Solution

```python
def dailyTemperatures(temperatures: list[int]) -> list[int]:
    n = len(temperatures)
    result = [0] * n
    stack = []  # stores indices, in decreasing temperature order (monotonic stack)

    for i, temp in enumerate(temperatures):
        # pop all days that are colder than today
        while stack and temperatures[stack[-1]] < temp:
            j = stack.pop()
            result[j] = i - j

        stack.append(i)

    return result
```

### Explanation

- The stack stores indices of days whose "next warmer day" hasn't been found yet.
- When we find a warmer day (`temp > temperatures[stack[-1]]`), we resolve all colder days on the stack.
- Days left in the stack at the end have no future warmer day → default 0.
- **Time:** O(n) | **Space:** O(n)

---

## Problem 3 — Min Stack (LeetCode #155)

Design a stack that supports `push`, `pop`, `top`, and `getMin` — all in O(1).

### Solution

```python
class MinStack:
    def __init__(self):
        self.stack = []
        self.min_stack = []  # tracks minimum at each state

    def push(self, val: int) -> None:
        self.stack.append(val)
        min_val = min(val, self.min_stack[-1] if self.min_stack else val)
        self.min_stack.append(min_val)

    def pop(self) -> None:
        self.stack.pop()
        self.min_stack.pop()

    def top(self) -> int:
        return self.stack[-1]

    def getMin(self) -> int:
        return self.min_stack[-1]
```

### Explanation

- We maintain a parallel `min_stack` where `min_stack[i]` stores the minimum of all elements from 0 to i.
- After every push/pop, the minimum is always `min_stack[-1]`.
- **Time:** O(1) for all operations | **Space:** O(n)

---

## Problem 4 — Sliding Window Maximum (LeetCode #239)

**Given** an array and window size `k`, return the maximum of each sliding window.

### Solution

```python
from collections import deque

def maxSlidingWindow(nums: list[int], k: int) -> list[int]:
    dq = deque()  # stores indices; front = index of current window's max
    result = []

    for i, num in enumerate(nums):
        # remove indices that are out of the current window
        while dq and dq[0] < i - k + 1:
            dq.popleft()

        # remove indices of elements smaller than current (they can never be max)
        while dq and nums[dq[-1]] < num:
            dq.pop()

        dq.append(i)

        if i >= k - 1:
            result.append(nums[dq[0]])

    return result
```

### Explanation

- The deque is **monotonically decreasing** — front always holds the index of the maximum.
- We remove the front if it's outside the current window.
- We remove the back while the back's value is smaller than the incoming element (it can never become the max).
- **Time:** O(n) | **Space:** O(k)

---

## Problem 5 — Evaluate Reverse Polish Notation (LeetCode #150)

**Given** tokens in RPN format (postfix), evaluate the expression.

### Solution

```python
def evalRPN(tokens: list[str]) -> int:
    stack = []
    ops = {
        '+': lambda a, b: a + b,
        '-': lambda a, b: a - b,
        '*': lambda a, b: a * b,
        '/': lambda a, b: int(a / b),  # truncate toward zero
    }

    for token in tokens:
        if token in ops:
            b = stack.pop()
            a = stack.pop()
            stack.append(ops[token](a, b))
        else:
            stack.append(int(token))

    return stack[0]
```

### Explanation

- Push numbers onto the stack.
- When an operator is encountered, pop the top two numbers, apply the operation, and push the result.
- Note the order: `b = stack.pop()` first, then `a = stack.pop()` — `a` is the left operand.
- **Time:** O(n) | **Space:** O(n)

---

## Key Takeaways

| Problem type | Structure | Key pattern |
|---|---|---|
| Bracket matching | Stack | Push open, match on close |
| Next greater element | Monotonic stack | Pop while stack top < current |
| Sliding window max | Monotonic deque | Remove out-of-window + smaller |
| BFS / level order | Queue (deque) | Enqueue neighbors |
| O(1) min/max | Parallel stack | Track min/max at each state |

**Monotonic stack** is the hardest concept here. Always ask: "What does it mean to pop from the stack?" Each pop means "the element being popped has found its answer in the current element."
