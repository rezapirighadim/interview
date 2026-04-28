# Bit Manipulation

## What is it?

Bit manipulation works directly with the binary representation of integers using **bitwise operators**. It's often used to replace modular arithmetic, flags, or set operations with extremely fast O(1) operations.

```
5  = 0101
3  = 0011
---------
5 & 3 = 0001 (AND)
5 | 3 = 0111 (OR)
5 ^ 3 = 0110 (XOR)
~5    = -6   (NOT, two's complement)
5 << 1 = 10  (left shift, multiply by 2)
5 >> 1 = 2   (right shift, divide by 2)
```

## Fundamental tricks

```python
# Check if bit i is set
(n >> i) & 1

# Set bit i
n |= (1 << i)

# Clear bit i
n &= ~(1 << i)

# Toggle bit i
n ^= (1 << i)

# Check if n is a power of 2
n > 0 and (n & (n - 1)) == 0

# Remove the lowest set bit
n & (n - 1)

# Get the lowest set bit
n & (-n)

# Count set bits (Brian Kernighan)
count = 0
while n:
    n &= n - 1   # removes lowest set bit each time
    count += 1
```

## When to use it?

- XOR problems: "find the element that appears once" (pairs cancel out)
- Checking/setting individual flags
- Power of 2 checks
- Counting set bits
- Subset enumeration (bitmask DP)
- The problem constraints are small (n ≤ 20 or 32) — bitmask over states

---

## Problem 1 — Single Number (LeetCode #136)

**Given** an array where every element appears twice except one, find that one.

### Solution

```python
def singleNumber(nums: list[int]) -> int:
    result = 0
    for num in nums:
        result ^= num
    return result
```

### Explanation

- XOR of a number with itself is 0: `a ^ a = 0`
- XOR of a number with 0 is itself: `a ^ 0 = a`
- XOR is commutative and associative, so all pairs cancel out, leaving the unique number.
- **Time:** O(n) | **Space:** O(1)

---

## Problem 2 — Number of 1 Bits (LeetCode #191)

**Given** an integer, return the number of `1` bits (Hamming weight).

### Solution

```python
def hammingWeight(n: int) -> int:
    count = 0
    while n:
        n &= n - 1  # remove the lowest set bit
        count += 1
    return count
```

### Explanation

- `n & (n - 1)` clears the **rightmost** set bit in `n` (it flips all bits up to and including the lowest set bit).
- We repeat until `n` becomes 0, counting how many set bits we removed.
- This runs in O(number of 1 bits), which is faster than shifting through all 32 bits.
- **Time:** O(k) where k = number of set bits | **Space:** O(1)

---

## Problem 3 — Counting Bits (LeetCode #338)

**Given** `n`, return an array where `result[i]` = number of 1 bits in `i`, for `0 <= i <= n`.

### Solution

```python
def countBits(n: int) -> list[int]:
    dp = [0] * (n + 1)

    for i in range(1, n + 1):
        dp[i] = dp[i >> 1] + (i & 1)
        # i >> 1 = i // 2 (drop last bit, already computed)
        # (i & 1) = 1 if i is odd, 0 if even

    return dp
```

### Explanation

- Key insight: the number of bits in `i` = bits in `i // 2` + the last bit of `i`.
- `i >> 1` shifts right by 1 (removes the last bit), and `i & 1` checks if the last bit is set.
- Since we compute in order, `dp[i >> 1]` is already known.
- **Time:** O(n) | **Space:** O(n)

---

## Problem 4 — Reverse Bits (LeetCode #190)

**Given** a 32-bit unsigned integer, reverse its bits.

### Solution

```python
def reverseBits(n: int) -> int:
    result = 0
    for _ in range(32):
        result = (result << 1) | (n & 1)  # shift result left, add current last bit of n
        n >>= 1                            # shift n right
    return result
```

### Explanation

- We extract the last bit of `n` with `n & 1`, and append it to the left of `result` by shifting `result` left first.
- After 32 iterations, all bits have been reversed.
- **Time:** O(32) = O(1) | **Space:** O(1)

---

## Problem 5 — Sum of Two Integers (LeetCode #371)

**Given** two integers `a` and `b`, return their sum **without using `+` or `-`**.

### Solution

```python
def getSum(a: int, b: int) -> int:
    mask = 0xFFFFFFFF  # 32-bit mask

    while b & mask:
        carry = (a & b) << 1  # carry bits
        a = a ^ b              # sum without carry
        b = carry

    return a if b == 0 else ~(a ^ mask)
```

### Explanation

- XOR computes addition **without** carry: `1 ^ 1 = 0`, `1 ^ 0 = 1`.
- AND then shift computes the **carry**: `(a & b) << 1`.
- We repeat until there's no carry left.
- The mask handles Python's arbitrary-precision integers, keeping us within 32 bits.
- **Time:** O(32) = O(1) | **Space:** O(1)

---

## Bitmask DP Bonus — Subsets

To iterate over all subsets of `n` elements using bitmasks:

```python
n = 4  # 4 elements, subsets are 0000 to 1111
for mask in range(1 << n):
    subset = []
    for i in range(n):
        if mask & (1 << i):
            subset.append(i)
    print(subset)
# Useful for small n (≤ 20) where 2^n subsets are feasible
```

---

## Key Takeaways

| Trick | Expression | Use case |
|---|---|---|
| Check bit i | `(n >> i) & 1` | Is the ith flag set? |
| Power of 2 | `n & (n-1) == 0` | Quick check |
| Remove lowest bit | `n & (n-1)` | Count set bits |
| XOR cancels pairs | `a ^ a = 0` | Find unique element |
| Add without `+` | XOR + carry | Interview trick |
| Subset iteration | `for mask in range(1 << n)` | Bitmask DP |

**XOR is the star of bit manipulation.** Whenever you see "find the unique element" or "pairs of identical values", XOR is almost certainly the answer.
