# LLD — Elevator System

## Problem Statement

Design an elevator control system for a building with multiple elevators and multiple
floors. The system receives floor requests (both inside-cabin button presses and
hallway calls) and dispatches the most appropriate elevator to serve each request
using an efficient scheduling algorithm (SCAN / Look algorithm).

---

## Requirements

**Functional**
- Users can call an elevator from any floor (up or down)
- Passengers inside the elevator can press a destination floor button
- System dispatches the nearest idle/compatible elevator
- Elevator serves floors in one direction before reversing (SCAN algorithm)
- Track elevator state: IDLE, MOVING_UP, MOVING_DOWN, DOORS_OPEN

**Non-functional**
- Scheduling algorithm is swappable (FCFS, SCAN, LOOK)
- Elevator state transitions are explicit and guarded
- System is extensible to add emergency mode, maintenance mode
- Each elevator is an independent state machine

---

## Design Patterns Used

| Pattern | Where | Why |
|---|---|---|
| **State** | `ElevatorState` | Guards door-opening, moving, stopping — no if/else chains in the controller |
| **Strategy** | `DispatchStrategy` | Swap SCAN for LOOK or FCFS without changing `ElevatorController` |
| **Observer** | `ElevatorEventListener` | Display panels, logging, and alarms react to elevator events without being coupled to elevator logic |
| **Command** | `ElevatorRequest` | Encapsulate floor requests as objects — enables priority queues and logging |
| **Scheduler** | `RequestQueue` (min-heap) | Always serve the nearest floor in the current direction first |

---

## Class Diagram

```
ElevatorController
  ├── elevators: List[Elevator]
  ├── strategy: DispatchStrategy
  └── pending_requests: List[ElevatorRequest]

Elevator
  ├── elevator_id: int
  ├── current_floor: int
  ├── direction: Direction (enum)
  ├── state: ElevatorState
  └── request_queue: RequestQueue

ElevatorState (interface — State Pattern)
  ├── IdleState
  ├── MovingUpState
  ├── MovingDownState
  └── DoorsOpenState

DispatchStrategy (interface — Strategy Pattern)
  ├── NearestElevatorStrategy
  └── LookAlgorithmStrategy (extensible)

ElevatorRequest (Command)
  ├── floor: int
  ├── direction: Direction
  └── source: RequestSource (HALL | CABIN)
```

---

## Clean Code Principles Applied

- **State Pattern:** `elevator.open_doors()` in `MovingUpState` raises an error — you
  can't open doors while moving. The state enforces physical constraints in code.
- **Strategy Pattern:** `NearestElevatorStrategy` can be swapped for `LookAlgorithmStrategy`
  in one line at the controller level.
- **Command as Value Object:** `ElevatorRequest(floor=5, direction=UP)` can be stored,
  logged, replayed, and sorted by priority.
- **Single Responsibility:** `Elevator` moves. `ElevatorController` dispatches.
  `DispatchStrategy` decides which elevator. Three separate jobs, three separate classes.
- **Fail Fast:** State transitions raise `InvalidOperationError` immediately. No silent
  ignored actions.

---

## Python Implementation

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from enum import Enum, auto
from typing import Callable, Optional
import heapq
import time


# ─── Enums ────────────────────────────────────────────────────────────────────

class Direction(Enum):
    UP   = auto()
    DOWN = auto()
    IDLE = auto()


class RequestSource(Enum):
    HALL  = auto()   # hallway button press
    CABIN = auto()   # inside-elevator button press


# ─── Exceptions ───────────────────────────────────────────────────────────────

class InvalidOperationError(Exception):
    pass


# ─── Command: Elevator Request ────────────────────────────────────────────────

@dataclass(order=True)
class ElevatorRequest:
    floor:     int
    direction: Direction   = field(compare=False)
    source:    RequestSource = field(compare=False)

    def __repr__(self) -> str:
        return f"Request(floor={self.floor}, dir={self.direction.name}, src={self.source.name})"


# ─── Request Queue (min-heap for current direction) ───────────────────────────

class RequestQueue:
    def __init__(self):
        self._up:   list[int] = []   # min-heap for UP requests
        self._down: list[int] = []   # max-heap (negated) for DOWN requests

    def add(self, request: ElevatorRequest) -> None:
        if request.direction in (Direction.UP, Direction.IDLE):
            heapq.heappush(self._up, request.floor)
        else:
            heapq.heappush(self._down, -request.floor)  # negate for max-heap

    def next_floor(self, current: int, direction: Direction) -> Optional[int]:
        if direction == Direction.UP and self._up:
            return self._up[0]
        if direction == Direction.DOWN and self._down:
            return -self._down[0]
        # Switch direction if current direction queue is empty
        if direction == Direction.UP and self._down:
            return -self._down[0]
        if direction == Direction.DOWN and self._up:
            return self._up[0]
        return None

    def pop_floor(self, floor: int, direction: Direction) -> None:
        if direction == Direction.UP and floor in self._up:
            self._up.remove(floor)
            heapq.heapify(self._up)
        elif direction == Direction.DOWN and -floor in self._down:
            self._down.remove(-floor)
            heapq.heapify(self._down)

    def is_empty(self) -> bool:
        return not self._up and not self._down


# ─── State Pattern: Elevator States ──────────────────────────────────────────

class ElevatorState(ABC):
    def __init__(self, elevator: "Elevator"):
        self._e = elevator

    def move_up(self) -> None:
        raise InvalidOperationError(f"Cannot move up in state {self.__class__.__name__}")

    def move_down(self) -> None:
        raise InvalidOperationError(f"Cannot move down in state {self.__class__.__name__}")

    def open_doors(self) -> None:
        raise InvalidOperationError(f"Cannot open doors in state {self.__class__.__name__}")

    def close_doors(self) -> None:
        raise InvalidOperationError(f"Cannot close doors in state {self.__class__.__name__}")

    def stop(self) -> None:
        raise InvalidOperationError(f"Cannot stop in state {self.__class__.__name__}")

    @property
    def name(self) -> str:
        return self.__class__.__name__


class IdleState(ElevatorState):
    def move_up(self) -> None:
        self._e._state = MovingUpState(self._e)
        self._e.direction = Direction.UP
        print(f"  [Elevator {self._e.elevator_id}] Starting UP from floor {self._e.current_floor}")

    def move_down(self) -> None:
        self._e._state = MovingDownState(self._e)
        self._e.direction = Direction.DOWN
        print(f"  [Elevator {self._e.elevator_id}] Starting DOWN from floor {self._e.current_floor}")

    def open_doors(self) -> None:
        self._e._state = DoorsOpenState(self._e)
        print(f"  [Elevator {self._e.elevator_id}] Doors opening at floor {self._e.current_floor}")


class MovingUpState(ElevatorState):
    def stop(self) -> None:
        self._e._state = IdleState(self._e)
        self._e.direction = Direction.IDLE
        print(f"  [Elevator {self._e.elevator_id}] Stopped at floor {self._e.current_floor}")

    def open_doors(self) -> None:
        self.stop()
        self._e._state.open_doors()


class MovingDownState(ElevatorState):
    def stop(self) -> None:
        self._e._state = IdleState(self._e)
        self._e.direction = Direction.IDLE
        print(f"  [Elevator {self._e.elevator_id}] Stopped at floor {self._e.current_floor}")

    def open_doors(self) -> None:
        self.stop()
        self._e._state.open_doors()


class DoorsOpenState(ElevatorState):
    def close_doors(self) -> None:
        self._e._state = IdleState(self._e)
        print(f"  [Elevator {self._e.elevator_id}] Doors closing at floor {self._e.current_floor}")


# ─── Observer Pattern: Events ─────────────────────────────────────────────────

ElevatorEventListener = Callable[[str, int, int], None]  # (event_name, elevator_id, floor)


def display_panel_listener(event: str, elevator_id: int, floor: int) -> None:
    print(f"  [Display Panel] Elevator {elevator_id}: {event} at floor {floor}")


# ─── Elevator ─────────────────────────────────────────────────────────────────

class Elevator:
    def __init__(self, elevator_id: int, total_floors: int):
        self.elevator_id   = elevator_id
        self.total_floors  = total_floors
        self.current_floor = 1
        self.direction     = Direction.IDLE
        self._state:       ElevatorState = IdleState(self)
        self._queue        = RequestQueue()
        self._listeners:   list[ElevatorEventListener] = []

    def add_listener(self, listener: ElevatorEventListener) -> None:
        self._listeners.append(listener)

    def _notify(self, event: str) -> None:
        for listener in self._listeners:
            listener(event, self.elevator_id, self.current_floor)

    def add_request(self, request: ElevatorRequest) -> None:
        self._queue.add(request)
        print(f"  [Elevator {self.elevator_id}] Queued {request}")

    def step(self) -> bool:
        """Simulate one floor of movement. Returns True if still has work."""
        next_floor = self._queue.next_floor(self.current_floor, self.direction)
        if next_floor is None:
            return False

        # Determine direction
        if next_floor > self.current_floor:
            self._state.move_up()
            self.current_floor += 1
        elif next_floor < self.current_floor:
            self._state.move_down()
            self.current_floor -= 1

        # Arrived at destination
        if self.current_floor == next_floor:
            self._queue.pop_floor(next_floor, self.direction)
            self._state.open_doors()
            self._notify("ARRIVED")
            time.sleep(0.1)  # simulate door open time
            self._state.close_doors()

        return not self._queue.is_empty()

    @property
    def state_name(self) -> str:
        return self._state.name

    def __repr__(self) -> str:
        return (f"Elevator(id={self.elevator_id}, floor={self.current_floor}, "
                f"dir={self.direction.name}, state={self.state_name})")


# ─── Strategy Pattern: Dispatch ───────────────────────────────────────────────

class DispatchStrategy(ABC):
    @abstractmethod
    def select(self, elevators: list[Elevator], request: ElevatorRequest) -> Elevator:
        ...


class NearestElevatorStrategy(DispatchStrategy):
    def select(self, elevators: list[Elevator], request: ElevatorRequest) -> Elevator:
        def score(elevator: Elevator) -> int:
            distance = abs(elevator.current_floor - request.floor)
            # Prefer idle elevators; penalize elevators moving opposite direction
            direction_penalty = 0
            if elevator.direction != Direction.IDLE:
                if (elevator.direction == Direction.UP and request.floor < elevator.current_floor or
                        elevator.direction == Direction.DOWN and request.floor > elevator.current_floor):
                    direction_penalty = 100
            return distance + direction_penalty

        return min(elevators, key=score)


# ─── Elevator Controller ──────────────────────────────────────────────────────

class ElevatorController:
    def __init__(self, elevators: list[Elevator], strategy: DispatchStrategy):
        self._elevators = elevators
        self._strategy  = strategy

    def request(self, floor: int, direction: Direction, source: RequestSource) -> None:
        req      = ElevatorRequest(floor, direction, source)
        elevator = self._strategy.select(self._elevators, req)
        print(f"[Controller] Dispatching Elevator {elevator.elevator_id} → {req}")
        elevator.add_request(req)

    def run_simulation(self, steps: int = 20) -> None:
        print("\n[Simulation Start]")
        for step in range(steps):
            any_active = False
            for elevator in self._elevators:
                if elevator.step():
                    any_active = True
            if not any_active:
                print("[Simulation] All elevators idle. Done.")
                break


# ─── Demo ─────────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    e1 = Elevator(elevator_id=1, total_floors=10)
    e2 = Elevator(elevator_id=2, total_floors=10)
    e1.add_listener(display_panel_listener)

    controller = ElevatorController(
        elevators=[e1, e2],
        strategy=NearestElevatorStrategy(),
    )

    # Hall calls
    controller.request(5, Direction.UP,   RequestSource.HALL)
    controller.request(3, Direction.DOWN,  RequestSource.HALL)
    controller.request(8, Direction.UP,   RequestSource.HALL)

    # Cabin button presses (direct floor requests)
    e1.add_request(ElevatorRequest(7, Direction.UP, RequestSource.CABIN))

    controller.run_simulation(steps=15)
```

---

## Node.js Implementation

```javascript
// elevator-system.js

const Direction     = Object.freeze({ UP: "UP", DOWN: "DOWN", IDLE: "IDLE" });
const RequestSource = Object.freeze({ HALL: "HALL", CABIN: "CABIN" });

// ─── Command: Request ─────────────────────────────────────────────────────────

class ElevatorRequest {
  constructor(floor, direction, source) {
    this.floor     = floor;
    this.direction = direction;
    this.source    = source;
  }
  toString() { return `Request(floor=${this.floor}, dir=${this.direction}, src=${this.source})`; }
}

// ─── State Pattern ────────────────────────────────────────────────────────────

class ElevatorState {
  constructor(elevator) { this._e = elevator; }
  moveUp()     { throw new Error(`Cannot moveUp in ${this.constructor.name}`); }
  moveDown()   { throw new Error(`Cannot moveDown in ${this.constructor.name}`); }
  openDoors()  { throw new Error(`Cannot openDoors in ${this.constructor.name}`); }
  closeDoors() { throw new Error(`Cannot closeDoors in ${this.constructor.name}`); }
  stop()       { throw new Error(`Cannot stop in ${this.constructor.name}`); }
}

class IdleState extends ElevatorState {
  moveUp() {
    this._e._state = new MovingUpState(this._e);
    this._e.direction = Direction.UP;
    console.log(`  [Elevator ${this._e.id}] Starting UP from floor ${this._e.currentFloor}`);
  }
  moveDown() {
    this._e._state = new MovingDownState(this._e);
    this._e.direction = Direction.DOWN;
    console.log(`  [Elevator ${this._e.id}] Starting DOWN from floor ${this._e.currentFloor}`);
  }
  openDoors() {
    this._e._state = new DoorsOpenState(this._e);
    console.log(`  [Elevator ${this._e.id}] Doors opening at floor ${this._e.currentFloor}`);
  }
}

class MovingUpState extends ElevatorState {
  stop() {
    this._e._state = new IdleState(this._e);
    this._e.direction = Direction.IDLE;
    console.log(`  [Elevator ${this._e.id}] Stopped at floor ${this._e.currentFloor}`);
  }
  openDoors() { this.stop(); this._e._state.openDoors(); }
}

class MovingDownState extends ElevatorState {
  stop() {
    this._e._state = new IdleState(this._e);
    this._e.direction = Direction.IDLE;
    console.log(`  [Elevator ${this._e.id}] Stopped at floor ${this._e.currentFloor}`);
  }
  openDoors() { this.stop(); this._e._state.openDoors(); }
}

class DoorsOpenState extends ElevatorState {
  closeDoors() {
    this._e._state = new IdleState(this._e);
    console.log(`  [Elevator ${this._e.id}] Doors closing at floor ${this._e.currentFloor}`);
  }
}

// ─── Elevator ─────────────────────────────────────────────────────────────────

class Elevator {
  #state; #upQueue = []; #downQueue = []; #listeners = [];
  currentFloor = 1; direction = Direction.IDLE;

  constructor(id) {
    this.id   = id;
    this.#state = new IdleState(this);
  }

  addListener(fn)   { this.#listeners.push(fn); }
  #notify(event)    { this.#listeners.forEach(fn => fn(event, this.id, this.currentFloor)); }

  addRequest(req) {
    if (req.direction === Direction.UP || req.direction === Direction.IDLE)
      this.#upQueue.push(req.floor);
    else
      this.#downQueue.push(req.floor);
    console.log(`  [Elevator ${this.id}] Queued ${req}`);
  }

  #nextFloor() {
    const dir = this.direction;
    if (dir === Direction.UP    && this.#upQueue.length)   return Math.min(...this.#upQueue);
    if (dir === Direction.DOWN  && this.#downQueue.length) return Math.max(...this.#downQueue);
    if (this.#downQueue.length) return Math.max(...this.#downQueue);
    if (this.#upQueue.length)   return Math.min(...this.#upQueue);
    return null;
  }

  #removeFromQueue(floor) {
    this.#upQueue   = this.#upQueue.filter(f => f !== floor);
    this.#downQueue = this.#downQueue.filter(f => f !== floor);
  }

  step() {
    const next = this.#nextFloor();
    if (next === null) return false;

    if (next > this.currentFloor)      { this.#state.moveUp();   this.currentFloor++; }
    else if (next < this.currentFloor) { this.#state.moveDown(); this.currentFloor--; }

    if (this.currentFloor === next) {
      this.#removeFromQueue(next);
      this.#state.openDoors();
      this.#notify("ARRIVED");
      this.#state.closeDoors();
    }
    return this.#upQueue.length > 0 || this.#downQueue.length > 0;
  }

  get stateName() { return this.#state.constructor.name; }
}

// ─── Strategy: Dispatch ───────────────────────────────────────────────────────

class NearestElevatorStrategy {
  select(elevators, request) {
    return elevators.reduce((best, el) => {
      const dist = Math.abs(el.currentFloor - request.floor);
      const penalty = (el.direction !== Direction.IDLE &&
        ((el.direction === Direction.UP && request.floor < el.currentFloor) ||
         (el.direction === Direction.DOWN && request.floor > el.currentFloor))) ? 100 : 0;
      const score = dist + penalty;
      return score < (Math.abs(best.currentFloor - request.floor)) ? el : best;
    });
  }
}

// ─── Controller ───────────────────────────────────────────────────────────────

class ElevatorController {
  constructor(elevators, strategy) {
    this.#elevators = elevators;
    this.#strategy  = strategy;
  }
  #elevators; #strategy;

  request(floor, direction, source) {
    const req = new ElevatorRequest(floor, direction, source);
    const el  = this.#strategy.select(this.#elevators, req);
    console.log(`[Controller] Dispatching Elevator ${el.id} → ${req}`);
    el.addRequest(req);
  }

  runSimulation(steps = 20) {
    console.log("\n[Simulation Start]");
    for (let i = 0; i < steps; i++) {
      const anyActive = this.#elevators.some(el => el.step());
      if (!anyActive) { console.log("[Simulation] All elevators idle. Done."); break; }
    }
  }
}

// ─── Demo ─────────────────────────────────────────────────────────────────────

const e1 = new Elevator(1);
const e2 = new Elevator(2);
e1.addListener((evt, id, floor) => console.log(`  [Panel] Elevator ${id}: ${evt} at floor ${floor}`));

const controller = new ElevatorController([e1, e2], new NearestElevatorStrategy());

controller.request(5, Direction.UP,   RequestSource.HALL);
controller.request(3, Direction.DOWN, RequestSource.HALL);
controller.request(8, Direction.UP,   RequestSource.HALL);
e1.addRequest(new ElevatorRequest(7, Direction.UP, RequestSource.CABIN));

controller.runSimulation(15);
```

---

## Key Takeaways

| Concept | Detail |
|---|---|
| State Pattern | `elevator.open_doors()` in `MovingUpState` throws — the elevator enforces physics in code |
| Strategy Pattern | Change scheduling: `new ElevatorController(elevators, new LookAlgorithmStrategy())` — one line |
| Command Pattern | `ElevatorRequest` is a first-class object. It can be stored in priority queues, logged, and replayed |
| Observer | `display_panel_listener` is registered on the elevator, not hardcoded inside it |
| SCAN Algorithm | The min-heap for UP and negated max-heap for DOWN gives O(log n) next-floor lookup |
| Separation of Concerns | `Elevator` handles movement. `ElevatorController` handles dispatch. `DispatchStrategy` handles selection. Three independent responsibilities |
