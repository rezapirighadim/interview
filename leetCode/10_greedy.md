# Greedy

## What is it?

A **greedy algorithm** makes the locally optimal choice at each step with the hope that these local choices lead to a globally optimal solution. Unlike DP, it doesn't explore all possibilities — it commits to one decision and never looks back.

The challenge is **proving** that the greedy choice is safe. The standard proof technique is an **exchange argument**: assume a better solution exists, show you can swap the greedy choice in without making it worse, arriving at a contradiction.

## When to use it?

- The problem has **optimal substructure** AND the greedy choice always leads to an optimal result
- Problems involving **intervals, scheduling, or ordering**
- Minimizing/maximizing with a **local rule** that works globally
- Phrases like: "minimum number of", "maximum coverage", "earliest deadline first"

## Greedy vs DP

| | Greedy | DP |
|---|---|---|
| Explores choices | Only the best local one | All possibilities |
| Reverses decisions | Never | Implicitly (picks best) |
| Speed | Usually O(n log n) | Usually O(n²) or O(n·k) |
| When it works | Must prove correctness | Always correct |

---

## Problem 1 — Jump Game (LeetCode #55)

**Given** an array where each element is the max jump from that position, return `True` if you can reach the last index.

### Solution

```python
def canJump(nums: list[int]) -> bool:
    max_reach = 0

    for i, jump in enumerate(nums):
        if i > max_reach:
            return False  # can't reach this index
        max_reach = max(max_reach, i + jump)

    return True
```

### Explanation

- Track the farthest index reachable so far (`max_reach`).
- If the current index `i` exceeds `max_reach`, we're stuck — return `False`.
- Otherwise, update `max_reach` with how far we can jump from `i`.
- **Time:** O(n) | **Space:** O(1)

---

## Problem 2 — Jump Game II (LeetCode #45)

**Given** the same array, return the **minimum** number of jumps to reach the last index.

### Solution

```python
def jump(nums: list[int]) -> int:
    jumps = 0
    current_end = 0  # end of current jump's range
    farthest = 0     # farthest we can reach so far

    for i in range(len(nums) - 1):
        farthest = max(farthest, i + nums[i])

        if i == current_end:  # must jump here — end of current level
            jumps += 1
            current_end = farthest

    return jumps
```

### Explanation

- Think of it like BFS levels. `current_end` is the boundary of the current level.
- When we reach `current_end`, we must take a jump — the next level's boundary is `farthest`.
- We only jump when forced to, always jumping as far as possible.
- **Time:** O(n) | **Space:** O(1)

---

## Problem 3 — Gas Station (LeetCode #134)

**Given** gas amounts and costs at each station on a circular route, find the starting station index that allows completing the circuit, or -1.

### Solution

```python
def canCompleteCircuit(gas: list[int], cost: list[int]) -> int:
    if sum(gas) < sum(cost):
        return -1  # impossible overall

    tank = 0
    start = 0

    for i in range(len(gas)):
        tank += gas[i] - cost[i]

        if tank < 0:
            # current start fails — try starting from next station
            start = i + 1
            tank = 0

    return start
```

### Explanation

- If total gas < total cost, it's impossible — return -1.
- Otherwise, a solution is **guaranteed** to exist and be unique.
- Greedy insight: if we run out of gas at station `i`, none of the stations from `start` to `i` can be a valid starting point (they'd all run out even earlier). So reset `start = i + 1`.
- **Time:** O(n) | **Space:** O(1)

---

## Problem 4 — Non-overlapping Intervals (LeetCode #435)

**Given** a list of intervals, return the minimum number of intervals to remove to make the rest non-overlapping.

### Solution

```python
def eraseOverlapIntervals(intervals: list[list[int]]) -> int:
    if not intervals:
        return 0

    intervals.sort(key=lambda x: x[1])  # sort by end time
    removed = 0
    last_end = intervals[0][1]

    for start, end in intervals[1:]:
        if start < last_end:
            # overlap: remove the current interval (it ends later, greedily keep earlier end)
            removed += 1
        else:
            last_end = end

    return removed
```

### Explanation

- Sort by **end time** — this is the classic "earliest deadline first" greedy strategy.
- When two intervals overlap, we remove the one that ends **later** (it causes more future conflicts). Since we sorted by end, the current interval always ends later, so we remove it.
- **Time:** O(n log n) | **Space:** O(1)

---

## Problem 5 — Partition Labels (LeetCode #763)

**Given** a string, partition it into as many parts as possible so each letter appears in only one part. Return the sizes of the parts.

### Solution

```python
def partitionLabels(s: str) -> list[int]:
    last = {char: i for i, char in enumerate(s)}  # last occurrence of each char

    result = []
    start = 0
    end = 0

    for i, char in enumerate(s):
        end = max(end, last[char])  # extend partition to cover this char's last occurrence

        if i == end:  # we've covered all chars in this partition
            result.append(end - start + 1)
            start = end + 1

    return result
```

### Explanation

- First pass: find the last occurrence of every character.
- Second pass: greedily extend the current partition's right boundary to include the last occurrence of every character seen so far.
- When `i == end`, we know no character in `[start, end]` appears later — safe to cut.
- **Time:** O(n) | **Space:** O(1) (26 letters max)

---

## Key Takeaways

| Problem type | Greedy strategy |
|---|---|
| Jump problems | Track max reachable index |
| Interval scheduling | Sort by end time (earliest deadline first) |
| Circular problems | Reset start when tank goes negative |
| String partitioning | Extend boundary to last occurrence |
| Minimize cost | Sort and process smallest first |

**The key question to ask:** "Does always picking the locally best option lead to a globally optimal result, and can I prove it?" If you can't prove it, consider DP. If you can (exchange argument), greedy will be faster and simpler.
