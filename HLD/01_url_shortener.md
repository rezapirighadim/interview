# HLD — URL Shortener (bit.ly)

## Problem Statement

Design a URL shortening service that converts long URLs into short, unique aliases
(e.g. `https://bit.ly/3xK9mQz`) and redirects users to the original URL when the
short link is visited. The system must handle massive read-heavy traffic globally with
low latency.

---

## Requirements

**Functional**
- `POST /shorten` — accept a long URL, return a unique short code (7 chars)
- `GET /{code}` — redirect to the original URL
- Custom aliases (e.g. `bit.ly/my-promo`) — optional, on request
- URL expiry — optional TTL per short link
- Analytics — click count per short URL (total + by day)

**Non-functional**
- 100M URLs created per day → ~1,160 writes/sec
- 10B redirects per day → ~115,000 reads/sec
- Read:write ratio ≈ 100:1 — heavily read-biased
- Redirect latency p99 < 10ms (cache hit path)
- Short codes must be collision-free and non-guessable
- 5 years of data retention → ~180B records total (manageable with sharding)

**Out of scope**
- User accounts and dashboards (UI)
- Link previews / screenshots

---

## Core Concepts / Design Decisions

### Short code generation — base62 vs MD5 vs counter

| Approach | Pros | Cons |
|---|---|---|
| **MD5 + truncate** | Stateless, deterministic for same URL | Collisions at 7 chars; long URL → same code (bad for multi-tenant) |
| **Global counter + base62** | Guaranteed unique, simple | Counter is a single point of failure; sequential codes are guessable |
| **Distributed counter (Snowflake-style)** | Unique, sortable, no SPOF | Complex; slight ordering leak |
| **Random base62 (6-7 chars)** | Non-guessable, simple | Must check DB for collision (rare but possible) |
| **Pre-generated key service** | Zero collision, zero latency for generation | Extra service to operate |

**Decision:** Pre-generated key service (Key Generation Service / KGS). A background
worker pre-generates millions of unique 7-char base62 codes and stores them in a
`keys_available` table. On each write request, the API pops one key atomically. This
completely eliminates collision checking at request time.

Base62 alphabet: `[0-9A-Za-z]` → 62^7 = ~3.5 trillion unique codes. At 100M/day,
that's 95+ years of codes without exhaustion.

```python
import string, random

ALPHABET = string.digits + string.ascii_letters  # 0-9A-Za-z

def generate_code(length: int = 7) -> str:
    return "".join(random.choices(ALPHABET, k=length))

def encode_base62(num: int) -> str:
    code = []
    while num:
        code.append(ALPHABET[num % 62])
        num //= 62
    return "".join(reversed(code)).zfill(7)
```

### 301 vs 302 redirect

| Status | Meaning | Browser caches it? | Use when |
|---|---|---|---|
| **301 Moved Permanently** | Resource is gone for good | Yes — browser never calls server again | Saving bandwidth; analytics not needed |
| **302 Found (Temporary)** | Resource moved temporarily | No — browser always calls server | Need every redirect hit to reach your server for analytics |

**Decision:** Use **302**. We need the redirect request to hit our servers so we can
increment click counters. 301 would hand off caching to the browser and we'd lose
visibility into usage.

### Caching strategy

Redirects are the hot path. A cache-aside pattern with Redis handles the read spike:
- Key: short code → Value: original URL
- TTL: 24h with lazy refresh (or match the link's expiry)
- Cache hit rate target: >99% (URLs follow power-law distribution — top 1% of links
  get 80%+ of traffic)

If cache miss: read from DB, write back to cache, return 302.

---

## Architecture Diagram

```
Client
  │
  ▼
[CDN / Global Load Balancer]  ← Anycast routing, closest PoP
  │
  ├──── POST /shorten ──────────────────────────────────────────┐
  │                                                             │
  ▼                                                             ▼
[API Gateway]                                          [API Gateway]
  │  (rate limiter,                                       │
  │   auth, throttle)                                     │
  │                                                       │
  ▼                                                       ▼
[Write Service]                                    [Redirect Service]
  │                                                       │
  ├── pops key from KGS ────►[Key Generation Service]    ├── L1: local in-process cache (10K hot URLs)
  │                               │ pre-fills             │
  ├── writes to primary DB        │ keys_available        ├── L2: Redis cluster
  │                               │                       │       (100M entries, ~50 bytes each = 5 GB)
  └── writes to Redis cache       ▼                       │
                          [PostgreSQL / MySQL]             └── L3: DB read replicas (shard by code prefix)
                          (primary, sharded)
                                  │
                          [Analytics Service]  ◄── async click event stream
                          (Kafka → Flink → ClickHouse)
                                  │
                          [Analytics DB]
                          (ClickHouse, columnar)
```

---

## Deep Dives

### Key Generation Service (KGS)

KGS runs as a separate service. A background cron generates batches of codes, inserts
them into `keys_available`. Each API server instance holds an in-memory buffer of
~1,000 pre-fetched keys to avoid round-trips per request.

```sql
CREATE TABLE keys_available (
    code     CHAR(7)     PRIMARY KEY,
    status   TINYINT     NOT NULL DEFAULT 0  -- 0=available, 1=used
);

CREATE TABLE keys_used (
    code         CHAR(7)      PRIMARY KEY,
    long_url     TEXT         NOT NULL,
    created_at   TIMESTAMP    NOT NULL,
    expires_at   TIMESTAMP,
    user_id      BIGINT,
    custom_alias BOOLEAN      DEFAULT FALSE
);
```

To pop a key atomically in PostgreSQL:

```sql
WITH picked AS (
    SELECT code FROM keys_available
    WHERE status = 0
    LIMIT 1
    FOR UPDATE SKIP LOCKED
)
UPDATE keys_available
SET status = 1
WHERE code = (SELECT code FROM picked)
RETURNING code;
```

`FOR UPDATE SKIP LOCKED` lets multiple API servers pop keys concurrently without
blocking each other.

### Database Schema and Sharding

```sql
CREATE TABLE urls (
    code         CHAR(7)      NOT NULL,
    long_url     TEXT         NOT NULL,
    user_id      BIGINT,
    created_at   TIMESTAMP    NOT NULL DEFAULT NOW(),
    expires_at   TIMESTAMP,
    PRIMARY KEY  (code)
);

CREATE INDEX idx_urls_user ON urls(user_id, created_at DESC);
```

At 100M URLs/day × 365 × 5 years = 182B rows. Shard on first 2 chars of `code`
(62^2 = 3,844 buckets, map to ~64 physical shards). Alternative: hash(`code`) % N.

Each shard row is ~200 bytes → 182B × 200 = ~36 TB total — fits in 64 shards × 600 GB
each, comfortable for SSDs.

### Redirect Service — Hot Path

```
1. Parse short code from URL path
2. Check in-process LRU cache (capacity 10K, TTL 60s)  — hit rate ~40%
3. Check Redis cluster                                  — hit rate ~55%
4. Read from DB read replica                            — hit rate ~5%
5. Return 302 Location: <long_url>
6. Async: emit click event to Kafka topic "url.clicks"
```

In-process cache is per-instance. Use a consistent hashing approach at the load
balancer to route the same short code to the same redirect server instance —
dramatically improves in-process cache hit rate.

### Analytics Pipeline

Click counting has two tiers:
- **Real-time approximate:** Kafka → streaming consumer increments a Redis counter
  per code per minute. Used for dashboards showing live click counts.
- **Accurate historical:** Kafka → Flink → ClickHouse (columnar OLAP). Used for
  "clicks per day per country/device" reports. ClickHouse can ingest 500K rows/sec
  and query 1B rows in under a second.

```
Kafka topic: url.clicks
  schema: { code, timestamp, ip, user_agent, referrer, country }

Flink job:
  - Tumbling window (1 min): count clicks per code → write to ClickHouse
  - Session window: unique visitors per code
```

Do not write analytics synchronously in the redirect path. Every millisecond matters
for p99 latency.

### Rate Limiting

Two layers:
1. **IP-level**: at API gateway — 100 shortens/IP/hour to prevent abuse
2. **User-level**: authenticated users get higher quota (10K/hour)
3. **Global**: circuit breaker if write service is overloaded

### Cache Invalidation

When a URL is deleted or expires:
1. Mark `expires_at` in DB
2. Delete the Redis key immediately
3. Redirect service checks `expires_at` on DB fallback; returns 410 Gone

---

## Scale Numbers

| Metric | Value |
|---|---|
| URL creates/day | 100M |
| URL creates/sec (peak 3×) | ~3,500/sec |
| Redirects/day | 10B |
| Redirects/sec (peak 3×) | ~350,000/sec |
| Short code space (7 chars base62) | 3.5 trillion |
| Years until code exhaustion | 95+ |
| Redis memory for 100M hot URLs | ~5 GB (50 bytes/entry) |
| DB storage over 5 years | ~36 TB |
| DB shards needed | 64 |
| KGS pre-generation buffer | 1M keys/batch |
| Target redirect p99 latency | <10 ms |
| Target cache hit rate | >99% |

---

## Common Follow-up Questions

**Q: How do you handle custom aliases?**
Custom aliases bypass the KGS. The user provides the code string; we check it doesn't
exist in `keys_used`, then insert it directly. Custom aliases skip the sequential key
pool entirely.

**Q: What if Redis goes down?**
Redirect service falls through to DB read replicas. Latency increases from ~5ms to
~20ms but the service stays up. Replicas are provisioned to handle 100% of redirect
traffic without cache. Redis recovery is fast since we just re-warm from DB reads.

**Q: How do you prevent enumeration (someone scraping all short URLs)?**
Use base62 random codes (not counter-based), rate limit by IP, require login for
bulk access. Counter-based codes like `000001, 000002` are trivially enumerable.

**Q: How does the KGS avoid giving out duplicate codes if it crashes mid-batch?**
The `FOR UPDATE SKIP LOCKED` + `status` flag ensures a code can only be handed out
once. On restart, KGS reads from `keys_available WHERE status=0` — already-used codes
are safely filtered out.

**Q: How do you scale the write path to 3,500/sec?**
Write service is stateless — scale horizontally. DB primary handles 3,500 inserts/sec
trivially (PostgreSQL handles 50K+ simple inserts/sec on good hardware). The KGS key
pop is the only serialized step; mitigate by pre-fetching keys into each API server's
in-memory buffer.

**Q: 301 vs 302 — when would you choose 301?**
If you want to reduce server load and the analytics requirement is dropped, 301 lets
browsers cache the redirect indefinitely. Useful for a link that's permanent and
click-count data isn't needed. In most real systems like bit.ly, 302 is the choice.
