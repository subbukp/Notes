# try/except — Error Handling in Python

---

## Basic Structure

```python
try:
    result = 10 / 0
except ZeroDivisionError:
    print("can't divide by zero")
```

Only catch what you expect. Never do `except Exception` blindly — you'll swallow bugs you didn't intend to.

---

## All the Blocks

```python
try:
    result = int("abc")
except ValueError as e:
    print(f"bad value: {e}")      # runs if ValueError raised
except TypeError as e:
    print(f"bad type: {e}")       # runs if TypeError raised
else:
    print("success")              # runs only if NO exception
finally:
    print("always runs")          # cleanup — always executes
```

- `else` — runs only if **nothing went wrong** in `try`
- `finally` — runs **always**, used for cleanup (files, DB connections, locks)

---

## Catching Multiple Exceptions

```python
# same handler for both
except (ValueError, TypeError) as e:
    print(f"bad input: {e}")
```

---

## Re-raising

```python
try:
    connect_to_db()
except ConnectionError as e:
    log_error(e)
    raise           # re-raises the same exception up the chain
```

Use bare `raise` to preserve the original traceback.  
**Don't** do `raise e` — that resets the traceback origin.

---

## Wrapping & Re-raising as a Different Exception

```python
try:
    raw = fetch_dicom_header()
except KeyError as e:
    raise ValueError(f"Missing DICOM field: {e}") from e
```

`from e` chains the exceptions — both appear in the traceback.  
Good for translating low-level errors into domain-level errors.

---

## Custom Exceptions

```python
class PatientNotFoundError(Exception):
    pass

class InvalidDICOMNameError(ValueError):
    pass

# raise it
raise PatientNotFoundError(f"No patient with MRN {mrn}")

# catch it
try:
    get_patient(mrn)
except PatientNotFoundError:
    return 404
```

- Inherit from `ValueError`, `RuntimeError` etc. when your error *is a kind of* that standard error
- Inherit from `Exception` for domain-specific ones with no standard equivalent

---

## What NOT to Do

```python
# BAD — swallows everything including real bugs
try:
    do_something()
except:
    pass

# BAD — too broad
try:
    do_something()
except Exception:
    pass

# BAD — hides root cause
except SomeError as e:
    raise SomeOtherError("failed") from None
```

---

## Mental Model

```
try      →  "attempt this risky thing"
except   →  "if THIS specific thing goes wrong, handle it"
else     →  "if nothing went wrong, do this"
finally  →  "no matter what, always do this"
```

`finally` is a **contract** — "I promise this will run."  
Used heavily with DB transactions, file handles, and locks.

---

## In Django / Backend Context

```python
try:
    patient = Patient.objects.get(mrn=mrn)
except Patient.DoesNotExist:
    return Response({"error": "not found"}, status=404)
except Patient.MultipleObjectsReturned:
    return Response({"error": "data integrity issue"}, status=500)
```

> Django ORM exceptions live on the model class itself —
> `Patient.DoesNotExist`, not `django.exceptions.DoesNotExist`

---

## Quick Reference

| Block     | When it runs                        | Use for                        |
|-----------|-------------------------------------|--------------------------------|
| `try`     | always                              | the risky operation            |
| `except`  | only if specified exception raised  | handling the error             |
| `else`    | only if NO exception raised         | success-path logic             |
| `finally` | always, no matter what              | cleanup (files, locks, DB)     |
