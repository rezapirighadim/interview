# LLD — Hotel Booking System

## Problem Statement

Design a hotel booking system where users can search for available rooms by date range and
type, make a reservation, cancel it, and pay. The hotel has different room categories with
different pricing. The system should prevent double-booking and support flexible pricing
rules (weekend surcharge, season pricing).

---

## Requirements

**Functional**
- Search available rooms by check-in/check-out date and room type
- Book a room (creates a reservation with a unique ID)
- Cancel a reservation (before check-in)
- Calculate the total price (base rate × nights × pricing modifier)
- View booking history for a user
- A room cannot be double-booked for overlapping dates

**Non-functional**
- Pricing rules must be swappable without touching booking logic
- Booking lifecycle transitions must be explicit and guarded
- System must scale to multiple hotels

---

## Design Patterns Used

| Pattern | Where | Why |
|---|---|---|
| **Strategy** | `PricingStrategy` | Plug in weekend/season/promo pricing without changing `Reservation` |
| **State** | `ReservationStatus` (guarded transitions) | Prevent illegal state changes (cancel an already-cancelled booking) |
| **Repository** | `RoomRepository`, `ReservationRepository` | Abstract storage — swap in-memory with DB without changing business logic |
| **Specification** | `AvailabilitySpec` | Encapsulate the "is room available?" query as a composable rule object |
| **Factory Method** | `ReservationFactory` | Centralize Reservation creation with ID generation and timestamp |

---

## Class Diagram

```
Hotel
  ├── rooms: RoomRepository
  └── reservations: ReservationRepository

Room
  ├── room_number: str
  ├── room_type: RoomType (enum)
  └── base_rate: float

Reservation
  ├── reservation_id: str
  ├── user: User
  ├── room: Room
  ├── check_in: date
  ├── check_out: date
  ├── status: ReservationStatus (enum + transitions)
  └── total_price: float

PricingStrategy (interface)
  ├── BasePricing
  ├── WeekendPricing
  └── SeasonPricing

RoomRepository (interface)
  └── InMemoryRoomRepository

ReservationRepository (interface)
  └── InMemoryReservationRepository

AvailabilitySpec
  └── is_satisfied_by(room, check_in, check_out, reservations) → bool
```

---

## Clean Code Principles Applied

- **Repository Pattern:** Business logic never queries raw data — it calls `repo.find_available()`.
  Switching from in-memory to PostgreSQL only changes the repository class.
- **Specification Pattern:** Availability rule is isolated, readable, and composable.
  You can combine `AvailabilitySpec AND RoomTypeSpec AND MaxPriceSpec`.
- **Guard Clauses in State Transitions:** `reservation.cancel()` raises `IllegalTransitionError`
  if the booking is already checked-in or cancelled. No silent failures.
- **Value Objects:** `DateRange` is immutable and validates that check-out > check-in.
- **Factory:** `ReservationFactory.create()` is the one place where IDs and timestamps are assigned.

---

## Python Implementation

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import date, datetime, timedelta
from enum import Enum, auto
from typing import Optional
import uuid


# ─── Value Objects ────────────────────────────────────────────────────────────

@dataclass(frozen=True)
class DateRange:
    check_in:  date
    check_out: date

    def __post_init__(self):
        if self.check_out <= self.check_in:
            raise ValueError("check_out must be after check_in")

    @property
    def nights(self) -> int:
        return (self.check_out - self.check_in).days

    def overlaps(self, other: "DateRange") -> bool:
        return self.check_in < other.check_out and other.check_in < self.check_out


# ─── Enums ────────────────────────────────────────────────────────────────────

class RoomType(Enum):
    STANDARD = auto()
    DELUXE   = auto()
    SUITE    = auto()


class ReservationStatus(Enum):
    PENDING    = auto()
    CONFIRMED  = auto()
    CANCELLED  = auto()
    CHECKED_IN = auto()
    COMPLETED  = auto()


VALID_TRANSITIONS = {
    ReservationStatus.PENDING:    {ReservationStatus.CONFIRMED, ReservationStatus.CANCELLED},
    ReservationStatus.CONFIRMED:  {ReservationStatus.CHECKED_IN, ReservationStatus.CANCELLED},
    ReservationStatus.CHECKED_IN: {ReservationStatus.COMPLETED},
    ReservationStatus.CANCELLED:  set(),
    ReservationStatus.COMPLETED:  set(),
}


# ─── Exceptions ───────────────────────────────────────────────────────────────

class RoomNotAvailableError(Exception): pass
class IllegalTransitionError(Exception): pass
class ReservationNotFoundError(Exception): pass


# ─── Domain Models ────────────────────────────────────────────────────────────

@dataclass(frozen=True)
class User:
    user_id: str
    name:    str
    email:   str


@dataclass
class Room:
    room_number: str
    room_type:   RoomType
    base_rate:   float   # per night

    def __repr__(self) -> str:
        return f"Room({self.room_number}, {self.room_type.name}, ${self.base_rate}/night)"


@dataclass
class Reservation:
    reservation_id: str
    user:           User
    room:           Room
    date_range:     DateRange
    total_price:    float
    status:         ReservationStatus = ReservationStatus.PENDING
    created_at:     datetime          = field(default_factory=datetime.now)

    def transition_to(self, new_status: ReservationStatus) -> None:
        allowed = VALID_TRANSITIONS.get(self.status, set())
        if new_status not in allowed:
            raise IllegalTransitionError(
                f"Cannot transition from {self.status.name} to {new_status.name}"
            )
        self.status = new_status

    def confirm(self)    -> None: self.transition_to(ReservationStatus.CONFIRMED)
    def cancel(self)     -> None: self.transition_to(ReservationStatus.CANCELLED)
    def check_in(self)   -> None: self.transition_to(ReservationStatus.CHECKED_IN)
    def check_out(self)  -> None: self.transition_to(ReservationStatus.COMPLETED)

    def __repr__(self) -> str:
        return (f"Reservation(id={self.reservation_id[:8]}, room={self.room.room_number}, "
                f"status={self.status.name}, price=${self.total_price:.2f})")


# ─── Strategy Pattern: Pricing ────────────────────────────────────────────────

class PricingStrategy(ABC):
    @abstractmethod
    def calculate(self, room: Room, date_range: DateRange) -> float:
        ...


class BasePricing(PricingStrategy):
    def calculate(self, room: Room, date_range: DateRange) -> float:
        return room.base_rate * date_range.nights


class WeekendPricing(PricingStrategy):
    """Wrap another strategy; add 20% surcharge for each weekend night."""

    def __init__(self, base: PricingStrategy, surcharge: float = 0.20):
        self._base      = base
        self._surcharge = surcharge

    def calculate(self, room: Room, date_range: DateRange) -> float:
        base_total = self._base.calculate(room, date_range)
        weekend_nights = sum(
            1 for i in range(date_range.nights)
            if (date_range.check_in + timedelta(days=i)).weekday() >= 5
        )
        return base_total + (room.base_rate * self._surcharge * weekend_nights)


# ─── Specification Pattern: Availability ─────────────────────────────────────

class AvailabilitySpec:
    """Encapsulates the 'is this room free?' business rule."""

    def is_satisfied_by(
        self,
        room: Room,
        date_range: DateRange,
        existing_reservations: list[Reservation],
    ) -> bool:
        return all(
            res.room.room_number != room.room_number
            or res.status == ReservationStatus.CANCELLED
            or not res.date_range.overlaps(date_range)
            for res in existing_reservations
        )


# ─── Repository Pattern ───────────────────────────────────────────────────────

class RoomRepository(ABC):
    @abstractmethod
    def find_by_type(self, room_type: RoomType) -> list[Room]: ...
    @abstractmethod
    def find_all(self) -> list[Room]: ...


class InMemoryRoomRepository(RoomRepository):
    def __init__(self, rooms: list[Room]):
        self._rooms = rooms

    def find_by_type(self, room_type: RoomType) -> list[Room]:
        return [r for r in self._rooms if r.room_type == room_type]

    def find_all(self) -> list[Room]:
        return list(self._rooms)


class ReservationRepository(ABC):
    @abstractmethod
    def save(self, reservation: Reservation) -> None: ...
    @abstractmethod
    def find_by_id(self, reservation_id: str) -> Optional[Reservation]: ...
    @abstractmethod
    def find_by_user(self, user_id: str) -> list[Reservation]: ...
    @abstractmethod
    def find_all(self) -> list[Reservation]: ...


class InMemoryReservationRepository(ReservationRepository):
    def __init__(self):
        self._store: dict[str, Reservation] = {}

    def save(self, reservation: Reservation) -> None:
        self._store[reservation.reservation_id] = reservation

    def find_by_id(self, reservation_id: str) -> Optional[Reservation]:
        return self._store.get(reservation_id)

    def find_by_user(self, user_id: str) -> list[Reservation]:
        return [r for r in self._store.values() if r.user.user_id == user_id]

    def find_all(self) -> list[Reservation]:
        return list(self._store.values())


# ─── Factory Method: Reservation ─────────────────────────────────────────────

class ReservationFactory:
    @staticmethod
    def create(
        user: User,
        room: Room,
        date_range: DateRange,
        pricing: PricingStrategy,
    ) -> Reservation:
        return Reservation(
            reservation_id=str(uuid.uuid4()),
            user=user,
            room=room,
            date_range=date_range,
            total_price=pricing.calculate(room, date_range),
        )


# ─── Booking Service (Application Layer) ─────────────────────────────────────

class BookingService:
    def __init__(
        self,
        room_repo:        RoomRepository,
        reservation_repo: ReservationRepository,
        pricing:          PricingStrategy,
    ):
        self._rooms        = room_repo
        self._reservations = reservation_repo
        self._pricing      = pricing
        self._availability = AvailabilitySpec()

    def search_available(self, room_type: RoomType, date_range: DateRange) -> list[Room]:
        candidates    = self._rooms.find_by_type(room_type)
        all_bookings  = self._reservations.find_all()
        return [
            r for r in candidates
            if self._availability.is_satisfied_by(r, date_range, all_bookings)
        ]

    def book(self, user: User, room: Room, date_range: DateRange) -> Reservation:
        all_bookings = self._reservations.find_all()
        if not self._availability.is_satisfied_by(room, date_range, all_bookings):
            raise RoomNotAvailableError(f"{room.room_number} is not available for {date_range}")

        reservation = ReservationFactory.create(user, room, date_range, self._pricing)
        reservation.confirm()
        self._reservations.save(reservation)
        print(f"[Booked] {reservation}")
        return reservation

    def cancel(self, reservation_id: str) -> None:
        reservation = self._reservations.find_by_id(reservation_id)
        if not reservation:
            raise ReservationNotFoundError(reservation_id)
        reservation.cancel()
        print(f"[Cancelled] {reservation}")

    def history(self, user: User) -> list[Reservation]:
        return self._reservations.find_by_user(user.user_id)


# ─── Demo ─────────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    rooms = [
        Room("101", RoomType.STANDARD, 80.0),
        Room("102", RoomType.STANDARD, 80.0),
        Room("201", RoomType.DELUXE,  150.0),
        Room("301", RoomType.SUITE,   300.0),
    ]

    room_repo        = InMemoryRoomRepository(rooms)
    reservation_repo = InMemoryReservationRepository()
    pricing          = WeekendPricing(BasePricing())

    service = BookingService(room_repo, reservation_repo, pricing)

    alice = User("u1", "Alice", "alice@example.com")
    bob   = User("u2", "Bob",   "bob@example.com")

    dr1 = DateRange(date(2025, 8, 1), date(2025, 8, 5))   # 4 nights
    dr2 = DateRange(date(2025, 8, 3), date(2025, 8, 7))   # overlaps with dr1

    available = service.search_available(RoomType.STANDARD, dr1)
    print(f"Available standard rooms: {[r.room_number for r in available]}")

    r1 = service.book(alice, rooms[0], dr1)

    # Bob tries to book the same room on overlapping dates
    try:
        service.book(bob, rooms[0], dr2)
    except RoomNotAvailableError as e:
        print(f"[Error] {e}")

    # Bob books the other standard room instead
    r2 = service.book(bob, rooms[1], dr2)

    service.cancel(r1.reservation_id)

    print("\nAlice's booking history:")
    for res in service.history(alice):
        print(f"  {res}")
```

---

## Node.js Implementation

```javascript
// hotel-booking.js

// ─── Value Objects ────────────────────────────────────────────────────────────

class DateRange {
  constructor(checkIn, checkOut) {
    if (checkOut <= checkIn) throw new Error("checkOut must be after checkIn");
    this.checkIn  = checkIn;
    this.checkOut = checkOut;
    Object.freeze(this);
  }

  get nights() {
    return Math.round((this.checkOut - this.checkIn) / 86_400_000);
  }

  overlaps(other) {
    return this.checkIn < other.checkOut && other.checkIn < this.checkOut;
  }
}

// ─── Enums ────────────────────────────────────────────────────────────────────

const RoomType = Object.freeze({ STANDARD: "STANDARD", DELUXE: "DELUXE", SUITE: "SUITE" });
const ResStatus = Object.freeze({
  PENDING: "PENDING", CONFIRMED: "CONFIRMED",
  CANCELLED: "CANCELLED", CHECKED_IN: "CHECKED_IN", COMPLETED: "COMPLETED",
});
const VALID_TRANSITIONS = {
  [ResStatus.PENDING]:    new Set([ResStatus.CONFIRMED, ResStatus.CANCELLED]),
  [ResStatus.CONFIRMED]:  new Set([ResStatus.CHECKED_IN, ResStatus.CANCELLED]),
  [ResStatus.CHECKED_IN]: new Set([ResStatus.COMPLETED]),
  [ResStatus.CANCELLED]:  new Set(),
  [ResStatus.COMPLETED]:  new Set(),
};

// ─── Domain Models ────────────────────────────────────────────────────────────

class Room {
  constructor(roomNumber, roomType, baseRate) {
    this.roomNumber = roomNumber;
    this.roomType   = roomType;
    this.baseRate   = baseRate;
  }
  toString() { return `Room(${this.roomNumber}, ${this.roomType}, $${this.baseRate}/night)`; }
}

class Reservation {
  #status;
  constructor(id, user, room, dateRange, totalPrice) {
    this.id         = id;
    this.user       = user;
    this.room       = room;
    this.dateRange  = dateRange;
    this.totalPrice = totalPrice;
    this.createdAt  = new Date();
    this.#status    = ResStatus.PENDING;
  }
  get status() { return this.#status; }

  transitionTo(newStatus) {
    if (!VALID_TRANSITIONS[this.#status]?.has(newStatus))
      throw new Error(`Cannot transition from ${this.#status} to ${newStatus}`);
    this.#status = newStatus;
  }
  confirm()   { this.transitionTo(ResStatus.CONFIRMED); }
  cancel()    { this.transitionTo(ResStatus.CANCELLED); }
  checkIn()   { this.transitionTo(ResStatus.CHECKED_IN); }
  checkOut()  { this.transitionTo(ResStatus.COMPLETED); }

  toString() {
    return `Reservation(id=${this.id.slice(0,8)}, room=${this.room.roomNumber}, status=${this.#status}, price=$${this.totalPrice.toFixed(2)})`;
  }
}

// ─── Strategy: Pricing ────────────────────────────────────────────────────────

class BasePricing {
  calculate(room, dateRange) { return room.baseRate * dateRange.nights; }
}

class WeekendPricing {
  constructor(base, surcharge = 0.20) { this._base = base; this._surcharge = surcharge; }
  calculate(room, dateRange) {
    const base = this._base.calculate(room, dateRange);
    let weekendNights = 0;
    for (let i = 0; i < dateRange.nights; i++) {
      const d = new Date(dateRange.checkIn.getTime() + i * 86_400_000);
      if (d.getDay() === 0 || d.getDay() === 6) weekendNights++;
    }
    return base + room.baseRate * this._surcharge * weekendNights;
  }
}

// ─── Specification: Availability ─────────────────────────────────────────────

class AvailabilitySpec {
  isSatisfiedBy(room, dateRange, existingReservations) {
    return existingReservations.every(
      (r) => r.room.roomNumber !== room.roomNumber
          || r.status === ResStatus.CANCELLED
          || !r.dateRange.overlaps(dateRange)
    );
  }
}

// ─── Repositories ─────────────────────────────────────────────────────────────

class InMemoryRoomRepository {
  #rooms;
  constructor(rooms) { this.#rooms = rooms; }
  findByType(type)  { return this.#rooms.filter((r) => r.roomType === type); }
  findAll()         { return [...this.#rooms]; }
}

class InMemoryReservationRepository {
  #store = new Map();
  save(reservation)      { this.#store.set(reservation.id, reservation); }
  findById(id)           { return this.#store.get(id) ?? null; }
  findByUser(userId)     { return [...this.#store.values()].filter((r) => r.user.userId === userId); }
  findAll()              { return [...this.#store.values()]; }
}

// ─── Factory ─────────────────────────────────────────────────────────────────

class ReservationFactory {
  static create(user, room, dateRange, pricing) {
    return new Reservation(
      crypto.randomUUID(),
      user, room, dateRange,
      pricing.calculate(room, dateRange)
    );
  }
}

// ─── Booking Service ──────────────────────────────────────────────────────────

class BookingService {
  #roomRepo; #resRepo; #pricing; #avail = new AvailabilitySpec();

  constructor(roomRepo, resRepo, pricing) {
    this.#roomRepo = roomRepo;
    this.#resRepo  = resRepo;
    this.#pricing  = pricing;
  }

  searchAvailable(roomType, dateRange) {
    const all = this.#resRepo.findAll();
    return this.#roomRepo.findByType(roomType).filter((r) => this.#avail.isSatisfiedBy(r, dateRange, all));
  }

  book(user, room, dateRange) {
    if (!this.#avail.isSatisfiedBy(room, dateRange, this.#resRepo.findAll()))
      throw new Error(`${room.roomNumber} is not available`);
    const reservation = ReservationFactory.create(user, room, dateRange, this.#pricing);
    reservation.confirm();
    this.#resRepo.save(reservation);
    console.log(`[Booked] ${reservation}`);
    return reservation;
  }

  cancel(reservationId) {
    const res = this.#resRepo.findById(reservationId);
    if (!res) throw new Error(`Reservation ${reservationId} not found`);
    res.cancel();
    console.log(`[Cancelled] ${res}`);
  }

  history(user) { return this.#resRepo.findByUser(user.userId); }
}

// ─── Demo ─────────────────────────────────────────────────────────────────────

const rooms = [
  new Room("101", RoomType.STANDARD, 80),
  new Room("102", RoomType.STANDARD, 80),
  new Room("201", RoomType.DELUXE,  150),
];

const service = new BookingService(
  new InMemoryRoomRepository(rooms),
  new InMemoryReservationRepository(),
  new WeekendPricing(new BasePricing())
);

const alice = { userId: "u1", name: "Alice", email: "alice@x.com" };
const bob   = { userId: "u2", name: "Bob",   email: "bob@x.com"   };

const dr1 = new DateRange(new Date("2025-08-01"), new Date("2025-08-05"));
const dr2 = new DateRange(new Date("2025-08-03"), new Date("2025-08-07"));

console.log("Available:", service.searchAvailable(RoomType.STANDARD, dr1).map(r => r.roomNumber));

const r1 = service.book(alice, rooms[0], dr1);

try {
  service.book(bob, rooms[0], dr2);
} catch (e) {
  console.log(`[Error] ${e.message}`);
}

service.book(bob, rooms[1], dr2);
service.cancel(r1.id);
```

---

## Key Takeaways

| Concept | Detail |
|---|---|
| Repository | `BookingService` never touches raw arrays — it talks to `RoomRepository`. Swapping to a DB = rewrite the repo only |
| Specification | `AvailabilitySpec.is_satisfied_by()` reads like a business rule, not a data query. It can be composed: `AvailabilitySpec AND MaxPriceSpec` |
| Value Object (DateRange) | Immutable, self-validating, carries domain behavior (`overlaps`, `nights`). Better than passing two raw `date` args everywhere |
| State Transitions | `VALID_TRANSITIONS` dict makes every legal flow visible in one place. Illegal transitions throw before any damage is done |
| Decorator Pricing | `WeekendPricing(BasePricing())` composes like Lego. Add `SeasonPricing(WeekendPricing(BasePricing()))` without touching any existing class |
