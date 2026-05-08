# DB Locks / Concurrency Control

---

## Why This Matters

Two workers read the same row at the same time. Both see `status = "pending"`. Both process it. Both save. You've now processed the same study twice. This is a **race condition** — and it's silent, no exceptions thrown.

Locks are how you prevent this.

---

## Types of Locks in PostgreSQL

**Row-level locks** — lock specific rows, not the whole table.  
**Table-level locks** — lock the entire table. Rare, usually from schema changes.

Day-to-day you only deal with row-level locks.

---

## `select_for_update()` — Pessimistic Locking

"Lock this row now, nobody else can touch it until my transaction ends."

```python
from django.db import transaction

with transaction.atomic():
    # row is locked the moment this query runs
    study = Study.objects.select_for_update().get(uid="1.2.3")

    if study.status == "pending":
        study.status = "processing"
        study.save()

# lock released here when transaction ends
```

Any other worker hitting `select_for_update()` on the same row will **block and wait** until the first transaction finishes.

**Mental model:** A bathroom with one key. You take the key, you're inside, nobody else can enter. You come out, someone else gets the key.

---

## `select_for_update(nowait=True)` — Don't Wait, Just Fail

```python
from django.db.utils import OperationalError

with transaction.atomic():
    try:
        study = Study.objects.select_for_update(nowait=True).get(uid="1.2.3")
    except OperationalError:
        # row is locked by someone else, skip it
        return
```

Instead of waiting, raises immediately if the row is locked. Good for background workers where "skip and retry later" is acceptable.

---

## `select_for_update(skip_locked=True)` — Skip Locked Rows

```python
with transaction.atomic():
    # get next available pending study not locked by another worker
    study = (
        Study.objects
        .select_for_update(skip_locked=True)
        .filter(status="pending")
        .first()
    )

    if study:
        study.status = "processing"
        study.save()
```

This is the **worker queue pattern**. Multiple Celery workers can run this simultaneously — each picks a different row, skips ones already being processed. No double-processing.

---

## Optimistic Locking — `version` Field Pattern

Pessimistic locking blocks workers. Optimistic locking doesn't lock at all — it detects conflicts at save time.

```python
class Study(models.Model):
    status = models.CharField(max_length=50)
    version = models.IntegerField(default=0)

# worker reads the row
study = Study.objects.get(uid="1.2.3")
original_version = study.version

# ... does some work ...

# saves only if nobody else has modified it since
updated = Study.objects.filter(
    uid="1.2.3",
    version=original_version        # version still the same?
).update(
    status="processed",
    version=original_version + 1    # bump the version
)

if updated == 0:
    raise Exception("conflict — someone else modified this row")
```

`update()` returns the number of rows affected. If `0`, someone else changed the row between your read and write — you lost the race, handle it.

**Mental model:** Google Docs conflict detection. Two people editing — last save wins, but you get warned there's a conflict.

---

## Pessimistic vs Optimistic — When to Use Which

```
Pessimistic (select_for_update)
  ✓ high contention — same rows fought over constantly
  ✓ you can't afford to retry on conflict
  ✗ workers wait/block — throughput drops under load
  ✗ risk of deadlocks if not careful

Optimistic (version field)
  ✓ low contention — conflicts are rare
  ✓ high throughput — no blocking
  ✗ you need retry logic for conflicts
  ✗ more code to manage
```

> In medical imaging pipelines — `skip_locked=True` is almost always  
> the right call for study/task queues.

---

## Deadlocks — And How to Avoid Them

A deadlock happens when two transactions are each waiting for the other to release a lock.

```
Worker A locks Study 1, wants Study 2
Worker B locks Study 2, wants Study 1
Both waiting forever → PostgreSQL kills one with DeadlockDetected
```

Fix — always acquire locks in the **same order**:

```python
# BAD — different workers lock in different orders
# Worker A: study_1 then study_2
# Worker B: study_2 then study_1 → deadlock

# GOOD — enforce consistent ordering
studies = Study.objects.select_for_update().filter(
    id__in=[id1, id2]
).order_by("id")   # consistent order, deadlock impossible
```

---

## Advisory Locks — PostgreSQL-Level Named Locks

For locking things that aren't DB rows — like "only one worker should run this job" or "only one sync per tenant at a time."

```python
from django.db import connection

def run_with_advisory_lock(lock_id, func):
    with connection.cursor() as cursor:
        cursor.execute("SELECT pg_try_advisory_lock(%s)", [lock_id])
        acquired = cursor.fetchone()[0]

        if not acquired:
            return  # someone else has this lock, skip

        try:
            func()
        finally:
            cursor.execute("SELECT pg_advisory_unlock(%s)", [lock_id])
```

Good for Celery beat jobs where you don't want two pods running the same scheduled task simultaneously.

---

## Common Mistakes

```python
# BAD — select_for_update outside a transaction does nothing
study = Study.objects.select_for_update().get(uid="1.2.3")
# no atomic() = no transaction = lock never acquired

# GOOD
with transaction.atomic():
    study = Study.objects.select_for_update().get(uid="1.2.3")

# BAD — lock held too long
with transaction.atomic():
    studies = Study.objects.select_for_update().filter(status="pending")
    time.sleep(10)          # lock held for 10s, everything else blocked
    for s in studies:
        process(s)

# GOOD — lock per unit, not bulk
for study in Study.objects.filter(status="pending"):
    with transaction.atomic():
        s = Study.objects.select_for_update(skip_locked=True).get(id=study.id)
        process(s)
```

---

## Quick Reference

```
select_for_update()                 →  lock row, others wait
select_for_update(nowait=True)      →  lock row, others get OperationalError
select_for_update(skip_locked=True) →  skip locked rows, worker queue pattern
version field + filter().update()   →  optimistic lock, no blocking
order_by() on lock queries          →  prevent deadlocks
pg_try_advisory_lock                →  named locks for non-row resources
```

---

## Mental Model Summary

```
No lock             →  race condition, silent double-processing
select_for_update   →  one at a time, others wait        (pessimistic)
skip_locked         →  each worker grabs a different row (queue)
nowait              →  try to lock, fail fast if can't
version field       →  no blocking, detect conflict at save time (optimistic)
advisory lock       →  lock on a concept, not a row
```
