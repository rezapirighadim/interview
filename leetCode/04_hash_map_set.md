# Hash Map / Hash Set

## What is it?

A **Hash Map** (dict in Python) stores key-value pairs with O(1) average lookup, insertion, and deletion. A **Hash Set** (set in Python) stores unique elements with the same O(1) properties.

These structures let you trade **space for time** — by remembering what you've seen, you can avoid repeated scanning and reduce O(n²) brute-force solutions to O(n).

## When to use it?

- You need to check **"have I seen this before?"** quickly
- You want to count **frequencies** of elements
- You need to look up a **complement** (e.g. `target - x`)
- You need to **group** elements by some property
- Detecting **duplicates** or **anagrams**
- Building an index or cache of computed values

## Core Python tools

```python
# Hash Map
freq = {}
freq[key] = freq.get(key, 0) + 1

from collections import Counter
freq = Counter(arr)           # frequency map in one line
freq = Counter(s)             # works on strings too

from collections import defaultdict
graph = defaultdict(list)     # avoids KeyError for missing keys

# Hash Set
seen = set()
seen.add(x)
x in seen                     # O(1) lookup
```

---

## Problem 1 — Two Sum (LeetCode #1)

**Given** an array and a target, return indices of two numbers that add up to target.

### Solution

```python
def twoSum(nums: list[int], target: int) -> list[int]:
    seen = {}  # value → index

    for i, num in enumerate(nums):
        complement = target - num

        if complement in seen:
            return [seen[complement], i]

        seen[num] = i

    return []
```

### Explanation

- For each number, the complement we need is `target - num`.
- If the complement is already in the map, we found the pair.
- Otherwise, store the current number and its index for future lookups.
- **Time:** O(n) | **Space:** O(n)

---

## Problem 2 — Group Anagrams (LeetCode #49)

**Given** a list of strings, group them by anagram.

### Solution

```python
from collections import defaultdict

def groupAnagrams(strs: list[str]) -> list[list[str]]:
    groups = defaultdict(list)

    for s in strs:
        key = tuple(sorted(s))  # anagrams have the same sorted form
        groups[key].append(s)

    return list(groups.values())
```

### Explanation

- Two strings are anagrams if and only if their sorted characters are identical.
- We use the sorted tuple as a dictionary key to group anagrams together.
- **Time:** O(n · k log k) where k = max string length | **Space:** O(n·k)

---

## Problem 3 — Top K Frequent Elements (LeetCode #347)

**Given** an array, return the `k` most frequent elements.

### Solution

```python
from collections import Counter

def topKFrequent(nums: list[int], k: int) -> list[int]:
    freq = Counter(nums)

    # bucket sort: index = frequency, value = list of numbers with that freq
    bucket = [[] for _ in range(len(nums) + 1)]
    for num, count in freq.items():
        bucket[count].append(num)

    result = []
    for i in range(len(bucket) - 1, 0, -1):
        result.extend(bucket[i])
        if len(result) >= k:
            return result[:k]

    return result
```

### Explanation

- Count frequencies with `Counter`.
- Use **bucket sort**: create a list where `bucket[f]` holds all numbers with frequency `f`.
- Iterate from the highest frequency bucket downward to collect the top k.
- **Time:** O(n) | **Space:** O(n) — better than the O(n log n) sort approach

---

## Problem 4 — Longest Consecutive Sequence (LeetCode #128)

**Given** an unsorted array, find the length of the longest consecutive sequence.

### Solution

```python
def longestConsecutive(nums: list[int]) -> int:
    num_set = set(nums)
    best = 0

    for num in num_set:
        # only start a sequence from the smallest number in that sequence
        if num - 1 not in num_set:
            length = 1
            while num + length in num_set:
                length += 1
            best = max(best, length)

    return best
```

### Explanation

- Convert to a set for O(1) lookup.
- Only begin counting from the **start** of a sequence (where `num - 1` is not in the set). This ensures each sequence is counted exactly once.
- Extend the sequence by checking consecutive numbers.
- **Time:** O(n) | **Space:** O(n)

---

## Problem 5 — Subarray Sum Equals K (LeetCode #560)

**Given** an array and integer `k`, count subarrays whose sum equals `k`.

### Solution

```python
from collections import defaultdict

def subarraySum(nums: list[int], k: int) -> int:
    prefix_count = defaultdict(int)
    prefix_count[0] = 1  # empty prefix
    prefix_sum = 0
    count = 0

    for num in nums:
        prefix_sum += num

        # if prefix_sum - k exists as a previous prefix sum,
        # the subarray between those two points sums to k
        count += prefix_count[prefix_sum - k]
        prefix_count[prefix_sum] += 1

    return count
```

### Explanation

- A subarray from index `i+1` to `j` sums to `k` if `prefix[j] - prefix[i] == k`, i.e. `prefix[i] == prefix[j] - k`.
- We count how many times each prefix sum has appeared so far.
- `prefix_count[0] = 1` handles the case where the subarray starts from index 0.
- **Time:** O(n) | **Space:** O(n)

---

## Key Takeaways

| Pattern | Tool | Example |
|---|---|---|
| Frequency counting | `Counter` | Top K, anagrams |
| Complement lookup | `dict` | Two Sum |
| Grouping | `defaultdict(list)` | Group anagrams |
| Membership test | `set` | Consecutive sequence |
| Prefix sum + lookup | `defaultdict(int)` | Subarray sum = k |

The universal question to ask yourself: **"If I stored what I've already computed or seen, could I answer the next query in O(1)?"** If yes, reach for a hash map.
