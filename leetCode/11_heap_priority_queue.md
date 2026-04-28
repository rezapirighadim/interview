# Heap & Priority Queue

## What is it?

A **heap** is a complete binary tree that maintains the **heap property**: in a min-heap, every parent is smaller than its children (so the root is always the minimum). Python's `heapq` module implements a **min-heap**.

A **priority queue** is the abstract concept — always gives you the most important (highest or lowest priority) element in O(log n).

```
Min-heap:        Max-heap (negate values):
     1                 -10
    / \               /   \
   3   2           -7     -8
  / \ / \         / \    /
 7  4 8  5       -3  -5 -6
```

## When to use it?

- **Top K** elements (largest, smallest, most frequent)
- **Kth** largest or smallest element
- **Merging K sorted lists**
- **Scheduling** (shortest job first, earliest deadline)
- Finding the **median** of a stream
- **Dijkstra's** shortest path algorithm

## Core Python tools

```python
import heapq

# Min-heap (default)
heap = []
heapq.heappush(heap, val)
min_val = heapq.heappop(heap)
min_val = heap[0]           # peek without popping

# Max-heap: negate values
heapq.heappush(heap, -val)
max_val = -heapq.heappop(heap)

# Build heap from list in O(n)
heapq.heapify(lst)

# Push and pop simultaneously (more efficient)
heapq.heappushpop(heap, val)

# Heap of tuples: sorted by first element
heapq.heappush(heap, (priority, item))
```

---

## Problem 1 — Kth Largest Element in an Array (LeetCode #215)

**Given** an array, return the kth largest element.

### Solution

```python
import heapq

def findKthLargest(nums: list[int], k: int) -> int:
    # maintain a min-heap of size k
    heap = []

    for num in nums:
        heapq.heappush(heap, num)
        if len(heap) > k:
            heapq.heappop(heap)  # remove the smallest

    return heap[0]  # root is the kth largest
```

### Explanation

- We keep only the `k` largest elements seen so far in a min-heap.
- When the heap exceeds `k`, we pop the minimum (which is too small to be top-k).
- The root of the heap is the smallest among the top-k, i.e., the kth largest.
- **Time:** O(n log k) | **Space:** O(k)

---

## Problem 2 — Top K Frequent Elements (LeetCode #347)

**Given** an array, return the `k` most frequent elements.

### Solution

```python
import heapq
from collections import Counter

def topKFrequent(nums: list[int], k: int) -> list[int]:
    freq = Counter(nums)

    # min-heap of (frequency, element) — keep top k by frequency
    heap = []
    for num, count in freq.items():
        heapq.heappush(heap, (count, num))
        if len(heap) > k:
            heapq.heappop(heap)

    return [num for count, num in heap]
```

### Explanation

- Same pattern as Kth Largest, but we use frequency as the priority.
- The heap maintains the `k` elements with the highest frequencies.
- **Time:** O(n log k) | **Space:** O(n)

---

## Problem 3 — Merge K Sorted Lists (LeetCode #23)

**Given** `k` sorted linked lists, merge them into one sorted list.

### Solution

```python
import heapq

class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next

def mergeKLists(lists: list[ListNode]) -> ListNode:
    heap = []

    for i, node in enumerate(lists):
        if node:
            heapq.heappush(heap, (node.val, i, node))

    dummy = ListNode()
    curr = dummy

    while heap:
        val, i, node = heapq.heappop(heap)
        curr.next = node
        curr = curr.next

        if node.next:
            heapq.heappush(heap, (node.next.val, i, node.next))

    return dummy.next
```

### Explanation

- Initialize the heap with the first node of each list.
- Each iteration: pop the minimum node, add it to the result, and push its successor.
- The `i` (list index) is used as a tiebreaker when values are equal (avoids comparing `ListNode` objects).
- **Time:** O(N log k) where N = total nodes | **Space:** O(k)

---

## Problem 4 — Find Median from Data Stream (LeetCode #295)

Design a structure that finds the median of a data stream in O(log n) per insert and O(1) per query.

### Solution

```python
import heapq

class MedianFinder:
    def __init__(self):
        self.small = []  # max-heap (negate): stores the smaller half
        self.large = []  # min-heap: stores the larger half

    def addNum(self, num: int) -> None:
        heapq.heappush(self.small, -num)

        # ensure every element in small <= every element in large
        if self.small and self.large and -self.small[0] > self.large[0]:
            heapq.heappush(self.large, -heapq.heappop(self.small))

        # balance sizes: small can have at most 1 more element than large
        if len(self.small) > len(self.large) + 1:
            heapq.heappush(self.large, -heapq.heappop(self.small))
        elif len(self.large) > len(self.small):
            heapq.heappush(self.small, -heapq.heappop(self.large))

    def findMedian(self) -> float:
        if len(self.small) > len(self.large):
            return -self.small[0]
        return (-self.small[0] + self.large[0]) / 2
```

### Explanation

- Split the data into two halves: `small` (max-heap) holds the lower half, `large` (min-heap) holds the upper half.
- `small` and `large` are kept balanced (differ by at most 1 element).
- Median = top of `small` if sizes differ, average of both tops if equal.
- **Time:** O(log n) per insert, O(1) per query | **Space:** O(n)

---

## Problem 5 — Task Scheduler (LeetCode #621)

**Given** a list of tasks (letters) and a cooldown `n`, find the minimum time to execute all tasks. Same tasks need at least `n` intervals between them.

### Solution

```python
import heapq
from collections import Counter, deque

def leastInterval(tasks: list[str], n: int) -> int:
    freq = Counter(tasks)
    max_heap = [-count for count in freq.values()]
    heapq.heapify(max_heap)

    time = 0
    cooldown_queue = deque()  # (count, available_at)

    while max_heap or cooldown_queue:
        time += 1

        if max_heap:
            count = 1 + heapq.heappop(max_heap)  # negate to get actual count, then decrement
            if count < 0:
                cooldown_queue.append((count, time + n))

        if cooldown_queue and cooldown_queue[0][1] == time:
            heapq.heappush(max_heap, cooldown_queue.popleft()[0])

    return time
```

### Explanation

- Always execute the most frequent remaining task (max-heap).
- After executing, put it in a cooldown queue with its next available time.
- If no task is available (heap is empty but cooldown queue isn't), we idle.
- **Time:** O(total_tasks · log 26) = O(n) | **Space:** O(26) = O(1)

---

## Key Takeaways

| Problem | Heap type | Pattern |
|---|---|---|
| Kth largest | Min-heap size k | Push all, pop when > k |
| Kth smallest | Max-heap size k | Negate + same pattern |
| Merge K sorted | Min-heap of (val, idx, node) | Always pop min, push successor |
| Median stream | Two heaps | Balance sizes |
| Scheduling | Max-heap + cooldown queue | Most frequent task first |

**The universal min-heap trick:** to get a max-heap in Python, **negate** all values when pushing and negate again when popping.
