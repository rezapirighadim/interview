# LLD — Vending Machine

## Problem Statement

Design a vending machine that holds multiple products at different prices, accepts coins
and notes, dispenses items when fully paid, and returns change. The machine must handle
different internal states correctly: idle, item selected, money inserted, dispensing.

---

## Requirements

**Functional**
- Display available products and their quantities
- Accept customer money (coins/notes) incrementally
- Let the customer select a product
- Dispense the product and return change when payment is sufficient
- Reject selection if item is out of stock or underpaid
- Support admin operations: restock items, collect cash

**Non-functional**
- Adding a new state (e.g. "maintenance mode") must not require rewriting existing states
- The machine must never dispense without full payment
- Pricing must be per-product (not global)

---

## Design Patterns Used

| Pattern | Where | Why |
|---|---|---|
| **State** | `VendingMachineState` | Each state encapsulates its own valid transitions and actions — no giant if/else chains |
| **Command** | `AdminCommand` | Encapsulate admin operations (restock, collect) as objects, enabling logging and undo |
| **Null Object** | `NoProduct` | Avoids `None` checks when no item is selected |
| **Iterator** | `inventory.__iter__` | Clean iteration over products without exposing internal dict |

---

## Class Diagram

```
VendingMachine
  ├── inventory: Inventory
  ├── cash_balance: float
  ├── current_state: VendingMachineState
  └── selected_product: Product | NoProduct

VendingMachineState (interface)
  ├── IdleState          → waiting for money/selection
  ├── HasMoneyState      → money inserted, waiting for selection
  ├── ItemSelectedState  → item chosen, waiting for full payment
  └── DispensingState    → dispensing + giving change

Product
  ├── name: str
  ├── price: float
  └── quantity: int

AdminCommand (interface)
  ├── RestockCommand
  └── CollectCashCommand
```

---

## Clean Code Principles Applied

- **State Pattern replaces conditionals:** Instead of `if machine.state == "idle": ... elif machine.state == "selected": ...`, each state class handles its own behavior. Adding "maintenance mode" = new class, zero existing changes.
- **Tell, Don't Ask:** The machine tells its current state to handle the action; it doesn't ask the state what it is.
- **Immutable Product:** Product data is a `dataclass(frozen=True)` — quantity lives in `Inventory`, not on the product.
- **Guard Clauses:** Each state method fails fast with a clear message before doing any work.
- **Command Pattern:** Admin operations are objects. They can be logged, queued, and replayed.

---

## Python Implementation

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Optional


# ─── Domain: Product ─────────────────────────────────────────────────────────

@dataclass(frozen=True)
class Product:
    name:  str
    price: float

    def __str__(self) -> str:
        return f"{self.name} (${self.price:.2f})"


# ─── Null Object: NoProduct ───────────────────────────────────────────────────

class _NoProduct:
    name  = "None"
    price = 0.0
    def __bool__(self) -> bool:
        return False
    def __str__(self) -> str:
        return "No product selected"

NO_PRODUCT = _NoProduct()


# ─── Inventory ────────────────────────────────────────────────────────────────

class Inventory:
    def __init__(self):
        self._stock: dict[Product, int] = {}

    def add(self, product: Product, quantity: int) -> None:
        self._stock[product] = self._stock.get(product, 0) + quantity

    def is_available(self, product: Product) -> bool:
        return self._stock.get(product, 0) > 0

    def dispense(self, product: Product) -> None:
        if not self.is_available(product):
            raise ValueError(f"{product.name} is out of stock")
        self._stock[product] -= 1

    def __iter__(self):
        return iter(self._stock.items())

    def display(self) -> None:
        print("\n── Available Products ──")
        for product, qty in self:
            status = f"qty: {qty}" if qty > 0 else "OUT OF STOCK"
            print(f"  {product}  [{status}]")
        print()


# ─── State Interface ──────────────────────────────────────────────────────────

class VendingMachineState(ABC):
    def __init__(self, machine: "VendingMachine"):
        self._m = machine

    def insert_money(self, amount: float) -> None:
        print(f"[State: {self.__class__.__name__}] Cannot insert money right now.")

    def select_product(self, product: Product) -> None:
        print(f"[State: {self.__class__.__name__}] Cannot select product right now.")

    def dispense(self) -> None:
        print(f"[State: {self.__class__.__name__}] Cannot dispense right now.")

    def cancel(self) -> None:
        print(f"[State: {self.__class__.__name__}] Nothing to cancel.")


# ─── Concrete States ──────────────────────────────────────────────────────────

class IdleState(VendingMachineState):
    def insert_money(self, amount: float) -> None:
        self._m.cash_balance += amount
        print(f"Inserted ${amount:.2f}. Balance: ${self._m.cash_balance:.2f}")
        self._m.transition_to(HasMoneyState(self._m))

    def cancel(self) -> None:
        print("Machine is idle. Nothing to cancel.")


class HasMoneyState(VendingMachineState):
    def insert_money(self, amount: float) -> None:
        self._m.cash_balance += amount
        print(f"Inserted ${amount:.2f}. Balance: ${self._m.cash_balance:.2f}")

    def select_product(self, product: Product) -> None:
        if not self._m.inventory.is_available(product):
            print(f"{product.name} is out of stock. Please select another item.")
            return
        self._m.selected_product = product
        print(f"Selected: {product}")
        if self._m.cash_balance >= product.price:
            self._m.transition_to(ItemSelectedState(self._m))
            self._m.current_state.dispense()  # auto-dispense if already paid
        else:
            shortage = product.price - self._m.cash_balance
            print(f"Please insert ${shortage:.2f} more.")
            self._m.transition_to(ItemSelectedState(self._m))

    def cancel(self) -> None:
        refund = self._m.cash_balance
        self._m.cash_balance = 0.0
        self._m.transition_to(IdleState(self._m))
        print(f"Cancelled. Refunding ${refund:.2f}")


class ItemSelectedState(VendingMachineState):
    def insert_money(self, amount: float) -> None:
        self._m.cash_balance += amount
        print(f"Inserted ${amount:.2f}. Balance: ${self._m.cash_balance:.2f}")
        product = self._m.selected_product
        if self._m.cash_balance >= product.price:
            self.dispense()

    def dispense(self) -> None:
        product = self._m.selected_product
        if self._m.cash_balance < product.price:
            shortage = product.price - self._m.cash_balance
            print(f"Insufficient funds. Please insert ${shortage:.2f} more.")
            return
        self._m.transition_to(DispensingState(self._m))
        self._m.current_state.dispense()

    def cancel(self) -> None:
        refund = self._m.cash_balance
        self._m.cash_balance    = 0.0
        self._m.selected_product = NO_PRODUCT
        self._m.transition_to(IdleState(self._m))
        print(f"Cancelled. Refunding ${refund:.2f}")


class DispensingState(VendingMachineState):
    def dispense(self) -> None:
        product = self._m.selected_product
        self._m.inventory.dispense(product)
        change = round(self._m.cash_balance - product.price, 2)
        self._m.cash_balance     = 0.0
        self._m.selected_product  = NO_PRODUCT
        print(f"Dispensing: {product.name}")
        if change > 0:
            print(f"Change returned: ${change:.2f}")
        self._m.transition_to(IdleState(self._m))


# ─── Command Pattern: Admin Operations ───────────────────────────────────────

class AdminCommand(ABC):
    @abstractmethod
    def execute(self) -> None:
        ...


class RestockCommand(AdminCommand):
    def __init__(self, machine: "VendingMachine", product: Product, quantity: int):
        self._machine   = machine
        self._product   = product
        self._quantity  = quantity

    def execute(self) -> None:
        self._machine.inventory.add(self._product, self._quantity)
        print(f"[Admin] Restocked {self._quantity}x {self._product.name}")


class CollectCashCommand(AdminCommand):
    def __init__(self, machine: "VendingMachine"):
        self._machine = machine

    def execute(self) -> None:
        collected = self._machine.total_cash_collected
        self._machine.total_cash_collected = 0.0
        print(f"[Admin] Collected ${collected:.2f} from machine")


# ─── Vending Machine (Context) ────────────────────────────────────────────────

class VendingMachine:
    def __init__(self):
        self.inventory:            Inventory = Inventory()
        self.cash_balance:         float = 0.0
        self.total_cash_collected: float = 0.0
        self.selected_product:     Product | _NoProduct = NO_PRODUCT
        self.current_state:        VendingMachineState = IdleState(self)

    def transition_to(self, state: VendingMachineState) -> None:
        self.current_state = state

    # ── Public Interface (delegates to state) ──
    def insert_money(self, amount: float) -> None:
        self.current_state.insert_money(amount)

    def select_product(self, product: Product) -> None:
        self.current_state.select_product(product)

    def cancel(self) -> None:
        self.current_state.cancel()

    def run_admin(self, command: AdminCommand) -> None:
        command.execute()


# ─── Demo ─────────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    machine = VendingMachine()

    cola  = Product("Cola",   1.50)
    chips = Product("Chips",  2.00)
    water = Product("Water",  1.00)

    machine.run_admin(RestockCommand(machine, cola,  5))
    machine.run_admin(RestockCommand(machine, chips, 3))
    machine.run_admin(RestockCommand(machine, water, 2))

    machine.inventory.display()

    # Scenario 1: successful purchase with change
    machine.insert_money(1.00)
    machine.insert_money(1.00)
    machine.select_product(cola)   # balance 2.00, cola costs 1.50 → dispense + 0.50 change

    print()

    # Scenario 2: insert money then cancel
    machine.insert_money(2.00)
    machine.cancel()               # refunds $2.00

    print()

    # Scenario 3: out of stock
    machine.run_admin(RestockCommand(machine, water, -2))  # drain water
    machine.insert_money(1.00)
    machine.select_product(water)  # out of stock message
    machine.cancel()
```

---

## Node.js Implementation

```javascript
// vending-machine.js

// ─── Domain: Product ─────────────────────────────────────────────────────────

class Product {
  constructor(name, price) {
    this.name  = name;
    this.price = price;
    Object.freeze(this);
  }
  toString() { return `${this.name} ($${this.price.toFixed(2)})`; }
}

// ─── Null Object ──────────────────────────────────────────────────────────────

const NO_PRODUCT = Object.freeze({ name: "None", price: 0, toString: () => "No product selected" });

// ─── Inventory ────────────────────────────────────────────────────────────────

class Inventory {
  #stock = new Map();

  add(product, quantity) {
    this.#stock.set(product, (this.#stock.get(product) ?? 0) + quantity);
  }

  isAvailable(product) {
    return (this.#stock.get(product) ?? 0) > 0;
  }

  dispense(product) {
    const qty = this.#stock.get(product) ?? 0;
    if (qty <= 0) throw new Error(`${product.name} is out of stock`);
    this.#stock.set(product, qty - 1);
  }

  display() {
    console.log("\n── Available Products ──");
    for (const [product, qty] of this.#stock) {
      const status = qty > 0 ? `qty: ${qty}` : "OUT OF STOCK";
      console.log(`  ${product}  [${status}]`);
    }
    console.log();
  }
}

// ─── State Base Class ─────────────────────────────────────────────────────────

class VendingMachineState {
  constructor(machine) { this._m = machine; }
  insertMoney(_)     { console.log(`[${this.constructor.name}] Cannot insert money now.`); }
  selectProduct(_)   { console.log(`[${this.constructor.name}] Cannot select product now.`); }
  dispense()         { console.log(`[${this.constructor.name}] Cannot dispense now.`); }
  cancel()           { console.log(`[${this.constructor.name}] Nothing to cancel.`); }
}

// ─── Concrete States ──────────────────────────────────────────────────────────

class IdleState extends VendingMachineState {
  insertMoney(amount) {
    this._m.cashBalance += amount;
    console.log(`Inserted $${amount.toFixed(2)}. Balance: $${this._m.cashBalance.toFixed(2)}`);
    this._m.transitionTo(new HasMoneyState(this._m));
  }
  cancel() { console.log("Machine is idle. Nothing to cancel."); }
}

class HasMoneyState extends VendingMachineState {
  insertMoney(amount) {
    this._m.cashBalance += amount;
    console.log(`Inserted $${amount.toFixed(2)}. Balance: $${this._m.cashBalance.toFixed(2)}`);
  }

  selectProduct(product) {
    if (!this._m.inventory.isAvailable(product)) {
      console.log(`${product.name} is out of stock.`);
      return;
    }
    this._m.selectedProduct = product;
    console.log(`Selected: ${product}`);
    this._m.transitionTo(new ItemSelectedState(this._m));
    if (this._m.cashBalance >= product.price) {
      this._m.currentState.dispense();
    } else {
      console.log(`Please insert $${(product.price - this._m.cashBalance).toFixed(2)} more.`);
    }
  }

  cancel() {
    const refund = this._m.cashBalance;
    this._m.cashBalance = 0;
    this._m.transitionTo(new IdleState(this._m));
    console.log(`Cancelled. Refunding $${refund.toFixed(2)}`);
  }
}

class ItemSelectedState extends VendingMachineState {
  insertMoney(amount) {
    this._m.cashBalance += amount;
    console.log(`Inserted $${amount.toFixed(2)}. Balance: $${this._m.cashBalance.toFixed(2)}`);
    if (this._m.cashBalance >= this._m.selectedProduct.price) this.dispense();
  }

  dispense() {
    const product = this._m.selectedProduct;
    if (this._m.cashBalance < product.price) {
      console.log(`Insufficient funds. Need $${(product.price - this._m.cashBalance).toFixed(2)} more.`);
      return;
    }
    this._m.transitionTo(new DispensingState(this._m));
    this._m.currentState.dispense();
  }

  cancel() {
    const refund = this._m.cashBalance;
    this._m.cashBalance     = 0;
    this._m.selectedProduct = NO_PRODUCT;
    this._m.transitionTo(new IdleState(this._m));
    console.log(`Cancelled. Refunding $${refund.toFixed(2)}`);
  }
}

class DispensingState extends VendingMachineState {
  dispense() {
    const product = this._m.selectedProduct;
    this._m.inventory.dispense(product);
    const change = parseFloat((this._m.cashBalance - product.price).toFixed(2));
    this._m.cashBalance     = 0;
    this._m.selectedProduct = NO_PRODUCT;
    console.log(`Dispensing: ${product.name}`);
    if (change > 0) console.log(`Change returned: $${change.toFixed(2)}`);
    this._m.transitionTo(new IdleState(this._m));
  }
}

// ─── Command Pattern: Admin ───────────────────────────────────────────────────

class RestockCommand {
  constructor(machine, product, quantity) {
    this._machine  = machine;
    this._product  = product;
    this._quantity = quantity;
  }
  execute() {
    this._machine.inventory.add(this._product, this._quantity);
    console.log(`[Admin] Restocked ${this._quantity}x ${this._product.name}`);
  }
}

// ─── Vending Machine (Context) ────────────────────────────────────────────────

class VendingMachine {
  cashBalance     = 0;
  selectedProduct = NO_PRODUCT;

  constructor() {
    this.inventory    = new Inventory();
    this.currentState = new IdleState(this);
  }

  transitionTo(state)  { this.currentState = state; }
  insertMoney(amount)  { this.currentState.insertMoney(amount); }
  selectProduct(p)     { this.currentState.selectProduct(p); }
  cancel()             { this.currentState.cancel(); }
  runAdmin(command)    { command.execute(); }
}

// ─── Demo ─────────────────────────────────────────────────────────────────────

const machine = new VendingMachine();
const cola  = new Product("Cola",  1.50);
const chips = new Product("Chips", 2.00);

machine.runAdmin(new RestockCommand(machine, cola,  5));
machine.runAdmin(new RestockCommand(machine, chips, 3));
machine.inventory.display();

machine.insertMoney(1.00);
machine.insertMoney(1.00);
machine.selectProduct(cola);   // dispense + $0.50 change

console.log();
machine.insertMoney(2.00);
machine.cancel();              // refund $2.00
```

---

## Key Takeaways

| Concept | Detail |
|---|---|
| State Pattern | Each state is self-contained. Transitions are explicit method calls — no if/else spaghetti |
| Adding "Maintenance" state | Create `MaintenanceState` class, override relevant methods. Zero changes to existing states |
| Command Pattern | Admin operations are objects — can be logged, queued, replayed, or undone |
| Null Object | `NO_PRODUCT` prevents null checks scattered across every state's `dispense()` method |
| Delegation | `VendingMachine` never decides behavior — it delegates to `currentState`. Pure context role |
