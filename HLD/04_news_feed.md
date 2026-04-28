# HLD — News Feed / Social Feed (Twitter/Instagram)

## Problem Statement

Design the home feed for a social platform with 500M daily active users. When a user
opens the app, they see a personalized, ranked list of posts from accounts they follow.
The feed must load in under 200ms, stay fresh as new posts appear, and handle the
extreme fan-out of celebrity accounts with 10M+ followers — without those accounts
making feed generation unbearably slow or expensive for everyone.

---

## Requirements

**Functional**
- Users follow other users (directed graph)
- Home feed shows posts from followed accounts, ranked (not pure chronological)
- Infinite scroll via cursor-based pagination
- New posts appear in feed within a few seconds (near-real-time)
- Supports text posts, images, videos (media referenced by URL, not stored in feed service)
- Feed can be filtered (e.g., hide reposts)

**Non-functional**
- 500M DAU
- 500K posts created/sec at peak (write spike from live events, time zones)
- Feed load latency: p99 < 200ms
- Feed staleness: new posts appear within 5 seconds for normal users
- Storage: posts are hot for ~7 days, warm for 30 days, cold archived after
- Availability > 99.99% — feed is the core product

**Out of scope**
- Post creation and media upload pipeline
- Search and discovery (Explore tab)
- Notifications for new activity (separate service)
- Ads injection (ads service overlays on top of organic feed)

---

## Core Concepts / Design Decisions

### Fan-out on Write vs Fan-out on Read vs Hybrid

This is the central design decision. Understanding all three — and when each breaks —
is what separates a senior-level answer from a junior one.

| Strategy | How it works | Pros | Cons |
|---|---|---|---|
| **Fan-out on Write** (push model) | When Alice posts, immediately write Alice's post ID into every follower's feed list | Feed reads are O(1) — pre-computed, just fetch the list | Write amplification: 1 post × 10M followers = 10M writes. Celebrity with 10M followers + 500K posts/sec = catastrophic |
| **Fan-out on Read** (pull model) | When Bob loads his feed, query every account Bob follows, merge and sort their recent posts | No write amplification; always fresh | Read is O(following_count) merges at request time. If Bob follows 5K accounts: 5K DB queries or a huge merge — too slow |
| **Hybrid** | Fan-out on write for normal users (< 10K followers). Fan-out on read for celebrities (> 10K followers). Merge at read time | Eliminates the write amplification problem for celebrities | Read path must handle the merge; extra complexity |

**Decision:** Hybrid. This is what Twitter, Instagram, and Facebook all use in practice.

### What counts as a "celebrity"?

Threshold is configurable — typically 10K–50K followers. At account creation, follower
count is 0 (regular user). When an account crosses the threshold, a backfill job
switches it to "celebrity mode" (removes their posts from pre-built feeds, marks
account as pull-on-read). The threshold matters only for write fan-out; reads always
merge the two lists.

---

## Architecture Diagram

```
User posts                    User loads home feed
     │                               │
     ▼                               ▼
[Post Service]                [Feed Read Service]
     │                               │
     ├──► store post in           ┌──┴─────────────────────────────┐
     │    [Post DB]               │                                │
     │    (sharded by user_id)    │   1. Fetch pre-built feed      │
     │                            │      from Redis sorted set     │
     │                            │      (fan-out-on-write users)  │
     └──► [Fan-out Service]       │                                │
               │                  │   2. Pull recent posts of      │
               │                  │      celebrity followees        │
               │                  │      from Post DB              │
    ┌──────────┤                  │                                │
    │          │                  │   3. Merge + rank              │
    │          │                  │                                │
    │  regular │ celebrity        │   4. Return page of post IDs   │
    │  (<10K   │ (>10K            │                                │
    │  followers)│followers)      └────────────────────────────────┘
    │          │                               │
    ▼          ▼                               ▼
[Fan-out   [Skip —                    [Post Hydration Service]
 Workers]   no pre-                    Batch fetch post content,
    │        built feed]               user info, media URLs
    │                                  from cache → DB
    │  write post_id to
    │  each follower's
    │  Redis sorted set
    │
    ▼
[Redis Cluster]            [Post DB]           [Graph DB / Follow Table]
 feed:{user_id} →           sharded by          user_id → [followee_ids]
 sorted set of post_ids     user_id             cached in Redis
 scored by rank_score
```

---

## Deep Dives

### Fan-out Service — Write Path for Normal Users

When a post is created, the Fan-out Service receives a Kafka event and distributes
the post to follower feeds.

```python
def fan_out(post_id: int, author_id: int, post_timestamp: int):
    # Skip if author is a celebrity — they use pull-on-read
    if author.follower_count > CELEBRITY_THRESHOLD:
        return

    follower_ids = graph_service.get_followers(author_id)  # paginated

    BATCH_SIZE = 1000
    for batch in chunks(follower_ids, BATCH_SIZE):
        for follower_id in batch:
            key = f"feed:{follower_id}"
            score = rank_score(post_id, post_timestamp, author_id, follower_id)
            redis.zadd(key, {post_id: score})
            redis.zremrangebyrank(key, 0, -(MAX_FEED_SIZE + 1))  # cap at 1000 posts
        time.sleep(0.001)  # throttle to avoid Redis hot write spike
```

`ZADD` with a score is what makes Redis sorted sets ideal here. The score can encode:
- Pure timestamp (chronological feed): `score = post_timestamp`
- Ranked feed: `score = ml_rank_score(engagement_signals, recency, author_affinity)`

`ZREMRANGEBYRANK` keeps only the most recent/highest-ranked 1,000 posts per user.
Older posts are loaded on-demand from Post DB if the user scrolls far enough.

Fan-out workers autoscale on Kafka consumer lag. At 500K posts/sec × avg 500
followers = 250M Redis writes/sec — requires horizontal scaling of workers and
Redis shards. In practice, the burst is much smaller (not all posts are from accounts
with 500 followers, and real peaks are shorter).

### Feed Read Service — Merging Write + Pull

```python
def get_feed(user_id: int, cursor: str, limit: int = 20) -> FeedPage:
    # 1. Fetch pre-built feed from Redis (posts from regular followees)
    feed_key   = f"feed:{user_id}"
    cursor_val = decode_cursor(cursor)

    pre_built = redis.zrangebyscore(
        feed_key,
        min=cursor_val,
        max="+inf",
        withscores=True,
        start=0,
        num=limit * 2   # fetch extra to account for removed posts
    )

    # 2. Identify celebrity followees and pull their recent posts
    celebrity_ids = graph_service.get_celebrity_followees(user_id)
    celebrity_posts = []
    for celeb_id in celebrity_ids:
        posts = post_db.get_recent_posts(celeb_id, since=cursor_val, limit=limit)
        celebrity_posts.extend(posts)

    # 3. Merge and re-rank
    all_posts = pre_built + celebrity_posts
    ranked    = rank_posts(all_posts, user_id)

    # 4. Return first `limit` posts, generate next cursor
    page      = ranked[:limit]
    next_cursor = encode_cursor(ranked[limit - 1].score if len(ranked) > limit else None)

    return FeedPage(posts=page, next_cursor=next_cursor)
```

The merge step is an in-memory k-way merge (like merging sorted arrays). If a user
follows 10 celebrities, that's 10 small sorted lists — trivially merged in microseconds.

### Redis Sorted Set for Feed Storage

```
Key:   feed:{user_id}
Type:  Sorted Set
Score: rank_score (float) — higher = shown first
Member: post_id (integer stored as string)

Commands:
  ZADD feed:42 1714521600.89 "post:9876"   # add/update post
  ZREVRANGEBYSCORE feed:42 +inf -inf       # get all posts (newest first)
  ZREVRANGEBYSCORE feed:42 (cursor -inf LIMIT 0 20   # cursor pagination
  ZREMRANGEBYRANK feed:42 0 -1001          # keep only top 1000
  ZCARD feed:42                            # count posts in feed
```

Memory estimate per user:
- Sorted set entry: ~80 bytes (score 8 bytes + member 20 bytes + pointers ~50 bytes)
- 1,000 posts per user × 80 bytes = 80 KB per user
- 500M DAU × 80 KB = 40 TB of Redis — too large for all users

**Solution:** Only cache feeds for **active users** (logged in within the last 7 days).
Roughly 30% of DAU have feeds in Redis = 150M users × 80 KB = 12 TB.
Use Redis Cluster with 24-node cluster × 512 GB RAM = 12 TB capacity — feasible.

For users with a cold/empty feed (new user, or cache expired), run a **feed bootstrap**:
pull recent posts from all followees and build the feed on first load. Cache for 7 days.

### Cursor-based Pagination

Do not use offset-based pagination (`LIMIT 20 OFFSET 40`). Offset pagination breaks
when new posts arrive (items shift, causing duplicates or gaps).

```
Cursor encodes the score (rank value) of the last item on the page:
  cursor = base64({"score": 1714521600.89, "post_id": 9876})

Next page fetch:
  ZREVRANGEBYSCORE feed:42 (1714521600.89 -inf LIMIT 0 20
  (exclusive lower bound = items strictly less than cursor score)

If two posts have the same score (tie in ranking):
  cursor = base64({"score": 1714521600.89, "post_id": 9876})
  filter: post_id < 9876 on tie-breaking
```

The `(` prefix in Redis makes the bound exclusive — critical for correct cursor behavior.

### Handling Hot Users (Celebrities) — Deep Dive

The problem: Lady Gaga has 50M followers. She posts. Fan-out service would need to
write 50M Redis entries. At 100µs per write: 5,000 seconds — clearly wrong.

**Solution already covered:** celebrities skip fan-out on write. Their posts are pulled
at read time. But what about celebrities who follow other celebrities? Bob (regular
user) follows 200 people: 190 regular + 10 celebrities. His feed read:
- Fetch pre-built feed (from the 190 regular followees' fan-out) from Redis: 1 call
- Fetch recent posts from 10 celebrities: 10 Post DB calls (or 1 batch call)
- Merge and rank: in-memory, trivial

The read cost is proportional to the number of celebrity followees, which is bounded
in practice (users follow a handful of celebrities, not thousands).

**Celebrity post caching:** Celebrity posts are read by millions of users. Cache the
most recent 100 posts per celebrity account in Redis:

```
Key:   posts:celeb:{celeb_user_id}
Type:  Sorted Set, scored by timestamp
TTL:   10 minutes (celebrities post infrequently)
```

This means even 50M users loading a celebrity's posts hit Redis, not the Post DB.

### Feed Ranking — Chronological vs ML Score

| Mode | Score | Pros | Cons |
|---|---|---|---|
| **Chronological** | unix timestamp | Predictable, users trust it | Bursty accounts drown out quieter ones |
| **ML Ranked** | model score (0.0–1.0) | Maximizes engagement | Users can't predict order; can create filter bubbles |
| **Hybrid** | `recency_weight × timestamp + engagement_weight × model_score` | Balances freshness and relevance | Two systems to maintain |

Twitter/X, Instagram, and LinkedIn all use ML-ranked feeds as default, with the option
to switch to chronological.

```python
def rank_score(post_id: int, timestamp: int, author_id: int, viewer_id: int) -> float:
    recency   = timestamp / MAX_TIMESTAMP  # normalized 0-1
    affinity  = social_graph.author_affinity(viewer_id, author_id)  # 0-1
    virality  = engagement_service.get_score(post_id)  # likes/shares normalized

    return 0.4 * recency + 0.4 * affinity + 0.2 * virality
```

In production this is a trained ML model (two-tower neural network for affinity,
LightGBM for ranking), not a hand-tuned formula. The score is computed offline and
stored with the post, not recomputed per user per load.

### Media Storage — CDN + Object Store

Feed service returns post IDs and metadata only — never raw media bytes.

```
Post object returned by feed:
{
    "post_id": 9876,
    "author":  { "user_id": 42, "username": "alice", "avatar_url": "https://cdn.example.com/avatars/42.jpg" },
    "text":    "Hello world",
    "media":   [
        { "type": "image", "url": "https://cdn.example.com/media/abc123_1080w.jpg" },
        { "type": "image", "url": "https://cdn.example.com/media/abc123_360w.jpg" }
    ],
    "created_at": 1714521600
}
```

Media pipeline (not the feed's responsibility, but relevant to complete the picture):
- Upload → Object Store (S3 / GCS) → transcoding job → multiple resolutions
- CDN (CloudFront / Akamai) sits in front of the object store
- CDN URL is stored with the post at creation time — feed service just returns it
- CDN handles the global edge caching; popular media served at <10ms from edge

### Post Hydration

Feed read service returns a list of post IDs and scores. A separate hydration step
fetches the actual post content, author info, and media URLs.

```python
def hydrate_posts(post_ids: list[int]) -> list[Post]:
    # Batch fetch from Redis cache first
    cached  = redis.mget([f"post:{pid}" for pid in post_ids])
    missing = [post_ids[i] for i, v in enumerate(cached) if v is None]

    # Fetch missing posts from Post DB (single query, not N queries)
    if missing:
        db_posts = post_db.batch_get(missing)
        for post in db_posts:
            redis.setex(f"post:{post.id}", 3600, serialize(post))

    return merge_and_order(cached, db_posts, post_ids)
```

This is a classic cache-aside batch pattern. Never do N individual DB calls for N
post IDs on a single feed page — always batch.

---

## Scale Numbers

| Metric | Value |
|---|---|
| DAU | 500M |
| Posts created/sec (peak) | 500K |
| Feed loads/sec (500M × 5 loads/day / 86400) | ~29,000/sec |
| Fan-out writes/sec (500K posts × avg 300 followers, 20% celebrity-filtered) | ~120M Redis writes/sec peak |
| Active user feeds in Redis | 150M |
| Redis memory for feeds (150M × 80 KB) | ~12 TB |
| Redis cluster nodes (512 GB RAM each) | 24 |
| Celebrity threshold (follower count) | 10K–50K |
| Max feed size per user (Redis) | 1,000 posts |
| Post cache TTL in Redis | 1 hour |
| Feed cache TTL (inactive user eviction) | 7 days |
| Post DB shards | 128 (sharded by user_id) |
| CDN cache hit rate (popular media) | > 99% |
| Feed load p99 latency target | < 200ms |

---

## Common Follow-up Questions

**Q: How do you handle a user with 500K following (someone who follows everyone)?**
Their read path merges posts from 500K followees — even 1 recent post per followee
is 500K items to sort. Cap the number of followees shown in the feed (e.g. the 1,000
accounts the user interacted with most recently). Or enforce a max-following limit
(Twitter has a 5,000 default limit). Alternatively, pre-compute the merge offline
for power-users.

**Q: What happens when a celebrity creates a new post — how soon do followers see it?**
Since celebrities use fan-on-read, the next feed load after the post is created will
include it — typically within seconds of the page refresh or app open. There is no
fan-out delay. The celebrity's recent posts are cached in Redis (`posts:celeb:{id}`)
with a short TTL (10 min), so the first load after a new post flushes the cache
and fetches the new post from DB.

**Q: How do you switch a user from regular to celebrity mode?**
A background job monitors follower counts. When user X crosses the threshold:
1. Mark X as "celebrity" in the user metadata store.
2. The fan-out service reads this flag — future posts skip fan-out.
3. Optionally, run a cleanup job to remove X's past posts from all pre-built feeds
   (or leave them; they'll age out of the sorted set naturally within 1,000-item cap).

**Q: How do you keep the feed fresh without polling?**
Server-Sent Events (SSE) or WebSocket connection from the mobile client to a
presence/push gateway. When a new post is fanned-out to a user's Redis sorted set,
emit a lightweight "feed updated" push notification. The client then pulls the latest
feed items. Do not push full post content over the WebSocket — just a signal to
trigger a pull.

**Q: How do you paginate correctly when new posts arrive between pages?**
Cursor-based pagination with a score: the cursor encodes the score of the last item
seen. New posts added above the cursor (higher score) don't affect retrieval of posts
below the cursor. The user may see a "X new posts — tap to refresh" banner, and
refreshing resets the cursor to the top of the feed.

**Q: How would you add a "Stories" feature without redesigning the feed?**
Stories are a separate data type with a 24h TTL. They live in their own Redis sorted
set per user, keyed by expiry time. The feed service fetches stories from a separate
endpoint that the client stitches into the top of the UI. Stories don't go through
the feed ranking — they're always shown first and expire on their own schedule.
