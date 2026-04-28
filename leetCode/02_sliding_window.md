# Sliding Window

## What is it?

Sliding Window is a technique for problems involving a **contiguous subarray or substring**. Instead of recomputing the result for every possible window from scratch (O(n²)), you maintain a window with two pointers and **slide** it across the data, adding one element on the right and removing one on the left — keeping the computation O(n).

Two variants exist:
- **Fixed-size window** — window length is given (e.g. "subarray of size k")
- **Variable-size window** — you expand/shrink the window to satisfy a condition

## When to use it?

- "Find the longest/shortest subarray/substring that satisfies X"
- "Find the maximum/minimum sum of a subarray of size k"
- "Count subarrays with at most/exactly k distinct elements"
- Input is a **string or array**, and you need something **contiguous**

## Templates

### Fixed-size window

```python
def fixed_window(arr, k):
    window_sum = sum(arr[:k])
    best = window_sum

    for i in range(k, len(arr)):
        window_sum += arr[i] - arr[i - k]  # add new, remove old
        best = max(best, window_sum)

    return best
```

### Variable-size window

```python
def variable_window(arr):
    left = 0
    state = {}   # track what's inside the window
    best = 0

    for right in range(len(arr)):
        # expand: include arr[right] in state
        state[arr[right]] = state.get(arr[right], 0) + 1

        # shrink: while window is invalid, move left forward
        while not is_valid(state):
            state[arr[left]] -= 1
            if state[arr[left]] == 0:
                del state[arr[left]]
            left += 1

        best = max(best, right - left + 1)

    return best
```

---

## Problem 1 — Best Time to Buy and Sell Stock (LeetCode #121)

**Given** daily prices, find the maximum profit from one buy and one sell.

### Solution

```python
def maxProfit(prices: list[int]) -> int:
    min_price = float('inf')
    max_profit = 0

    for price in prices:
        if price < min_price:
            min_price = price
        else:
            max_profit = max(max_profit, price - min_price)

    return max_profit
```

### Explanation

- We track the lowest price seen so far (`min_price`).
- At each day, we check if selling today gives a better profit.
- This is essentially a window where the left pointer is the best buy day.
- **Time:** O(n) | **Space:** O(1)

---

## Problem 2 — Longest Substring Without Repeating Characters (LeetCode #3)

**Given** a string, find the length of the longest substring with all unique characters.

### Solution

```python
def lengthOfLongestSubstring(s: str) -> int:
    char_index = {}  # char → last seen index
    left = 0
    best = 0

    for right, char in enumerate(s):
        # if char is in window, jump left past its last occurrence
        if char in char_index and char_index[char] >= left:
            left = char_index[char] + 1

        char_index[char] = right
        best = max(best, right - left + 1)

    return best
```

### Explanation

- We store the last seen index of each character.
- When a duplicate is found **inside the current window**, we move `left` to just after the previous occurrence.
- `right - left + 1` is the current window size.
- **Time:** O(n) | **Space:** O(min(n, alphabet_size))

---

## Problem 3 — Minimum Window Substring (LeetCode #76)

**Given** strings `s` and `t`, find the smallest window in `s` that contains all characters of `t`.

### Solution

```python
from collections import Counter

def minWindow(s: str, t: str) -> str:
    if not t or not s:
        return ""

    need = Counter(t)       # how many of each char we still need
    missing = len(t)        # total characters still needed
    left = 0
    best_start, best_len = 0, float('inf')

    for right, char in enumerate(s):
        if need[char] > 0:
            missing -= 1
        need[char] -= 1

        if missing == 0:  # valid window found
            # shrink from left
            while need[s[left]] < 0:
                need[s[left]] += 1
                left += 1

            if right - left + 1 < best_len:
                best_len = right - left + 1
                best_start = left

            # break window to keep searching
            need[s[left]] += 1
            missing += 1
            left += 1

    return s[best_start:best_start + best_len] if best_len != float('inf') else ""
```

### Explanation

- `need` tracks the deficit of each character. When `need[c] > 0`, we still need more of `c`.
- `missing` counts how many total characters are still short. When it hits 0, the window is valid.
- Once valid, we shrink from the left as much as possible, then record the window.
- **Time:** O(n + m) | **Space:** O(m) where m = len(t)

---

## Problem 4 — Longest Repeating Character Replacement (LeetCode #424)

**Given** a string and integer `k`, you can replace at most `k` characters. Find the longest substring with all same characters after replacements.

### Solution

```python
def characterReplacement(s: str, k: int) -> int:
    count = {}
    left = 0
    max_count = 0  # count of most frequent char in window
    best = 0

    for right in range(len(s)):
        count[s[right]] = count.get(s[right], 0) + 1
        max_count = max(max_count, count[s[right]])

        # window size - max_count = characters we need to replace
        window_size = right - left + 1
        if window_size - max_count > k:
            count[s[left]] -= 1
            left += 1

        best = max(best, right - left + 1)

    return best
```

### Explanation

- The key insight: a window is valid if `(window size) - (most frequent char count) <= k`.
- We track the max frequency in the window. If the window becomes invalid, shrink from the left.
- `max_count` never decreases — this is a subtle optimization that keeps the window size monotonically growing.
- **Time:** O(n) | **Space:** O(1) (26 letters max)

---

## Problem 5 — Maximum Average Subarray I (LeetCode #643)

**Given** an array and integer `k`, find the maximum average of any contiguous subarray of length `k`.

### Solution

```python
def findMaxAverage(nums: list[int], k: int) -> float:
    window_sum = sum(nums[:k])
    best = window_sum

    for i in range(k, len(nums)):
        window_sum += nums[i] - nums[i - k]
        best = max(best, window_sum)

    return best / k
```

### Explanation

- Classic fixed-size window: initialize with the first `k` elements.
- Slide one step at a time: add the incoming element, subtract the outgoing one.
- **Time:** O(n) | **Space:** O(1)

---

## Key Takeaways

| Problem type | Strategy |
|---|---|
| Fixed window size | Init first k, then slide |
| Longest valid window | Expand right, shrink left when invalid |
| Shortest valid window | Expand right until valid, shrink left as much as possible |
| "At most k distinct" | Shrink when distinct count > k |

The core loop is always: **right moves every iteration, left only moves when the window breaks a constraint.**
