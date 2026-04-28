# LLD — Parking Lot

## Problem Statement

Design a multi-floor parking lot that can park cars, motorcycles, and trucks. It should
assign the nearest available spot, track occupancy, and generate a ticket on entry and a
bill on exit.

---

## Requirements

**Functional**
- Support multiple vehicle types: Motorcycle, Car, Truck
- Support multiple spot sizes: Small, Medium, Large
- A vehicle fits only into a spot of the same size or larger
- Issue a parking ticket on entry with timestamp
- Calculate the fee on exit based on time parked
- Track real-time availability per floor and spot type

**Non-functional**
- Thread-safe assignment (multiple entrances)
- Easily extendable pricing strategy
- Open for new vehicle types without modifying existing classes

---

## Design Patterns Used

| Pattern | Where | Why |
|---|---|---|
| **Strategy** | `PricingStrategy` | Swap hourly/flat/weekend pricing without touching the core |
| **Factory Method** | `VehicleFactory` | Decouple vehicle creation from the parking logic |
| **Singleton** | `ParkingLot` | One global instance manages all floors |
| **Observer** | `ParkingObserver` | Notify dashboard/display boards when occupancy changes |
| **Template Method** | `BaseVehicle` | Shared structure, subclasses provide spot size requirement |

---

## Class Diagram

```
ParkingLot (Singleton)
  └── floors: List[ParkingFloor]
        └── spots: List[ParkingSpot]
              ├── SpotSize (enum): SMALL | MEDIUM | LARGE
              └── Vehicle (Abstract)
                    ├── Motorcycle  → needs SMALL
                    ├── Car         → needs MEDIUM
                    └── Truck       → needs LARGE

ParkingTicket
  ├── ticket_id
  ├── vehicle
  ├── spot
  └── entry_time

PricingStrategy (interface)
  ├── HourlyPricing
  └── FlatRatePricing

ParkingObserver (interface)
  └── DisplayBoard
```

---

## Clean Code Principles Applied

- **Single Responsibility:** `ParkingSpot` only knows about occupancy. `PricingStrategy` only
  calculates fees. `ParkingTicket` only holds entry data.
- **Open/Closed:** Adding a `FlatRatePricing` requires zero changes to existing classes.
- **Dependency Inversion:** `ParkingLot` depends on the `PricingStrategy` interface, not a
  concrete implementation.
- **DRY:** Spot search logic lives in one place (`_find_available_spot`).
- **Fail Fast:** Raise meaningful exceptions (`ParkingLotFullError`) instead of returning `None`.

---

## Python Implementation

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum, auto
from threading import Lock
from typing import Optional
import uuid


# ─── Enums ───────────────────────────────────────────────────────────────────

class SpotSize(Enum):
    SMALL = 1
    MEDIUM = 2
    LARGE = 3


class VehicleType(Enum):
    MOTORCYCLE = auto()
    CAR = auto()
    TRUCK = auto()


# ─── Exceptions ──────────────────────────────────────────────────────────────

class ParkingLotFullError(Exception):
    pass


class InvalidTicketError(Exception):
    pass


# ─── Strategy Pattern: Pricing ────────────────────────────────────────────────

class PricingStrategy(ABC):
    @abstractmethod
    def calculate(self, entry: datetime, exit: datetime, spot_size: SpotSize) -> float:
        ...


class HourlyPricing(PricingStrategy):
    RATES = {SpotSize.SMALL: 2.0, SpotSize.MEDIUM: 3.5, SpotSize.LARGE: 5.0}

    def calculate(self, entry: datetime, exit: datetime, spot_size: SpotSize) -> float:
        hours = max(1, (exit - entry).seconds // 3600)
        return hours * self.RATES[spot_size]


class FlatRatePricing(PricingStrategy):
    def calculate(self, entry: datetime, exit: datetime, spot_size: SpotSize) -> float:
        return 10.0  # flat daily rate regardless of time


# ─── Observer Pattern ─────────────────────────────────────────────────────────

class ParkingObserver(ABC):
    @abstractmethod
    def on_spot_changed(self, floor: int, available: int, total: int) -> None:
        ...


class DisplayBoard(ParkingObserver):
    def on_spot_changed(self, floor: int, available: int, total: int) -> None:
        print(f"[Display] Floor {floor}: {available}/{total} spots available")


# ─── Domain Models ────────────────────────────────────────────────────────────

class Vehicle(ABC):
    def __init__(self, license_plate: str, vehicle_type: VehicleType):
        self.license_plate = license_plate
        self.vehicle_type = vehicle_type

    @property
    @abstractmethod
    def required_spot_size(self) -> SpotSize:
        ...

    def __repr__(self) -> str:
        return f"{self.vehicle_type.name}({self.license_plate})"


class Motorcycle(Vehicle):
    def __init__(self, license_plate: str):
        super().__init__(license_plate, VehicleType.MOTORCYCLE)

    @property
    def required_spot_size(self) -> SpotSize:
        return SpotSize.SMALL


class Car(Vehicle):
    def __init__(self, license_plate: str):
        super().__init__(license_plate, VehicleType.CAR)

    @property
    def required_spot_size(self) -> SpotSize:
        return SpotSize.MEDIUM


class Truck(Vehicle):
    def __init__(self, license_plate: str):
        super().__init__(license_plate, VehicleType.TRUCK)

    @property
    def required_spot_size(self) -> SpotSize:
        return SpotSize.LARGE


# ─── Factory Pattern: Vehicle Creation ───────────────────────────────────────

class VehicleFactory:
    _creators = {
        VehicleType.MOTORCYCLE: Motorcycle,
        VehicleType.CAR: Car,
        VehicleType.TRUCK: Truck,
    }

    @staticmethod
    def create(vehicle_type: VehicleType, license_plate: str) -> Vehicle:
        creator = VehicleFactory._creators.get(vehicle_type)
        if not creator:
            raise ValueError(f"Unknown vehicle type: {vehicle_type}")
        return creator(license_plate)


# ─── Parking Spot ─────────────────────────────────────────────────────────────

@dataclass
class ParkingSpot:
    spot_id: str
    floor: int
    size: SpotSize
    vehicle: Optional[Vehicle] = field(default=None)

    @property
    def is_available(self) -> bool:
        return self.vehicle is None

    def park(self, vehicle: Vehicle) -> None:
        if not self.is_available:
            raise ValueError(f"Spot {self.spot_id} is already occupied")
        self.vehicle = vehicle

    def vacate(self) -> None:
        self.vehicle = None

    def can_fit(self, vehicle: Vehicle) -> bool:
        # A spot can fit a vehicle if the spot is >= the vehicle's required size
        return self.is_available and self.size.value >= vehicle.required_spot_size.value


# ─── Parking Ticket ───────────────────────────────────────────────────────────

@dataclass
class ParkingTicket:
    ticket_id: str
    vehicle: Vehicle
    spot: ParkingSpot
    entry_time: datetime = field(default_factory=datetime.now)

    @staticmethod
    def generate(vehicle: Vehicle, spot: ParkingSpot) -> "ParkingTicket":
        return ParkingTicket(
            ticket_id=str(uuid.uuid4())[:8].upper(),
            vehicle=vehicle,
            spot=spot,
        )


# ─── Parking Floor ────────────────────────────────────────────────────────────

class ParkingFloor:
    def __init__(self, floor_number: int, spots: list[ParkingSpot]):
        self.floor_number = floor_number
        self._spots = spots

    def find_available_spot(self, vehicle: Vehicle) -> Optional[ParkingSpot]:
        return next((s for s in self._spots if s.can_fit(vehicle)), None)

    @property
    def available_count(self) -> int:
        return sum(1 for s in self._spots if s.is_available)

    @property
    def total_count(self) -> int:
        return len(self._spots)


# ─── Singleton Pattern: Parking Lot ──────────────────────────────────────────

class ParkingLot:
    _instance: Optional["ParkingLot"] = None
    _lock = Lock()

    def __new__(cls) -> "ParkingLot":
        with cls._lock:
            if cls._instance is None:
                cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self):
        if hasattr(self, "_initialized"):
            return
        self._initialized = True
        self._floors: list[ParkingFloor] = []
        self._active_tickets: dict[str, ParkingTicket] = {}
        self._pricing: PricingStrategy = HourlyPricing()
        self._observers: list[ParkingObserver] = []
        self._op_lock = Lock()

    def configure(self, floors: list[ParkingFloor], pricing: PricingStrategy) -> None:
        self._floors = floors
        self._pricing = pricing

    def add_observer(self, observer: ParkingObserver) -> None:
        self._observers.append(observer)

    def _notify_observers(self, floor: ParkingFloor) -> None:
        for obs in self._observers:
            obs.on_spot_changed(floor.floor_number, floor.available_count, floor.total_count)

    def park(self, vehicle: Vehicle) -> ParkingTicket:
        with self._op_lock:
            for floor in self._floors:
                spot = floor.find_available_spot(vehicle)
                if spot:
                    spot.park(vehicle)
                    ticket = ParkingTicket.generate(vehicle, spot)
                    self._active_tickets[ticket.ticket_id] = ticket
                    self._notify_observers(floor)
                    print(f"[Entry] {vehicle} parked at Floor {spot.floor}, "
                          f"Spot {spot.spot_id}. Ticket: {ticket.ticket_id}")
                    return ticket
            raise ParkingLotFullError(f"No available spot for {vehicle}")

    def exit(self, ticket_id: str) -> float:
        with self._op_lock:
            ticket = self._active_tickets.pop(ticket_id, None)
            if not ticket:
                raise InvalidTicketError(f"Ticket {ticket_id} not found")

            exit_time = datetime.now()
            fee = self._pricing.calculate(ticket.entry_time, exit_time, ticket.spot.size)
            ticket.spot.vacate()

            # find the floor to notify observers
            for floor in self._floors:
                if ticket.spot.floor == floor.floor_number:
                    self._notify_observers(floor)
                    break

            print(f"[Exit] {ticket.vehicle} — Duration: "
                  f"{(exit_time - ticket.entry_time).seconds}s — Fee: ${fee:.2f}")
            return fee


# ─── Demo ─────────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    # Build spots: floor 1 has 2 small, 2 medium, 1 large
    spots_f1 = [
        ParkingSpot("F1-S1", 1, SpotSize.SMALL),
        ParkingSpot("F1-S2", 1, SpotSize.SMALL),
        ParkingSpot("F1-M1", 1, SpotSize.MEDIUM),
        ParkingSpot("F1-M2", 1, SpotSize.MEDIUM),
        ParkingSpot("F1-L1", 1, SpotSize.LARGE),
    ]
    floor1 = ParkingFloor(1, spots_f1)

    lot = ParkingLot()
    lot.configure([floor1], HourlyPricing())
    lot.add_observer(DisplayBoard())

    moto = VehicleFactory.create(VehicleType.MOTORCYCLE, "MOTO-01")
    car  = VehicleFactory.create(VehicleType.CAR,        "CAR-99")
    truck = VehicleFactory.create(VehicleType.TRUCK,     "TRUCK-X")

    t1 = lot.park(moto)
    t2 = lot.park(car)
    t3 = lot.park(truck)

    lot.exit(t1.ticket_id)
    lot.exit(t2.ticket_id)
```

---

## Node.js Implementation

```javascript
// parking-lot.js
import { randomUUID } from "crypto";

// ─── Enums ───────────────────────────────────────────────────────────────────

const SpotSize  = Object.freeze({ SMALL: 1, MEDIUM: 2, LARGE: 3 });
const VehicleType = Object.freeze({ MOTORCYCLE: "MOTORCYCLE", CAR: "CAR", TRUCK: "TRUCK" });

// ─── Custom Errors ────────────────────────────────────────────────────────────

class ParkingLotFullError extends Error {}
class InvalidTicketError  extends Error {}

// ─── Strategy Pattern: Pricing ────────────────────────────────────────────────

class HourlyPricing {
  #rates = { [SpotSize.SMALL]: 2.0, [SpotSize.MEDIUM]: 3.5, [SpotSize.LARGE]: 5.0 };

  calculate(entryTime, exitTime, spotSize) {
    const hours = Math.max(1, Math.floor((exitTime - entryTime) / 3_600_000));
    return hours * this.#rates[spotSize];
  }
}

class FlatRatePricing {
  calculate(_entry, _exit, _size) { return 10.0; }
}

// ─── Observer Pattern ─────────────────────────────────────────────────────────

class DisplayBoard {
  onSpotChanged(floorNum, available, total) {
    console.log(`[Display] Floor ${floorNum}: ${available}/${total} spots available`);
  }
}

// ─── Domain Models ────────────────────────────────────────────────────────────

class Vehicle {
  constructor(licensePlate, vehicleType, requiredSpotSize) {
    this.licensePlate    = licensePlate;
    this.vehicleType     = vehicleType;
    this.requiredSpotSize = requiredSpotSize;
  }
  toString() { return `${this.vehicleType}(${this.licensePlate})`; }
}

// ─── Factory Pattern ─────────────────────────────────────────────────────────

class VehicleFactory {
  static #creators = {
    [VehicleType.MOTORCYCLE]: (lp) => new Vehicle(lp, VehicleType.MOTORCYCLE, SpotSize.SMALL),
    [VehicleType.CAR]:        (lp) => new Vehicle(lp, VehicleType.CAR,        SpotSize.MEDIUM),
    [VehicleType.TRUCK]:      (lp) => new Vehicle(lp, VehicleType.TRUCK,      SpotSize.LARGE),
  };

  static create(vehicleType, licensePlate) {
    const creator = this.#creators[vehicleType];
    if (!creator) throw new Error(`Unknown vehicle type: ${vehicleType}`);
    return creator(licensePlate);
  }
}

// ─── Parking Spot ─────────────────────────────────────────────────────────────

class ParkingSpot {
  #vehicle = null;

  constructor(spotId, floor, size) {
    this.spotId = spotId;
    this.floor  = floor;
    this.size   = size;
  }

  get isAvailable() { return this.#vehicle === null; }
  get vehicle()     { return this.#vehicle; }

  canFit(vehicle) {
    return this.isAvailable && this.size >= vehicle.requiredSpotSize;
  }

  park(vehicle) {
    if (!this.isAvailable) throw new Error(`Spot ${this.spotId} is occupied`);
    this.#vehicle = vehicle;
  }

  vacate() { this.#vehicle = null; }
}

// ─── Parking Ticket ───────────────────────────────────────────────────────────

class ParkingTicket {
  constructor(vehicle, spot) {
    this.ticketId  = randomUUID().slice(0, 8).toUpperCase();
    this.vehicle   = vehicle;
    this.spot      = spot;
    this.entryTime = Date.now();
  }
}

// ─── Parking Floor ────────────────────────────────────────────────────────────

class ParkingFloor {
  constructor(floorNumber, spots) {
    this.floorNumber = floorNumber;
    this.#spots = spots;
  }
  #spots;

  findAvailableSpot(vehicle) {
    return this.#spots.find((s) => s.canFit(vehicle)) ?? null;
  }

  get availableCount() { return this.#spots.filter((s) => s.isAvailable).length; }
  get totalCount()     { return this.#spots.length; }
}

// ─── Singleton Pattern: Parking Lot ──────────────────────────────────────────

class ParkingLot {
  static #instance = null;

  #floors   = [];
  #tickets  = new Map();   // ticketId → ParkingTicket
  #pricing  = new HourlyPricing();
  #observers = [];

  constructor() {
    if (ParkingLot.#instance) return ParkingLot.#instance;
    ParkingLot.#instance = this;
  }

  static getInstance() {
    if (!ParkingLot.#instance) new ParkingLot();
    return ParkingLot.#instance;
  }

  configure(floors, pricing) {
    this.#floors  = floors;
    this.#pricing = pricing;
  }

  addObserver(observer) { this.#observers.push(observer); }

  #notifyObservers(floor) {
    this.#observers.forEach((o) =>
      o.onSpotChanged(floor.floorNumber, floor.availableCount, floor.totalCount)
    );
  }

  park(vehicle) {
    for (const floor of this.#floors) {
      const spot = floor.findAvailableSpot(vehicle);
      if (spot) {
        spot.park(vehicle);
        const ticket = new ParkingTicket(vehicle, spot);
        this.#tickets.set(ticket.ticketId, ticket);
        this.#notifyObservers(floor);
        console.log(`[Entry] ${vehicle} → Floor ${spot.floor}, Spot ${spot.spotId}. Ticket: ${ticket.ticketId}`);
        return ticket;
      }
    }
    throw new ParkingLotFullError(`No available spot for ${vehicle}`);
  }

  exit(ticketId) {
    const ticket = this.#tickets.get(ticketId);
    if (!ticket) throw new InvalidTicketError(`Ticket ${ticketId} not found`);

    this.#tickets.delete(ticketId);
    const exitTime = Date.now();
    const fee = this.#pricing.calculate(ticket.entryTime, exitTime, ticket.spot.size);
    ticket.spot.vacate();

    const floor = this.#floors.find((f) => f.floorNumber === ticket.spot.floor);
    if (floor) this.#notifyObservers(floor);

    const durationSec = Math.round((exitTime - ticket.entryTime) / 1000);
    console.log(`[Exit] ${ticket.vehicle} — Duration: ${durationSec}s — Fee: $${fee.toFixed(2)}`);
    return fee;
  }
}

// ─── Demo ─────────────────────────────────────────────────────────────────────

const spots = [
  new ParkingSpot("F1-S1", 1, SpotSize.SMALL),
  new ParkingSpot("F1-S2", 1, SpotSize.SMALL),
  new ParkingSpot("F1-M1", 1, SpotSize.MEDIUM),
  new ParkingSpot("F1-M2", 1, SpotSize.MEDIUM),
  new ParkingSpot("F1-L1", 1, SpotSize.LARGE),
];

const lot = ParkingLot.getInstance();
lot.configure([new ParkingFloor(1, spots)], new HourlyPricing());
lot.addObserver(new DisplayBoard());

const moto  = VehicleFactory.create(VehicleType.MOTORCYCLE, "MOTO-01");
const car   = VehicleFactory.create(VehicleType.CAR,        "CAR-99");
const truck = VehicleFactory.create(VehicleType.TRUCK,      "TRUCK-X");

const t1 = lot.park(moto);
const t2 = lot.park(car);
const t3 = lot.park(truck);

lot.exit(t1.ticketId);
lot.exit(t2.ticketId);
```

---

## Key Takeaways

| Concept | Detail |
|---|---|
| Strategy | Pricing is injected, not hard-coded — swap at runtime without touching `ParkingLot` |
| Singleton | One `ParkingLot` across the whole app — use private constructor + static instance |
| Factory | Vehicle creation is centralized — adding `Van` type only touches `VehicleFactory` |
| Observer | `DisplayBoard` is decoupled — it reacts to events without `ParkingLot` knowing about screens |
| Thread safety | Python uses `threading.Lock`; Node.js is single-threaded but the pattern documents the intent |
| Fail Fast | `ParkingLotFullError` and `InvalidTicketError` prevent silent null bugs |
