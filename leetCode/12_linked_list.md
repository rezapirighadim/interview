# Linked List

## What is it?

A **linked list** is a linear data structure where each node holds a value and a pointer to the next node. Unlike arrays, there is no random access — you traverse from the head.

```
head → [1] → [2] → [3] → [4] → [5] → None
```

Variations:
- **Singly linked list** — each node has one `next` pointer
- **Doubly linked list** — each node has `next` and `prev`
- **Circular linked list** — last node points back to head

## When to use linked list techniques?

- **Reverse** a list or a portion of it
- **Detect/find** a cycle (Floyd's algorithm)
- **Find the middle** of a list
- **Merge** two sorted lists
- **Remove** Nth node from end
- **Reorder** nodes in a specific pattern

## Core patterns

### Fast & Slow Pointers (Floyd's)

```python
slow = head
fast = head

while fast and fast.next:
    slow = slow.next
    fast = fast.next.next
# When fast reaches end, slow is at the middle
```

### Dummy Head

```python
dummy = ListNode(0)
dummy.next = head
curr = dummy
# ... build list ...
return dummy.next  # skips the dummy
```

### In-place Reversal

```python
prev = None
curr = head

while curr:
    next_node = curr.next
    curr.next = prev
    prev = curr
    curr = next_node

return prev  # new head
```

---

## Problem 1 — Reverse Linked List (LeetCode #206)

**Given** the head of a singly linked list, reverse it and return the new head.

### Solution

```python
def reverseList(head: ListNode) -> ListNode:
    prev = None
    curr = head

    while curr:
        next_node = curr.next  # save next
        curr.next = prev       # reverse pointer
        prev = curr            # advance prev
        curr = next_node       # advance curr

    return prev
```

### Explanation

- We walk the list once, reversing each `next` pointer as we go.
- `prev` eventually becomes the new head (the old tail).
- Always save `curr.next` **before** redirecting the pointer, or you lose the rest of the list.
- **Time:** O(n) | **Space:** O(1)

---

## Problem 2 — Linked List Cycle (LeetCode #141)

**Given** a linked list, return `True` if it has a cycle.

### Solution

```python
def hasCycle(head: ListNode) -> bool:
    slow = fast = head

    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next

        if slow == fast:
            return True

    return False
```

### Explanation

- Floyd's cycle detection: `slow` moves 1 step, `fast` moves 2 steps.
- If there is a cycle, `fast` will lap `slow` and they will meet inside the cycle.
- If there is no cycle, `fast` reaches `None`.
- **Time:** O(n) | **Space:** O(1)

---

## Problem 3 — Merge Two Sorted Lists (LeetCode #21)

**Given** two sorted linked lists, merge them into one sorted list.

### Solution

```python
def mergeTwoLists(list1: ListNode, list2: ListNode) -> ListNode:
    dummy = ListNode(0)
    curr = dummy

    while list1 and list2:
        if list1.val <= list2.val:
            curr.next = list1
            list1 = list1.next
        else:
            curr.next = list2
            list2 = list2.next
        curr = curr.next

    curr.next = list1 or list2  # attach the remaining list

    return dummy.next
```

### Explanation

- Use a dummy head to simplify edge cases (empty list, etc.).
- Compare the current heads and attach the smaller one.
- When one list is exhausted, attach the rest of the other — no need to loop.
- **Time:** O(m + n) | **Space:** O(1)

---

## Problem 4 — Remove Nth Node From End (LeetCode #19)

**Given** a linked list and `n`, remove the nth node from the end and return the head.

### Solution

```python
def removeNthFromEnd(head: ListNode, n: int) -> ListNode:
    dummy = ListNode(0)
    dummy.next = head
    fast = slow = dummy

    # advance fast n+1 steps so the gap between fast and slow is n
    for _ in range(n + 1):
        fast = fast.next

    # move both until fast reaches None
    while fast:
        fast = fast.next
        slow = slow.next

    # slow is now just before the node to remove
    slow.next = slow.next.next

    return dummy.next
```

### Explanation

- We use a **two-pointer gap trick**: move `fast` n+1 steps ahead so when `fast` hits `None`, `slow` is right before the target node.
- The dummy head handles the edge case where we remove the head itself.
- **Time:** O(n) | **Space:** O(1)

---

## Problem 5 — Reorder List (LeetCode #143)

**Given** a list L₀→L₁→…→Lₙ, reorder it to L₀→Lₙ→L₁→Lₙ₋₁→…

### Solution

```python
def reorderList(head: ListNode) -> None:
    # Step 1: find middle
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next

    # Step 2: reverse second half
    prev = None
    curr = slow.next
    slow.next = None  # cut the list in half
    while curr:
        next_node = curr.next
        curr.next = prev
        prev = curr
        curr = next_node
    second = prev

    # Step 3: merge two halves
    first = head
    while second:
        tmp1, tmp2 = first.next, second.next
        first.next = second
        second.next = tmp1
        first = tmp1
        second = tmp2
```

### Explanation

- Three clear steps: find middle → reverse second half → merge alternately.
- This is a combination of three fundamental linked list operations.
- No extra space needed — all done in-place.
- **Time:** O(n) | **Space:** O(1)

---

## Key Takeaways

| Technique | Used for |
|---|---|
| Fast & slow pointers | Cycle detection, finding middle |
| Dummy head | Simplifies edge cases (empty list, remove head) |
| In-place reversal | Reverse full or partial list |
| Two-pointer gap | Nth from end |
| Split + reverse + merge | Reorder list |

**The dummy head is your best friend** in linked list problems. It eliminates special handling for the head node being removed or modified, and almost every problem becomes cleaner with it.
