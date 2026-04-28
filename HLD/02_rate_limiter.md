# HLD — Distributed Rate Limiter

## Problem Statement

Design a rate limiting system that enforces request quotas across a distributed fleet
of API servers. The limiter must be correct under concurrent load (no race conditions),
operate with sub-millisecond overhead, and support multiple granularities: per-user,
per-IP, and per-endpoint. Clients that exceed their quota receive a 429 with actionable
retry headers.

---

## Requirements

**Functional**
- Enforce limits per user ID, per IP address, and per endpoint
- Configurable limits: e.g. 1,000 requests/user/hour, 100 requests/IP/minute
- Return `429 Too Many Requests` with `Retry-After` and `X-RateLimit-*` headers
- Limits apply across all API server instances (distributed, not per-instance)
- Support burst allowance (token bucket semantics)

**Non-functional**
- Decision latency < 5ms p99 (one Redis round-trip)
- No false positives under concurrent requests from the same user
- Graceful degradation: if Redis is unavailable, fail open (allow traffic) to avoid
  an outage caused by the limiter itself
- Hot-reloadable config — change limits without a deploy

**Out of scope**
- DDoS mitigation at the network layer (that's a WAF / BGP blackholing problem)
- Per-tenant billing metering (different problem domain)

---

## Core Concepts / Design Decisions

### Algorithm Comparison

| Algorithm | Memory | Allows burst | Smooth output | Distributed-friendly | Accuracy |
|---|---|---|---|---|---|
| **Token Bucket** | O(1) per key | Yes — up to bucket size | No | Yes (Redis counter) | High |
| **Leaky Bucket** | O(1) per key + queue | No — fixed output rate | Yes — constant drain | Hard (queue is stateful) | High |
| **Fixed Window Counter** | O(1) per key | Yes — double at boundary | No | Yes (Redis INCR) | Low at window edges |
| **Sliding Window Log** | O(n) — stores timestamps | No burst after quota | Very accurate | Expensive at scale | Highest |
| **Sliding Window Counter** | O(1) per key | Partial — weighted | Near-accurate | Yes (two Redis keys) | High |

**Decision:** Use **sliding window counter** for per-user and per-endpoint limits.
Use **token bucket** for burst-tolerant endpoints. Both are implementable atomically
in Redis without Lua scripts for the basic case.

### Why not leaky bucket?

Leaky bucket requires a per-user in-memory queue to meter output. In a distributed
environment that queue can't live on individual servers. Moving it to Redis turns it
into a complex queue-per-user — operationally painful and memory-intensive at millions
of users.

### Fixed window problem

A user gets 1,000 req/hour. With a fixed window, they can make 1,000 requests at
23:59 and another 1,000 at 00:01 — 2,000 requests in 2 minutes, 2× the intended rate.
Sliding window eliminates this spike.

---

## Algorithm Deep Dives

### Token Bucket (Redis)

```
bucket_key = "tb:{user_id}:{endpoint}"
tokens     = GET bucket_key
if tokens is None:
    tokens = MAX_TOKENS
refill = (now - last_refill) * REFILL_RATE
tokens = min(MAX_TOKENS, tokens + refill)
if tokens >= 1:
    tokens -= 1
    SET bucket_key {tokens, now}
    ALLOW
else:
    DENY
```

The problem: reading tokens, computing refill, and writing back is a read-modify-write
cycle. Two concurrent requests can both read `tokens=1`, both decrement, and both
be allowed — that is a race condition. **Fix: Lua script** (executes atomically on
Redis, no context switch between read and write).

```lua
-- token_bucket.lua
-- KEYS[1] = bucket key
-- ARGV[1] = now (unix ms), ARGV[2] = max_tokens, ARGV[3] = refill_rate (tokens/ms)

local data     = redis.call("HMGET", KEYS[1], "tokens", "last_refill")
local tokens   = tonumber(data[1]) or tonumber(ARGV[2])
local last     = tonumber(data[2]) or tonumber(ARGV[1])
local elapsed  = tonumber(ARGV[1]) - last
local refill   = elapsed * tonumber(ARGV[3])

tokens = math.min(tonumber(ARGV[2]), tokens + refill)

if tokens >= 1 then
    tokens = tokens - 1
    redis.call("HMSET", KEYS[1], "tokens", tokens, "last_refill", ARGV[1])
    redis.call("PEXPIRE", KEYS[1], 3600000)
    return {1, math.floor(tokens)}
else
    return {0, 0}
end
```

The Lua script runs as a single Redis command — atomically. No WATCH/MULTI/EXEC needed.

### Sliding Window Counter (Redis)

More accurate than fixed window, O(1) memory, near-zero computation:

```
# Two buckets: current minute and previous minute
# Weight previous bucket by how far through the current minute we are.

now           = current_unix_timestamp
window_size   = 60  # seconds
current_slot  = floor(now / window_size)
prev_slot     = current_slot - 1
elapsed       = now % window_size  # seconds into current window
weight        = 1 - (elapsed / window_size)

current_key   = "sw:{user_id}:{endpoint}:{current_slot}"
prev_key      = "sw:{user_id}:{endpoint}:{prev_slot}"

current_count = GET current_key  or 0
prev_count    = GET prev_key     or 0

estimated_count = prev_count * weight + current_count

if estimated_count < LIMIT:
    INCR current_key
    EXPIRE current_key  window_size * 2
    ALLOW
else:
    DENY
```

This is a good approximation. It assumes requests in the previous window were spread
uniformly — the error is at most a few percent in practice.

Atomic version as Lua script:

```lua
-- sliding_window.lua
-- KEYS[1]=current_key, KEYS[2]=prev_key
-- ARGV[1]=limit, ARGV[2]=weight, ARGV[3]=window_ttl_sec

local curr  = tonumber(redis.call("GET", KEYS[1])) or 0
local prev  = tonumber(redis.call("GET", KEYS[2])) or 0
local est   = prev * tonumber(ARGV[2]) + curr

if est < tonumber(ARGV[1]) then
    redis.call("INCR", KEYS[1])
    redis.call("EXPIRE", KEYS[1], tonumber(ARGV[3]))
    return {1, tonumber(ARGV[1]) - math.floor(est) - 1}
else
    return {0, 0}
end
```

### Sliding Window Log

Store the exact timestamp of each request. On each new request, remove all timestamps
older than the window, then count remaining entries.

```python
def is_allowed(user_id: str, limit: int, window_sec: int) -> bool:
    key = f"log:{user_id}"
    now = time.time()
    cutoff = now - window_sec

    pipe = redis.pipeline()
    pipe.zremrangebyscore(key, 0, cutoff)         # remove old entries
    pipe.zadd(key, {str(uuid4()): now})            # add current timestamp
    pipe.zcard(key)                                # count entries in window
    pipe.expire(key, window_sec)
    results = pipe.execute()

    count = results[2]
    if count > limit:
        redis.zrem(key, str(uuid4()))              # undo the add
        return False
    return True
```

Memory cost: O(requests_in_window). At 1,000 req/user/hour across 10M users that's
potentially 10B Redis sorted set entries — prohibitive. Use only for low-volume,
high-accuracy use cases (e.g. payment endpoints with very low limits).

---

## Architecture Diagram

```
Client Request
      │
      ▼
[CDN / Edge]  ← can do coarse IP blocking (edge rate limiting via Cloudflare/Nginx)
      │
      ▼
[API Gateway]  ←── Rate Limiter Middleware is injected here
      │                │
      │                ├── resolve identity: user_id from JWT or IP from header
      │                ├── build Redis key(s) for user + IP + endpoint
      │                ├── execute Lua script (atomic check + increment)
      │                └── attach X-RateLimit-* headers to response
      │
      │   allow                          deny
      ├──────────────────►[Service A]    └──► 429 + Retry-After header
      │
      ▼
[Redis Cluster]  ← sharded by key, each shard handles millions of counters
  ├── shard 0: user_* keys
  ├── shard 1: ip_* keys
  └── shard 2: endpoint_* keys

[Config Service]  ← stores rate limit rules, polled every 30s
  └── rules: { endpoint, user_tier, limit, window }
```

---

## Where to Place the Rate Limiter

| Placement | Pros | Cons |
|---|---|---|
| **Edge / CDN** | Stops traffic before it hits your infra | Limited logic, hard to access user identity |
| **API Gateway** | Central enforcement, sees all traffic | Single point of configuration, latency hop |
| **Per-service middleware** | Finer control per domain | Duplicated logic, inconsistent enforcement |
| **Sidecar proxy (Envoy)** | Language-agnostic, config-driven | More infra complexity (service mesh) |

**Decision:** Primary enforcement at API Gateway + secondary coarse IP limiting at the
CDN/edge. Service-level limiters are added only for critical resources (e.g. payment
service self-protects with its own stricter limits).

---

## Deep Dives

### Per-user vs Per-IP vs Per-endpoint

These are independent dimensions — all three can be active simultaneously. Check all
relevant keys; deny if any one fires.

```python
def check_all_limits(request) -> RateLimitResult:
    user_id  = request.user_id or "anonymous"
    ip       = request.client_ip
    endpoint = request.path  # e.g. "/v1/search"

    checks = [
        ("user",     f"rl:user:{user_id}",           USER_LIMIT,     WINDOW_SEC),
        ("ip",       f"rl:ip:{ip}",                  IP_LIMIT,       WINDOW_SEC),
        ("endpoint", f"rl:ep:{endpoint}:{user_id}",  ENDPOINT_LIMIT, WINDOW_SEC),
    ]

    for dimension, key, limit, window in checks:
        allowed, remaining = sliding_window_check(key, limit, window)
        if not allowed:
            return RateLimitResult(allowed=False, dimension=dimension, remaining=0)

    return RateLimitResult(allowed=True, remaining=min(r for _, r in ...))
```

Different user tiers get different limits — read from Config Service:

```json
{
  "tiers": {
    "free":       { "requests_per_hour": 100 },
    "pro":        { "requests_per_hour": 10000 },
    "enterprise": { "requests_per_hour": 1000000 }
  },
  "endpoints": {
    "/v1/search":  { "additional_limit": 50,    "window_sec": 60 },
    "/v1/payment": { "additional_limit": 10,    "window_sec": 3600 }
  }
}
```

### Response Headers

Always return rate limit state — clients need this to self-throttle:

```
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1714521600        ← unix timestamp when window resets
Retry-After: 43                      ← seconds until the client may retry
X-RateLimit-Policy: sliding-window
```

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1714521600
```

### Handling Distributed Race Conditions

The critical insight: all checks and increments happen **inside a single Lua script**
on a **single Redis node** (for a given key). Redis is single-threaded for command
execution, so Lua scripts are inherently atomic. There is no distributed race condition
for a single key.

The subtle case: the user's key lives on a specific Redis shard. All requests for
`user_id=abc` hash to the same shard — correct. If you use Redis Cluster, ensure
the key design lands the same logical entity on the same shard (avoid cross-slot
Lua calls).

```
key prefix: "rl:{user_id}" → always same slot
BAD:  KEYS[1]="rl:user:abc", KEYS[2]="rl:ip:1.2.3.4"  ← two different slots
GOOD: run two separate Lua calls on their respective shards
```

### Fail Open vs Fail Closed

```python
def is_request_allowed(key: str, limit: int) -> bool:
    try:
        result = redis.eval(LUA_SCRIPT, 1, key, limit, ...)
        return result[0] == 1
    except RedisConnectionError:
        # Fail open: better to allow excess traffic than to block all users
        # during a Redis outage. Log and alert immediately.
        metrics.increment("rate_limiter.redis_error")
        return True
    except Exception as e:
        log.error("Unexpected rate limiter error", exc_info=e)
        return True
```

Some systems fail closed for payment endpoints specifically — a configurable policy.

### Preventing Redis Hotspots

A single very-popular API key (e.g. a large enterprise customer) can hammer one Redis
shard. Solutions:
1. **Key sharding:** Append a random suffix 0–9 to the key, distribute counter across
   10 slots, sum them to get the total. Increases Redis calls but distributes load.
2. **Local counters:** Each API server keeps a local counter and syncs to Redis every
   100ms. Slight inaccuracy (up to N_servers × 100ms_requests) but eliminates hot keys.
   Good for high-volume, low-precision needs.

---

## Scale Numbers

| Metric | Value |
|---|---|
| API requests/sec | 500,000 |
| Rate limiter decisions/sec | 500,000 |
| Redis ops/sec (3 checks × 500K) | 1.5M ops/sec |
| Redis p99 command latency | < 1ms |
| Rate limiter overhead to request | < 3ms p99 |
| Keys in Redis (10M users × 3 dimensions) | ~30M keys |
| Memory per key (sliding window counter) | ~100 bytes |
| Total Redis memory | ~3 GB |
| Redis shards needed | 3–5 |
| Lua script size | < 1 KB |

---

## Common Follow-up Questions

**Q: How do you handle rate limiting for unauthenticated requests?**
Fall back to IP address as the identity. Group IP ranges (CIDR /24) if a single IP
represents a large NAT (e.g. a corporate office). Be aware that IPv6 clients may have
unique IPs per request — limit on /48 prefix for IPv6.

**Q: Can you rate limit without Redis?**
Yes: in a single-process service, an in-memory token bucket is fine. For distributed
systems, you need shared state. Alternatives to Redis: Memcached (less atomic), a
dedicated rate limit service, or approximate client-side limiting (good enough for
some ML inference APIs).

**Q: How do you test that the Lua script is correct under concurrency?**
Use `redis-cli --pipe` or a test harness that fires N goroutines simultaneously
against a test Redis. Assert that exactly `limit` requests were allowed, no more.
Unit-test the Lua logic in isolation using `redis.call` mock.

**Q: What is the difference between rate limiting and throttling?**
Rate limiting enforces a quota over time (1000 req/hour). Throttling controls
throughput (max 100 req/sec) — it smooths bursts. Token bucket does both: the bucket
capacity controls burst, the refill rate controls sustained throughput.

**Q: How do you handle a celebrity user with huge traffic without overwhelming their Redis shard?**
Use local counters with periodic Redis sync (option 2 in the hotspot section above).
Accept the trade-off: up to `sync_interval × n_servers` extra requests may slip through.
For payment-critical endpoints, accept the hot key and add Redis Cluster read replicas.
