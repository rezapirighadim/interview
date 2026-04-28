# Intervals

## What is it?

Interval problems involve ranges `[start, end]` and typically ask you to merge overlapping ranges, insert a new range, or check if ranges conflict.

Two intervals `[a, b]` and `[c, d]` **overlap** if `a <= d` and `c <= b`.

```
[1, 4] and [3, 6] → overlap  → merge to [1, 6]
[1, 2] and [4, 6] → no overlap → keep as is
```

## When to use it?

- "Merge overlapping intervals"
- "Insert a new interval"
- "Find the minimum number of intervals to cover a range"
- Meeting room scheduling problems
- "Remove minimum intervals to make the rest non-overlapping"

## Core technique: sort first

Almost every interval problem starts with sorting by **start time**. After sorting, you only need to compare the current interval with the last merged interval.

```python
intervals.sort(key=lambda x: x[0])
```

---

## Problem 1 — Merge Intervals (LeetCode #56)

**Given** a list of intervals, merge all overlapping intervals.

### Solution

```python
def merge(intervals: list[list[int]]) -> list[list[int]]:
    intervals.sort(key=lambda x: x[0])
    merged = [intervals[0]]

    for start, end in intervals[1:]:
        last_end = merged[-1][1]

        if start <= last_end:
            # overlapping: extend the last interval
            merged[-1][1] = max(last_end, end)
        else:
            # no overlap: start a new interval
            merged.append([start, end])

    return merged
```

### Explanation

- After sorting by start, two intervals overlap only if the current `start <= last merged end`.
- When they overlap, extend the merged interval's end to `max(last_end, end)`.
- `max` is important: the current interval might be completely inside the last merged one.
- **Time:** O(n log n) | **Space:** O(n)

---

## Problem 2 — Insert Interval (LeetCode #57)

**Given** a sorted list of non-overlapping intervals and a new interval, insert it and merge if necessary.

### Solution

```python
def insert(intervals: list[list[int]], newInterval: list[int]) -> list[list[int]]:
    result = []
    i = 0
    n = len(intervals)

    # add all intervals that come entirely before newInterval
    while i < n and intervals[i][1] < newInterval[0]:
        result.append(intervals[i])
        i += 1

    # merge all overlapping intervals into newInterval
    while i < n and intervals[i][0] <= newInterval[1]:
        newInterval[0] = min(newInterval[0], intervals[i][0])
        newInterval[1] = max(newInterval[1], intervals[i][1])
        i += 1

    result.append(newInterval)

    # add all remaining intervals that come after
    while i < n:
        result.append(intervals[i])
        i += 1

    return result
```

### Explanation

- Three phases: before the new interval (no overlap), during (merge), after (no overlap).
- The middle while loop handles all intervals that overlap with `newInterval`, expanding it as needed.
- **Time:** O(n) | **Space:** O(n)

---

## Problem 3 — Non-overlapping Intervals (LeetCode #435)

**Given** a list of intervals, return the minimum number to remove so the rest don't overlap.

### Solution

```python
def eraseOverlapIntervals(intervals: list[list[int]]) -> int:
    intervals.sort(key=lambda x: x[1])  # sort by END time
    removed = 0
    last_end = float('-inf')

    for start, end in intervals:
        if start >= last_end:
            last_end = end  # no conflict, keep this interval
        else:
            removed += 1   # conflict: remove the one ending later (which is current, since sorted by end)

    return removed
```

### Explanation

- Sort by **end time** (greedy: keep intervals that end earliest, as they cause fewer future conflicts).
- If current start >= last kept end → no overlap → keep it.
- If overlap → remove current (it ends later than or equal to the previous kept interval, so it's the worse choice).
- **Time:** O(n log n) | **Space:** O(1)

---

## Problem 4 — Meeting Rooms II (LeetCode #253)

**Given** meeting time intervals, find the minimum number of conference rooms required.

### Solution

```python
import heapq

def minMeetingRooms(intervals: list[list[int]]) -> int:
    if not intervals:
        return 0

    intervals.sort(key=lambda x: x[0])
    rooms = []  # min-heap of end times

    for start, end in intervals:
        if rooms and rooms[0] <= start:
            heapq.heapreplace(rooms, end)  # reuse the earliest-ending room
        else:
            heapq.heappush(rooms, end)     # need a new room

    return len(rooms)
```

### Explanation

- Sort by start time. The heap tracks when each room becomes free (end times).
- For each meeting, if the earliest-ending room is free (`rooms[0] <= start`), reuse it.
- Otherwise, allocate a new room.
- The heap size at the end = number of rooms needed.
- **Time:** O(n log n) | **Space:** O(n)

---

## Problem 5 — Minimum Number of Arrows to Burst Balloons (LeetCode #452)

**Given** balloons represented as `[start, end]`, find the minimum number of arrows to burst all of them. An arrow at position `x` bursts all balloons where `start <= x <= end`.

### Solution

```python
def findMinArrowShots(points: list[list[int]]) -> int:
    points.sort(key=lambda x: x[1])  # sort by end
    arrows = 1
    arrow_pos = points[0][1]

    for start, end in points[1:]:
        if start > arrow_pos:
            # current balloon not hit by last arrow
            arrows += 1
            arrow_pos = end

    return arrows
```

### Explanation

- Sort by end position. Shoot the first arrow at the end of the first balloon.
- This arrow also hits all balloons that start before or at that position.
- When a balloon starts after the current arrow position, shoot a new arrow at that balloon's end.
- **Time:** O(n log n) | **Space:** O(1)

---

## Key Takeaways

| Problem | Sort by | Strategy |
|---|---|---|
| Merge intervals | Start | Compare current start vs last end |
| Insert interval | (already sorted) | Three-phase linear scan |
| Minimum removals | End | Greedy keep earliest ending |
| Meeting rooms | Start | Min-heap of end times |
| Arrow bursting | End | Greedy shoot at earliest end |

**One key insight:** sorting by **end time** is the classic greedy move for scheduling problems. It maximizes the number of non-overlapping events you can fit, which minimizes removals/conflicts.
