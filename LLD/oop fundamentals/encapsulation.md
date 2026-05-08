# OOP — Encapsulation

## The Core Idea

> Encapsulation = Data hiding + Controlled access

Group data and behaviour into a single unit and **restrict direct access** to internal details. External code only sees what it needs to.

---

## 1. Real-World Analogy — ATM

```mermaid
flowchart LR
    User -->|deposit| ATM
    User -->|withdraw| ATM
    User -->|checkBalance| ATM
    ATM -->|controlled access| Vault[(balance\nprivate)]

    style Vault fill:#1F4E79,color:#fff
    style ATM fill:#2E75B6,color:#fff
```

You don't walk into the vault and change numbers yourself. You interact through a well-defined interface — the ATM. The bank can change how it stores or validates data internally — none of that affects how you use the ATM.

---

## 2. Why Encapsulation Matters

```mermaid
mindmap
  root((Encapsulation))
    Data Hiding
      Private fields
      No direct external access
    Controlled Access
      Validation in setters
      Business rules enforced
    Maintainability
      Change internals freely
      External code unaffected
    Security
      No invalid states
      Sensitive data protected
```

---

## 3. Access Modifiers

```mermaid
classDiagram
    class BankAccount {
        -balance: float          ← private
        #owner: str              ← protected
        +account_number: str     ← public
        +deposit(amount)
        +withdraw(amount)
        +get_balance()
    }
```

| Modifier | Symbol | Accessible From |
|---|---|---|
| `private` | `__` (dunder) | Same class only |
| `protected` | `_` (single underscore) | Class + subclasses |
| `public` | no prefix | Anywhere |

> **Rule of thumb** — make everything private by default, then selectively expose what needs to be public.

```python
class BankAccount:
    def __init__(self, owner: str):
        self.__balance = 0.0      # private   — no external access
        self._owner = owner       # protected — subclasses can access
        self.account_id = "ACC-1" # public    — anyone can read

account = BankAccount("Alice")
print(account.__balance)   # ❌ AttributeError
print(account._owner)      # ✅ works (but convention says: don't)
print(account.account_id)  # ✅ works
```

---

## 4. Getters and Setters

```mermaid
sequenceDiagram
    participant Caller
    participant BankAccount

    Caller->>BankAccount: set_amount(amount)
    BankAccount->>BankAccount: validate(amount)
    alt invalid
        BankAccount-->>Caller: raise ValueError
    else valid
        BankAccount->>BankAccount: __amount = amount
        BankAccount-->>Caller: OK
    end

    Caller->>BankAccount: get_balance()
    BankAccount-->>Caller: __balance (read-only)
```

```python
class BankAccount:
    def __init__(self):
        self.__balance = 0.0

    # Getter — read-only access
    def get_balance(self) -> float:
        return self.__balance

    # Setter — with validation
    def deposit(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Deposit amount must be positive.")
        self.__balance += amount
```

---

## 5. Complete Example — BankAccount

```mermaid
classDiagram
    class BankAccount {
        <<encapsulated>>
        -__balance: float
        -__owner: str
        +deposit(amount: float)
        +withdraw(amount: float)
        +get_balance() float
    }
```

```python
class BankAccount:
    def __init__(self, owner: str, initial_balance: float = 0.0):
        self.__owner = owner
        self.__balance = initial_balance

    def deposit(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Deposit must be positive.")
        self.__balance += amount
        print(f"Deposited ${amount:.2f}. New balance: ${self.__balance:.2f}")

    def withdraw(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Withdrawal must be positive.")
        if amount > self.__balance:
            raise ValueError("Insufficient funds.")
        self.__balance -= amount
        print(f"Withdrew ${amount:.2f}. Remaining: ${self.__balance:.2f}")

    def get_balance(self) -> float:
        return self.__balance


account = BankAccount("Alice", 100.0)
account.deposit(50.0)      # ✅ Deposited $50.00
account.withdraw(200.0)    # ❌ ValueError: Insufficient funds
account.__balance = 9999   # ❌ AttributeError — cannot reach private field
```

---

## 6. Practical Example — PaymentProcessor

```mermaid
flowchart TD
    Caller -->|"PaymentProcessor('4111...1111', 200)"| Constructor
    Constructor -->|__mask_card| Masking["**** **** **** 1111\n(original discarded)"]
    Masking --> Store["__masked_card stored"]
    Caller -->|process_payment| Process["validate + charge"]
    Process --> Log["logs masked card only\nnever raw number"]

    style Store fill:#1F4E79,color:#fff
    style Masking fill:#2E75B6,color:#fff
```

```python
class PaymentProcessor:
    def __init__(self, card_number: str, amount: float):
        self.__masked_card = self.__mask_card(card_number)  # raw number never stored
        self.__amount = amount

    def __mask_card(self, card_number: str) -> str:
        # private method — caller never needs to know this exists
        return "**** **** **** " + card_number[-4:]

    def process_payment(self) -> None:
        print(f"Processing ${self.__amount:.2f} for card {self.__masked_card}")


processor = PaymentProcessor("4111111111111111", 150.00)
processor.process_payment()
# Processing $150.00 for card **** **** **** 1111

print(processor.__masked_card)   # ❌ AttributeError
processor.__mask_card("...")     # ❌ AttributeError
```

### Why This Works

```mermaid
flowchart LR
    A["Raw card number\nenters constructor"] -->|immediately| B["__mask_card\nprivate method"]
    B --> C["masked string\nstored internally"]
    A -.->|never stored| X["❌ discarded"]
    C --> D["process_payment\npublic method"]
    D --> E["logs masked\ncard only ✅"]

    style X fill:#C00000,color:#fff
    style E fill:#375623,color:#fff
```

---

## 7. Encapsulation vs No Encapsulation

```mermaid
flowchart TD
    subgraph Without ["❌ Without Encapsulation"]
        W1[External code] -->|account.balance = -999| W2[Direct field access]
        W2 --> W3[Invalid state — no validation]
    end

    subgraph With ["✅ With Encapsulation"]
        E1[External code] -->|account.withdraw| E2[Public method]
        E2 -->|validates| E3{amount valid?}
        E3 -->|Yes| E4[Update balance]
        E3 -->|No| E5[Raise ValueError]
    end
```

---

## Quick Reference

| Concept | What it does | Example |
|---|---|---|
| Private field | Hides data from outside | `self.__balance` |
| Protected field | Accessible to subclasses | `self._owner` |
| Getter | Read-only access to private field | `get_balance()` |
| Setter / method | Write access with validation | `deposit()`, `withdraw()` |
| Private method | Internal logic, not part of public API | `__mask_card()` |

---

## Summary — All Four Concepts So Far

```mermaid
mindmap
  root((OOP))
    Classes & Objects
      Class = blueprint
      Object = instance
      Each object has independent state
    Enums
      Fixed set of named constants
      Type-safe
      str Enum for serialization
      int Enum for ordering
      MRO determines method priority
    Interfaces
      Define the what not the how
      Contract between components
      Enables polymorphism
      Promotes loose coupling
      Dependency injection
    Encapsulation
      Data hiding
      Controlled access via methods
      Private fields and methods
      Validation at the boundary
      External code sees only what it needs
```

> **Classes** give you structure. **Enums** give you safe constants. **Interfaces** give you contracts. **Encapsulation** protects your internal state.