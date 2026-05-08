# OOP — Polymorphism

## The Core Idea

> Polymorphism = Same method name, different behaviour depending on the object invoking it.

Write code that targets a **common type**. The actual behaviour is determined by the **concrete implementation**.

---

## 1. Real-World Analogy — Universal Remote

```mermaid
flowchart LR
    Remote["Universal Remote\n(same interface)"] -->|powerOn| TV
    Remote -->|powerOn| AC[Air Conditioner]
    Remote -->|powerOn| Proj[Projector]

    TV -->|turns on screen| R1[Display image]
    AC -->|starts cooling| R2[Adjust temperature]
    Proj -->|warms up lamp| R3[Project image]

    style Remote fill:#1F4E79,color:#fff
```

Same buttons. Different behaviour per device. The interface never changes — the receiver decides what happens.

---

## 2. Why Polymorphism Matters

```mermaid
mindmap
  root((Polymorphism))
    Loose Coupling
      Interact with abstractions
      Not specific implementations
    Flexibility
      Add new behaviours
      Without modifying existing code
      Open-Closed Principle
    Scalability
      Grow features
      Minimal impact on existing code
    Extensibility
      Plug in new implementations
      Core logic stays untouched
```

---

## 3. Two Forms of Polymorphism

```mermaid
flowchart TD
    P[Polymorphism] --> CT[Compile-time\nMethod Overloading]
    P --> RT[Runtime\nMethod Overriding]

    CT --> CT1[Same method name\ndifferent parameters]
    CT --> CT2[Resolved before\nprogram runs]

    RT --> RT1[Child overrides\nparent method]
    RT --> RT2[Resolved while\nprogram runs]

    style RT fill:#1F4E79,color:#fff
    style CT fill:#2E75B6,color:#fff
```

---

## 4. Compile-time Polymorphism — Method Overloading

Same method name, different parameter lists. Compiler picks the right one **before the program runs**.

```mermaid
sequenceDiagram
    participant Caller
    participant Calculator

    Caller->>Calculator: add(1, 2)
    Calculator-->>Caller: int 3

    Caller->>Calculator: add(1.5, 2.5)
    Calculator-->>Caller: float 4.0

    Caller->>Calculator: add(1, 2, 3)
    Calculator-->>Caller: int 6

    Note over Calculator: Decision made at compile time
```

```python
# Python doesn't have native overloading
# but achieves it via default args or type checks

class Calculator:
    def add(self, a, b, c=None):
        if c is not None:
            return a + b + c
        return a + b

calc = Calculator()
print(calc.add(1, 2))       # 3
print(calc.add(1.5, 2.5))   # 4.0
print(calc.add(1, 2, 3))    # 6
```

> Python resolves this at **runtime** via duck typing. Java/C++ do true compile-time overloading with separate method signatures.

---

## 5. Runtime Polymorphism — Method Overriding

Child class overrides a parent method. The decision of **which version runs** is made at runtime based on the actual object type.

```mermaid
classDiagram
    class Notification {
        <<abstract>>
        #recipient: str
        #message: str
        +send()*
        +format_header() str
    }

    class EmailNotification {
        -subject: str
        +send()
    }

    class SMSNotification {
        -phone_number: str
        +send()
    }

    class PushNotification {
        -device_token: str
        +send()
    }

    Notification <|-- EmailNotification
    Notification <|-- SMSNotification
    Notification <|-- PushNotification
```

### Dynamic Dispatch Flow

```mermaid
sequenceDiagram
    participant Loop
    participant Ref as "Notification\n(reference)"
    participant Actual as "Actual object\nat runtime"

    Loop->>Ref: call send()
    Ref->>Actual: which type am I?
    Actual-->>Ref: EmailNotification
    Ref->>Actual: invoke EmailNotification.send()
    Actual-->>Loop: sends via SMTP

    Loop->>Ref: call send()
    Ref->>Actual: which type am I?
    Actual-->>Ref: SMSNotification
    Ref->>Actual: invoke SMSNotification.send()
    Actual-->>Loop: sends via carrier
```

```python
from abc import ABC, abstractmethod
from datetime import datetime

class Notification(ABC):
    def __init__(self, recipient: str, message: str):
        self.recipient = recipient
        self.message = message

    def format_header(self) -> str:
        ts = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        return f"[{ts}] To: {self.recipient}"

    @abstractmethod
    def send(self) -> None:
        pass


class EmailNotification(Notification):
    def __init__(self, recipient: str, message: str, subject: str):
        super().__init__(recipient, message)
        self.subject = subject

    def send(self) -> None:
        print(f"{self.format_header()} | Email | Subject: {self.subject} | {self.message}")


class SMSNotification(Notification):
    def send(self) -> None:
        print(f"{self.format_header()} | SMS | {self.message[:160]}")


class PushNotification(Notification):
    def __init__(self, recipient: str, message: str, device_token: str):
        super().__init__(recipient, message)
        self.device_token = device_token

    def send(self) -> None:
        print(f"{self.format_header()} | Push | Token: {self.device_token} | {self.message}")


# All stored as Notification references — runtime picks the right send()
notifications: list[Notification] = [
    EmailNotification("doctor@hospital.com", "Report ready", "Patient Report"),
    SMSNotification("+1234567890", "Appointment confirmed for tomorrow."),
    PushNotification("patient_01", "Take your medication now.", "TOKEN-XYZ"),
]

for n in notifications:
    n.send()   # same call — different behaviour each time
```

```
[2025-01-01 10:00:00] To: doctor@hospital.com | Email | Subject: Patient Report | Report ready
[2025-01-01 10:00:00] To: +1234567890 | SMS | Appointment confirmed for tomorrow.
[2025-01-01 10:00:00] To: patient_01 | Push | Token: TOKEN-XYZ | Take your medication now.
```

> The variable type says `Notification`. The behaviour says `EmailNotification`, `SMSNotification`, `PushNotification`. That's runtime polymorphism.

---

## 6. Polymorphism with Interfaces vs Abstract Classes

```mermaid
flowchart TD
    Q1{"Are the classes\nfundamentally different\nbut share a capability?"}
    Q1 -->|Yes| IFACE["Use Interface\ncan-do relationship\nSendable, Exportable"]
    Q1 -->|No| Q2{"Are they a family\nsharing common logic?"}
    Q2 -->|Yes| ABS["Use Abstract Class\nis-a relationship\nNotification, MediaPlayer"]
    Q2 -->|No| BOTH["Use Both\nAbstract class for shared logic\nInterface for shared capability"]

    style IFACE fill:#2E75B6,color:#fff
    style ABS fill:#1F4E79,color:#fff
    style BOTH fill:#375623,color:#fff
```

### Side-by-side Comparison

```mermaid
classDiagram
    class Sendable {
        <<interface>>
        +send()
    }

    class Notification {
        <<abstract>>
        #recipient: str
        #message: str
        +format_header() str
        +send()*
    }

    class EmailNotification {
        +send()
    }

    class Invoice {
        +send()
    }

    class Report {
        +send()
    }

    Sendable <|.. Invoice : can send
    Sendable <|.. Report  : can send
    Notification <|-- EmailNotification : is a notification
    EmailNotification ..|> Sendable : also can send
```

| Aspect | Interface | Abstract Class |
|---|---|---|
| Relationship | **can-do** (capability) | **is-a** (family) |
| Shared behaviour | ❌ contract only | ✅ concrete methods + fields |
| Multiple | ✅ implement many | ❌ extend only one |
| Use when | Unrelated classes share a capability | Related classes share logic |
| Example | `Sendable` — Email, Invoice, Report | `Notification` — Email, SMS, Push |

> In practice, **use both** — abstract class for shared logic, interface for shared capability.

---

## 7. Compile-time vs Runtime Summary

```mermaid
flowchart LR
    subgraph Compile["Compile-time (Overloading)"]
        C1["add(int, int)"]
        C2["add(float, float)"]
        C3["add(int, int, int)"]
        C4["Resolved by\ncompiler / signature"]
    end

    subgraph Runtime["Runtime (Overriding)"]
        R1["notification.send()"]
        R2["Actual type checked\nat runtime"]
        R3["EmailNotification.send()\nor SMSNotification.send()\nor PushNotification.send()"]
    end
```

| | Compile-time | Runtime |
|---|---|---|
| Also called | Method Overloading | Method Overriding / Dynamic Dispatch |
| Decided | Before program runs | While program runs |
| Based on | Parameter types/count | Actual object type |
| Requires | Same class | Parent-child relationship |
| Power | Low–Medium | High |

---

## Quick Reference

```mermaid
flowchart TD
    SameMethodName[Same method name called] --> CT{Parameter\ndifference?}
    CT -->|Yes| OL[Overloading\nCompile-time]
    CT -->|No| OR{Child overrides\nparent method?}
    OR -->|Yes| OV[Overriding\nRuntime]
    OR -->|No| NA[Not polymorphism]

    style OL fill:#2E75B6,color:#fff
    style OV fill:#1F4E79,color:#fff
```

---

## Summary — All Seven OOP Concepts

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
      Prefer composition for has-a
    Polymorphism
      Same interface different behaviour
      Compile-time = overloading
      Runtime = overriding
      Write once works for all subtypes
```

> **Classes** give structure. **Enums** give safe constants. **Interfaces** give contracts. **Encapsulation** protects state. **Abstraction** simplifies complexity. **Inheritance** enables reuse. **Polymorphism** makes it all flexible.