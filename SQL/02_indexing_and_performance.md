# Indexing & Query Performance

> Indexes are the single highest-leverage tool for query performance. Know when they help, when they hurt, and how to read a query plan.

---

## B-Tree Index Internals (What You Need to Know)

A B-tree index is a balanced tree where:
- **Leaf nodes** hold actual index entries: the column value + a pointer (heap tuple ID) to the actual row.
- **Internal nodes** hold separator keys that route searches to the right leaf.
- The tree stays balanced — all leaf nodes are at the same depth.

**What this means for performance:**
- Point lookup: O(log n) — follow internal nodes down to a leaf.
- Range scan: O(log n + k) — find the start leaf, then scan sibling leaves for k matching rows.
- Full scan through an index is slower than a seq scan on small tables — the planner knows this.

**What B-tree indexes support:** `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `IN`, `IS NULL`, `LIKE 'prefix%'` (prefix match only).

**What they do NOT support:** `LIKE '%suffix'`, `LIKE '%middle%'`, regex, full-text search — use a different index type for those.

---

## When Indexes Help vs. Hurt

| Situation | Index helps? | Why |
|---|---|---|
| High-cardinality filter (`user_id = 42`) | Yes | Eliminates most rows immediately |
| Low-cardinality filter (`status = 'active'` with 90% active) | No | Seq scan + filter is cheaper than random I/O to fetch most rows |
| ORDER BY + LIMIT on indexed column | Yes | Index already sorted; skip the sort |
| JOIN on FK column | Yes | Avoids nested-loop full scan of the inner table |
| Heavy write table (logging, events) | Caution | Every INSERT/UPDATE/DELETE must update all indexes on that table |
| Small table (< ~10K rows) | No | Planner will seq scan regardless; index maintenance is wasted |
| Aggregates on unfiltered column (`SUM(revenue)`) | No | Must touch all rows anyway |
| Covering index (all needed columns in index) | Yes | Query answered from index alone; no heap access |

---

## Composite Index Column Order — The Leftmost Prefix Rule

A composite index `(a, b, c)` can be used for queries filtering on:
- `a` alone
- `a, b` together
- `a, b, c` together

It **cannot** be used for:
- `b` alone
- `c` alone
- `b, c` together

```sql
CREATE INDEX idx_orders_user_status ON orders (user_id, status);

-- USES the index (leading column present):
SELECT * FROM orders WHERE user_id = 5;
SELECT * FROM orders WHERE user_id = 5 AND status = 'shipped';

-- DOES NOT use the index:
SELECT * FROM orders WHERE status = 'shipped';  -- missing user_id
```

**Rule of thumb for column order:**
1. Equality filters first (`=` conditions).
2. Range filters last (`>`, `<`, `BETWEEN`).
3. Columns in `ORDER BY` after that, if possible.

```sql
-- Query: WHERE user_id = 5 AND created_at > '2024-01-01' ORDER BY created_at

-- Good index order:
CREATE INDEX ON orders (user_id, created_at);
-- user_id is equality → goes first. created_at handles both range filter and ORDER BY.
```

---

## Covering Indexes

A covering index includes all columns a query needs — the query is answered entirely from the index without touching the heap (table rows).

```sql
-- Query:
SELECT email, full_name FROM users WHERE status = 'active';

-- Without covering index: index scan on status, then heap fetch for email and full_name
CREATE INDEX idx_users_status ON users (status);

-- With covering index: no heap access at all
CREATE INDEX idx_users_status_covering ON users (status) INCLUDE (email, full_name);
```

PostgreSQL uses `INCLUDE` to add non-searchable columns to a leaf node. The planner shows `Index Only Scan` in EXPLAIN when a covering index is used.

---

## Index Types

| Type | Syntax | Best for |
|---|---|---|
| B-tree (default) | `CREATE INDEX ON t (col)` | Equality, range, sorting — use this 95% of the time |
| Unique | `CREATE UNIQUE INDEX ON t (col)` | Enforce uniqueness + index in one step |
| Partial | `CREATE INDEX ON t (col) WHERE condition` | Index a subset of rows; much smaller, much faster |
| Composite | `CREATE INDEX ON t (col1, col2)` | Multi-column filters; leftmost prefix rule applies |
| Hash | `CREATE INDEX USING HASH ON t (col)` | Equality only; no range support; rarely worth it over B-tree |
| GIN | `CREATE INDEX USING GIN ON t (col)` | Arrays, JSONB keys, full-text search (`tsvector`) |
| GiST | `CREATE INDEX USING GIST ON t (col)` | Geometric types, range types, full-text |
| Full-text | `CREATE INDEX ON t USING GIN (to_tsvector('english', body))` | Text search with `@@` operator |

**Partial index example — the best bang-for-buck optimization most teams miss:**

```sql
-- 95% of orders are 'delivered'. Queries almost always filter on 'pending' or 'processing'.
CREATE INDEX idx_orders_active ON orders (created_at)
    WHERE status IN ('pending', 'processing');

-- Result: tiny index covering only 5% of rows. Lookups are dramatically faster.
```

---

## Reading EXPLAIN / EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE
SELECT u.email, COUNT(o.order_id)
FROM users u
JOIN orders o ON u.user_id = o.user_id
WHERE o.status = 'pending'
GROUP BY u.email;
```

**Key nodes to recognize:**

| Node | Meaning | When you see it |
|---|---|---|
| `Seq Scan` | Full table scan | No usable index, or table is tiny |
| `Index Scan` | Uses B-tree, fetches heap rows | Selective filter on indexed column |
| `Index Only Scan` | Covering index — no heap access | All needed columns are in the index |
| `Bitmap Index Scan` + `Bitmap Heap Scan` | Multiple index lookups batched | Low-selectivity or OR conditions |
| `Hash Join` | Build hash table of smaller input, probe with larger | Large unsorted joins |
| `Merge Join` | Both inputs sorted on join key | When both sides are already sorted (or index-sorted) |
| `Nested Loop` | For each outer row, scan inner | Small outer result set + inner has good index |
| `Sort` | Explicit sort step | No index on ORDER BY column |
| `Aggregate` | GROUP BY or aggregation | Normal; check it comes after filtering |

**What to look for:**

```
-> Seq Scan on orders  (cost=0.00..45000.00 rows=1200000 width=20)
                              Filter: (status = 'pending')
                              Rows Removed by Filter: 1180000
```

- `Rows Removed by Filter` on a `Seq Scan` is the danger signal — you're reading 1.2M rows to keep 20K. Add an index.
- `cost=X..Y`: X is startup cost, Y is total cost. Compare Y across plan nodes to find the bottleneck.
- `actual time=X..Y`: X is first row latency, Y is total actual time. High difference between `rows=estimate` and `actual rows` means stale statistics — run `ANALYZE`.

```sql
-- After adding index:
EXPLAIN ANALYZE SELECT ...;
-- Look for: Index Scan or Bitmap Index Scan replacing Seq Scan
-- Look for: actual rows close to estimated rows (good statistics)
-- Look for: Index Only Scan if you added a covering index
```

---

## The N+1 Query Problem

**What it is:** Your application runs 1 query to fetch N parent records, then N additional queries to fetch each parent's children — one per row.

```python
# N+1 in Python/ORM pseudocode
orders = db.query("SELECT * FROM orders WHERE user_id = 5")  # 1 query
for order in orders:
    items = db.query(f"SELECT * FROM order_items WHERE order_id = {order.id}")  # N queries
```

**How to detect it:**
- DB query logs show the same query pattern repeated hundreds of times with only the ID changing.
- APM tools (Datadog, New Relic) show "N+1 detected" or a spike in query count without a spike in data volume.
- `pg_stat_statements` shows a single query template with very high `calls` count.

**Fix 1: JOIN — fetch everything in one query**

```sql
SELECT o.order_id, o.created_at, oi.product_id, oi.quantity
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.user_id = 5;
```

**Fix 2: Subquery / IN — two queries, no per-row loop**

```sql
-- Query 1: get order IDs
SELECT order_id FROM orders WHERE user_id = 5;

-- Query 2: get all items for those orders
SELECT * FROM order_items WHERE order_id IN (101, 102, 103);
```

**Fix 3: Batch fetch in application (for ORMs)**

```python
order_ids = [o.id for o in orders]
items = db.query("SELECT * FROM order_items WHERE order_id = ANY(%s)", [order_ids])
# Then group items by order_id in Python — 2 queries total, not N+1
```

---

## 5 Common Slow Query Patterns with Fixes

### Pattern 1: Function on Indexed Column

```sql
-- BAD: the index on created_at cannot be used — function wraps the column
SELECT * FROM orders WHERE DATE(created_at) = '2024-03-15';

-- FIXED: rewrite to a range condition the index can use
SELECT * FROM orders
WHERE created_at >= '2024-03-15'
  AND created_at <  '2024-03-16';
```

---

### Pattern 2: LIKE with Leading Wildcard

```sql
-- BAD: leading % means B-tree cannot skip to the right position
SELECT * FROM users WHERE email LIKE '%@gmail.com';

-- FIXED option A: reverse the string and index, use trailing % instead
CREATE INDEX idx_users_email_reversed ON users (reverse(email));
SELECT * FROM users WHERE reverse(email) LIKE reverse('%@gmail.com');
-- = WHERE reverse(email) LIKE 'moc.liamg@%'

-- FIXED option B: full-text index for true substring search
CREATE INDEX idx_users_email_trgm ON users USING GIN (email gin_trgm_ops);
SELECT * FROM users WHERE email LIKE '%@gmail.com';  -- now uses GIN trigram index
```

---

### Pattern 3: SELECT * with Unused Columns

```sql
-- BAD: fetches all columns including large TEXT/JSONB fields
SELECT * FROM products WHERE category_id = 10;

-- FIXED: select only what you need — enables covering index, reduces network I/O
SELECT product_id, name, price FROM products WHERE category_id = 10;
```

---

### Pattern 4: Missing Index on FK (Slow Joins)

```sql
-- BAD: orders.user_id has no index — every JOIN forces a seq scan of orders
SELECT u.email, COUNT(*) FROM users u JOIN orders o ON u.user_id = o.user_id GROUP BY u.email;

-- FIXED: add an index on the FK side
CREATE INDEX idx_orders_user_id ON orders (user_id);
```

PostgreSQL does NOT automatically create indexes on foreign key columns. You must add them manually. This is one of the most common performance oversights in production schemas.

---

### Pattern 5: OR Conditions Breaking Index Use

```sql
-- BAD: OR across different columns; planner may seq scan
SELECT * FROM issues WHERE status = 'open' OR priority = 'critical';

-- FIXED option A: UNION ALL (each branch can use its own index)
SELECT * FROM issues WHERE status = 'open'
UNION ALL
SELECT * FROM issues WHERE priority = 'critical' AND status <> 'open';

-- FIXED option B: separate indexes + bitmap scan (PostgreSQL does this automatically for indexed columns)
CREATE INDEX ON issues (status);
CREATE INDEX ON issues (priority);
-- PostgreSQL will use a Bitmap OR scan combining both indexes
```

---

## Pagination: OFFSET vs. Cursor-Based

### OFFSET — Simple but Breaks at Scale

```sql
-- Page 1
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 0;

-- Page 500 (10,000th row)
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 9980;
```

**The problem:** `OFFSET 9980` tells PostgreSQL to scan and discard 9,980 rows before returning 20. As the page number grows, the query gets linearly slower — even with an index on `created_at`. At OFFSET 1,000,000 on a large table, this query takes seconds.

Additional problem: if a new post is inserted while a user is paginating, rows shift — the user sees duplicates or skips rows.

### Cursor-Based Pagination — Correct and Fast

```sql
-- First page (no cursor)
SELECT post_id, title, created_at
FROM posts
ORDER BY created_at DESC, post_id DESC  -- tie-break with post_id for stability
LIMIT 20;

-- Next page: use the last row's values as the cursor
-- (last row had created_at = '2024-03-10 12:00:00', post_id = 4451)
SELECT post_id, title, created_at
FROM posts
WHERE (created_at, post_id) < ('2024-03-10 12:00:00', 4451)  -- row comparison
ORDER BY created_at DESC, post_id DESC
LIMIT 20;
```

**Why this is fast:** the `WHERE` clause with an index on `(created_at DESC, post_id DESC)` jumps directly to the right position — no rows are discarded. Performance is O(log n) regardless of page depth.

| | OFFSET | Cursor-based |
|---|---|---|
| Implementation complexity | Low | Medium |
| Performance at page 1 | Same | Same |
| Performance at page 1000 | Degrades linearly | Constant |
| Stable under inserts | No (rows shift) | Yes |
| Random access (jump to page N) | Yes | No |
| REST API friendliness | Easy (`?page=5`) | Requires opaque cursor token |

**When to use OFFSET:** admin panels, internal tools, small datasets, or when random-page access is required. Everywhere else, use cursor-based.

---

## Statistics and the Query Planner

PostgreSQL's planner decides between index scan and seq scan by estimating row counts. Those estimates come from statistics gathered by `ANALYZE` (runs automatically via autovacuum, but you can run it manually after bulk loads).

```sql
-- Force statistics update after a large data load
ANALYZE orders;

-- Inspect what the planner knows about a column
SELECT n_distinct, correlation
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
```

- `n_distinct`: estimated number of unique values. Negative means fraction of total rows (e.g., `-0.05` = 5% distinct).
- `correlation`: how well the physical order of rows matches the column's sort order. Correlation near 1 or -1 means an index scan is cheap (rows are clustered). Near 0 means random I/O — the planner may prefer a seq scan even with an index.

**When estimates are badly wrong:** the planner chooses a bad plan. Symptoms: `EXPLAIN ANALYZE` shows `rows=1` estimated but `rows=50000` actual. Fix:

```sql
-- Increase statistics target for that column (default is 100 samples)
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
ANALYZE orders;
```

---

## Index Maintenance

Indexes degrade over time. Dead tuples from UPDATEs and DELETEs leave bloat inside index pages, and `HOT` updates (heap-only tuples) cannot always avoid index entries.

```sql
-- Check index bloat (rough estimate)
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- Rebuild an index without locking the table
REINDEX INDEX CONCURRENTLY idx_orders_user_id;

-- Remove an index that is never used
SELECT indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND schemaname = 'public'
ORDER BY pg_relation_size(indexrelid) DESC;
-- Any index with idx_scan = 0 since the last stats reset is a candidate for removal.
```

`REINDEX CONCURRENTLY` requires PostgreSQL 12+. It builds the new index alongside the old one, then swaps them — no table lock. Use it for production rebuilds.

**Rule:** audit unused indexes before every major release. An index that is never scanned still pays the write overhead on every INSERT/UPDATE/DELETE.
