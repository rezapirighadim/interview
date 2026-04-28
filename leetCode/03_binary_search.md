# Binary Search

## What is it?

Binary Search repeatedly halves the search space to find a target in **O(log n)** time. Instead of scanning every element, you compare the middle element with the target and eliminate half the remaining candidates each step.

The classic form works on a **sorted array**, but the real power of binary search is applying it to any **monotonic decision space** — even when the array isn't sorted, as long as you can ask "is this value feasible?" and the answer flips cleanly from No to Yes (or vice versa) at some threshold.

## When to use it?

- Searching in a **sorted array**
- Finding the **first/last** position of a value
- The problem asks for a **minimum/maximum** value that satisfies a condition
- You see the phrase "find the smallest X such that..." or "find the largest X such that..."
- The answer space is a range of integers and you can write a `feasible(x)` function

## Templates

### Standard binary search

```python
def binary_search(arr, target):
    left, right = 0, len(arr) - 1

    while left <= right:
        mid = left + (right - left) // 2  # avoids overflow

        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return -1  # not found
```

### Find leftmost (first occurrence)

```python
def lower_bound(arr, target):
    left, right = 0, len(arr)  # right is exclusive

    while left < right:
        mid = (left + right) // 2
        if arr[mid] < target:
            left = mid + 1
        else:
            right = mid  # keep mid as a candidate

    return left
```

### Binary search on answer space

```python
def binary_search_on_answer(lo, hi):
    while lo < hi:
        mid = (lo + hi) // 2
        if feasible(mid):
            hi = mid        # mid works, try smaller
        else:
            lo = mid + 1    # mid doesn't work, go bigger

    return lo  # smallest value that is feasible
```

---

## Problem 1 — Binary Search (LeetCode #704)

**Given** a sorted array and a target, return its index or -1.

### Solution

```python
def search(nums: list[int], target: int) -> int:
    left, right = 0, len(nums) - 1

    while left <= right:
        mid = (left + right) // 2

        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return -1
```

### Explanation

- `left <= right`: we keep searching while there are elements to check.
- We always move `left` or `right` by at least 1, so the loop always terminates.
- **Time:** O(log n) | **Space:** O(1)

---

## Problem 2 — Find Minimum in Rotated Sorted Array (LeetCode #153)

**Given** a rotated sorted array, find the minimum element.

### Solution

```python
def findMin(nums: list[int]) -> int:
    left, right = 0, len(nums) - 1

    while left < right:
        mid = (left + right) // 2

        if nums[mid] > nums[right]:
            # minimum is in the right half
            left = mid + 1
        else:
            # minimum is in the left half (including mid)
            right = mid

    return nums[left]
```

### Explanation

- Compare `nums[mid]` with `nums[right]` (not `nums[left]`) to detect the rotation.
- If `nums[mid] > nums[right]`, the rotation point (minimum) is to the right of mid.
- Otherwise, mid could itself be the minimum, so we keep it (`right = mid`, not `mid - 1`).
- **Time:** O(log n) | **Space:** O(1)

---

## Problem 3 — Search in Rotated Sorted Array (LeetCode #33)

**Given** a rotated sorted array, search for a target.

### Solution

```python
def search(nums: list[int], target: int) -> int:
    left, right = 0, len(nums) - 1

    while left <= right:
        mid = (left + right) // 2

        if nums[mid] == target:
            return mid

        # left half is sorted
        if nums[left] <= nums[mid]:
            if nums[left] <= target < nums[mid]:
                right = mid - 1
            else:
                left = mid + 1
        # right half is sorted
        else:
            if nums[mid] < target <= nums[right]:
                left = mid + 1
            else:
                right = mid - 1

    return -1
```

### Explanation

- One half of the array is always sorted after a rotation. Identify which half, then check if target falls in it.
- If yes, narrow to that half. If no, narrow to the other half.
- **Time:** O(log n) | **Space:** O(1)

---

## Problem 4 — Koko Eating Bananas (LeetCode #875)

**Given** piles of bananas and `h` hours, find the minimum eating speed `k` such that Koko can eat all bananas in time.

### Solution

```python
import math

def minEatingSpeed(piles: list[int], h: int) -> int:
    def feasible(speed):
        # total hours needed at this speed
        return sum(math.ceil(pile / speed) for pile in piles) <= h

    left, right = 1, max(piles)

    while left < right:
        mid = (left + right) // 2
        if feasible(mid):
            right = mid    # try slower
        else:
            left = mid + 1  # need faster

    return left
```

### Explanation

- The answer lies in the range `[1, max(piles)]`. Binary search on this range.
- `feasible(speed)` checks if eating at `speed` bananas/hour finishes within `h` hours.
- We look for the **minimum** speed that is feasible → classic "find leftmost True" pattern.
- **Time:** O(n log m) where m = max(piles) | **Space:** O(1)

---

## Problem 5 — Find First and Last Position (LeetCode #34)

**Given** a sorted array, return the start and end positions of a target value.

### Solution

```python
def searchRange(nums: list[int], target: int) -> list[int]:
    def lower_bound(target):
        left, right = 0, len(nums)
        while left < right:
            mid = (left + right) // 2
            if nums[mid] < target:
                left = mid + 1
            else:
                right = mid
        return left

    start = lower_bound(target)

    if start == len(nums) or nums[start] != target:
        return [-1, -1]

    end = lower_bound(target + 1) - 1
    return [start, end]
```

### Explanation

- `lower_bound(target)` returns the index of the first element `>= target`.
- For the end, we find the first element `> target` and subtract 1.
- Using the same function twice is clean and avoids writing a separate upper-bound helper.
- **Time:** O(log n) | **Space:** O(1)

---

## Key Takeaways

| Pattern | When to use | right boundary |
|---|---|---|
| Exact match | Find a specific value | `left <= right` |
| Find leftmost | First occurrence or first feasible | `right = mid` |
| Find rightmost | Last occurrence | `left = mid` |
| Answer space | Min/max satisfying a condition | Binary search on integers |

**The most common mistake:** using `right = mid - 1` when you need `right = mid` (cutting out a valid candidate). Draw a small example and trace the loop before coding.
