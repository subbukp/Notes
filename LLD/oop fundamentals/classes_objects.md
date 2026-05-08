# OOP — Classes & Objects

## The Core Question
Every OOP system starts with: **How do I represent real-world entities in code?**  
Classes and Objects are the answer.

---

## 1. What is a Class?

A **class** is a blueprint/template for creating objects. It defines:
- **Attributes** — the data/state an object holds
- **Methods** — the actions/behaviour an object can perform

> A class is not an object itself. It's the recipe, not the cake.

### Key Characteristics
- Groups related data and actions together
- Can be instantiated into many independent objects
- Each object shares the same structure but has its own state

### Example — `MedicalScan` class

```python
class MedicalScan:
    def __init__(self, scan_id: str, patient_id: str, scan_type: str):
        self.scan_id = scan_id
        self.patient_id = patient_id
        self.scan_type = scan_type       # e.g. "X-Ray", "CT", "MRI"
        self.results = {}
        self.is_processed = False

    def mark_processed(self, results: dict) -> None:
        self.results = results
        self.is_processed = True

    def display_status(self) -> None:
        status = "Processed" if self.is_processed else "Pending"
        print(f"[{self.scan_id}] Patient: {self.patient_id} | Status: {status}")
```

The `MedicalScan` class defines what every scan object looks like and what it can do — but it's just a definition until you instantiate it.

---

## 2. What is an Object?

An **object** is an instance of a class — the actual thing you interact with.

```python
scan_a = MedicalScan("SCAN-001", "PAT-101", "X-Ray")
scan_b = MedicalScan("SCAN-002", "PAT-102", "CT")

scan_a.mark_processed({"findings": ["pleural effusion"], "confidence": 0.91})

scan_a.display_status()  # [SCAN-001] Patient: PAT-101 | Status: Processed
scan_b.display_status()  # [SCAN-002] Patient: PAT-102 | Status: Pending
```

> `scan_a` and `scan_b` are both `MedicalScan` objects — same structure, completely independent state.

---

## 3. Practical Example — Hospital User Access

### The Scenario
A hospital portal needs to manage user access. Each user belongs to a department, has a role, and can only access records within their permitted scope. Once a user is deactivated, they should not be able to access any records.

**Without classes** — you'd track user IDs, departments, roles, and active status in separate dicts/arrays with no clean way to enforce rules like "deactivated users cannot access records."

**With classes** — the `HospitalUser` owns its data and enforces the rules internally.

```python
class HospitalUser:
    def __init__(self, user_id: str, department: str, role: str):
        self.user_id = user_id
        self.department = department     # e.g. "Radiology", "Cardiology"
        self.role = role                 # e.g. "doctor", "admin", "technician"
        self.is_active = True
        self.accessible_records = []

    def grant_access(self, record_id: str) -> None:
        if not self.is_active:
            print(f"User {self.user_id} is deactivated. Access denied.")
            return
        self.accessible_records.append(record_id)
        print(f"Access granted to {record_id} for {self.user_id}")

    def deactivate(self) -> None:
        self.is_active = False
        self.accessible_records = []
        print(f"User {self.user_id} deactivated.")


# Usage
doctor = HospitalUser("USR-001", "Radiology", "doctor")
doctor.grant_access("SCAN-001")  # ✅ Access granted

doctor.deactivate()
doctor.grant_access("SCAN-002")  # ❌ Access denied
```

### Why This Design Works
- **Encapsulates state** — department, role, active status, and accessible records live together
- **Enforces business rules** — `grant_access()` checks `is_active` before allowing access
- **Reusable** — one class handles every user across all departments
- **Easy to extend** — need to add MFA, audit log, or login history later? Just add new fields/methods

---

## Quick Reference

| Concept | Definition | Example |
|---|---|---|
| Class | Blueprint/template | `MedicalScan`, `HospitalUser` |
| Object | Instance of a class | `scan_a`, `doctor` |
| Attribute | Data the object holds | `scan_id`, `department`, `role` |
| Method | Action the object can do | `mark_processed()`, `deactivate()` |
| Instantiation | Creating an object from a class | `MedicalScan("SCAN-001", "PAT-101", "X-Ray")` |

---

## Key Takeaway

> A class is the definition. An object is the thing.  
> Each object has its own state but shares the same structure and behaviour as every other object created from that class.