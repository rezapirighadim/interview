# HLD — Notification Service at Scale

## Problem Statement

Design a notification platform that delivers 1B+ notifications per day across push
(mobile), email, and SMS channels. The system must prioritize transactional messages
(OTP, payment receipt) over marketing messages, survive third-party provider outages,
and respect per-user channel preferences and rate limits — all at sub-second delivery
for high-priority notifications.

This document focuses on distributed architecture and scale. For class-level design
(Strategy, Chain of Responsibility, Builder patterns), see `LLD/04_notification_service.md`.

---

## Requirements

**Functional**
- Accept notification requests from any internal service via an event API
- Deliver via: mobile push (APNs / FCM), email (SendGrid / SES), SMS (Twilio / Vonage)
- Priority tiers: CRITICAL (OTP, security alerts) > TRANSACTIONAL (receipts, confirmations) > MARKETING
- Per-user preference service: user can opt out of a channel or a notification type
- Deduplication: same notification must not be delivered twice (idempotency)
- Retry on delivery failure with exponential backoff and a dead letter queue (DLQ)
- Provider failover: if Twilio is down, automatically route SMS through Vonage

**Non-functional**
- 1B notifications/day → ~11,600/sec average, peak ~50,000/sec
- CRITICAL notifications: delivered in < 5 seconds end-to-end
- MARKETING notifications: best effort, < 30 minutes for a blast to 100M users
- At-least-once delivery semantics (idempotency handled by deduplication layer)
- No single provider dependency — all channels have hot standby providers

**Out of scope**
- Notification content management / template CMS (separate service)
- User-facing unsubscribe UI (separate service, writes to preference store)

---

## Core Concepts / Design Decisions

### Event-driven, not request-driven

The caller (Order Service, Auth Service, etc.) publishes an event and moves on.
Notification delivery is asynchronous. The caller never blocks on delivery.

```
# What the Order Service does:
event = {
    "idempotency_key": "order-placed:order_id=789:user_id=42",
    "event_type": "ORDER_PLACED",
    "user_id": 42,
    "priority": "TRANSACTIONAL",
    "payload": { "order_id": "789", "total": "$49.99" },
    "channels": ["push", "email"]
}
kafka.produce(topic="notification.requests", key=str(user_id), value=event)
```

Kafka is the backbone — it buffers spikes, enables replay, and decouples producers
from workers completely.

### One Kafka topic per channel, not per event type

| Topology | Pros | Cons |
|---|---|---|
| Topic per event type | Fine-grained, easy to trace | N topics × 3 channels = too many topics; hard to add channels |
| Topic per channel | Worker pools can specialize; scale each independently | Event metadata must include routing intent |
| Single topic | Simplest | One slow channel (SMS) can back-pressure email delivery |

**Decision:** One topic per channel (`notif.push`, `notif.email`, `notif.sms`).
Within each topic, use Kafka partitions keyed by `user_id % num_partitions` to maintain
per-user ordering. Priority tiers get **separate Kafka topics**: `notif.push.critical`,
`notif.push.transactional`, `notif.push.marketing`. Workers always drain critical before
transactional, transactional before marketing.

---

## Architecture Diagram

```
Internal Services (Order, Auth, Payment, ...)
        │
        │  produce events
        ▼
[Notification API Service]
  ├── validates event schema
  ├── checks idempotency store (Redis SET, 24h TTL)
  ├── fetches user preferences from Preference Service
  ├── filters channels (opt-out, do-not-disturb, rate limit)
  └── routes to Kafka topics by (channel × priority)

        │
    ┌───┴───────────────────────────────────┐
    │                                       │
    ▼                                       ▼
[Kafka Cluster]                        [Kafka Cluster]
 notif.push.critical                    notif.sms.critical
 notif.push.transactional               notif.sms.transactional
 notif.push.marketing                   notif.sms.marketing
 notif.email.*                          ...
    │
    │  consumed by
    ▼
[Push Worker Pool]     [Email Worker Pool]     [SMS Worker Pool]
  K8s Deployment,        K8s Deployment,         K8s Deployment,
  autoscaled by          autoscaled by           autoscaled by
  Kafka consumer         Kafka consumer          Kafka consumer
  lag metric             lag metric              lag metric
    │                        │                       │
    │                        │                       │
    ▼                        ▼                       ▼
[APNs / FCM]           [SendGrid / SES]        [Twilio]
    │  fail                  │  fail                 │  fail
    ▼                        ▼                       ▼
[FCM fallback]         [SES fallback]          [Vonage fallback]
    │                        │                       │
    └───────────────────────►▼◄──────────────────────┘
                      [Delivery Log DB]
                      (write: delivery status, timestamp, provider used)
                      (read: used by support/debugging tools)

    any retry → [DLQ Topic: notif.dlq] → [DLQ Worker] → alerts / manual re-drive
```

---

## Deep Dives

### Notification API Service — Ingestion

The API service is the smart router. It does all enrichment before producing to Kafka,
so workers are dumb and fast.

```python
def ingest(event: NotificationEvent) -> IngestResult:
    # 1. Idempotency check — reject duplicates immediately
    dup_key = f"notif:dedup:{event.idempotency_key}"
    if not redis.set(dup_key, 1, nx=True, ex=86400):
        return IngestResult(status="DUPLICATE_SKIPPED")

    # 2. User preferences
    prefs = preference_service.get(event.user_id)
    active_channels = [
        ch for ch in event.channels
        if prefs.is_opted_in(ch, event.event_type)
        and not prefs.is_do_not_disturb(ch)
    ]
    if not active_channels:
        return IngestResult(status="ALL_CHANNELS_OPTED_OUT")

    # 3. Per-user rate limit check per channel
    for ch in active_channels:
        if not rate_limiter.allow(event.user_id, ch, event.priority):
            active_channels.remove(ch)

    # 4. Produce to Kafka — one message per channel
    for ch in active_channels:
        topic = f"notif.{ch}.{event.priority.lower()}"
        kafka.produce(topic=topic, key=str(event.user_id), value=event)

    return IngestResult(status="ACCEPTED", channels=active_channels)
```

The idempotency key must be set by the caller, not generated internally. The caller
knows what makes an event unique (e.g., `order-placed:{order_id}` — there must be
exactly one such notification regardless of how many times the upstream retries).

### Worker Pool Design

Workers are simple Kafka consumers. They are stateless and horizontally scalable.
Each worker instance processes one Kafka partition at a time (Kafka consumer group
semantics guarantee this).

```python
class PushWorker:
    def __init__(self, providers: list[PushProvider]):
        self.providers = providers  # primary first, fallbacks after

    def process(self, message: KafkaMessage):
        event = NotificationEvent.from_kafka(message)
        payload = template_service.render(event)

        for provider in self.providers:
            try:
                result = provider.send(event.user_id, payload)
                delivery_log.write(event, provider.name, "SUCCESS")
                message.commit()
                return
            except ProviderTemporaryError as e:
                log.warning(f"{provider.name} failed: {e}. Trying next provider.")
                continue
            except ProviderPermanentError as e:
                delivery_log.write(event, provider.name, "PERMANENT_FAILURE")
                message.commit()  # don't retry — bad token, invalid number, etc.
                return

        # All providers failed — send to DLQ
        kafka.produce(topic="notif.dlq", key=message.key, value=message.value)
        message.commit()
```

Key points:
- **Commit only after delivery or DLQ** — Kafka offset commit = acknowledgement.
  Never commit before attempting delivery.
- **Provider failover** is synchronous within the same worker call. No separate
  scheduler needed for the primary → fallback transition.
- **Permanent failures** (invalid APNs token, non-existent phone number) must not
  be retried. The provider's error code distinguishes these from transient failures.

### Retry Strategy with Exponential Backoff

For transient failures that exhaust all providers, the worker sends to DLQ and relies
on a separate DLQ worker for retries. This avoids blocking the main consumer.

```
Retry schedule for TRANSACTIONAL notifications:
  Attempt 1: immediate (in-worker provider failover)
  Attempt 2: 30s delay    (DLQ worker picks up, re-publishes to main topic)
  Attempt 3: 5min delay
  Attempt 4: 30min delay
  Attempt 5: 2hr delay
  → Give up, write FAILED to delivery log, fire alert to on-call

For CRITICAL notifications:
  Attempt 1: immediate
  Attempt 2: 5s
  Attempt 3: 30s
  → Give up after 3 attempts; alert immediately

For MARKETING notifications:
  Attempt 1: immediate
  Attempt 2: 1hr delay
  → Give up; acceptable loss for bulk campaigns
```

The DLQ topic stores the original event plus retry metadata (attempt count, next retry
time). DLQ worker polls and re-injects events into the main topic when `next_retry_time`
has passed.

```
Event in DLQ:
{
  "original_event": { ... },
  "attempt":        2,
  "next_retry_at":  1714521930,
  "last_error":     "Twilio 503: Service Unavailable"
}
```

### Deduplication (Idempotency Keys)

Two levels of deduplication are needed:

1. **Ingestion-level** (API Service): Redis `SET NX` on the `idempotency_key` with 24h
   TTL. Blocks duplicate events from even entering Kafka.

2. **Delivery-level** (Workers): Before calling the provider, check if we already have
   a SUCCESS record in `delivery_log` for this `idempotency_key`. Handles the case
   where the worker delivered successfully but crashed before committing the Kafka offset
   (causing the message to be re-processed).

```python
def is_already_delivered(idempotency_key: str) -> bool:
    return delivery_log.exists(idempotency_key, status="SUCCESS")
```

### User Preference Service

The preference service is in the hot path (called per notification). It must be fast.

```
Architecture:
  - Source of truth: PostgreSQL (user_id, channel, event_type, opted_in, updated_at)
  - Read cache: Redis hash per user_id, TTL 15 minutes
  - Cache invalidation: when user updates preferences, write to DB + delete Redis key

Preference schema:
  user_id    | channel | event_type      | opted_in | do_not_disturb_start | do_not_disturb_end
  42         | push    | ORDER_PLACED    | true     | 22:00                | 08:00
  42         | sms     | MARKETING       | false    | null                 | null
  42         | email   | *               | true     | null                 | null
```

Do-not-disturb (DND) is checked at ingestion time using the user's local timezone.
CRITICAL notifications bypass DND — an OTP cannot wait until 8am.

### Rate Limiting Per User Per Channel

Beyond opt-in preferences, the platform enforces hard limits to prevent spam:

```
Limits enforced at ingestion (API Service):
  MARKETING:     max 3/day/channel/user
  TRANSACTIONAL: max 20/day/channel/user
  CRITICAL:      unlimited (OTPs, security alerts)

Implementation: Redis counter with 24h rolling window (same sliding window technique
as the rate limiter HLD).

Key: "notif:rl:{user_id}:{channel}:{priority}:{day_bucket}"
```

### Provider Failover — Twilio → Vonage

Circuit breaker pattern per provider:

```python
class ProviderCircuitBreaker:
    FAILURE_THRESHOLD = 10      # failures in 60s → open circuit
    RECOVERY_TIMEOUT  = 30      # seconds before trying again (half-open)

    def __init__(self, provider: SmsProvider):
        self.provider = provider
        self.state    = "CLOSED"
        self.failures = 0
        self.opened_at = None

    def send(self, user_id: int, message: str) -> bool:
        if self.state == "OPEN":
            if time.time() - self.opened_at > self.RECOVERY_TIMEOUT:
                self.state = "HALF_OPEN"
            else:
                raise ProviderCircuitOpenError(self.provider.name)

        try:
            result = self.provider.send(user_id, message)
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
                self.failures = 0
            return result
        except Exception:
            self.failures += 1
            if self.failures >= self.FAILURE_THRESHOLD:
                self.state = "OPEN"
                self.opened_at = time.time()
            raise
```

The worker's provider list is `[twilio_breaker, vonage_breaker]`. If Twilio's circuit
is open, it skips immediately to Vonage without a timeout wait.

### Scaling the Marketing Blast (100M users in 30 minutes)

At 100M notifications in 30 minutes, that's ~55,555 notifications/sec. This exceeds
normal throughput. Strategy:

1. **Pre-segment users** — Marketing service pre-computes the target user list hours
   in advance, stores in an S3 file (`users_to_notify_campaign_123.parquet`).

2. **Batch ingestion job** — A Spark/Flink job reads the S3 file, calls the
   Notification API in bulk (not one-by-one), which produces to Kafka.

3. **Autoscale workers** — Kafka consumer lag metric triggers K8s HPA to scale
   from 20 to 200 push worker pods.

4. **Provider rate limits** — FCM and APNs have per-sender rate limits. Use
   multiple sender accounts if needed. Spread send rate over the 30-minute window.

```
Marketing blast flow:
  Campaign Service → S3 (user list) → Batch Ingestion Job → Kafka notif.push.marketing
  → (200 workers) → FCM / APNs → devices
```

Marketing notifications can tolerate being throttled — workers process them at
a controlled rate rather than as fast as possible, to avoid hammering providers.

---

## Scale Numbers

| Metric | Value |
|---|---|
| Notifications/day | 1B |
| Average throughput | 11,600/sec |
| Peak throughput | ~50,000/sec |
| CRITICAL notification e2e latency | < 5 seconds |
| MARKETING blast for 100M users | < 30 minutes |
| Kafka topics | ~12 (3 channels × 3 priorities + DLQ + events) |
| Kafka partitions per topic | 100–200 |
| Worker pods per channel (normal) | 20–50 |
| Worker pods per channel (peak/blast) | 200–500 |
| Deduplication Redis TTL | 24 hours |
| Preference cache (Redis) TTL | 15 minutes |
| Delivery log retention | 90 days |
| DLQ max retry attempts (transactional) | 5 |
| Provider failover SLA (circuit opens) | 10 failures in 60s |

---

## Common Follow-up Questions

**Q: How do you guarantee exactly-once delivery vs at-least-once?**
Exactly-once is not achievable without the provider supporting idempotent endpoints
(most SMS/push providers do not). At-least-once + idempotency key at the delivery
layer gives effectively-once semantics. The only gap is if the provider delivers twice
but reports a failure on the second call — rare enough to accept.

**Q: What happens if the Notification API goes down?**
Callers should use an outbox pattern or retry producing to Kafka. Kafka itself is
highly available (3-way replication). If the API is temporarily unreachable, the
upstream service can queue events locally and retry. For CRITICAL notifications,
the calling service should have a fallback (e.g., Auth Service directly calls
Twilio for OTP if Notification Service is unreachable).

**Q: How do you handle users in different timezones for marketing blasts?**
Store the user's timezone in the preference service. The campaign scheduler creates
sub-batches per timezone and schedules each sub-batch to send at the configured
local time (e.g., 10am local). This increases campaign scheduling complexity
but dramatically improves open rates and reduces complaints.

**Q: How do you monitor delivery health?**
Three key metrics:
1. **Delivery rate**: `delivered / attempted` per channel per priority. Alert if < 95%
2. **E2E latency**: time from Kafka produce to provider ACK. Alert if CRITICAL p99 > 5s
3. **DLQ growth rate**: messages landing in DLQ per minute. Alert if increasing.

All three are Kafka consumer lag + custom metrics emitted by workers → Prometheus → Grafana.

**Q: How is this different from the LLD notification service file?**
The LLD file covers object-oriented design: Strategy pattern for channels, Chain of
Responsibility for pipeline steps, Builder for constructing Notification objects. It
runs in a single process. This HLD file covers distributed systems concerns: Kafka
topology, worker autoscaling, provider failover with circuit breakers, deduplication
across machines, and handling 1B events/day. Both perspectives are needed in a full
system design interview.
