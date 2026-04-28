# LLD — Notification Service

## Problem Statement

Design a notification service that sends messages to users via multiple channels (Email,
SMS, Push). Users can subscribe to specific event types and choose their preferred channels.
The system should support rate limiting, retry logic, and be easily extensible with new
channel types.

---

## Requirements

**Functional**
- Users subscribe to event types (e.g. ORDER_PLACED, PAYMENT_FAILED)
- When an event is published, all subscribed users are notified
- Each user can have multiple preferred channels (Email, SMS, Push)
- Support rate limiting per user per channel
- Support retry with exponential backoff on delivery failure

**Non-functional**
- Adding a new channel (e.g. Slack, WhatsApp) requires zero changes to existing code
- Decouple the event producer from the notification delivery
- Notification sending should be non-blocking (async)

---

## Design Patterns Used

| Pattern | Where | Why |
|---|---|---|
| **Observer** | `EventBus` + `NotificationHandler` | Publishers don't know subscribers; subscribers react to events they care about |
| **Strategy** | `NotificationChannel` | Swap Email/SMS/Push at runtime; adding Slack = new class only |
| **Chain of Responsibility** | `RateLimiter → RetryHandler → ChannelSender` | Each step handles its concern and passes to the next |
| **Builder** | `NotificationBuilder` | Construct complex `Notification` objects step by step without a 10-arg constructor |
| **Template Method** | `BaseChannel.send()` | Shared pre/post logic; subclasses only implement `_deliver()` |

---

## Class Diagram

```
EventBus
  └── subscribers: dict[EventType → List[Subscriber]]
        └── NotificationHandler (Subscriber)
              └── builds Notification via NotificationBuilder
                    └── dispatches to NotificationDispatcher
                          └── pipeline: RateLimiter → RetryHandler → Channel

NotificationChannel (Strategy interface)
  ├── EmailChannel
  ├── SMSChannel
  └── PushChannel

Notification (immutable)
  ├── recipient: User
  ├── event: Event
  ├── channel: NotificationChannel
  └── message: str

User
  ├── user_id: str
  ├── email: str
  ├── phone: str
  └── preferred_channels: List[NotificationChannel]
```

---

## Clean Code Principles Applied

- **Single Responsibility:** `EventBus` only routes events. `RateLimiter` only checks limits.
  `RetryHandler` only handles retries. Each class does one thing.
- **Open/Closed:** `SlackChannel` is a new file, not a modification of any existing file.
- **Dependency Inversion:** `NotificationDispatcher` depends on `NotificationChannel` interface.
  It never imports `EmailChannel` or `SMSChannel` directly.
- **Builder:** `NotificationBuilder` prevents constructors with 6+ positional arguments.
- **Fail Fast + Logging:** Each pipeline step logs clearly before raising or retrying.

---

## Python Implementation

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from enum import Enum, auto
from typing import Callable, Optional
import time
import random


# ─── Domain ───────────────────────────────────────────────────────────────────

class EventType(Enum):
    ORDER_PLACED   = auto()
    PAYMENT_FAILED = auto()
    SHIPMENT_SENT  = auto()
    PROMO          = auto()


@dataclass(frozen=True)
class Event:
    event_type: EventType
    payload:    dict
    occurred_at: datetime = field(default_factory=datetime.now)


@dataclass
class User:
    user_id:           str
    name:              str
    email:             str
    phone:             str
    preferred_channels: list[str] = field(default_factory=list)  # channel names


# ─── Strategy Pattern: Channels ───────────────────────────────────────────────

class NotificationChannel(ABC):
    @property
    @abstractmethod
    def name(self) -> str: ...

    def send(self, user: User, message: str) -> bool:
        """Template Method: shared logging + subclass delivery."""
        print(f"  [{self.name}] → {user.name}: {message[:60]}...")
        return self._deliver(user, message)

    @abstractmethod
    def _deliver(self, user: User, message: str) -> bool:
        """Subclasses implement the actual delivery."""
        ...


class EmailChannel(NotificationChannel):
    @property
    def name(self) -> str: return "Email"

    def _deliver(self, user: User, message: str) -> bool:
        # Simulate occasional failure
        success = random.random() > 0.1
        if success:
            print(f"    ✓ Email sent to {user.email}")
        else:
            print(f"    ✗ Email failed for {user.email}")
        return success


class SMSChannel(NotificationChannel):
    @property
    def name(self) -> str: return "SMS"

    def _deliver(self, user: User, message: str) -> bool:
        success = random.random() > 0.15
        if success:
            print(f"    ✓ SMS sent to {user.phone}")
        else:
            print(f"    ✗ SMS failed for {user.phone}")
        return success


class PushChannel(NotificationChannel):
    @property
    def name(self) -> str: return "Push"

    def _deliver(self, user: User, message: str) -> bool:
        print(f"    ✓ Push notification sent to {user.user_id}")
        return True


# ─── Builder Pattern: Notification ───────────────────────────────────────────

@dataclass(frozen=True)
class Notification:
    recipient: User
    event:     Event
    channel:   NotificationChannel
    message:   str
    created_at: datetime = field(default_factory=datetime.now)


class NotificationBuilder:
    def __init__(self):
        self._recipient: Optional[User]               = None
        self._event:     Optional[Event]              = None
        self._channel:   Optional[NotificationChannel] = None
        self._message:   Optional[str]                = None

    def for_user(self, user: User) -> "NotificationBuilder":
        self._recipient = user
        return self

    def about_event(self, event: Event) -> "NotificationBuilder":
        self._event = event
        return self

    def via_channel(self, channel: NotificationChannel) -> "NotificationBuilder":
        self._channel = channel
        return self

    def with_message(self, message: str) -> "NotificationBuilder":
        self._message = message
        return self

    def build(self) -> Notification:
        if not all([self._recipient, self._event, self._channel, self._message]):
            raise ValueError("Notification is missing required fields")
        return Notification(
            recipient=self._recipient,
            event=self._event,
            channel=self._channel,
            message=self._message,
        )


# ─── Chain of Responsibility: Pipeline ───────────────────────────────────────

class NotificationHandler(ABC):
    def __init__(self, next_handler: Optional["NotificationHandler"] = None):
        self._next = next_handler

    def handle(self, notification: Notification) -> bool:
        if self._next:
            return self._next.handle(notification)
        return True  # end of chain


class RateLimiter(NotificationHandler):
    """Allow max N notifications per user per channel per minute."""

    def __init__(self, max_per_minute: int, next_handler: Optional[NotificationHandler] = None):
        super().__init__(next_handler)
        self._max = max_per_minute
        self._log: dict[str, list[float]] = {}  # key → timestamps

    def handle(self, notification: Notification) -> bool:
        key = f"{notification.recipient.user_id}:{notification.channel.name}"
        now = time.time()
        window_start = now - 60

        timestamps = self._log.get(key, [])
        timestamps = [t for t in timestamps if t > window_start]  # keep last 60s

        if len(timestamps) >= self._max:
            print(f"  [RateLimiter] Blocked: {key} has hit the rate limit")
            return False

        timestamps.append(now)
        self._log[key] = timestamps
        return super().handle(notification)


class RetryHandler(NotificationHandler):
    """Retry delivery up to max_retries times with exponential backoff."""

    def __init__(self, max_retries: int = 3, next_handler: Optional[NotificationHandler] = None):
        super().__init__(next_handler)
        self._max_retries = max_retries

    def handle(self, notification: Notification) -> bool:
        for attempt in range(1, self._max_retries + 1):
            success = super().handle(notification)
            if success:
                return True
            wait = 2 ** attempt * 0.1  # 0.2s, 0.4s, 0.8s
            print(f"  [Retry] Attempt {attempt} failed. Retrying in {wait:.1f}s...")
            time.sleep(wait)
        print(f"  [Retry] All {self._max_retries} attempts failed.")
        return False


class ChannelDelivery(NotificationHandler):
    """Final handler: delegates to the channel's send method."""

    def handle(self, notification: Notification) -> bool:
        return notification.channel.send(notification.recipient, notification.message)


# ─── Observer Pattern: Event Bus ─────────────────────────────────────────────

Subscriber = Callable[[Event, list[User]], None]


class EventBus:
    _instance: Optional["EventBus"] = None

    def __new__(cls) -> "EventBus":
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._subscribers: dict[EventType, list[Subscriber]] = {}
        return cls._instance

    def subscribe(self, event_type: EventType, handler: Subscriber) -> None:
        self._subscribers.setdefault(event_type, []).append(handler)

    def publish(self, event: Event, recipients: list[User]) -> None:
        print(f"\n[EventBus] Publishing: {event.event_type.name}")
        handlers = self._subscribers.get(event.event_type, [])
        for handler in handlers:
            handler(event, recipients)


# ─── Notification Service ─────────────────────────────────────────────────────

class NotificationService:
    def __init__(self, channels: dict[str, NotificationChannel]):
        self._channels = channels
        self._pipeline = RateLimiter(
            max_per_minute=5,
            next_handler=RetryHandler(
                max_retries=2,
                next_handler=ChannelDelivery()
            )
        )

    def _message_for(self, event: Event) -> str:
        templates = {
            EventType.ORDER_PLACED:   "Your order has been placed successfully!",
            EventType.PAYMENT_FAILED: "Payment failed. Please update your payment method.",
            EventType.SHIPMENT_SENT:  "Your order is on its way!",
            EventType.PROMO:          "Check out our latest offers!",
        }
        return templates.get(event.event_type, "You have a new notification.")

    def notify(self, event: Event, recipients: list[User]) -> None:
        message = self._message_for(event)
        for user in recipients:
            for channel_name in user.preferred_channels:
                channel = self._channels.get(channel_name)
                if not channel:
                    continue
                notification = (
                    NotificationBuilder()
                    .for_user(user)
                    .about_event(event)
                    .via_channel(channel)
                    .with_message(message)
                    .build()
                )
                self._pipeline.handle(notification)


# ─── Demo ─────────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    channels = {
        "Email": EmailChannel(),
        "SMS":   SMSChannel(),
        "Push":  PushChannel(),
    }

    service = NotificationService(channels)

    bus = EventBus()
    bus.subscribe(EventType.ORDER_PLACED,   service.notify)
    bus.subscribe(EventType.PAYMENT_FAILED, service.notify)

    users = [
        User("u1", "Alice", "alice@example.com", "+1-555-0001", ["Email", "Push"]),
        User("u2", "Bob",   "bob@example.com",   "+1-555-0002", ["SMS"]),
    ]

    bus.publish(Event(EventType.ORDER_PLACED,   {"order_id": "ORD-001"}), users)
    bus.publish(Event(EventType.PAYMENT_FAILED, {"order_id": "ORD-002"}), users)
```

---

## Node.js Implementation

```javascript
// notification-service.js

const EventTypeEnum = Object.freeze({
  ORDER_PLACED:   "ORDER_PLACED",
  PAYMENT_FAILED: "PAYMENT_FAILED",
  SHIPMENT_SENT:  "SHIPMENT_SENT",
});

// ─── Strategy: Channels ───────────────────────────────────────────────────────

class NotificationChannel {
  get name() { throw new Error("Abstract"); }

  send(user, message) {
    console.log(`  [${this.name}] → ${user.name}: ${message.slice(0, 60)}...`);
    return this._deliver(user, message);
  }

  _deliver(_user, _message) { throw new Error("Abstract"); }
}

class EmailChannel extends NotificationChannel {
  get name() { return "Email"; }
  _deliver(user) {
    const ok = Math.random() > 0.1;
    console.log(ok ? `    ✓ Email sent to ${user.email}` : `    ✗ Email failed`);
    return ok;
  }
}

class SMSChannel extends NotificationChannel {
  get name() { return "SMS"; }
  _deliver(user) {
    const ok = Math.random() > 0.15;
    console.log(ok ? `    ✓ SMS sent to ${user.phone}` : `    ✗ SMS failed`);
    return ok;
  }
}

class PushChannel extends NotificationChannel {
  get name() { return "Push"; }
  _deliver(user) {
    console.log(`    ✓ Push sent to ${user.userId}`);
    return true;
  }
}

// ─── Builder: Notification ────────────────────────────────────────────────────

class NotificationBuilder {
  #recipient = null; #event = null; #channel = null; #message = null;

  forUser(user)       { this.#recipient = user;    return this; }
  aboutEvent(event)   { this.#event     = event;   return this; }
  viaChannel(channel) { this.#channel   = channel; return this; }
  withMessage(msg)    { this.#message   = msg;     return this; }

  build() {
    if (!this.#recipient || !this.#event || !this.#channel || !this.#message)
      throw new Error("Notification missing required fields");
    return Object.freeze({
      recipient: this.#recipient,
      event:     this.#event,
      channel:   this.#channel,
      message:   this.#message,
      createdAt: new Date(),
    });
  }
}

// ─── Chain of Responsibility ──────────────────────────────────────────────────

class NotificationHandlerBase {
  constructor(next = null) { this._next = next; }
  handle(notification) { return this._next ? this._next.handle(notification) : true; }
}

class RateLimiter extends NotificationHandlerBase {
  #maxPerMin; #log = new Map();

  constructor(maxPerMin, next) { super(next); this.#maxPerMin = maxPerMin; }

  handle(notification) {
    const key   = `${notification.recipient.userId}:${notification.channel.name}`;
    const now   = Date.now();
    const times = (this.#log.get(key) ?? []).filter(t => now - t < 60_000);

    if (times.length >= this.#maxPerMin) {
      console.log(`  [RateLimiter] Blocked: ${key}`);
      return false;
    }
    times.push(now);
    this.#log.set(key, times);
    return super.handle(notification);
  }
}

class RetryHandler extends NotificationHandlerBase {
  #maxRetries;
  constructor(maxRetries, next) { super(next); this.#maxRetries = maxRetries; }

  async handle(notification) {
    for (let i = 1; i <= this.#maxRetries; i++) {
      const ok = await super.handle(notification);
      if (ok) return true;
      const wait = 2 ** i * 100;
      console.log(`  [Retry] Attempt ${i} failed. Retrying in ${wait}ms...`);
      await new Promise(r => setTimeout(r, wait));
    }
    console.log(`  [Retry] All ${this.#maxRetries} attempts exhausted.`);
    return false;
  }
}

class ChannelDelivery extends NotificationHandlerBase {
  handle(notification) {
    return notification.channel.send(notification.recipient, notification.message);
  }
}

// ─── Observer: Event Bus ──────────────────────────────────────────────────────

class EventBus {
  static #instance = null;
  #subscribers = new Map();

  static getInstance() {
    if (!EventBus.#instance) EventBus.#instance = new EventBus();
    return EventBus.#instance;
  }

  subscribe(eventType, handler) {
    const handlers = this.#subscribers.get(eventType) ?? [];
    handlers.push(handler);
    this.#subscribers.set(eventType, handlers);
  }

  async publish(event, recipients) {
    console.log(`\n[EventBus] Publishing: ${event.type}`);
    for (const handler of this.#subscribers.get(event.type) ?? [])
      await handler(event, recipients);
  }
}

// ─── Notification Service ─────────────────────────────────────────────────────

const TEMPLATES = {
  [EventTypeEnum.ORDER_PLACED]:   "Your order has been placed successfully!",
  [EventTypeEnum.PAYMENT_FAILED]: "Payment failed. Please update your payment method.",
};

class NotificationService {
  #channels; #pipeline;

  constructor(channels) {
    this.#channels = channels;
    this.#pipeline = new RateLimiter(5, new RetryHandler(2, new ChannelDelivery()));
  }

  async notify(event, recipients) {
    const message = TEMPLATES[event.type] ?? "You have a new notification.";
    for (const user of recipients) {
      for (const channelName of user.preferredChannels) {
        const channel = this.#channels[channelName];
        if (!channel) continue;
        const notification = new NotificationBuilder()
          .forUser(user).aboutEvent(event).viaChannel(channel).withMessage(message).build();
        await this.#pipeline.handle(notification);
      }
    }
  }
}

// ─── Demo ─────────────────────────────────────────────────────────────────────

(async () => {
  const channels = { Email: new EmailChannel(), SMS: new SMSChannel(), Push: new PushChannel() };
  const service  = new NotificationService(channels);

  const bus = EventBus.getInstance();
  bus.subscribe(EventTypeEnum.ORDER_PLACED,   (e, r) => service.notify(e, r));
  bus.subscribe(EventTypeEnum.PAYMENT_FAILED, (e, r) => service.notify(e, r));

  const users = [
    { userId: "u1", name: "Alice", email: "alice@x.com", phone: "+1-555-0001", preferredChannels: ["Email", "Push"] },
    { userId: "u2", name: "Bob",   email: "bob@x.com",   phone: "+1-555-0002", preferredChannels: ["SMS"] },
  ];

  await bus.publish({ type: EventTypeEnum.ORDER_PLACED,   payload: { orderId: "ORD-001" } }, users);
  await bus.publish({ type: EventTypeEnum.PAYMENT_FAILED, payload: { orderId: "ORD-002" } }, users);
})();
```

---

## Key Takeaways

| Concept | Detail |
|---|---|
| Observer (Event Bus) | Producer publishes to a bus; consumers react. Zero coupling between order service and notification service |
| Strategy (Channels) | Adding Slack = `class SlackChannel extends NotificationChannel`. No existing code changes |
| Chain of Responsibility | `RateLimiter → RetryHandler → ChannelDelivery` — each step is composable and testable in isolation |
| Builder | `NotificationBuilder` prevents a fragile 6-argument constructor. Each step is readable and optional |
| Template Method | `BaseChannel.send()` logs before calling `_deliver()` — avoids duplicating logging in every channel |
| Singleton Event Bus | One bus per application — registered subscribers persist across modules |
