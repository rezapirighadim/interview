# Two Pointers

## What is it?

Two Pointers is a technique where you use two index variables that move through a data structure — usually an array or string — to solve problems in **O(n)** time instead of the naive **O(n²)** brute force.

The two pointers can move:
- **Toward each other** — one starts at the left, one at the right (opposite ends)
- **In the same direction** — both start at the left but at different speeds (fast/slow)

## When to use it?

- The input is a **sorted array** or you can sort it first
- You need to find a **pair or triplet** that satisfies some condition (sum, difference)
- You need to check for **palindromes**
- You need to **remove duplicates** in-place
- You see phrases like: "find two numbers that sum to", "is palindrome", "remove in-place"

## How it works (template)

```python
def two_pointer_template(arr):
    left, right = 0, len(arr) - 1

    while left < right:
        current = arr[left] + arr[right]

        if current == target:
            # found answer
            return [left, right]
        elif current < target:
            left += 1   # need bigger sum → move left forward
        else:
            right -= 1  # need smaller sum → move right backward

    return []
```

---

## Problem 1 — Two Sum II (LeetCode #167)

**Given** a 1-indexed sorted array, find two numbers that add up to `target`. Return their indices.

### Solution

```python
def twoSum(numbers: list[int], target: int) -> list[int]:
    left, right = 0, len(numbers) - 1

    while left < right:
        s = numbers[left] + numbers[right]

        if s == target:
            return [left + 1, right + 1]  # 1-indexed
        elif s < target:
            left += 1
        else:
            right -= 1

    return []
```

### Explanation

- Because the array is **sorted**, if the sum is too small we move `left` right (increase it), if too big we move `right` left (decrease it).
- We never miss a valid pair because we systematically narrow the window.
- **Time:** O(n) | **Space:** O(1)

---

## Problem 2 — Valid Palindrome (LeetCode #125)

**Given** a string, return `True` if it is a palindrome (ignoring non-alphanumeric characters and case).

### Solution

```python
def isPalindrome(s: str) -> bool:
    left, right = 0, len(s) - 1

    while left < right:
        # skip non-alphanumeric
        while left < right and not s[left].isalnum():
            left += 1
        while left < right and not s[right].isalnum():
            right -= 1

        if s[left].lower() != s[right].lower():
            return False

        left += 1
        right -= 1

    return True
```

### Explanation

- We skip non-alphanumeric characters using inner `while` loops before comparing.
- If any pair of characters doesn't match, it's not a palindrome.
- **Time:** O(n) | **Space:** O(1)

---

## Problem 3 — 3Sum (LeetCode #15)

**Given** an array, return all unique triplets that sum to zero.

### Solution

```python
def threeSum(nums: list[int]) -> list[list[int]]:
    nums.sort()
    result = []

    for i in range(len(nums) - 2):
        # skip duplicates for the first element
        if i > 0 and nums[i] == nums[i - 1]:
            continue

        left, right = i + 1, len(nums) - 1

        while left < right:
            s = nums[i] + nums[left] + nums[right]

            if s == 0:
                result.append([nums[i], nums[left], nums[right]])
                # skip duplicates for second and third elements
                while left < right and nums[left] == nums[left + 1]:
                    left += 1
                while left < right and nums[right] == nums[right - 1]:
                    right -= 1
                left += 1
                right -= 1
            elif s < 0:
                left += 1
            else:
                right -= 1

    return result
```

### Explanation

- Sort first so we can use two pointers.
- Fix one element with the outer loop, then use two pointers for the remaining pair.
- Skip duplicate values at each pointer to avoid returning duplicate triplets.
- **Time:** O(n²) | **Space:** O(1) (excluding output)

---

## Problem 4 — Container With Most Water (LeetCode #11)

**Given** an array of heights, find two lines that form a container holding the most water.

### Solution

```python
def maxArea(height: list[int]) -> int:
    left, right = 0, len(height) - 1
    max_water = 0

    while left < right:
        width = right - left
        water = width * min(height[left], height[right])
        max_water = max(max_water, water)

        # move the shorter side — moving the taller side can only decrease area
        if height[left] < height[right]:
            left += 1
        else:
            right -= 1

    return max_water
```

### Explanation

- Water is limited by the **shorter** wall, so we move the shorter pointer inward hoping to find a taller wall.
- Moving the taller pointer inward would only reduce the width without any chance of increasing the height limit.
- **Time:** O(n) | **Space:** O(1)

---

## Problem 5 — Remove Duplicates from Sorted Array (LeetCode #26)

**Given** a sorted array, remove duplicates in-place and return the count of unique elements.

### Solution

```python
def removeDuplicates(nums: list[int]) -> int:
    if not nums:
        return 0

    slow = 0  # points to the last unique element written

    for fast in range(1, len(nums)):
        if nums[fast] != nums[slow]:
            slow += 1
            nums[slow] = nums[fast]

    return slow + 1
```

### Explanation

- `slow` marks where the next unique value should be written.
- `fast` scans ahead looking for values different from `nums[slow]`.
- When a new unique value is found, we advance `slow` and copy it over.
- **Time:** O(n) | **Space:** O(1)

---

## Key Takeaways

| Situation | Pointer Movement |
|---|---|
| Sum too small | Move left pointer right |
| Sum too big | Move right pointer left |
| Non-alphanumeric | Skip with inner while |
| Duplicates | Skip while adjacent equals |
| Remove in-place | Slow/fast same direction |
