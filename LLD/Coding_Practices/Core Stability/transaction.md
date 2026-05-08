# DB Transactions — Consistency in Django

---

## What is a Transaction?

A transaction is a group of DB operations that either **all succeed** or **all fail together**. No partial states.

**Mental model:** Bank transfer. You debit account A and credit account B. If the credit fails after the debit, you've lost money. A transaction ensures both happen or neither happens.

---

## `atomic()` — The Core Tool

```python
from django.db import transaction

# as a context manager
with transaction.atomic():
    patient = Patient.objects.create(mrn="123", name="John")
    study = Study.objects.create(patient=patient, uid="1.2.3")
    # if Study.create fails, Patient.create is also rolled back
```

```python
# as a decorator
@transaction.atomic
def create_patient_with_study(data):
    patient = Patient.objects.create(**data["patient"])
    study = Study.objects.create(patient=patient, **data["study"])
    return patient
```

If any exception is raised inside `atomic()`, everything rolls back automatically.

---

## Nesting — Savepoints

`atomic()` can be nested. Inner blocks create **savepoints** — you can roll back just the inner block without losing the outer transaction.

```python
with transaction.atomic():
    patient = Patient.objects.create(mrn="123")

    try:
        with transaction.atomic():      # savepoint created here
            study = Study.objects.create(uid="bad_uid")
    except IntegrityError:
        pass    # inner rolled back, patient still saved

    # patient survives, study didn't
    Audit.objects.create(patient=patient, note="study creation failed")
```

---

## `on_commit()` — Run Code Only After Transaction Commits

Critical for side effects like sending emails, triggering Celery tasks, or pushing to external APIs.

```python
from django.db import transaction

def create_patient(data):
    with transaction.atomic():
        patient = Patient.objects.create(**data)

        # BAD — fires even if transaction rolls back
        send_welcome_email(patient.id)

        # GOOD — only fires after successful commit
        transaction.on_commit(
            lambda: send_welcome_email(patient.id)
        )
```

> Use `on_commit()` for any side effect that can't be undone —  
> emails, Celery tasks, external API calls, webhooks.

---

## `select_for_update()` — Lock Rows During Transaction

Prevents race conditions when multiple workers read-then-write the same row.

```python
with transaction.atomic():
    # locks this row until transaction ends
    patient = Patient.objects.select_for_update().get(mrn="123")
    patient.status = "processing"
    patient.save()
    # another worker trying to get this row will WAIT here
```

Without `select_for_update()`, two workers can read the same row simultaneously and overwrite each other's changes.

---

## When Transactions Fail Silently — The Broken Connection Problem

```python
# BAD — exception swallowed inside atomic(), transaction is now broken
with transaction.atomic():
    try:
        Patient.objects.create(mrn=duplicate_mrn)  # IntegrityError
    except IntegrityError:
        pass  # swallowed — but transaction is now in a broken state

    # this will ALSO fail with TransactionManagementError
    Audit.objects.create(note="attempted duplicate")
```

Fix: use a nested `atomic()` to isolate the risky operation.

```python
with transaction.atomic():
    try:
        with transaction.atomic():      # savepoint isolates the error
            Patient.objects.create(mrn=duplicate_mrn)
    except IntegrityError:
        pass  # savepoint rolled back, outer transaction still healthy

    Audit.objects.create(note="attempted duplicate")  # works fine
```

---

## `ATOMIC_REQUESTS` — Wrap Every Request in a Transaction

In `settings.py`:

```python
DATABASES = {
    "default": {
        ...
        "ATOMIC_REQUESTS": True,   # every HTTP request is one transaction
    }
}
```

Every view is automatically wrapped in `atomic()`. If the view raises an exception, the whole request rolls back.

> Be careful with long requests — long transactions hold DB locks.  
> This is how you get `idle in transaction` sessions piling up.

---

## Common Mistakes

```python
# BAD — irreversible side effect inside atomic
with transaction.atomic():
    patient.save()
    requests.post("https://external-api.com/notify")  # can't undo this
    # DB rolls back but the API call already happened

# GOOD — defer to on_commit
with transaction.atomic():
    patient.save()
    transaction.on_commit(
        lambda: requests.post("https://external-api.com/notify")
    )

# BAD — transaction wrapping too much (holds locks too long)
with transaction.atomic():
    for study in Study.objects.all():    # could be thousands
        process_and_save(study)          # locks rows for the whole loop

# GOOD — transaction per unit of work
for study in Study.objects.all():
    with transaction.atomic():
        process_and_save(study)          # lock held briefly per study
```

---

## Quick Reference

```
atomic()                →  wrap operations, auto rollback on exception
atomic() nested         →  creates a savepoint, partial rollback possible
on_commit()             →  run side effects only after successful commit
select_for_update()     →  lock rows to prevent race conditions
ATOMIC_REQUESTS=True    →  wrap every HTTP request in a transaction
```

---

## Mental Model Summary

```
No transaction   →  each query is independent, partial failures possible
atomic()         →  all or nothing — success commits, exception rolls back
savepoint        →  checkpoint inside a transaction, inner-only rollback
on_commit()      →  "only do this if everything went through"
select_for_update→  "nobody else touch this row until I'm done"
```
