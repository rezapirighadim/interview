# Advanced SQL — Window Functions, CTEs, and Tricky Queries

> Window functions and CTEs are where SQL interviews separate juniors from seniors. Learn the pattern, then the syntax.

---

## Window Functions

A window function computes a value for each row using a set of rows related to that row — without collapsing them like GROUP BY does. Each row keeps its identity; the window result is just another column.

```sql
function_name() OVER (
    PARTITION BY col   -- divide rows into groups (optional)
    ORDER BY col       -- define order within the group
    ROWS/RANGE BETWEEN ...  -- define the frame (optional)
)
```

---

### ROW_NUMBER — Unique sequential number per partition

**Use case:** Get the most recent order per user (top-N per group).

```sql
SELECT *
FROM (
    SELECT
        order_id,
        user_id,
        total_amount,
        created_at,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM orders
) sub
WHERE rn = 1;
```

`ROW_NUMBER` always produces unique numbers — ties get arbitrary ordering. Use `RANK` or `DENSE_RANK` if ties matter.

---

### RANK and DENSE_RANK — Ranking with ties

**Use case:** Find the top 3 highest-paid employees per department, counting ties correctly.

```sql
SELECT *
FROM (
    SELECT
        emp_id,
        dept_id,
        salary,
        RANK()       OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk,
        DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS dense_rnk
    FROM employees
) sub
WHERE dense_rnk <= 3;
```

| salary | RANK | DENSE_RANK |
|---|---|---|
| 100k | 1 | 1 |
| 90k  | 2 | 2 |
| 90k  | 2 | 2 |
| 80k  | 4 | 3 |  ← RANK skips 3; DENSE_RANK does not |

Use `DENSE_RANK` when "top 3 salary levels" is what you mean. Use `RANK` when "3rd place in a competition" is what you mean (a tie means no one gets 3rd).

---

### LAG and LEAD — Access adjacent rows

**Use case:** Calculate month-over-month revenue change.

```sql
WITH monthly AS (
    SELECT
        DATE_TRUNC('month', created_at) AS month,
        SUM(total_amount)               AS revenue
    FROM orders
    GROUP BY 1
)
SELECT
    month,
    revenue,
    LAG(revenue)  OVER (ORDER BY month) AS prev_month_revenue,
    revenue - LAG(revenue) OVER (ORDER BY month) AS delta,
    ROUND(
        100.0 * (revenue - LAG(revenue) OVER (ORDER BY month))
              / NULLIF(LAG(revenue) OVER (ORDER BY month), 0),
        2
    ) AS pct_change
FROM monthly
ORDER BY month;
```

`LAG(col, n, default)` — look n rows back. `LEAD(col, n, default)` — look n rows forward. The third argument is the default when the window goes out of bounds (NULL by default).

---

### SUM OVER PARTITION — Running totals and aggregates within groups

**Use case:** Show each order alongside the running total of that user's spend.

```sql
SELECT
    order_id,
    user_id,
    total_amount,
    SUM(total_amount) OVER (
        PARTITION BY user_id
        ORDER BY created_at
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total,
    AVG(total_amount) OVER (
        PARTITION BY user_id
        ORDER BY created_at
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_7_order_avg
FROM orders;
```

**Frame types:**
- `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — running total from start to current row.
- `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` — 7-row rolling window.
- `RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW` — time-based rolling window (needs `ORDER BY` on a date column).

---

## CTEs — Common Table Expressions

```sql
WITH cte_name AS (
    SELECT ...
),
another_cte AS (
    SELECT ... FROM cte_name ...
)
SELECT * FROM another_cte;
```

CTEs are named subqueries. They improve readability; in PostgreSQL 12+, the planner inlines non-recursive CTEs by default (same execution plan as a subquery). Add `MATERIALIZED` to force a separate execution if you need to scan a CTE result multiple times without recomputing.

---

### Recursive CTE — Org Hierarchy and Graph Traversal

**Use case:** Find all reports under a given manager (org chart).

```sql
-- employees table: emp_id, name, manager_id
WITH RECURSIVE org_tree AS (
    -- Base case: the starting employee
    SELECT emp_id, name, manager_id, 0 AS depth
    FROM employees
    WHERE emp_id = 42  -- start from manager with ID 42

    UNION ALL

    -- Recursive case: join direct reports
    SELECT e.emp_id, e.name, e.manager_id, ot.depth + 1
    FROM employees e
    JOIN org_tree ot ON e.manager_id = ot.emp_id
)
SELECT emp_id, name, depth
FROM org_tree
ORDER BY depth, name;
```

**Use case:** Find the full path in a category tree.

```sql
WITH RECURSIVE category_path AS (
    SELECT category_id, name, parent_id, ARRAY[name] AS path
    FROM categories
    WHERE category_id = 55  -- leaf node

    UNION ALL

    SELECT c.category_id, c.name, c.parent_id, ARRAY[c.name] || cp.path
    FROM categories c
    JOIN category_path cp ON c.category_id = cp.parent_id
)
SELECT path
FROM category_path
WHERE parent_id IS NULL;  -- root reached
-- Result: {'Electronics', 'Computers', 'Laptops', 'Gaming Laptops'}
```

**Safety:** add `WHERE depth < 100` or use `CYCLE` detection (PostgreSQL 14+) to prevent infinite loops on cyclic data.

```sql
WITH RECURSIVE ... (
    ...
    UNION ALL
    SELECT ... WHERE depth < 50  -- cycle guard
)
```

---

## Classic Interview Query Problems

### 1. Nth Highest Salary

```sql
-- Find the 3rd highest distinct salary
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 2;  -- OFFSET N-1

-- Or with DENSE_RANK to handle ties correctly:
SELECT salary
FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS dr
    FROM employees
) sub
WHERE dr = 3
LIMIT 1;
```

---

### 2. Employees Who Earn More Than Their Manager

```sql
SELECT e.emp_id, e.name, e.salary, m.salary AS manager_salary
FROM employees e
JOIN employees m ON e.manager_id = m.emp_id
WHERE e.salary > m.salary;
```

Self-join on the same table. The alias trick (`e` for employee, `m` for manager) is the key — make sure you can explain why two aliases are needed.

---

### 3. Find Duplicate Rows and Keep Only One

```sql
-- Identify duplicates: find email addresses that appear more than once
SELECT email, COUNT(*) AS cnt
FROM users
GROUP BY email
HAVING COUNT(*) > 1;

-- Delete duplicates, keep the row with the lowest user_id
DELETE FROM users
WHERE user_id NOT IN (
    SELECT MIN(user_id)
    FROM users
    GROUP BY email
);

-- With CTE (cleaner, same effect):
WITH ranked AS (
    SELECT user_id,
           ROW_NUMBER() OVER (PARTITION BY email ORDER BY user_id) AS rn
    FROM users
)
DELETE FROM users
WHERE user_id IN (SELECT user_id FROM ranked WHERE rn > 1);
```

---

### 4. Running Total and Rolling Average

```sql
SELECT
    order_date,
    daily_revenue,
    SUM(daily_revenue) OVER (ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total,
    ROUND(
        AVG(daily_revenue) OVER (ORDER BY order_date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW),
        2
    ) AS rolling_7day_avg
FROM (
    SELECT
        DATE(created_at)  AS order_date,
        SUM(total_amount) AS daily_revenue
    FROM orders
    GROUP BY DATE(created_at)
) daily
ORDER BY order_date;
```

---

### 5. Longest Streak of Consecutive Login Days

```sql
WITH daily_logins AS (
    -- Deduplicate: one row per user per day
    SELECT DISTINCT user_id, DATE(logged_in_at) AS login_date
    FROM user_sessions
),
grouped AS (
    -- Subtract a sequential row number to get a group identifier.
    -- If logins are consecutive, login_date - rn stays constant.
    SELECT
        user_id,
        login_date,
        login_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date)::INT * INTERVAL '1 day')
            AS streak_group
    FROM daily_logins
),
streak_lengths AS (
    SELECT
        user_id,
        streak_group,
        COUNT(*) AS streak_len,
        MIN(login_date) AS streak_start,
        MAX(login_date) AS streak_end
    FROM grouped
    GROUP BY user_id, streak_group
)
SELECT user_id, streak_len, streak_start, streak_end
FROM streak_lengths
ORDER BY streak_len DESC;
```

**The key insight:** subtracting a row number from a date collapses consecutive dates into the same group value. Non-consecutive dates produce different group values. `COUNT(*)` per group = streak length.

---

### 6. Find Gaps in a Sequence

```sql
-- Find missing IDs in an order sequence (assuming sequential order IDs)
SELECT
    order_id + 1            AS gap_start,
    next_order_id - 1       AS gap_end,
    next_order_id - order_id - 1 AS gap_size
FROM (
    SELECT
        order_id,
        LEAD(order_id) OVER (ORDER BY order_id) AS next_order_id
    FROM orders
) sub
WHERE next_order_id - order_id > 1;
```

**Generalizing to date gaps:**

```sql
-- Find days with no orders
WITH date_series AS (
    SELECT generate_series(
        (SELECT MIN(DATE(created_at)) FROM orders),
        (SELECT MAX(DATE(created_at)) FROM orders),
        INTERVAL '1 day'
    )::DATE AS day
),
order_days AS (
    SELECT DISTINCT DATE(created_at) AS order_day FROM orders
)
SELECT ds.day AS missing_day
FROM date_series ds
LEFT JOIN order_days od ON ds.day = od.order_day
WHERE od.order_day IS NULL
ORDER BY ds.day;
```

---

### 7. Pivot / Crosstab a Table

**Goal:** Turn row-level monthly data into columns.

```sql
-- Input: (year, month, revenue)
-- Output: one row per year, columns for Jan, Feb, ..., Dec

SELECT
    year,
    SUM(CASE WHEN month = 1  THEN revenue ELSE 0 END) AS jan,
    SUM(CASE WHEN month = 2  THEN revenue ELSE 0 END) AS feb,
    SUM(CASE WHEN month = 3  THEN revenue ELSE 0 END) AS mar,
    SUM(CASE WHEN month = 4  THEN revenue ELSE 0 END) AS apr,
    SUM(CASE WHEN month = 5  THEN revenue ELSE 0 END) AS may,
    SUM(CASE WHEN month = 6  THEN revenue ELSE 0 END) AS jun,
    SUM(CASE WHEN month = 7  THEN revenue ELSE 0 END) AS jul,
    SUM(CASE WHEN month = 8  THEN revenue ELSE 0 END) AS aug,
    SUM(CASE WHEN month = 9  THEN revenue ELSE 0 END) AS sep,
    SUM(CASE WHEN month = 10 THEN revenue ELSE 0 END) AS oct,
    SUM(CASE WHEN month = 11 THEN revenue ELSE 0 END) AS nov,
    SUM(CASE WHEN month = 12 THEN revenue ELSE 0 END) AS dec
FROM (
    SELECT
        EXTRACT(YEAR  FROM created_at)::INT AS year,
        EXTRACT(MONTH FROM created_at)::INT AS month,
        SUM(total_amount) AS revenue
    FROM orders
    GROUP BY 1, 2
) monthly
GROUP BY year
ORDER BY year;
```

PostgreSQL also has the `crosstab()` function in the `tablefunc` extension for dynamic pivot, but the `CASE WHEN` approach is what interviewers expect you to know cold.

---

### 8. Most Recent Record Per Group (Top-N Per Group)

Three equivalent approaches — know all three:

```sql
-- Approach A: ROW_NUMBER (most flexible, handles top-N where N > 1)
SELECT *
FROM (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM orders
) sub
WHERE rn = 1;

-- Approach B: DISTINCT ON (PostgreSQL-specific, very clean for top-1)
SELECT DISTINCT ON (user_id)
    user_id, order_id, total_amount, created_at
FROM orders
ORDER BY user_id, created_at DESC;

-- Approach C: NOT EXISTS (works in any SQL dialect)
SELECT o.*
FROM orders o
WHERE NOT EXISTS (
    SELECT 1 FROM orders o2
    WHERE o2.user_id = o.user_id
      AND o2.created_at > o.created_at
);
```

`DISTINCT ON` is the idiomatic PostgreSQL answer. `ROW_NUMBER` is the portable answer. Know the trade-off.

---

## Transactions: ACID

| Property | Meaning | How the DB enforces it |
|---|---|---|
| **Atomicity** | All operations in a transaction succeed, or none do | Rollback log (WAL/undo log) |
| **Consistency** | Transaction brings DB from one valid state to another | Constraints, triggers, FK checks |
| **Isolation** | Concurrent transactions don't interfere with each other | Locking, MVCC (Multi-Version Concurrency Control) |
| **Durability** | Committed transactions survive crashes | Write-ahead log (WAL) flushed to disk |

```sql
BEGIN;
    UPDATE accounts SET balance = balance - 500 WHERE account_id = 1;
    UPDATE accounts SET balance = balance + 500 WHERE account_id = 2;
    -- If the second UPDATE fails, the first is rolled back automatically
COMMIT;
-- or ROLLBACK; to undo everything
```

---

## Isolation Levels

PostgreSQL implements isolation via MVCC: readers don't block writers; each transaction sees a snapshot.

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| Read Uncommitted | Possible (PG treats as Read Committed) | Possible | Possible |
| **Read Committed** (PG default) | Not possible | Possible | Possible |
| Repeatable Read | Not possible | Not possible | Possible (PG: also prevented) |
| Serializable | Not possible | Not possible | Not possible |

**Anomaly definitions:**

- **Dirty read:** Reading uncommitted data from another transaction that might be rolled back.
- **Non-repeatable read:** Reading the same row twice in one transaction and getting different values (another transaction committed a change in between).
- **Phantom read:** Running the same range query twice in one transaction and getting different sets of rows (another transaction inserted/deleted rows in between).

```sql
-- Set isolation level for the current transaction:
BEGIN ISOLATION LEVEL REPEATABLE READ;
    SELECT balance FROM accounts WHERE account_id = 1;
    -- ... some processing ...
    SELECT balance FROM accounts WHERE account_id = 1;  -- guaranteed same result
COMMIT;
```

**When to use each:**
- **Read Committed:** default; fine for most OLTP operations.
- **Repeatable Read:** reporting queries that run multiple SELECTs and need a consistent snapshot, financial calculations.
- **Serializable:** critical financial operations, inventory that absolutely cannot oversell, anywhere you'd otherwise use application-level locking.

---

## Deadlocks

**What causes them:** Two transactions each hold a lock the other needs.

```
Transaction A: locks Row 1, then tries to lock Row 2
Transaction B: locks Row 2, then tries to lock Row 1
→ Both wait forever = deadlock
```

**PostgreSQL detects deadlocks** automatically (deadlock_timeout, default 1s) and kills one transaction with:
```
ERROR: deadlock detected
DETAIL: Process 12345 waits for ShareLock on transaction 67890; blocked by process 67890.
```

**How to detect in production:**

```sql
-- View current locks and waiting queries
SELECT
    blocked.pid,
    blocked.query,
    blocking.pid   AS blocking_pid,
    blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.cardinality(pg_blocking_pids(blocked.pid)) > 0;
```

**How to prevent deadlocks:**

1. **Always acquire locks in the same order.** If Transaction A always locks users before orders, and Transaction B does too, they can never deadlock on those two resources.

    ```sql
    -- Both transactions: lock the lower account_id first
    BEGIN;
    SELECT * FROM accounts WHERE account_id = LEAST(1, 2)    FOR UPDATE;
    SELECT * FROM accounts WHERE account_id = GREATEST(1, 2) FOR UPDATE;
    ```

2. **Keep transactions short.** Long transactions hold locks longer, increasing the window for conflict.

3. **Use `SELECT ... FOR UPDATE SKIP LOCKED`** for queue-like patterns — skip rows someone else is processing rather than waiting.

    ```sql
    -- Worker claiming a job from a queue
    BEGIN;
    SELECT job_id FROM jobs
    WHERE status = 'pending'
    ORDER BY created_at
    LIMIT 1
    FOR UPDATE SKIP LOCKED;
    -- process the job ...
    UPDATE jobs SET status = 'done' WHERE job_id = ?;
    COMMIT;
    ```

4. **Use advisory locks** for application-level mutexes when you need to serialize a business operation without locking a table row.

    ```sql
    SELECT pg_advisory_xact_lock(user_id);  -- released automatically at COMMIT/ROLLBACK
    ```

5. **Retry on deadlock.** In application code, catch the deadlock error and retry the entire transaction — it is always safe to retry because the losing transaction was fully rolled back.
