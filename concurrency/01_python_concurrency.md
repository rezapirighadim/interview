# Python Concurrency Patterns

## When to Use What

The three models are not interchangeable. Pick wrong and you get either slower code or code that doesn't actually run in parallel.

| Workload | Best tool | Why |
|---|---|---|
| I/O-bound (network, disk) — many threads | `threading` | Threads release GIL on I/O; simple to reason about |
| I/O-bound — highly concurrent async | `asyncio` | Single thread, cooperative scheduling, massive scale |
| CPU-bound (computation, data crunching) | `multiprocessing` | Separate processes bypass GIL entirely |
| Mix of CPU + I/O | `ProcessPoolExecutor` + `asyncio` | Run blocking CPU work off the event loop |
| Quick parallel map over data | `concurrent.futures` | Simple API wrapping both thread and process pools |

**Rule of thumb:** if you're waiting on something external → threads or async. If you're computing something heavy → processes.

---

## The GIL

The Global Interpreter Lock is a mutex inside CPython that allows only one thread to execute Python bytecode at a time. It exists to protect CPython's reference counting from race conditions.

**What the GIL actually blocks:**
- Two threads running Python bytecode simultaneously in the same process

**What the GIL does NOT block:**
- I/O operations (threads release the GIL while waiting on a syscall)
- C extension code that releases the GIL explicitly (NumPy, OpenCV, etc.)
- Multiple processes (each has its own GIL)

**Common misconceptions:**
- "Threading is useless in Python" — False. For I/O-bound work, threads are effective and simple.
- "asyncio is faster than threading for I/O" — Depends. asyncio scales better (no thread overhead), but threading is often good enough and easier to read.
- "multiprocessing always wins for CPU work" — Interprocess communication has overhead. For small tasks the IPC cost can dominate.

```python
import threading
import time

# GIL demonstration: two CPU-bound threads don't actually run in parallel
counter = 0

def cpu_work(n):
    global counter
    for _ in range(n):
        counter += 1  # This is NOT atomic despite the GIL; the GIL releases between bytecodes

t1 = threading.Thread(target=cpu_work, args=(1_000_000,))
t2 = threading.Thread(target=cpu_work, args=(1_000_000,))
t1.start(); t2.start()
t1.join(); t2.join()
# counter will NOT reliably be 2_000_000 — race condition!
print(counter)  # could be less than 2_000_000
```

---

## threading Module

### Thread

```python
import threading

def worker(name, delay):
    print(f"{name} starting")
    time.sleep(delay)
    print(f"{name} done")

# Daemon threads die when the main thread exits
t = threading.Thread(target=worker, args=("worker-1", 1), daemon=True)
t.start()
t.join()  # Block until thread finishes
```

### Lock — protecting shared state

```python
import threading

class SafeCounter:
    def __init__(self):
        self._count = 0
        self._lock = threading.Lock()

    def increment(self):
        with self._lock:          # always use context manager — releases on exception
            self._count += 1

    @property
    def value(self):
        with self._lock:
            return self._count

counter = SafeCounter()
threads = [threading.Thread(target=counter.increment) for _ in range(1000)]
for t in threads: t.start()
for t in threads: t.join()
print(counter.value)  # always 1000
```

### RLock — reentrant lock

Use when the same thread may try to acquire the same lock recursively (e.g., a method that calls another method that also acquires the lock).

```python
import threading

class TreeNode:
    def __init__(self, value):
        self.value = value
        self.children = []
        self._lock = threading.RLock()  # same thread can acquire multiple times

    def add_child(self, child):
        with self._lock:
            self.children.append(child)

    def add_and_log(self, child):
        with self._lock:           # acquires lock
            self.add_child(child)  # acquires lock again — RLock allows this
            print(f"Added {child.value} to {self.value}")
```

### Semaphore — limiting concurrency

```python
import threading
import time
import random

# Allow at most 3 concurrent "connections"
sem = threading.Semaphore(3)

def fetch(url):
    with sem:  # blocks if 3 others are already inside
        print(f"Fetching {url}")
        time.sleep(random.uniform(0.5, 1.5))
        print(f"Done {url}")

threads = [threading.Thread(target=fetch, args=(f"http://example.com/{i}",)) for i in range(10)]
for t in threads: t.start()
for t in threads: t.join()
```

### Event — signaling between threads

```python
import threading
import time

ready = threading.Event()

def producer():
    print("Producer: preparing data...")
    time.sleep(2)
    ready.set()  # signal all waiting threads
    print("Producer: signaled")

def consumer(name):
    print(f"{name}: waiting for data")
    ready.wait()  # blocks until event is set
    print(f"{name}: processing data")

t_prod = threading.Thread(target=producer)
t_cons = [threading.Thread(target=consumer, args=(f"Consumer-{i}",)) for i in range(3)]

for t in t_cons: t.start()
t_prod.start()
for t in t_cons: t.join()
t_prod.join()
```

### Condition — wait for a specific state

```python
import threading
import time

class BoundedBuffer:
    def __init__(self, capacity):
        self._buf = []
        self._cap = capacity
        self._cond = threading.Condition()

    def put(self, item):
        with self._cond:
            while len(self._buf) >= self._cap:
                self._cond.wait()       # release lock, sleep, re-acquire when notified
            self._buf.append(item)
            self._cond.notify_all()

    def get(self):
        with self._cond:
            while not self._buf:
                self._cond.wait()
            item = self._buf.pop(0)
            self._cond.notify_all()
            return item
```

---

## multiprocessing Module

### Pool — parallel map

```python
from multiprocessing import Pool
import math

def is_prime(n):
    if n < 2: return False
    for i in range(2, int(math.sqrt(n)) + 1):
        if n % i == 0: return False
    return True

if __name__ == "__main__":
    numbers = range(10_000, 11_000)
    with Pool(processes=4) as pool:
        results = pool.map(is_prime, numbers)  # distributes work across 4 processes
    primes = [n for n, p in zip(numbers, results) if p]
    print(f"Found {len(primes)} primes")
```

### Process + Queue — interprocess communication

```python
from multiprocessing import Process, Queue
import time

def worker(task_queue, result_queue):
    while True:
        task = task_queue.get()
        if task is None:  # poison pill
            break
        result = task ** 2
        result_queue.put(result)

if __name__ == "__main__":
    tasks = Queue()
    results = Queue()

    procs = [Process(target=worker, args=(tasks, results)) for _ in range(4)]
    for p in procs: p.start()

    for i in range(20):
        tasks.put(i)
    for _ in procs:
        tasks.put(None)  # one poison pill per worker

    for p in procs: p.join()

    while not results.empty():
        print(results.get())
```

### Pipe — bidirectional communication

```python
from multiprocessing import Process, Pipe

def child(conn):
    msg = conn.recv()
    conn.send(f"echo: {msg}")
    conn.close()

if __name__ == "__main__":
    parent_conn, child_conn = Pipe()
    p = Process(target=child, args=(child_conn,))
    p.start()
    parent_conn.send("hello from parent")
    print(parent_conn.recv())
    p.join()
```

### Shared Memory — zero-copy sharing

```python
from multiprocessing import Process, Value, Array
import ctypes

def increment(val):
    for _ in range(1000):
        with val.get_lock():  # Value has a built-in lock
            val.value += 1

if __name__ == "__main__":
    shared_val = Value(ctypes.c_int, 0)
    procs = [Process(target=increment, args=(shared_val,)) for _ in range(4)]
    for p in procs: p.start()
    for p in procs: p.join()
    print(shared_val.value)  # 4000
```

---

## asyncio

### Core concepts

```python
import asyncio

# A coroutine is defined with async def
# It doesn't run until awaited or scheduled as a task
async def fetch_data(url: str) -> str:
    await asyncio.sleep(1)  # simulates I/O without blocking
    return f"data from {url}"

# gather runs coroutines concurrently (not sequentially)
async def main():
    results = await asyncio.gather(
        fetch_data("http://api.example.com/users"),
        fetch_data("http://api.example.com/orders"),
        fetch_data("http://api.example.com/products"),
    )
    # All three run concurrently — total ~1 second, not 3
    for r in results: print(r)

asyncio.run(main())
```

### Tasks — fire and forget (with handle)

```python
import asyncio

async def background_job(name, delay):
    await asyncio.sleep(delay)
    print(f"{name} complete")

async def main():
    # create_task schedules immediately, doesn't block
    t1 = asyncio.create_task(background_job("job-1", 2))
    t2 = asyncio.create_task(background_job("job-2", 1))

    print("Tasks scheduled, doing other work...")
    await asyncio.sleep(0.5)
    print("Still going...")

    await t1  # wait for specific task
    await t2

asyncio.run(main())
```

### asyncio.Queue — producer-consumer

```python
import asyncio
import random

async def producer(queue: asyncio.Queue, n_items: int):
    for i in range(n_items):
        item = random.randint(1, 100)
        await queue.put(item)
        print(f"Produced: {item}")
        await asyncio.sleep(0.1)
    # Signal consumers to stop
    for _ in range(3):  # number of consumers
        await queue.put(None)

async def consumer(name: str, queue: asyncio.Queue):
    while True:
        item = await queue.get()
        if item is None:
            queue.task_done()
            break
        print(f"{name} consumed: {item}")
        await asyncio.sleep(0.2)
        queue.task_done()

async def main():
    queue = asyncio.Queue(maxsize=5)
    prod = asyncio.create_task(producer(queue, 10))
    consumers = [asyncio.create_task(consumer(f"C{i}", queue)) for i in range(3)]
    await asyncio.gather(prod, *consumers)

asyncio.run(main())
```

### asyncio.wait — first-done and timeout patterns

```python
import asyncio

async def risky_call(name, delay):
    await asyncio.sleep(delay)
    return f"{name} done"

async def main():
    tasks = {
        asyncio.create_task(risky_call("fast", 1)),
        asyncio.create_task(risky_call("slow", 5)),
        asyncio.create_task(risky_call("medium", 2)),
    }

    # Return as soon as first one finishes
    done, pending = await asyncio.wait(tasks, return_when=asyncio.FIRST_COMPLETED)
    for t in done:
        print(f"First result: {t.result()}")
    for t in pending:
        t.cancel()  # cancel remaining

asyncio.run(main())
```

---

## Classic Concurrency Problems

### Producer-Consumer with threading.Queue

```python
import threading
import queue
import time
import random

def producer(q: queue.Queue, n_items: int):
    for i in range(n_items):
        item = random.randint(1, 100)
        q.put(item)
        print(f"Produced: {item}")
        time.sleep(0.05)
    q.put(None)  # sentinel

def consumer(q: queue.Queue):
    while True:
        item = q.get()
        if item is None:
            q.task_done()
            break
        print(f"Consumed: {item}")
        time.sleep(0.1)
        q.task_done()

q = queue.Queue(maxsize=10)
p = threading.Thread(target=producer, args=(q, 20))
c = threading.Thread(target=consumer, args=(q,))
p.start(); c.start()
p.join(); c.join()
```

### Thread Pool with concurrent.futures

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import time

def download(url: str) -> str:
    time.sleep(0.5)  # simulate network call
    return f"content of {url}"

urls = [f"http://example.com/page/{i}" for i in range(20)]

with ThreadPoolExecutor(max_workers=5) as executor:
    future_to_url = {executor.submit(download, url): url for url in urls}
    for future in as_completed(future_to_url):
        url = future_to_url[future]
        try:
            data = future.result()
            print(f"Got: {data}")
        except Exception as e:
            print(f"Failed {url}: {e}")
```

### Rate-Limited API Caller

```python
import threading
import time
import queue

class RateLimiter:
    """Allow at most `calls` calls per `period` seconds."""
    def __init__(self, calls: int, period: float):
        self._sem = threading.Semaphore(calls)
        self._calls = calls
        self._period = period

    def __enter__(self):
        self._sem.acquire()
        return self

    def __exit__(self, *args):
        # Release after `period` seconds in a background thread
        def release():
            time.sleep(self._period)
            self._sem.release()
        threading.Thread(target=release, daemon=True).start()

def call_api(endpoint: str, limiter: RateLimiter):
    with limiter:
        print(f"Calling {endpoint} at {time.time():.2f}")
        time.sleep(0.1)  # actual call

limiter = RateLimiter(calls=3, period=1.0)  # 3 calls per second
threads = [threading.Thread(target=call_api, args=(f"/endpoint/{i}", limiter)) for i in range(10)]
for t in threads: t.start()
for t in threads: t.join()
```

### In-Process Pub/Sub

```python
import threading
from collections import defaultdict
from typing import Callable

class EventBus:
    def __init__(self):
        self._subscribers: dict[str, list[Callable]] = defaultdict(list)
        self._lock = threading.Lock()

    def subscribe(self, topic: str, handler: Callable):
        with self._lock:
            self._subscribers[topic].append(handler)

    def publish(self, topic: str, data):
        with self._lock:
            handlers = list(self._subscribers[topic])
        for handler in handlers:
            # run handlers in separate threads so a slow handler doesn't block
            threading.Thread(target=handler, args=(data,), daemon=True).start()

bus = EventBus()
bus.subscribe("order.created", lambda d: print(f"Email service: order {d['id']}"))
bus.subscribe("order.created", lambda d: print(f"Inventory service: reserve {d['item']}"))

bus.publish("order.created", {"id": 42, "item": "widget"})
import time; time.sleep(0.1)  # let daemon threads finish
```

### Dining Philosophers — Deadlock and Solution

```python
import threading
import time
import random

# DEADLOCK version (don't use in production)
def philosopher_deadlock(idx, left_fork, right_fork):
    for _ in range(3):
        left_fork.acquire()   # everyone grabs left
        right_fork.acquire()  # then waits for right → deadlock
        print(f"Philosopher {idx} eating")
        time.sleep(random.uniform(0.1, 0.3))
        right_fork.release()
        left_fork.release()

# SOLUTION: impose a global ordering on lock acquisition
def philosopher_safe(idx, forks):
    n = len(forks)
    left, right = idx, (idx + 1) % n
    # Always acquire the lower-numbered fork first
    first, second = (left, right) if left < right else (right, left)
    for _ in range(3):
        forks[first].acquire()
        forks[second].acquire()
        print(f"Philosopher {idx} eating")
        time.sleep(random.uniform(0.1, 0.3))
        forks[second].release()
        forks[first].release()
        time.sleep(random.uniform(0.1, 0.3))  # think

n = 5
forks = [threading.Lock() for _ in range(n)]
philosophers = [
    threading.Thread(target=philosopher_safe, args=(i, forks)) for i in range(n)
]
for p in philosophers: p.start()
for p in philosophers: p.join()
print("No deadlock!")
```

---

## Common Interview Questions

**Q: What is the GIL and does it make Python thread-safe?**

The GIL ensures only one thread runs Python bytecode at once, but it does NOT make your code thread-safe. The GIL can be released between any two bytecodes. Operations that look atomic in Python (like `x += 1`) are multiple bytecodes and can be interrupted. Use `threading.Lock` for shared mutable state.

**Q: When would you use multiprocessing over asyncio?**

CPU-bound work. asyncio is cooperative single-threaded concurrency — it can't parallelize computation. multiprocessing spawns real OS processes, each with its own Python interpreter, bypassing the GIL.

**Q: How does asyncio differ from threading?**

Threading is preemptive (OS decides when to switch) and uses real OS threads. asyncio is cooperative (coroutines yield control explicitly via `await`) and runs on a single thread. asyncio has lower overhead per concurrent task but requires `async/await` throughout your call stack.

**Q: What's the difference between `asyncio.gather` and `asyncio.wait`?**

`gather` takes coroutines, wraps them in tasks, runs them concurrently, returns results in input order, and by default propagates the first exception. `wait` takes tasks/futures, returns two sets (done, pending), and lets you control when to stop (`FIRST_COMPLETED`, `FIRST_EXCEPTION`, `ALL_COMPLETED`).

**Q: How do you share data between multiprocessing processes?**

Options in order of preference: `multiprocessing.Queue` (safe, designed for IPC), `multiprocessing.Pipe` (faster, point-to-point), `multiprocessing.Value`/`Array` (shared memory, needs explicit locking), `multiprocessing.Manager` (proxy objects, slowest but most flexible).

**Q: What is a thread-safe data structure in Python's standard library?**

`queue.Queue` is thread-safe (uses internal locks). `collections.deque` operations are thread-safe for individual append/pop due to the GIL, but not for compound operations. Regular `list`, `dict` etc. are not safe for concurrent modification.

---

## 5 Subtle Bugs to Watch For

### 1. Race condition on compound operations

```python
# BUG: read-check-write is not atomic
if key not in shared_dict:
    shared_dict[key] = expensive_computation()
# Two threads can both pass the `if` check before either writes

# FIX: use a lock around the whole compound operation
with lock:
    if key not in shared_dict:
        shared_dict[key] = expensive_computation()
```

### 2. Deadlock from inconsistent lock ordering

```python
# BUG: thread A holds lock_a, waits for lock_b
#      thread B holds lock_b, waits for lock_a
def transfer_a_to_b():
    with lock_a:
        with lock_b: ...

def transfer_b_to_a():
    with lock_b:       # inconsistent order
        with lock_a: ...

# FIX: always acquire locks in the same global order
def transfer(src_lock, dst_lock):
    first, second = sorted([src_lock, dst_lock], key=id)
    with first:
        with second: ...
```

### 3. Forgetting to cancel tasks in asyncio

```python
# BUG: pending tasks keep running after you're done with them
async def bad():
    tasks = [asyncio.create_task(work()) for _ in range(10)]
    done, pending = await asyncio.wait(tasks, return_when=asyncio.FIRST_COMPLETED)
    return done.pop().result()
    # pending tasks are leaked — they run until the event loop closes

# FIX: cancel pending tasks
async def good():
    tasks = [asyncio.create_task(work()) for _ in range(10)]
    done, pending = await asyncio.wait(tasks, return_when=asyncio.FIRST_COMPLETED)
    for t in pending:
        t.cancel()
    return done.pop().result()
```

### 4. Using mutable default arguments in threads

```python
# BUG: all threads share the same list object
def worker(results=[]):  # mutable default is created ONCE
    results.append(1)
    return results

# FIX
def worker(results=None):
    if results is None:
        results = []
    results.append(1)
    return results
```

### 5. Starvation with unfair locks

```python
# BUG: if many threads continuously acquire a lock,
# a low-priority thread may never get it (starvation)
# Python's threading.Lock is not fair (not FIFO)

# FIX: use a Queue to enforce FIFO ordering
request_queue = queue.Queue()

def submit_task(fn):
    event = threading.Event()
    request_queue.put((fn, event))
    event.wait()  # caller blocks until its turn

def dispatcher():
    while True:
        fn, event = request_queue.get()
        fn()
        event.set()  # wake up the caller
```
