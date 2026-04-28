# Interview Prep — Algorithms, System Design & Career

> This repo was built for Iranian developers who are navigating a tough job market and need a focused, no-fluff resource to prepare for technical interviews at top companies. Everything here is practical: pattern-first explanations, real LeetCode problems, full system design walkthroughs, and honest career advice for the international job search.

---

## What's inside

### LeetCode Patterns (`/leetCode`)

Each file covers one algorithm pattern: the intuition behind it, a reusable template, and 3–5 representative problems solved step-by-step.

| # | Pattern | Key idea |
|---|---------|----------|
| 01 | [Two Pointers](leetCode/01_two_pointers.md) | Reduce O(n²) pair search to O(n) on sorted arrays |
| 02 | [Sliding Window](leetCode/02_sliding_window.md) | Track a moving subarray/substring without recomputing |
| 03 | [Binary Search](leetCode/03_binary_search.md) | Halve the search space on sorted or monotonic input |
| 04 | [Hash Map / Set](leetCode/04_hash_map_set.md) | Trade space for O(1) lookups; frequency counting |
| 05 | [Stack & Queue](leetCode/05_stack_queue.md) | LIFO/FIFO order; monotonic stack tricks |
| 06 | [Recursion & Backtracking](leetCode/06_recursion_backtracking.md) | Explore all paths; prune early |
| 07 | [Trees — BFS & DFS](leetCode/07_trees_bfs_dfs.md) | Level-order vs depth-first traversal |
| 08 | [Graphs](leetCode/08_graphs.md) | BFS/DFS on adjacency lists; cycle detection; topological sort |
| 09 | [Dynamic Programming](leetCode/09_dynamic_programming.md) | Memoization and tabulation; state machine thinking |
| 10 | [Greedy](leetCode/10_greedy.md) | Local optimal choices that guarantee global optimum |
| 11 | [Heap / Priority Queue](leetCode/11_heap_priority_queue.md) | Top-K problems; streaming medians |
| 12 | [Linked List](leetCode/12_linked_list.md) | Pointer manipulation; fast & slow pointers |
| 13 | [Intervals](leetCode/13_intervals.md) | Sort-then-merge; sweep line |
| 14 | [Bit Manipulation](leetCode/14_bit_manipulation.md) | XOR tricks; masking; power-of-two checks |
| 15 | [Trie](leetCode/15_trie.md) | Prefix matching; autocomplete |
| 16 | [Math & Number Theory](leetCode/16_math_number_theory.md) | GCD, primes, modular arithmetic |

---

### Low-Level Design (`/LLD`)

Each file is a complete LLD interview walkthrough: requirements, class diagram, design patterns used, full Python implementation, and extension points.

| # | System | Highlights |
|---|--------|-----------|
| 01 | [Parking Lot](LLD/01_parking_lot.md) | Strategy pattern for pricing, thread-safe spot assignment |
| 02 | [LRU Cache](LLD/02_lru_cache.md) | Doubly-linked list + hash map; O(1) get & put |
| 03 | [Vending Machine](LLD/03_vending_machine.md) | State machine pattern; inventory & payment flow |
| 04 | [Notification Service](LLD/04_notification_service.md) | Observer pattern; multi-channel delivery, rate limiting, retry |
| 05 | [Hotel Booking](LLD/05_hotel_booking.md) | Repository pattern; date-range availability; concurrency |
| 06 | [Elevator System](LLD/06_elevator_system.md) | Scheduler strategies (SCAN/LOOK); request queuing |

---

### High-Level Design (`/HLD`)

Distributed system design at scale — the architecture, trade-offs, data flows, and numbers. Each file covers a system asked at senior+ interviews.

| # | System | Scale & Focus |
|---|--------|--------------|
| 01 | [URL Shortener](HLD/01_url_shortener.md) | 100M creates/day, 10B redirects/day; base62, KGS, CDN caching |
| 02 | [Rate Limiter](HLD/02_rate_limiter.md) | Token bucket vs sliding window; Redis Lua scripts; distributed edge placement |
| 03 | [Notification Service at Scale](HLD/03_notification_service_at_scale.md) | 1B+ notifications/day; Kafka, priority queues, DLQ, provider failover |
| 04 | [News Feed](HLD/04_news_feed.md) | 500M DAU; fan-out on write vs read vs hybrid; Redis sorted sets |

---

### SQL & Database Design (`/SQL`)

From schema fundamentals to advanced query patterns — covering the full range of database questions asked in backend interviews.

| # | Topic | What's covered |
|---|-------|---------------|
| 01 | [Schema Design](SQL/01_schema_design.md) | 1NF–BCNF normalization; 4 classic schema problems (e-commerce, social, hotel, Jira) |
| 02 | [Indexing & Performance](SQL/02_indexing_and_performance.md) | B-tree internals, composite indexes, EXPLAIN ANALYZE, N+1, cursor pagination |
| 03 | [Advanced Queries](SQL/03_advanced_queries.md) | Window functions, recursive CTEs, 8 classic interview problems, isolation levels |

---

### Concurrency Patterns (`/concurrency`)

Threading, multiprocessing, async, and channels — the patterns that show up in system design and coding interviews for backend roles.

| File | Language | What's covered |
|------|----------|---------------|
| [Python Concurrency](concurrency/01_python_concurrency.md) | Python | GIL, threading, multiprocessing, asyncio, producer-consumer, dining philosophers |
| [Go Concurrency](concurrency/02_go_concurrency.md) | Go | Goroutines, channels, select, sync, pipeline, fan-out/fan-in, worker pool |

---

### Behavioral Interview Guide (`/behavioral`)

| File | What's covered |
|------|---------------|
| [Behavioral Guide](behavioral/guide.md) | STAR method, 12 common themes with real example answers, special section for Iranian/MENA developers on framing regional experience for international audiences |

---

### Mock Interview Question Bank (`/mock`)

| File | What's covered |
|------|---------------|
| [Question Bank](mock/question_bank.md) | 8 coding sessions, 6 LLD sessions, 4 HLD sessions, 4 behavioral rounds — all with collapsible hints and interviewer rubrics. Includes a solo mock interview ritual. |

---

### Career & Job Search (`/career`)

| File | What's covered |
|------|---------------|
| [Resume & LinkedIn](career/resume_and_linkedin.md) | Resume bullet rewrites, framing regional experience, LinkedIn optimization, cold outreach templates, salary negotiation for remote/international roles |

---

## How to use this repo

1. **Start with LeetCode patterns** — work through `/leetCode` in order. Each pattern builds on the previous. Understand the template before touching a real problem.
2. **Drill 2–3 problems per pattern** on your own before looking at the solution.
3. **Move to LLD** — attempt the design yourself first (draw classes, pick patterns), then compare.
4. **Study HLD** — read once for the concepts, then re-read and ask: "why this choice and not that one?"
5. **Do SQL alongside HLD** — schema design questions come up in almost every backend system design round.
6. **Run mock sessions** from `/mock` — set a timer, talk out loud, write before you type.
7. **Read the behavioral guide** before your first real interview, not after. Most engineers find out they needed it too late.

---

## Contributing

Pull requests are welcome. If you have a clean explanation for a pattern, an LLD problem, or real interview experience to share, open a PR. Keep the style consistent: intuition first, template second, examples third. No fluff.

---

*Built with care for every developer who is restarting — hang in there.*
