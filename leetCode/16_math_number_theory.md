# Math & Number Theory

## What is it?

Mathematical techniques in LeetCode rely on properties of numbers — divisibility, prime factorization, modular arithmetic, GCD, and geometric reasoning. These problems often have elegant O(1) or O(√n) solutions that brute force completely misses.

## Key concepts

### GCD and LCM

```python
import math
math.gcd(a, b)          # built-in GCD (Euclidean algorithm)
a * b // math.gcd(a, b) # LCM
```

### Modular Arithmetic

```python
# (a + b) % m = (a % m + b % m) % m
# (a * b) % m = (a % m) * (b % m) % m
# Useful when answers can be very large

MOD = 10**9 + 7
result = (a * b) % MOD
```

### Prime Checking

```python
def is_prime(n):
    if n < 2:
        return False
    for i in range(2, int(n**0.5) + 1):  # only check up to √n
        if n % i == 0:
            return False
    return True
```

### Sieve of Eratosthenes (all primes up to n)

```python
def sieve(n):
    is_prime = [True] * (n + 1)
    is_prime[0] = is_prime[1] = False
    for i in range(2, int(n**0.5) + 1):
        if is_prime[i]:
            for j in range(i*i, n+1, i):
                is_prime[j] = False
    return [i for i in range(n+1) if is_prime[i]]
```

---

## Problem 1 — Reverse Integer (LeetCode #7)

**Given** a 32-bit signed integer, reverse its digits. Return 0 if it overflows.

### Solution

```python
def reverse(x: int) -> int:
    INT_MAX = 2**31 - 1
    INT_MIN = -(2**31)

    sign = -1 if x < 0 else 1
    x = abs(x)

    reversed_x = int(str(x)[::-1]) * sign

    if reversed_x < INT_MIN or reversed_x > INT_MAX:
        return 0

    return reversed_x
```

### Explanation

- Convert to string, reverse, convert back. Handle the sign separately.
- Check 32-bit overflow bounds at the end.
- Python doesn't overflow natively, so we check manually.
- **Time:** O(log x) — number of digits | **Space:** O(log x)

---

## Problem 2 — Happy Number (LeetCode #202)

**Given** an integer, determine if it is "happy". A happy number eventually reaches 1 when repeatedly replacing it with the sum of squares of its digits.

### Solution

```python
def isHappy(n: int) -> bool:
    def digit_square_sum(num):
        total = 0
        while num:
            num, digit = divmod(num, 10)
            total += digit ** 2
        return total

    slow = n
    fast = digit_square_sum(n)

    while fast != 1 and slow != fast:
        slow = digit_square_sum(slow)
        fast = digit_square_sum(digit_square_sum(fast))

    return fast == 1
```

### Explanation

- If the number is not happy, it will eventually enter a cycle.
- We use Floyd's cycle detection: slow advances once, fast advances twice.
- If they meet at 1, it's happy. If they meet elsewhere, it's a cycle.
- **Time:** O(log n) | **Space:** O(1)

---

## Problem 3 — Count Primes (LeetCode #204)

**Given** `n`, return the number of primes less than `n`.

### Solution

```python
def countPrimes(n: int) -> int:
    if n < 2:
        return 0

    is_prime = [True] * n
    is_prime[0] = is_prime[1] = False

    for i in range(2, int(n**0.5) + 1):
        if is_prime[i]:
            for j in range(i * i, n, i):
                is_prime[j] = False

    return sum(is_prime)
```

### Explanation

- Sieve of Eratosthenes: mark every multiple of each prime as composite.
- Start inner loop at `i * i` (smaller multiples were already marked by previous primes).
- Only check up to `√n` in the outer loop.
- **Time:** O(n log log n) | **Space:** O(n)

---

## Problem 4 — Pow(x, n) (LeetCode #50)

Implement `x^n` (power function).

### Solution

```python
def myPow(x: float, n: int) -> float:
    def fast_pow(base, exp):
        if exp == 0:
            return 1.0
        if exp % 2 == 0:
            half = fast_pow(base, exp // 2)
            return half * half
        else:
            return base * fast_pow(base, exp - 1)

    if n < 0:
        x = 1 / x
        n = -n

    return fast_pow(x, n)
```

### Explanation

- **Fast exponentiation (binary exponentiation):** instead of multiplying `x` n times, we square repeatedly.
- `x^n = (x^(n/2))^2` if n is even — cuts the problem in half each time.
- This achieves O(log n) instead of O(n).
- **Time:** O(log n) | **Space:** O(log n) recursion depth

---

## Problem 5 — Excel Sheet Column Number (LeetCode #171)

**Given** a column title like `"AB"`, return its corresponding number.

### Solution

```python
def titleToNumber(columnTitle: str) -> int:
    result = 0

    for char in columnTitle:
        result = result * 26 + (ord(char) - ord('A') + 1)

    return result
```

### Explanation

- This is a **base-26** number system where A=1, B=2, ..., Z=26.
- Each character contributes its value, shifted left by multiplying by 26.
- `"AB" = 1*26 + 2 = 28`
- Exactly like converting a binary or hexadecimal string to decimal.
- **Time:** O(n) | **Space:** O(1)

---

## Bonus — Common Math Tricks

```python
# Integer square root without float
import math
isqrt = math.isqrt(n)     # Python 3.8+

# Check if perfect square
math.isqrt(n) ** 2 == n

# Sum of 1 to n
n * (n + 1) // 2

# Number of digits in n
len(str(n))   # or  math.floor(math.log10(n)) + 1

# All factors of n
factors = []
for i in range(1, int(n**0.5) + 1):
    if n % i == 0:
        factors.append(i)
        if i != n // i:
            factors.append(n // i)
```

---

## Key Takeaways

| Topic | Key fact |
|---|---|
| Primality | Only check up to √n |
| All primes ≤ n | Sieve of Eratosthenes: O(n log log n) |
| Fast power | Square repeatedly: O(log n) |
| GCD | Euclidean: `math.gcd(a, b)` |
| Cycle detection | Use Floyd's (fast/slow) even in math sequences |
| Base conversion | Treat digit string as positional number system |

**The √n pattern is everywhere in math problems.** Factors come in pairs around √n, primes only need to be checked up to √n, and many number theory proofs hinge on this symmetry.
