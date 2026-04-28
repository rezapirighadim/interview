# Dynamic Programming

## What is it?

Dynamic Programming (DP) solves complex problems by breaking them into **overlapping subproblems** and storing the result of each so it's never recomputed. It is essentially **recursion + memoization**, or equivalently, **smart iteration with a table**.

Two approaches:
- **Top-down (memoization)** — write the recursive solution, cache results in a dict/array.
- **Bottom-up (tabulation)** — fill a table iteratively from the smallest subproblems up.

The key insight: `dp[i]` = answer for input of size `i`, built from smaller `dp` values.

## When to use it?

- You see **overlapping subproblems** (same sub-input computed multiple times in brute force)
- The problem has **optimal substructure** (optimal solution contains optimal solutions to subproblems)
- Phrases like: "minimum/maximum", "number of ways", "is it possible to", "longest/shortest"
- You have already written a recursive solution that is too slow

## Template

```python
# Top-down
from functools import lru_cache

@lru_cache(maxsize=None)
def dp(i):
    if base_case(i):
        return base_value
    return combine(dp(i-1), dp(i-2), ...)  # recurrence relation

# Bottom-up
dp = [0] * (n + 1)
dp[0] = base_value_0
dp[1] = base_value_1

for i in range(2, n + 1):
    dp[i] = combine(dp[i-1], dp[i-2], ...)
```

---

## Problem 1 — Climbing Stairs (LeetCode #70)

**Given** `n` stairs, you can climb 1 or 2 steps at a time. How many distinct ways to reach the top?

### Solution

```python
def climbStairs(n: int) -> int:
    if n <= 2:
        return n

    prev2, prev1 = 1, 2

    for _ in range(3, n + 1):
        curr = prev1 + prev2
        prev2 = prev1
        prev1 = curr

    return prev1
```

### Explanation

- To reach stair `i`, you either came from `i-1` (one step) or `i-2` (two steps).
- Recurrence: `dp[i] = dp[i-1] + dp[i-2]` — this is Fibonacci!
- We optimize space by only keeping the last two values.
- **Time:** O(n) | **Space:** O(1)

---

## Problem 2 — House Robber (LeetCode #198)

**Given** houses with money, you cannot rob two adjacent houses. Find the maximum you can rob.

### Solution

```python
def rob(nums: list[int]) -> int:
    if not nums:
        return 0
    if len(nums) == 1:
        return nums[0]

    prev2, prev1 = 0, 0

    for num in nums:
        curr = max(prev1, prev2 + num)
        prev2 = prev1
        prev1 = curr

    return prev1
```

### Explanation

- At each house: either skip it (take `prev1`) or rob it (take `prev2 + num`).
- `prev2` = best result two houses ago, `prev1` = best result from previous house.
- We don't need the full array — just the last two states.
- **Time:** O(n) | **Space:** O(1)

---

## Problem 3 — Longest Common Subsequence (LeetCode #1143)

**Given** two strings, find the length of their longest common subsequence.

### Solution

```python
def longestCommonSubsequence(text1: str, text2: str) -> int:
    m, n = len(text1), len(text2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if text1[i-1] == text2[j-1]:
                dp[i][j] = dp[i-1][j-1] + 1   # characters match: extend LCS
            else:
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])  # skip one character

    return dp[m][n]
```

### Explanation

- `dp[i][j]` = length of LCS of `text1[:i]` and `text2[:j]`.
- If the current characters match, we extend the LCS from the diagonal.
- If not, we take the best from skipping one character in either string.
- **Time:** O(m·n) | **Space:** O(m·n) — can be reduced to O(n)

---

## Problem 4 — 0/1 Knapsack / Coin Change (LeetCode #322)

**Given** coin denominations and an amount, find the minimum number of coins to make that amount.

### Solution

```python
def coinChange(coins: list[int], amount: int) -> int:
    dp = [float('inf')] * (amount + 1)
    dp[0] = 0  # 0 coins needed for amount 0

    for a in range(1, amount + 1):
        for coin in coins:
            if coin <= a:
                dp[a] = min(dp[a], dp[a - coin] + 1)

    return dp[amount] if dp[amount] != float('inf') else -1
```

### Explanation

- `dp[a]` = minimum coins to make amount `a`.
- For each amount, try every coin: if we use this coin, we need `dp[a - coin] + 1` coins total.
- Initialize with `inf` to mean "impossible", except `dp[0] = 0`.
- **Time:** O(amount · len(coins)) | **Space:** O(amount)

---

## Problem 5 — Longest Increasing Subsequence (LeetCode #300)

**Given** an array, find the length of the longest strictly increasing subsequence.

### Solution (O(n log n) with patience sorting)

```python
import bisect

def lengthOfLIS(nums: list[int]) -> int:
    tails = []  # tails[i] = smallest tail element of all increasing subsequences of length i+1

    for num in nums:
        pos = bisect.bisect_left(tails, num)
        if pos == len(tails):
            tails.append(num)
        else:
            tails[pos] = num  # replace to keep tails as small as possible

    return len(tails)
```

### Explanation

- `tails` is not the actual LIS — it's a "patience sort" structure that tracks the smallest possible tail for each length.
- `bisect_left` finds where `num` should go. If it extends the longest subsequence, append. Otherwise, replace the existing tail to maintain the best possible values for future elements.
- `len(tails)` = length of the LIS.
- **Time:** O(n log n) | **Space:** O(n)

---

## Common DP Patterns

| Pattern | Recurrence | Example |
|---|---|---|
| Linear (1D) | `dp[i] = f(dp[i-1], dp[i-2])` | Climbing Stairs, House Robber |
| 2D grid/string | `dp[i][j] = f(dp[i-1][j], dp[i][j-1])` | LCS, Edit Distance |
| Knapsack | `dp[i] = max/min(dp[i], dp[i-w] + v)` | Coin Change, Partition Equal Subset |
| Interval DP | `dp[i][j] = f(dp[i][k], dp[k+1][j])` | Burst Balloons, Matrix Chain |
| State machine | `dp[state] = transition` | Buy/Sell Stock with Cooldown |

## Key Takeaways

1. **Identify the state**: what info is needed to describe a subproblem uniquely?
2. **Write the recurrence**: how does `dp[i]` relate to smaller `dp` values?
3. **Define base cases**: what is the answer for the smallest input?
4. **Determine the order**: ensure smaller subproblems are solved before larger ones.
5. **Optimize space**: if `dp[i]` only depends on `dp[i-1]` and `dp[i-2]`, you only need two variables.
