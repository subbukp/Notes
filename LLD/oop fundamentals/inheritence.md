# OOP — Inheritance

## The Core Idea

> Inheritance = Define common logic once in a base class, extend or specialise in derived classes.

Enables **code reuse**, **logical hierarchy**, and is a prerequisite for **polymorphism**.

---

## 1. Real-World Analogy — User System

```mermaid
classDiagram
    class User {
        -username: str
        -email: str
        +login()
        +logout()
    }

    class Admin {
        +delete_user()
        +ban_user()
    }

    class Customer {
        +browse_products()
        +add_to_cart()
    }

    class Vendor {
        +add_product()
        +manage_inventory()
    }

    User <|-- Admin
    User <|-- Customer
    User <|-- Vendor
```

`User` holds common attributes and behaviour. `Admin`, `Customer`, `Vendor` inherit everything from `User` and add their own role-specific behaviour.

---

## 2. Why Inheritance Matters

```mermaid
mindmap
  root((Inheritance))
    Code Reusability
      DRY principle
      Write common logic once
      All subclasses get it free
    Logical Hierarchy
      Models is-a relationships
      ElectricCar is a Vehicle
      Admin is a User
    Ease of Maintenance
      Fix bug in parent once
      All subclasses get the fix
    Polymorphism
      Treat subclasses as parent type
      Swap implementations freely
```

---

## 3. How It Works

```mermaid
flowchart TD
    Parent["Parent Class\n(superclass)"] -->|inherits| Child["Child Class\n(subclass)"]

    Child --> R[Inherits all non-private\nfields and methods]
    Child --> O[Can override\ninherited methods]
    Child --> E[Can extend with\nnew fields and methods]
```

### Example — Vehicle Hierarchy

```mermaid
classDiagram
    class Vehicle {
        #make: str
        #model: str
        #year: int
        +start_engine()
        +stop_engine()
        +display_info()
    }

    class ElectricCar {
        -battery_capacity: int
        +charge_battery()
        +start_engine()
    }

    class GasCar {
        -fuel_tank_size: float
        +fill_tank()
        +start_engine()
    }

    Vehicle <|-- ElectricCar
    Vehicle <|-- GasCar
```

```python
class Vehicle:
    def __init__(self, make: str, model: str, year: int):
        self.make = make
        self.model = model
        self.year = year

    def start_engine(self) -> None:
        print(f"{self.make} {self.model}: engine started.")

    def stop_engine(self) -> None:
        print(f"{self.make} {self.model}: engine stopped.")

    def display_info(self) -> None:
        print(f"{self.year} {self.make} {self.model}")


class ElectricCar(Vehicle):
    def __init__(self, make: str, model: str, year: int, battery: int):
        super().__init__(make, model, year)
        self.battery_capacity = battery

    def charge_battery(self) -> None:
        print(f"Charging {self.battery_capacity}kWh battery.")

    def start_engine(self) -> None:           # override
        print(f"{self.make} {self.model}: electric motor engaged silently.")


class GasCar(Vehicle):
    def __init__(self, make: str, model: str, year: int, tank: float):
        super().__init__(make, model, year)
        self.fuel_tank_size = tank

    def fill_tank(self) -> None:
        print(f"Filling {self.fuel_tank_size}L tank.")

    def start_engine(self) -> None:           # override
        print(f"{self.make} {self.model}: combustion engine roaring.")
```

---

## 4. Types of Inheritance

### Single Inheritance

```mermaid
flowchart LR
    Vehicle --> ElectricCar

    style Vehicle fill:#1F4E79,color:#fff
    style ElectricCar fill:#2E75B6,color:#fff
```

One parent, one child. Most common pattern. Supported by all languages.

---

### Multi-level Inheritance

```mermaid
flowchart LR
    Vehicle --> Car --> ElectricCar

    style Vehicle fill:#1F4E79,color:#fff
    style Car fill:#2E75B6,color:#fff
    style ElectricCar fill:#4472C4,color:#fff
```

Each level adds more specialisation. Keep chains shallow — 2–3 levels max.

---

### Hierarchical Inheritance

```mermaid
classDiagram
    class Vehicle

    class ElectricCar {
        -battery_capacity: int
    }
    class GasCar {
        -fuel_tank_size: float
    }
    class HybridCar {
        -battery_capacity: int
        -fuel_tank_size: float
    }
    class TeslaModel3
    class NissanLeaf

    Vehicle <|-- ElectricCar
    Vehicle <|-- GasCar
    Vehicle <|-- HybridCar
    ElectricCar <|-- TeslaModel3
    ElectricCar <|-- NissanLeaf
```

Multiple children from one parent. Very common, perfectly natural.

---

### Multiple Inheritance — The Diamond Problem

```mermaid
classDiagram
    class Vehicle {
        +start()
    }
    class Machine {
        +start()
    }
    class ElectricCar {
        +start()
    }

    Vehicle <|-- ElectricCar
    Machine <|-- ElectricCar
```

```mermaid
flowchart TD
    EC[ElectricCar.start?] --> V[Vehicle.start] & M[Machine.start]
    V & M --> Q["❓ Which one runs?"]

    style Q fill:#C00000,color:#fff
```

| Language | How it handles it |
|---|---|
| Python | MRO — C3 linearization (left to right) |
| C++ | Virtual inheritance (complex) |
| Java / C# | Not allowed — single class inheritance only |

---

## 5. When to Use Inheritance

```mermaid
flowchart TD
    Q1{"Is it an\nis-a relationship?"}
    Q1 -->|Yes| Q2{"Does parent define\nshared logic children need?"}
    Q1 -->|No| COMP["Use Composition\n(has-a / uses-a)"]

    Q2 -->|Yes| Q3{"Hierarchy shallow?\n2-3 levels"}
    Q2 -->|No| COMP

    Q3 -->|Yes| USE["✅ Use Inheritance"]
    Q3 -->|No| COMP

    style USE fill:#375623,color:#fff
    style COMP fill:#2E75B6,color:#fff
```

### Use Inheritance When
- Clear **is-a** relationship → `Dog` is an `Animal`, `Admin` is a `User`
- Parent has common behaviour children should share
- Hierarchy is **shallow** (2–3 levels max)
- Child does not violate behaviour expected from parent

### Avoid Inheritance When
- Relationship is **has-a** or **uses-a** → `Car` *has* an `Engine`, not *is* an `Engine`
- You need runtime flexibility to swap behaviour → use composition + dependency injection
- You want to combine behaviours from multiple sources dynamically
- Deep hierarchy that makes changes risky

> When in doubt, **start with composition**. Refactoring composition → inheritance is easy. Untangling a deep inheritance tree is not.

---

## 6. Inheritance vs Composition

```mermaid
flowchart LR
    subgraph Inheritance["Inheritance (is-a)"]
        I1[ElectricCar] -->|extends| I2[Vehicle]
    end

    subgraph Composition["Composition (has-a)"]
        C1[Car] -->|has a| C2[Engine]
        C1 -->|has a| C3[Logger]
    end
```

---

## 7. Practical Example — Notification System

```mermaid
classDiagram
    class Notification {
        <<abstract>>
        #recipient: str
        #message: str
        #timestamp: str
        +send()*
        +format_header() str
    }

    class EmailNotification {
        -subject: str
        +send()
    }

    class SMSNotification {
        -max_chars: int
        +send()
    }

    class PushNotification {
        -device_token: str
        -priority: str
        +send()
    }

    Notification <|-- EmailNotification
    Notification <|-- SMSNotification
    Notification <|-- PushNotification
```

### Notification Flow

```mermaid
flowchart TD
    N[Notification created] --> FH[format_header\ninherited from base]
    FH --> Branch{Which type?}
    Branch -->|Email| E["add subject\nsend via SMTP"]
    Branch -->|SMS| S["truncate to 160 chars\nsend via carrier"]
    Branch -->|Push| P["attach device token\nsend via FCN"]
    E & S & P --> Done[Delivered ✅]
```

```python
from abc import ABC, abstractmethod
from datetime import datetime

class Notification(ABC):
    def __init__(self, recipient: str, message: str):
        self.recipient = recipient
        self.message = message
        self.timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    # concrete — shared by all channels
    def format_header(self) -> str:
        return f"[{self.timestamp}] To: {self.recipient}"

    @abstractmethod
    def send(self) -> None:
        pass


class EmailNotification(Notification):
    def __init__(self, recipient: str, message: str, subject: str):
        super().__init__(recipient, message)
        self.subject = subject

    def send(self) -> None:
        print(f"{self.format_header()} | Subject: {self.subject}")
        print(f"Body: {self.message}")


class SMSNotification(Notification):
    MAX_CHARS = 160

    def send(self) -> None:
        truncated = self.message[:self.MAX_CHARS]
        print(f"{self.format_header()} | SMS: {truncated}")


class PushNotification(Notification):
    def __init__(self, recipient: str, message: str, device_token: str, priority: str = "normal"):
        super().__init__(recipient, message)
        self.device_token = device_token
        self.priority = priority

    def send(self) -> None:
        print(f"{self.format_header()} | Device: {self.device_token} | Priority: {self.priority}")
        print(f"Payload: {self.message}")


# Adding a new channel — zero changes to existing code
class SlackNotification(Notification):
    def __init__(self, recipient: str, message: str, webhook_url: str):
        super().__init__(recipient, message)
        self.webhook_url = webhook_url

    def send(self) -> None:
        print(f"{self.format_header()} | Webhook: {self.webhook_url}")
        print(f"Slack message: {self.message}")


# All treated as Notification — polymorphism
notifications = [
    EmailNotification("doctor@hospital.com", "Report ready", "Patient Report"),
    SMSNotification("+1234567890", "Your appointment is confirmed for tomorrow at 10am."),
    PushNotification("patient_01", "Take your medication", "TOKEN-XYZ", "high"),
]

for n in notifications:
    n.send()   # each calls its own send() — parent reference, child behaviour
```

---

## Quick Reference

| Concept | What it means | Example |
|---|---|---|
| Single inheritance | One parent, one child | `ElectricCar(Vehicle)` |
| Multi-level | Chain of parent → child → grandchild | `Vehicle → Car → ElectricCar` |
| Hierarchical | Multiple children, one parent | `Email, SMS, Push all extend Notification` |
| Multiple | One child, multiple parents | `class D(B, C)` — Python only via MRO |
| `super()` | Call parent class method | `super().__init__(...)` |
| Override | Redefine parent method in child | `start_engine()` in `ElectricCar` |
| Extend | Add new methods/fields in child | `charge_battery()` in `ElectricCar` |

---

## Summary — All Six OOP Concepts

```mermaid
mindmap
  root((OOP))
    Classes & Objects
      Class = blueprint
      Object = instance
      Independent state per object
    Enums
      Fixed set of named constants
      str Enum for serialization
      int Enum for ordering
      MRO = left to right lookup
    Interfaces
      Contract only
      Enables polymorphism
      Dependency injection
    Encapsulation
      Data hiding
      Controlled access via methods
      Protects internal state
    Abstraction
      Hides complexity
      Abstract class = contract + shared logic
      Public API = clean surface
    Inheritance
      is-a relationship
      DRY — write once reuse everywhere
      Override to specialise
      Extend to add new behaviour
      Prefer composition for has-a
```

> **Classes** give structure. **Enums** give safe constants. **Interfaces** give contracts. **Encapsulation** protects state. **Abstraction** simplifies complexity. **Inheritance** enables reuse.