# LLD — LRU Cache

## Problem Statement

Design an LRU (Least Recently Used) Cache with O(1) `get` and `put` operations. When
the cache is full and a new item is inserted, the least recently used item must be evicted.
The design should be clean enough to be extended with TTL support, statistics, and
different eviction policies.

---

## Requirements

**Functional**
- `get(key)` — return the value if key exists (and mark it as recently used), else return -1
- `put(key, value)` — insert or update the key. If at capacity, evict the LRU item first
- Capacity is fixed at construction time

**Non-functional**
- Both `get` and `put` must be O(1)
- Support easy extension: LFU eviction, TTL expiry, statistics tracking

---

## Design Patterns Used

| Pattern | Where | Why |
|---|---|---|
| **Doubly Linked List + HashMap** | Core data structure | O(1) remove + O(1) lookup |
| **Strategy** | `EvictionPolicy` | Swap LRU for LFU or FIFO without changing the cache |
| **Decorator** | `StatsCache` | Add hit/miss tracking around the core cache with zero coupling |
| **Null Object** | `CacheMiss` sentinel | Avoid returning `None`/`-1` and checking everywhere |

---

## Class Diagram

```
EvictionPolicy (interface)
  ├── LRUPolicy
  └── FIFOPolicy (extensible)

Cache (interface)
  ├── LRUCache
  │     ├── _map: dict[key → Node]
  │     └── _list: DoublyLinkedList
  │               ├── head (sentinel)
  │               └── tail (sentinel)
  └── StatsCache (Decorator)
        └── wraps: Cache
```

---

## Clean Code Principles Applied

- **Single Responsibility:** `DoublyLinkedList` only manages node order. `LRUCache` only
  manages the eviction logic. `StatsCache` only tracks metrics.
- **Open/Closed:** Adding TTL support means writing a new `TTLCache(cache)` decorator —
  zero existing code changes.
- **Encapsulation:** The doubly linked list internals are private. Callers only see `get`/`put`.
- **Meaningful Names:** `move_to_front`, `evict_lru`, `_detach`, `_prepend` — each name
  describes exactly one action.
- **Sentinel nodes:** `head` and `tail` dummy nodes eliminate all edge case checks in
  the list manipulation code.

---

## Python Implementation

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Any, Optional
import time


# ─── Domain: Doubly Linked List Node ─────────────────────────────────────────

class _Node:
    __slots__ = ("key", "value", "prev", "next", "created_at")

    def __init__(self, key: Any = None, value: Any = None):
        self.key   = key
        self.value = value
        self.prev: Optional[_Node] = None
        self.next: Optional[_Node] = None
        self.created_at = time.time()


# ─── Doubly Linked List (internal utility) ───────────────────────────────────

class _DoublyLinkedList:
    """Head = most recent. Tail = least recent (LRU candidate)."""

    def __init__(self):
        self._head = _Node()   # sentinel
        self._tail = _Node()   # sentinel
        self._head.next = self._tail
        self._tail.prev = self._head

    def prepend(self, node: _Node) -> None:
        """Insert node right after head (mark as most recent)."""
        node.next = self._head.next
        node.prev = self._head
        self._head.next.prev = node
        self._head.next = node

    def detach(self, node: _Node) -> None:
        """Remove node from its current position."""
        node.prev.next = node.next
        node.next.prev = node.prev

    def move_to_front(self, node: _Node) -> None:
        self.detach(node)
        self.prepend(node)

    def pop_lru(self) -> Optional[_Node]:
        """Remove and return the node just before the tail sentinel."""
        lru = self._tail.prev
        if lru is self._head:
            return None
        self.detach(lru)
        return lru


# ─── Cache Interface ─────────────────────────────────────────────────────────

class Cache(ABC):
    @abstractmethod
    def get(self, key: Any) -> Any:
        ...

    @abstractmethod
    def put(self, key: Any, value: Any) -> None:
        ...

    @abstractmethod
    def __len__(self) -> int:
        ...


# ─── Core LRU Cache ──────────────────────────────────────────────────────────

class LRUCache(Cache):
    MISS = object()  # Null Object sentinel — avoids if-None checks everywhere

    def __init__(self, capacity: int):
        if capacity <= 0:
            raise ValueError("Capacity must be a positive integer")
        self._capacity = capacity
        self._map: dict[Any, _Node] = {}
        self._list = _DoublyLinkedList()

    def get(self, key: Any) -> Any:
        node = self._map.get(key)
        if node is None:
            return LRUCache.MISS
        self._list.move_to_front(node)  # recently used → move to front
        return node.value

    def put(self, key: Any, value: Any) -> None:
        if key in self._map:
            node = self._map[key]
            node.value = value
            self._list.move_to_front(node)
        else:
            if len(self._map) >= self._capacity:
                self._evict()
            node = _Node(key, value)
            self._map[key] = node
            self._list.prepend(node)

    def _evict(self) -> None:
        lru_node = self._list.pop_lru()
        if lru_node:
            del self._map[lru_node.key]

    def __len__(self) -> int:
        return len(self._map)

    @property
    def capacity(self) -> int:
        return self._capacity


# ─── Decorator Pattern: Stats ─────────────────────────────────────────────────

@dataclass
class CacheStats:
    hits:   int = 0
    misses: int = 0

    @property
    def total(self) -> int:
        return self.hits + self.misses

    @property
    def hit_rate(self) -> float:
        return self.hits / self.total if self.total else 0.0

    def __repr__(self) -> str:
        return (f"CacheStats(hits={self.hits}, misses={self.misses}, "
                f"hit_rate={self.hit_rate:.1%})")


class StatsCache(Cache):
    """Decorator: wraps any Cache and tracks hit/miss statistics."""

    def __init__(self, cache: Cache):
        self._cache = cache
        self.stats  = CacheStats()

    def get(self, key: Any) -> Any:
        result = self._cache.get(key)
        if result is LRUCache.MISS:
            self.stats.misses += 1
        else:
            self.stats.hits += 1
        return result

    def put(self, key: Any, value: Any) -> None:
        self._cache.put(key, value)

    def __len__(self) -> int:
        return len(self._cache)


# ─── Decorator Pattern: TTL ───────────────────────────────────────────────────

class TTLCache(Cache):
    """Decorator: evicts keys that have exceeded their time-to-live."""

    def __init__(self, cache: Cache, ttl_seconds: float):
        self._cache = cache
        self._ttl   = ttl_seconds
        self._timestamps: dict[Any, float] = {}

    def get(self, key: Any) -> Any:
        if self._is_expired(key):
            self._timestamps.pop(key, None)
            return LRUCache.MISS
        return self._cache.get(key)

    def put(self, key: Any, value: Any) -> None:
        self._timestamps[key] = time.time()
        self._cache.put(key, value)

    def _is_expired(self, key: Any) -> bool:
        ts = self._timestamps.get(key)
        return ts is not None and (time.time() - ts) > self._ttl

    def __len__(self) -> int:
        return len(self._cache)


# ─── Demo ─────────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    base  = LRUCache(capacity=3)
    cache = StatsCache(base)

    cache.put("a", 1)
    cache.put("b", 2)
    cache.put("c", 3)

    print(cache.get("a"))          # 1 (hit) — 'a' now most recent
    cache.put("d", 4)              # evicts 'b' (LRU)
    print(cache.get("b"))          # MISS
    print(cache.get("c"))          # 3 (hit)
    print(cache.stats)             # CacheStats(hits=2, misses=1, hit_rate=66.7%)
```

---

## Node.js Implementation

```javascript
// lru-cache.js

// ─── Doubly Linked List Node ──────────────────────────────────────────────────

class _Node {
  constructor(key = null, value = null) {
    this.key   = key;
    this.value = value;
    this.prev  = null;
    this.next  = null;
  }
}

// ─── Doubly Linked List ───────────────────────────────────────────────────────

class _DoublyLinkedList {
  constructor() {
    this._head = new _Node();  // sentinel
    this._tail = new _Node();  // sentinel
    this._head.next = this._tail;
    this._tail.prev = this._head;
  }

  prepend(node) {
    node.next = this._head.next;
    node.prev = this._head;
    this._head.next.prev = node;
    this._head.next = node;
  }

  detach(node) {
    node.prev.next = node.next;
    node.next.prev = node.prev;
  }

  moveToFront(node) {
    this.detach(node);
    this.prepend(node);
  }

  popLRU() {
    const lru = this._tail.prev;
    if (lru === this._head) return null;
    this.detach(lru);
    return lru;
  }
}

// ─── Null Object sentinel ─────────────────────────────────────────────────────

const CACHE_MISS = Symbol("CACHE_MISS");

// ─── Core LRU Cache ──────────────────────────────────────────────────────────

class LRUCache {
  static MISS = CACHE_MISS;

  #capacity;
  #map;
  #list;

  constructor(capacity) {
    if (capacity <= 0) throw new Error("Capacity must be a positive integer");
    this.#capacity = capacity;
    this.#map  = new Map();
    this.#list = new _DoublyLinkedList();
  }

  get(key) {
    const node = this.#map.get(key);
    if (!node) return CACHE_MISS;
    this.#list.moveToFront(node);
    return node.value;
  }

  put(key, value) {
    if (this.#map.has(key)) {
      const node = this.#map.get(key);
      node.value = value;
      this.#list.moveToFront(node);
    } else {
      if (this.#map.size >= this.#capacity) this.#evict();
      const node = new _Node(key, value);
      this.#map.set(key, node);
      this.#list.prepend(node);
    }
  }

  #evict() {
    const lru = this.#list.popLRU();
    if (lru) this.#map.delete(lru.key);
  }

  get size()     { return this.#map.size; }
  get capacity() { return this.#capacity; }
}

// ─── Decorator: Stats ─────────────────────────────────────────────────────────

class StatsCache {
  #cache;
  #hits   = 0;
  #misses = 0;

  constructor(cache) { this.#cache = cache; }

  get(key) {
    const result = this.#cache.get(key);
    result === CACHE_MISS ? this.#misses++ : this.#hits++;
    return result;
  }

  put(key, value) { this.#cache.put(key, value); }

  get stats() {
    const total   = this.#hits + this.#misses;
    const hitRate = total ? ((this.#hits / total) * 100).toFixed(1) + "%" : "N/A";
    return { hits: this.#hits, misses: this.#misses, hitRate };
  }

  get size() { return this.#cache.size; }
}

// ─── Decorator: TTL ───────────────────────────────────────────────────────────

class TTLCache {
  #cache;
  #ttl;
  #timestamps = new Map();

  constructor(cache, ttlMs) {
    this.#cache = cache;
    this.#ttl   = ttlMs;
  }

  get(key) {
    if (this.#isExpired(key)) {
      this.#timestamps.delete(key);
      return CACHE_MISS;
    }
    return this.#cache.get(key);
  }

  put(key, value) {
    this.#timestamps.set(key, Date.now());
    this.#cache.put(key, value);
  }

  #isExpired(key) {
    const ts = this.#timestamps.get(key);
    return ts !== undefined && Date.now() - ts > this.#ttl;
  }

  get size() { return this.#cache.size; }
}

// ─── Demo ─────────────────────────────────────────────────────────────────────

const base  = new LRUCache(3);
const cache = new StatsCache(base);

cache.put("a", 1);
cache.put("b", 2);
cache.put("c", 3);

console.log(cache.get("a"));    // 1  (hit) — 'a' is now most recent
cache.put("d", 4);              // evicts 'b' (LRU)
console.log(cache.get("b"));    // Symbol(CACHE_MISS)
console.log(cache.get("c"));    // 3  (hit)
console.log(cache.stats);       // { hits: 2, misses: 1, hitRate: '66.7%' }
```

---

## Key Takeaways

| Concept | Detail |
|---|---|
| HashMap + DLL | The only data structure combination that gives O(1) for both lookup and ordered removal |
| Sentinel nodes | Dummy head/tail eliminate all if-empty edge cases inside the list |
| Decorator | `StatsCache(TTLCache(LRUCache(3)))` — stack behaviors without inheritance explosion |
| Null Object | `CACHE_MISS` sentinel is safer than `None`/`-1` — avoids ambiguity when `None` is a valid cached value |
| `__slots__` | On `_Node` in Python: reduces memory by ~40% when there are millions of cache nodes |
