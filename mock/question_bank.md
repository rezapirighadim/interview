# Mock Interview Question Bank

> Run these like real interviews. Set a timer. Talk out loud. Write on paper first. Treat silence as failure.

---

## How to Use This Bank

Pick a session. Set the timer before you read the prompt. Do not look at the collapsible sections until time is up or you're genuinely stuck (and even then — the "stuck" hint is a last resort, not a crutch).

The goal isn't to get the right answer. The goal is to practice the process of getting to the right answer under pressure, out loud, on the first try.

---

## Coding Mock Sessions

### Coding Session 1 — Easy | 20 Minutes

**Problem: Two Sum**

Given an array of integers `nums` and an integer `target`, return the indices of the two numbers such that they add up to `target`.

You may assume that each input has exactly one solution, and you may not use the same element twice. You can return the answer in any order.

**Examples:**
```
Input: nums = [2, 7, 11, 15], target = 9
Output: [0, 1]
Explanation: nums[0] + nums[1] = 2 + 7 = 9

Input: nums = [3, 2, 4], target = 6
Output: [1, 2]

Input: nums = [3, 3], target = 6
Output: [0, 1]
```

**Constraints:**
- 2 <= nums.length <= 10^4
- -10^9 <= nums[i] <= 10^9
- -10^9 <= target <= 10^9
- Only one valid answer exists.

**Follow-up:** Can you solve it in O(n) time complexity?

<details>
<summary>What the interviewer is looking for</summary>

- Brute force mentioned, then immediately improved (O(n²) → O(n))
- Hash map used correctly — store complement, not the number itself
- Edge case awareness: duplicates (the [3,3] case), negative numbers
- Clean variable naming — does `seen`, `complement`, `target - num` read clearly?
- No off-by-one errors when storing/checking index
- If they jump straight to O(n) without mentioning brute force, they should at least acknowledge they're skipping it and why

</details>

<details>
<summary>Stuck? One hint.</summary>

As you scan each number, ask yourself: "What number would make this work?" Store that instead of the number you have.

</details>

---

### Coding Session 2 — Easy | 20 Minutes

**Problem: Valid Parentheses**

Given a string `s` containing just the characters `'('`, `')'`, `'{'`, `'}'`, `'['` and `']'`, determine if the input string is valid.

An input string is valid if:
- Open brackets must be closed by the same type of brackets.
- Open brackets must be closed in the correct order.
- Every close bracket has a corresponding open bracket of the same type.

**Examples:**
```
Input: s = "()"
Output: true

Input: s = "()[]{}"
Output: true

Input: s = "(]"
Output: false

Input: s = "([)]"
Output: false

Input: s = "{[]}"
Output: true
```

**Constraints:**
- 1 <= s.length <= 10^4
- `s` consists of parentheses only: `'()[]{}'`

<details>
<summary>What the interviewer is looking for</summary>

- Stack identified quickly as the right structure — no over-engineering
- Correct handling of close bracket with empty stack (should return false, not crash)
- Clean mapping between open/close brackets — dictionary/map preferred over if/else chains
- Final check: stack must be empty at the end, not just "no error thrown"
- Good candidates ask about edge cases before coding: empty string, single character, odd length (fast false)
- A map like `{')': '(', '}': '{', ']': '['}` is cleaner than two separate checks

</details>

<details>
<summary>Stuck? One hint.</summary>

When you see a closing bracket, you need to know what the most recent unclosed bracket was. Which data structure gives you the most recent thing first?

</details>

---

### Coding Session 3 — Medium | 35 Minutes

**Problem: Longest Substring Without Repeating Characters**

Given a string `s`, find the length of the longest substring without repeating characters.

**Examples:**
```
Input: s = "abcabcbb"
Output: 3
Explanation: "abc" is the longest substring without repeating characters.

Input: s = "bbbbb"
Output: 1
Explanation: "b" is the longest substring.

Input: s = "pwwkew"
Output: 3
Explanation: "wke" is the answer (not "pwke" — "pwke" has 'p','w','k','e', which is 4, but 'e' is not repeated — actually "pwke" is valid too, the answer is 3 from "wke" or "kew")

Input: s = ""
Output: 0
```

**Constraints:**
- 0 <= s.length <= 5 * 10^4
- `s` consists of English letters, digits, symbols and spaces.

<details>
<summary>What the interviewer is looking for</summary>

- Sliding window pattern identified — two pointers, one window
- Set or HashMap used to track characters in current window
- Critical distinction: Set tells you if a character is in window, HashMap tells you WHERE it is (skip-ahead optimization)
- The skip-ahead: when you find a repeat, instead of moving left pointer one step at a time, jump to (previous_index + 1). This is the medium-difficulty insight.
- Correct max update: update max EVERY iteration, not just when you shrink the window
- Edge cases: empty string, all same character, all unique characters
- Space complexity awareness: O(min(n, charset_size)) not O(n)

</details>

<details>
<summary>Stuck? One hint.</summary>

You're maintaining a "window" of valid characters. When you add a character that already exists, where does your window's left edge need to move to? You don't have to move it one step at a time.

</details>

---

### Coding Session 4 — Medium | 35 Minutes

**Problem: Product of Array Except Self**

Given an integer array `nums`, return an array `answer` such that `answer[i]` is equal to the product of all the elements of `nums` except `nums[i]`.

The product of any prefix or suffix of `nums` is guaranteed to fit in a 32-bit integer.

You must write an algorithm that runs in O(n) time and **without using the division operation**.

**Examples:**
```
Input: nums = [1, 2, 3, 4]
Output: [24, 12, 8, 6]

Input: nums = [-1, 1, 0, -3, 3]
Output: [0, 0, 9, 0, 0]
```

**Constraints:**
- 2 <= nums.length <= 10^5
- -30 <= nums[i] <= 30
- The product of any prefix or suffix of `nums` is guaranteed to fit in a 32-bit integer.

**Follow-up:** Can you solve it in O(1) extra space complexity? (The output array doesn't count as extra space.)

<details>
<summary>What the interviewer is looking for</summary>

- Division approach mentioned and immediately ruled out (constraint recognition)
- Prefix/suffix product insight: answer[i] = (product of everything to the left) * (product of everything to the right)
- Two-pass solution: first pass builds prefix products, second pass multiplies in suffix products using a running variable
- The O(1) space optimization: doing this with one output array and one running suffix variable
- Good candidates name their variables clearly: `prefix`, `suffix`, `left_products`, `right_products`
- Common mistake: off-by-one in the prefix/suffix arrays (prefix[0] = 1, not nums[0])

</details>

<details>
<summary>Stuck? One hint.</summary>

What if you made two passes through the array? One left-to-right, one right-to-left. What could you compute on each pass that, when combined, gives you the answer?

</details>

---

### Coding Session 5 — Medium | 35 Minutes

**Problem: 3Sum**

Given an integer array `nums`, return all the triplets `[nums[i], nums[j], nums[k]]` such that `i != j`, `i != k`, `j != k`, and `nums[i] + nums[j] + nums[k] == 0`.

The solution set must not contain duplicate triplets.

**Examples:**
```
Input: nums = [-1, 0, 1, 2, -1, -4]
Output: [[-1, -1, 2], [-1, 0, 1]]

Input: nums = [0, 1, 1]
Output: []

Input: nums = [0, 0, 0]
Output: [[0, 0, 0]]
```

**Constraints:**
- 3 <= nums.length <= 3000
- -10^5 <= nums[i] <= 10^5

<details>
<summary>What the interviewer is looking for</summary>

- Sorting first — this is the key insight that enables two-pointer
- Fix one element, two-pointer the rest: O(n²) total
- Correct duplicate skipping: after fixing nums[i], skip if nums[i] == nums[i-1]; after finding a valid triplet, skip duplicates for both left and right pointers
- Hash set approach works but is O(n²) space — interviewer may ask if you can do better
- Common mistakes: not skipping duplicates (produces duplicate triplets), off-by-one in pointer bounds, forgetting to sort first
- [0,0,0,0] case: should produce [[0,0,0]] once, not multiple times

</details>

<details>
<summary>Stuck? One hint.</summary>

Sort the array first. Then for each element, treat it as a fixed "third wheel" and use two pointers on the rest of the array to find pairs that sum to its negative.

</details>

---

### Coding Session 6 — Medium | 35 Minutes

**Problem: Binary Tree Level Order Traversal**

Given the root of a binary tree, return the level order traversal of its nodes' values (i.e., from left to right, level by level).

**Examples:**
```
Input: root = [3, 9, 20, null, null, 15, 7]
Output: [[3], [9, 20], [15, 7]]

Input: root = [1]
Output: [[1]]

Input: root = []
Output: []
```

**Constraints:**
- The number of nodes in the tree is in the range [0, 2000].
- -1000 <= Node.val <= 1000

<details>
<summary>What the interviewer is looking for</summary>

- BFS (queue) identified as the correct approach — DFS with level tracking works but is less natural
- Key technique: snapshot the queue size at the start of each level iteration to know when one level ends and the next begins
- Handling empty tree (null root) without crashing — check before starting
- Output structure: list of lists, not a flat list
- Good candidates articulate the "why": BFS processes nodes in the order we want, level by level
- Bonus: can they flip this to right-to-left? Can they do zigzag level order? Shows they understand the pattern

</details>

<details>
<summary>Stuck? One hint.</summary>

Use a queue. At the start of processing each level, check how many nodes are currently in the queue — that tells you exactly how many nodes belong to the current level.

</details>

---

### Coding Session 7 — Hard | 45 Minutes

**Problem: Median of Two Sorted Arrays**

Given two sorted arrays `nums1` and `nums2` of size `m` and `n` respectively, return the median of the two sorted arrays.

The overall run time complexity must be O(log(m+n)).

**Examples:**
```
Input: nums1 = [1, 3], nums2 = [2]
Output: 2.00000
Explanation: merged array = [1, 2, 3], median = 2

Input: nums1 = [1, 2], nums2 = [3, 4]
Output: 2.50000
Explanation: merged array = [1, 2, 3, 4], median = (2 + 3) / 2 = 2.5
```

**Constraints:**
- nums1.length == m
- nums2.length == n
- 0 <= m <= 1000
- 0 <= n <= 1000
- 1 <= m + n <= 2000
- -10^6 <= nums1[i], nums2[i] <= 10^6

<details>
<summary>What the interviewer is looking for</summary>

- Merge approach (O(m+n)) mentioned and acknowledged as not meeting the constraint
- Binary search on the smaller array — partition approach
- Understanding of what a "correct partition" means: all elements on the left of partition are ≤ all elements on the right
- Correct handling of edge cases: one array empty, partition at boundary (using -infinity/+infinity as sentinels)
- Even vs odd total length handled correctly
- Many strong candidates can describe the algorithm verbally but struggle to implement the boundary conditions — that's okay, walk through it
- A perfect implementation in 45 minutes is exceptional. Getting the approach right and working through it methodically is sufficient.

</details>

<details>
<summary>Stuck? One hint.</summary>

You're looking for the right place to "cut" both arrays so the left halves together equal the right halves together in count. Binary search on the cut position in the smaller array, and derive the cut position in the larger array from it.

</details>

---

### Coding Session 8 — Hard | 45 Minutes

**Problem: Word Ladder**

A transformation sequence from word `beginWord` to word `endWord` using a dictionary `wordList` is a sequence `beginWord -> s1 -> s2 -> ... -> sk` such that:
- Every adjacent pair of words differs by a single letter.
- Every `si` for `1 <= i <= k` is in `wordList`. Note that `beginWord` does not need to be in `wordList`.
- `sk == endWord`

Given two words, `beginWord` and `endWord`, and a dictionary `wordList`, return the number of words in the shortest transformation sequence from `beginWord` to `endWord`, or 0 if no such sequence exists.

**Examples:**
```
Input: beginWord = "hit", endWord = "cog", wordList = ["hot","dot","dog","lot","log","cog"]
Output: 5
Explanation: "hit" -> "hot" -> "dot" -> "dog" -> "cog" (5 words)

Input: beginWord = "hit", endWord = "cog", wordList = ["hot","dot","dog","lot","log"]
Output: 0
Explanation: "cog" is not in wordList, so no valid transformation sequence exists.
```

**Constraints:**
- 1 <= beginWord.length <= 10
- endWord.length == beginWord.length
- 1 <= wordList.length <= 5000
- wordList[i].length == beginWord.length
- beginWord, endWord, and wordList[i] consist of lowercase English letters.
- beginWord != endWord
- All the words in wordList are unique.

<details>
<summary>What the interviewer is looking for</summary>

- Problem modeled as a graph: words are nodes, edges between words that differ by one letter
- BFS for shortest path — DFS gives A path, not necessarily the shortest
- Word set (hash set) for O(1) lookup and marking visited — not the original list
- Character substitution approach: for each word, try replacing each character with a-z and check if it's in the word set. More efficient than comparing all pairs.
- Removing words from the set as they're visited (instead of a separate visited set) — elegant
- Edge case: endWord not in wordList → return 0 immediately
- Return value is number of WORDS (including begin and end), not number of transformations

</details>

<details>
<summary>Stuck? One hint.</summary>

Model this as a graph where each word is a node. Two words are connected if they differ by exactly one letter. You want the shortest path from beginWord to endWord. What algorithm finds shortest paths?

</details>

---

## LLD (Low-Level Design) Mock Sessions

*Format: 45 minutes. Spend 5-10 minutes clarifying requirements, 30-35 minutes designing.*

---

### LLD Session 1 — Design a Parking Lot | 45 Minutes

**Prompt:**

Design an object-oriented system for a multi-floor parking lot. The lot has multiple floors, each with multiple spots. Spots can accommodate different vehicle types (motorcycle, car, large/truck). The system should:
- Track available spots
- Assign the nearest available spot to an entering vehicle
- Release spots when a vehicle exits
- Calculate parking fee based on time parked

Design the classes, their attributes, methods, and relationships. You can use any language, but use real syntax, not pseudocode.

<details>
<summary>What the interviewer is looking for</summary>

**Clarifying questions they should ask:**
- Is "nearest" by floor first? By spot number within floor?
- Is there a maximum capacity per vehicle type?
- Fee calculation: flat rate? Hourly? Different rates by vehicle type?
- Concurrent access? (thread safety)

**Key design elements:**
- `ParkingLot`, `ParkingFloor`, `ParkingSpot`, `Vehicle`, `Ticket` classes
- `VehicleType` enum (MOTORCYCLE, CAR, LARGE)
- `SpotType` enum matching vehicle types
- `Ticket` holds entry time, spot reference, vehicle — needed for fee calculation
- Clean separation: `ParkingSpot` doesn't know about fees. A `FeeCalculator` does.
- Strategy pattern opportunity: different fee strategies (hourly, flat, premium hours)
- `ParkingLot` manages floor/spot assignment — shouldn't be the vehicle's job
- Thread safety: spot assignment should be atomic (spot.assign() should be synchronized or use CAS)

**Red flags:**
- God class: ParkingLot that does everything including fee calculation
- No distinction between spot type and vehicle type
- No ticket/record of entry — can't calculate duration without it

</details>

<details>
<summary>Stuck? One hint.</summary>

Start with the nouns in the problem: parking lot, floor, spot, vehicle, ticket. Each is likely a class. Then figure out what each knows and what each does — and whether any responsibility is in the wrong place.

</details>

---

### LLD Session 2 — Design a Rate Limiter | 45 Minutes

**Prompt:**

Design a rate limiter that can be used as middleware in a web service. The limiter should:
- Allow configuration of max requests per time window per user
- Support at least two strategies: fixed window and sliding window
- Be thread-safe for concurrent requests
- Return a response indicating whether the request is allowed and how many requests remain

Design the interfaces, classes, and data structures. Discuss the tradeoffs of each strategy.

<details>
<summary>What the interviewer is looking for</summary>

**Clarifying questions:**
- Single server or distributed? (changes the storage layer entirely)
- Per user, per IP, per API key?
- What to return on rate limit: 429 immediately, or queue?
- Accuracy vs performance tradeoff acceptable?

**Key design elements:**
- `RateLimiter` interface with `isAllowed(userId)` returning a `RateLimitResult`
- `FixedWindowRateLimiter`: counter per (userId, window_start). Simple but boundary burst problem.
- `SlidingWindowRateLimiter`: log of timestamps per user. More accurate. Higher memory.
- `TokenBucketRateLimiter`: bonus — smooth out bursts
- Thread safety: synchronized blocks or `AtomicInteger` for fixed window; `ConcurrentHashMap` with `synchronized` on per-user list for sliding window
- Cleanup: who deletes old timestamp logs? Background thread? Lazy cleanup on access?

**Tradeoffs discussion expected:**
- Fixed window: memory O(1) per user, but allows 2x burst at window boundary
- Sliding window: accurate, but O(requests_per_window) memory per user
- Token bucket: memory O(1), handles bursts gracefully, more complex implementation

**Red flags:**
- No thread safety discussion
- No interface — just one concrete implementation
- No discussion of what happens in distributed systems (local storage breaks everything)

</details>

<details>
<summary>Stuck? One hint.</summary>

Start by defining the interface first: what does `isAllowed` take in? What does it return? Then implement fixed window first (simplest), and once that works, talk about why sliding window is more accurate.

</details>

---

### LLD Session 3 — Design a Library Management System | 45 Minutes

**Prompt:**

Design a system for a library that manages books, members, and borrowing. The system should:
- Catalog books (a book can have multiple copies)
- Allow members to search for books by title, author, or ISBN
- Allow members to borrow and return books
- Track due dates and calculate fines for overdue returns
- Place reservations on books that are currently checked out

Design the classes and their interactions. Focus on the borrowing and reservation workflows.

<details>
<summary>What the interviewer is looking for</summary>

**Clarifying questions:**
- How many copies of a book can exist?
- Can a member borrow multiple books? Is there a limit?
- Is reservation first-come-first-served?
- Fine calculation: per day? Grace period?

**Key design elements:**
- `Book` (metadata: ISBN, title, author) vs `BookItem` (physical copy: barcode, status)
- `Member` with borrow history, active loans, reservations
- `BorrowRecord` (or `Loan`): links member, book item, issue date, due date, return date
- `Reservation` queue: ordered list of members waiting for a book
- `Catalog` for search — consider search indexing (in memory: inverted index by title tokens)
- State machine for `BookItem`: AVAILABLE → BORROWED → RETURNED; AVAILABLE → RESERVED → BORROWED
- Fine calculation lives in a `FineCalculator`, not in `Member` or `BookItem`
- When a book is returned and there's a reservation queue, who notifies the next member? Observer pattern or a `ReservationService`.

**Red flags:**
- Book and BookItem conflated — one class for both
- No reservation queue concept
- Member knows how to calculate their own fines (wrong responsibility)

</details>

<details>
<summary>Stuck? One hint.</summary>

There's an important distinction between a book (the concept: "Harry Potter") and a copy of a book (the physical item on the shelf). These probably need to be different classes. Start there and everything else follows.

</details>

---

### LLD Session 4 — Design an Elevator System | 45 Minutes

**Prompt:**

Design the control system for a building with multiple elevators. The system should:
- Handle requests from floors (up/down buttons) and from inside elevators (destination floor buttons)
- Dispatch the optimal elevator to a floor request
- Move elevators efficiently (minimize wait time and total travel)
- Handle the elevator door open/close states and emergency stop

Design the classes, states, and the dispatch algorithm. You don't need to implement a perfect scheduling algorithm, but explain your choice.

<details>
<summary>What the interviewer is looking for</summary>

**Clarifying questions:**
- How many elevators? How many floors?
- Priority users (e.g., emergency, VIP)?
- Can we assume all elevators start at ground floor?

**Key design elements:**
- `ElevatorCar` with state machine: IDLE, MOVING_UP, MOVING_DOWN, DOOR_OPEN
- `Floor` with up/down request buttons
- `ElevatorController` (dispatcher): receives floor requests, decides which elevator responds
- `Request` types: `ExternalRequest` (from floor buttons) vs `InternalRequest` (from inside car)
- Dispatch algorithms to discuss:
  - Nearest car: simple, low fairness
  - SCAN (elevator algorithm): service all requests in one direction before reversing — good throughput, poor for edge floors
  - LOOK: like SCAN but reverses when no more requests in current direction
- `ElevatorCar` maintains a sorted set of destination floors in its current direction

**State transitions must be explicit:**
- IDLE → receives request → MOVING
- MOVING → reaches floor with request → DOOR_OPEN
- DOOR_OPEN → timeout/close button → MOVING or IDLE

**Red flags:**
- No dispatcher — each elevator manages all requests itself
- No state machine for elevator (doors open while moving, etc.)
- Picking a dispatch algorithm without justifying the tradeoff

</details>

<details>
<summary>Stuck? One hint.</summary>

Think of the elevator as a state machine first. What states can it be in? What triggers each transition? Once you have the states, the dispatcher's job becomes clearer: given a floor request, which elevator's current state and position makes it cheapest to respond?

</details>

---

### LLD Session 5 — Design a Task Queue with Workers | 45 Minutes

**Prompt:**

Design a task queue system where:
- Clients submit tasks with a type and payload
- Worker threads pick up and process tasks
- Tasks can have priorities (higher priority tasks run first)
- Failed tasks are retried up to a configurable max retry count
- The system exposes a way to check the status of a submitted task

Design the classes and concurrency model. You can use thread primitives (locks, semaphores, etc.) or higher-level constructs.

<details>
<summary>What the interviewer is looking for</summary>

**Clarifying questions:**
- Tasks CPU-bound or I/O-bound? (affects thread pool sizing advice)
- Single process or distributed workers?
- Persistence: in-memory queue or backed by storage?
- What's a "failed" task — exception thrown? Timeout?

**Key design elements:**
- `Task` with: id, type, payload, priority, status, retry count, max retries
- `TaskStatus` enum: PENDING, RUNNING, COMPLETED, FAILED, RETRYING
- `PriorityBlockingQueue` (or equivalent) as the queue — thread-safe, ordered by priority
- `Worker` thread: poll queue, execute task, handle exceptions, resubmit on failure
- `TaskExecutor` interface: `execute(Task)` — different types of tasks have different executors
- `WorkerPool`: manages N worker threads
- `TaskRegistry` (ConcurrentHashMap<taskId, Task>): stores tasks for status lookup
- Retry logic: exponential backoff, max retry threshold, move to dead-letter queue on exhaustion
- Thread safety: `PriorityBlockingQueue` handles queue safety; task status updates need synchronization or atomic operations

**Red flags:**
- Single-threaded worker
- No retry logic
- No way to query task status after submission
- Race condition in status updates (multiple workers, no synchronization)

</details>

<details>
<summary>Stuck? One hint.</summary>

The queue needs to be thread-safe and support priority ordering. Java has `PriorityBlockingQueue`, Python has `queue.PriorityQueue`. Start with the data model for a Task (what does a task know about itself?), then figure out how workers interact with it.

</details>

---

### LLD Session 6 — Design a Notification System | 45 Minutes

**Prompt:**

Design a notification system that:
- Sends notifications through multiple channels: email, SMS, push notification
- Users can configure their notification preferences per channel per notification type
- Supports templates for notification content
- Handles delivery failures with retries
- Tracks delivery status per notification per channel

Design the core classes and the notification dispatch flow.

<details>
<summary>What the interviewer is looking for</summary>

**Clarifying questions:**
- Real-time or batched?
- Volume expectations? (changes whether you need a queue)
- Who creates notifications — internal events or external API calls?
- Template variables — static or dynamic?

**Key design elements:**
- `Notification` with: id, type (ORDER_SHIPPED, PAYMENT_FAILED, etc.), recipient, template variables, timestamp
- `NotificationChannel` interface with `send(Notification, Recipient)` → implemented by `EmailChannel`, `SMSChannel`, `PushChannel`
- `UserPreferences`: maps (userId, notificationType) → list of channels
- `NotificationTemplate`: template string per (notificationType, channel) — email template looks different from SMS
- `NotificationDispatcher`: looks up user prefs, renders template per channel, dispatches
- `DeliveryRecord`: tracks (notificationId, channel, status, attempts, lastAttemptTime)
- Retry: exponential backoff per channel, separate retry queue or scheduled re-check

**Design patterns:**
- Strategy: each channel is a strategy
- Observer: events trigger notifications
- Template Method: render() base with channel-specific overrides

**Red flags:**
- One monolithic `sendNotification()` method with if/else for each channel
- No user preference system — sending to all channels always
- No delivery tracking or retry

</details>

<details>
<summary>Stuck? One hint.</summary>

The sending mechanism for email, SMS, and push are all different — that's a hint to use an interface or abstract class. Define `NotificationChannel` first, then have each concrete channel implement it. The dispatcher just iterates over a user's preferred channels and calls the same method on each.

</details>

---

## HLD (High-Level Design) Mock Sessions

*Format: 45 minutes. Requirements 5 min, estimation 5-10 min, design 25-30 min, deep dive 5 min.*

---

### HLD Session 1 — Design a URL Shortener | 45 Minutes

**Prompt:**

Design a URL shortening service like bit.ly. Users can submit a long URL and receive a short URL (e.g., `short.ly/xK2p9`). When someone visits the short URL, they are redirected to the original. The service should:

- Generate short URLs (7 characters)
- Support 100 million URL creation requests per day
- Support 10 billion redirect requests per day (100:1 read:write ratio)
- URLs should not expire by default (but support optional expiry)
- Short codes must be unique

Estimate capacity, design the system, and identify the critical bottlenecks.

<details>
<summary>What the interviewer is looking for</summary>

**Estimation:**
- Writes: 100M/day = ~1,200/sec
- Reads: 10B/day = ~115,000/sec
- Storage: 100M URLs/day * 365 * 5 years = ~180B records. Each record ~500 bytes → ~90 TB

**Key design decisions:**
- Short code generation: hash (MD5, first 7 chars) vs counter-based vs random
  - Hash: collision prone at scale; counter: easy to enumerate; random: preferred with collision check
- 7 chars, 62 symbols (a-z, A-Z, 0-9): 62^7 = 3.5 trillion combinations — enough
- Read path: User → CDN (cache frequent redirects) → Load Balancer → App Server → Cache (Redis) → DB
- Write path: App Server → ID Generator → DB write
- ID Generation: single point of failure concern. Options: UUID, Snowflake ID, pre-generated ID ranges per server
- DB: write-once read-many. SQL works. Key-value store (Cassandra, DynamoDB) at this scale is preferred for the redirect path.
- Cache: cache the hot short codes (top 20% serve 80% of redirects). Redis. TTL = 24 hours. Cache miss → DB → cache.
- 301 vs 302 redirect: 301 is permanent (browser caches, reduces load), 302 is temporary (you keep the redirect traffic for analytics)

**Deep dive options:**
- Analytics: how do you count clicks without slowing down redirects? Async, stream to Kafka, count in batch.
- Custom aliases: add a separate table lookup before the random generation path.
- Expiry: TTL on cache matches URL expiry, background job cleans DB.

**Red flags:**
- No estimation
- No cache in the read path (completely unrealistic at 115K rps)
- No discussion of the code generation collision risk
- Single database without replication

</details>

<details>
<summary>Stuck? One hint.</summary>

The read:write ratio is 100:1. That asymmetry should drive your architecture — the read path needs to be extremely fast and heavily cached. Design the read path first, then worry about writes.

</details>

---

### HLD Session 2 — Design Twitter/X Feed | 45 Minutes

**Prompt:**

Design the core news feed feature for Twitter. Users can post tweets (280 characters), follow other users, and see a feed of tweets from people they follow, sorted by time (or relevance — you choose).

Key requirements:
- 300 million active users
- 500 million tweets per day
- Users follow on average 200 accounts, are followed by on average 200 accounts
- Feed must load in under 200ms for 99th percentile
- Some users have 100M followers (celebrities)

Identify the core challenge and design around it.

<details>
<summary>What the interviewer is looking for</summary>

**The core challenge:** The "fan-out" problem. When a celebrity with 100M followers posts a tweet, do you write to 100M user feeds immediately (fan-out on write) or compute feeds lazily on read (fan-out on read)?

**Fan-out on write (push model):**
- On tweet: write tweet to all followers' pre-computed feeds
- Read: just read the pre-computed feed (fast)
- Problem: a celebrity with 100M followers = 100M writes per tweet. Twitter has celebrity accounts.

**Fan-out on read (pull model):**
- On tweet: just store the tweet
- On feed read: pull tweets from all accounts the user follows, merge and sort
- Problem: following 1000 accounts = 1000 DB reads per feed load. Too slow.

**Hybrid approach (what Twitter actually does):**
- Regular users: fan-out on write — pre-compute and store feeds in cache (Redis list per user)
- Celebrities: fan-out on read — don't pre-compute; on read, merge celebrity tweets in at query time
- Threshold: >X followers → pull model for that account's tweets

**Data model:**
- `tweets` table: tweet_id, user_id, content, timestamp
- `follows` table: follower_id, followee_id
- `user_feed` cache (Redis): sorted set per user, ordered by tweet timestamp, contains tweet IDs

**Storage:**
- 500M tweets/day * 280 chars * 5 years ≈ 360 TB (just text; media handled separately)
- Media: object store (S3), CDN for delivery

**Red flags:**
- No acknowledgment of the celebrity/fan-out problem
- Proposing pure fan-out on write with no concern for 100M-follower accounts
- No cache in the read path
- Single DB with no sharding discussion

</details>

<details>
<summary>Stuck? One hint.</summary>

Ask yourself: what happens when someone with 100 million followers posts a tweet? If you write to every follower's feed in real time, how long does that take? That tension between "precompute everything" and "compute on demand" is the central design challenge here.

</details>

---

### HLD Session 3 — Design a Distributed Cache | 45 Minutes

**Prompt:**

Design a distributed in-memory caching system similar to Redis or Memcached. The system should:
- Support GET and SET operations with optional TTL
- Handle 1 million requests per second
- Store up to 10 TB of data across nodes
- Ensure high availability (no single point of failure)
- Handle node failures and additions gracefully

Design the architecture, data distribution strategy, and failure handling.

<details>
<summary>What the interviewer is looking for</summary>

**Data distribution:**
- Naive modulo hashing: `hash(key) % N`. Problem: adding/removing a node invalidates ~100% of keys.
- Consistent hashing: keys and nodes on a ring. Adding/removing a node only invalidates 1/N of keys.
- Virtual nodes (vnodes): each physical node owns multiple points on the ring. Balances load when nodes have different capacities or during failures.

**Replication:**
- Single primary per key: simple, but node failure loses that key until recovery
- Replication factor R: each key stored on R nodes. Primary handles write, secondaries serve reads or take over on failure.
- Write-through vs write-behind: depends on consistency requirements

**Failure handling:**
- Node failure detected via heartbeat (gossip protocol between nodes)
- When node fails: consistent hashing means its keys are served by successor node (if replicated there)
- Re-replication: when a node recovers or a new node joins, redistribute keys

**Client vs server-side routing:**
- Clients know the ring topology (Memcached style): client computes node, connects directly. Fast, but clients need to be updated on topology changes.
- Proxy/router (Redis Cluster): client talks to any node, which redirects if needed.

**TTL implementation:**
- Lazy expiration: check TTL on GET, delete if expired
- Active expiration: background thread periodically scans and deletes expired keys
- Combination of both at scale

**Eviction policies:** LRU, LFU, TTL-only — explain the tradeoff

**Red flags:**
- Modulo hashing with no concern for resharding
- No replication discussion
- No failure detection mechanism
- No eviction policy discussion (memory fills up)

</details>

<details>
<summary>Stuck? One hint.</summary>

The hardest problem here isn't storage or reads — it's what happens when you add or remove a node. With naive hashing, adding one server reshuffles almost everything. Consistent hashing was invented to solve exactly this. Start there.

</details>

---

### HLD Session 4 — Design a Ride-Sharing Service | 45 Minutes

**Prompt:**

Design the core matching system for a ride-sharing service like Snapp or Uber. Focus on:
- Riders can request a ride from location A to location B
- Drivers broadcast their real-time location
- The system matches a rider to the nearest available driver
- The system shows riders an ETA before they confirm
- Handle 5 million active drivers sending location updates every 5 seconds

Design the location tracking system, the matching algorithm, and the request/accept flow.

<details>
<summary>What the interviewer is looking for</summary>

**Location updates:**
- 5M drivers * 1 update/5 seconds = 1M writes/second for location
- Can't store every update in a relational DB — too slow and too much data
- In-memory store for current driver locations: Redis with geospatial commands (`GEOADD`, `GEORADIUS`)
- Historical location: stream updates to Kafka, consume for analytics/tracking

**Geospatial indexing:**
- GeoHash: divides earth into grid cells (variable precision). Drivers in same cell are nearby. Can expand search to adjacent cells if no drivers found.
- Quadtree: recursive space subdivision. Better for non-uniform density (city centers have many drivers, rural areas few).
- S2 Geometry: used by Uber. Hierarchical cell system.
- For interview: geohash is sufficient and easy to reason about.

**Matching flow:**
1. Rider requests ride → Rider Service
2. Rider Service gets rider location → queries Driver Location Service for nearby available drivers
3. Sort by ETA (not just distance — traffic matters) → call Maps API or internal routing service
4. Offer ride to best matched driver (or top 3 in parallel if one declines)
5. Driver accepts → trip created → both parties notified

**WebSocket for real-time:**
- Driver app maintains WebSocket to server for receiving ride requests
- Rider app maintains WebSocket for driver location updates during trip

**ETA calculation:**
- Simple: distance / average speed. Fast but inaccurate.
- Better: call a routing engine (internal OSRM, or external Maps API). Cache common routes.

**Surge pricing trigger:**
- Demand/supply ratio per geohash cell. If drivers < threshold in a cell, apply multiplier.

**Red flags:**
- No geospatial indexing — just querying all drivers and calculating distance
- No WebSocket or push mechanism — polling is too slow for real-time
- Offering ride to one driver and blocking until they respond (creates long waits)
- No discussion of driver location update scale

</details>

<details>
<summary>Stuck? One hint.</summary>

The core problem is: given a rider's location, find nearby available drivers fast. You can't query every driver in the database and calculate distances one by one. Think about how to pre-organize the geographic space so nearby drivers can be found in O(log n) or better.

</details>

---

## Behavioral Mock Sessions

*Format: 30 minutes, 3 questions. Give 5-8 minutes per answer. Use STAR. Don't rush to action — set up the situation.*

---

### Behavioral Session 1 — Ownership & Failure

**Time: 30 minutes (3 questions, ~10 minutes each)**

**Question 1:**
Tell me about a time you took on a problem that was clearly outside your job description. What made you step up, and what happened?

**Question 2:**
Describe the most significant professional mistake you've made. What was it, what was the impact, and what specifically did you change afterward?

**Question 3:**
Tell me about a project you owned where something went badly wrong. Walk me through exactly what you did from the moment you realized things were off track.

<details>
<summary>What the interviewer is looking for</summary>

**Q1 — Ownership:**
- Proactive identification of a problem (not assigned to them)
- Clear reasoning for why they stepped in
- Specific actions they took — not "I helped the team"
- Outcome with measurable impact
- Red flag: "I asked my manager what I should do" as the first action

**Q2 — Failure/self-awareness:**
- Genuine mistake with real consequences — not a humblebrag ("I work too hard")
- Personal accountability — not "the team made a mistake"
- Specific behavioral change, not just "I learned to be more careful"
- Shows they can talk about failure without becoming defensive
- Red flag: can't name a real mistake; blames external factors

**Q3 — Crisis ownership:**
- Early recognition of the problem (good engineers don't deny bad signals)
- Systematic approach to triage, not panic
- Communication to stakeholders throughout
- Post-mortem or process change at the end
- Red flag: waited too long to escalate; no lesson extracted

</details>

<details>
<summary>Stuck on structuring your answer?</summary>

For each question: 2-3 sentences of context (Situation + Task), then spend most of your time on exactly what YOU did (Action), then close with what changed (Result + lesson). Resist the urge to explain context for more than 60 seconds.

</details>

---

### Behavioral Session 2 — Conflict & Communication

**Time: 30 minutes (3 questions, ~10 minutes each)**

**Question 1:**
Tell me about a time you had a serious disagreement with a colleague about a technical approach. How did you handle it, and what was the outcome?

**Question 2:**
Describe a time you had to deliver bad news to a stakeholder or client — a missed deadline, a bug in production, a feature that couldn't be built. How did you handle the conversation?

**Question 3:**
Tell me about a time you had to push back on a request from a manager or senior leader. What happened?

<details>
<summary>What the interviewer is looking for</summary>

**Q1 — Technical conflict:**
- Direct communication — didn't avoid the conflict or passive-aggressively comply
- Used data and reasoning, not emotion or seniority
- Listened to the other perspective (shows they're not just stubborn)
- Comfortable when the decision went against them — could still execute
- Red flag: "I just let them do it their way" with no attempt to raise the concern

**Q2 — Delivering bad news:**
- Came with information and options, not just the problem
- Communicated early — didn't wait until it was impossible to course correct
- Tone: matter-of-fact, not panicked or apologetic to the point of incoherence
- Proposed a path forward
- Red flag: blamed others in the story; softened the bad news to the point of misleading

**Q3 — Pushing back upward:**
- Used evidence and framing, not just "I don't think that's right"
- Respectful but clear — didn't just capitulate
- Knew when to stop — once the decision was made, committed
- Red flag: never pushes back ("I always trust management"); OR always pushes back and resents every outcome

</details>

<details>
<summary>Stuck on structuring your answer?</summary>

For conflict stories: be specific about what the actual disagreement was. "We disagreed on the approach" is not specific enough. "He wanted to use a polling model, I wanted WebSockets, because our use case required sub-100ms updates" — that's specific. Specificity = credibility.

</details>

---

### Behavioral Session 3 — Growth & Adaptability

**Time: 30 minutes (3 questions, ~10 minutes each)**

**Question 1:**
Tell me about a time you had to learn something completely new under real time pressure. What was your strategy and how long did it take before you were useful?

**Question 2:**
Describe a situation where the requirements or direction of your project changed significantly mid-way. How did you respond?

**Question 3:**
Tell me about a time you received feedback that was hard to hear. What did you do with it?

<details>
<summary>What the interviewer is looking for</summary>

**Q1 — Learning agility:**
- Has a systematic approach (not just "I Googled it")
- Time-boxed and practical — didn't try to master everything before starting
- Measured their own progress ("useful in X days/weeks")
- Built something as part of learning, not just read docs
- Red flag: "I'm a quick learner" with no process described

**Q2 — Adaptability:**
- Acknowledged the change without blaming stakeholders for "changing their minds"
- Re-scoped cleanly — shows structured thinking
- Communicated the impact of the change on timeline/scope — didn't just absorb it silently
- Stayed productive during the ambiguity
- Red flag: resents the change; "we just had to start over"

**Q3 — Receiving feedback:**
- Names the specific feedback, not just "I got some feedback"
- Didn't immediately defend themselves in the story
- Changed a specific behavior or practice
- Can point to an observable outcome from the change
- Red flag: story about a misunderstanding where they were actually right; no behavior change

</details>

<details>
<summary>Stuck on structuring your answer?</summary>

For feedback stories: the hardest and most impressive version is behavioral feedback (how you communicate, how you work with others), not just technical feedback (fix this bug, write better tests). If you have a story about changing how you interact with people — not just your code — use that one.

</details>

---

### Behavioral Session 4 — Impact & Cross-Functional Work

**Time: 30 minutes (3 questions, ~10 minutes each)**

**Question 1:**
Tell me about the most impactful technical project you've worked on. How did you measure the impact, and what would have happened if you hadn't done it?

**Question 2:**
Describe a time you had to work closely with non-technical stakeholders to deliver a product or feature. What made it work, and what would you do differently?

**Question 3:**
Tell me about a time you helped someone else on your team grow — a junior engineer, a peer, or even someone more senior than you.

<details>
<summary>What the interviewer is looking for</summary>

**Q1 — Impact:**
- Quantified: time saved, revenue, users affected, error rates, latency — something measurable
- Shows counterfactual thinking: what was the cost of NOT doing this?
- Demonstrates they care about business outcomes, not just technical elegance
- Red flag: impact is vague ("things went smoother"); can't connect technical work to business value

**Q2 — Cross-functional:**
- Adapted communication style for non-technical audience
- Proactively reached out — didn't wait for stakeholders to come to them
- Managed expectations with honesty, not just optimism
- Something specific they'd do differently (shows reflection)
- Red flag: "they just didn't understand how software works" — condescending framing

**Q3 — Mentorship:**
- Specific person, specific growth area
- Had a method — not just "I answered their questions"
- Can describe what changed in the other person's work
- Learned something themselves from the process
- Red flag: the story is about how great they are, not what the other person achieved

</details>

<details>
<summary>Stuck on structuring your answer?</summary>

For impact stories: before you finish your answer, ask yourself: "Did I say a number?" Revenue, users, percentage improvement, time saved, errors reduced — something quantifiable. If you didn't, add it. "The page load time went from 4.2 seconds to 0.8 seconds" is infinitely more memorable than "it got much faster."

</details>

---

## How to Run a Mock Interview Alone

Most people don't do this. That's why most people still feel underprepared the day before the interview.

### The Ritual

**Before you start:**
1. Close everything except a timer and a blank document or paper
2. Write the problem down — do not just read it from the screen. Writing forces slower processing.
3. Set the timer. Do NOT start the timer and then re-read the problem for 5 minutes. Start the timer when you pick up the pen.

**During the session:**
4. Talk out loud. Narrate your thinking: "I'm considering a hash map here because lookup is O(1)..." This is not optional. Silent mock interviews do not prepare you for real ones. The discomfort of narrating is exactly what you need to practice.
5. If you're stuck, say so out loud: "I'm not sure about the edge case here — let me think through a small example." This is what strong candidates do. Silence looks like giving up.
6. For coding: write the solution on paper before typing it. This removes the crutch of IDE autocomplete and forces real syntax recall.

**After time is up:**
7. Before looking at any hints or solutions, write a brief self-review:
   - Did I clarify requirements before starting?
   - Did I state the time/space complexity?
   - Did I handle edge cases?
   - Was my explanation clear, or would I have lost the interviewer?
   - Did I finish, or did I run out of time? Why?

### The Self-Review Checklist

**For coding:**
- [ ] Clarified input/output format and constraints before coding
- [ ] Started with brute force, then optimized
- [ ] Stated time and space complexity for my solution
- [ ] Tested with: normal case, empty/null input, single element, duplicates, negatives
- [ ] Talked through my logic while coding, not after

**For LLD:**
- [ ] Asked 3-5 clarifying questions before touching the design
- [ ] Drew a class diagram or entity list before writing methods
- [ ] Identified at least one design pattern (and justified why)
- [ ] Discussed at least one tradeoff
- [ ] Mentioned thread safety if the system is concurrent

**For HLD:**
- [ ] Did back-of-envelope estimation for scale, storage, bandwidth
- [ ] Drew an architecture diagram before going deep on any component
- [ ] Identified the single hardest technical challenge and explained how the design addresses it
- [ ] Discussed at least one alternative approach and why you didn't choose it
- [ ] Didn't over-engineer — matched complexity to the requirements stated

**For behavioral:**
- [ ] Gave a specific Situation (named the company, team, project)
- [ ] Clearly stated what MY task or role was
- [ ] Used "I" more than "we" in the Action section
- [ ] Ended with a measurable Result
- [ ] Answered in under 8 minutes (practice this — over-long answers lose interviewers)

### Frequency and Format

- Do one full mock session 3x per week minimum during active prep
- Alternate between coding, design, and behavioral — don't just do the one you're comfortable with
- Once a week: do a "real" mock with another person (peer, friend, mock platform). The nervous energy of being watched is irreplaceable.
- Record yourself at least once. Watch the playback without skipping. It's uncomfortable. It's also the single most useful thing you can do.

### The Night Before

Do not cram. Do one easy coding problem to warm up your hands, then review your notes on 2-3 behavioral stories. Sleep. The interview tests your thinking under pressure — you cannot think well when you're exhausted.

---

*The best mock interview is the one you actually do, alone, on a timer, talking out loud, even though it feels awkward. That feeling is the prep working.*
