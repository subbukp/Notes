# Python `logging` — Errors + Flow Tracking

---

## Why Not Just `print()`?

`print()` has no level, no timestamp, no module info, can't be turned off in production, can't write to files. Logging gives you all of that.

---

## The 5 Levels

```python
import logging

logging.debug("detailed internal state")     # dev only
logging.info("user logged in")               # normal flow
logging.warning("disk space low")            # something's off
logging.error("failed to save patient")      # something broke
logging.critical("DB is down")               # system-level failure
```

| Level      | When to use                              | Audience         |
|------------|------------------------------------------|------------------|
| `DEBUG`    | internal state, step-by-step details     | development only |
| `INFO`     | normal things happening                  | flow tracking    |
| `WARNING`  | something smells off, but recovered      | not broken yet   |
| `ERROR`    | something broke                          | handle this      |
| `CRITICAL` | system-level failure                     | wake someone up  |

---

## Always Use a Named Logger, Never the Root Logger

```python
# BAD — root logger, shared globally, messy
import logging
logging.info("something")

# GOOD — module-level named logger
import logging
logger = logging.getLogger(__name__)

logger.info("something")
```

`__name__` gives you the full module path like `qure_platform_api.patients.views`  
— so you always know exactly where the log came from.

---

## Basic Setup

```python
import logging

logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s | %(levelname)s | %(name)s | %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)

logger = logging.getLogger(__name__)
logger.info("app started")
```

Output:
```
2024-03-15 10:23:01 | INFO | myapp.views | app started
```

---

## Logging Exceptions — Always Use `exc_info` or `exception()`

```python
try:
    patient = Patient.objects.get(mrn=mrn)
except Patient.DoesNotExist as e:
    # BAD — logs the message, loses the traceback
    logger.error("patient not found")

    # GOOD — logs message + full traceback
    logger.error("patient not found for mrn=%s", mrn, exc_info=True)

    # ALSO GOOD — shorthand for error + exc_info=True
    logger.exception("patient not found for mrn=%s", mrn)
```

> `logger.exception()` is just `logger.error()` with `exc_info=True` baked in.  
> Use it inside `except` blocks.

---

## Flow Tracking with INFO and DEBUG

```python
logger = logging.getLogger(__name__)

def parse_patient_name(fullname):
    logger.debug("parse_patient_name called with: %s", fullname)

    if not fullname:
        logger.warning("empty fullname received, returning empty string")
        return ""

    parts = fullname.split("^")
    logger.debug("split result: %s", parts)

    result = f"{parts[1]} {parts[0]}".strip()
    logger.info("parsed name: %s → %s", fullname, result)
    return result
```

Rule of thumb:
- `DEBUG` → internal state, variable values, step-by-step
- `INFO` → "this thing completed successfully"
- `WARNING` → unexpected input that you recovered from

---

## Structured Logging with Extra Context

```python
logger.info(
    "dicom study ingested",
    extra={
        "patient_id": patient.id,
        "study_uid": study_uid,
        "modality": modality,
    }
)
```

> In Datadog, `extra` fields become searchable attributes —  
> critical for filtering logs by `patient_id`, `study_uid` etc.

---

## Logger Hierarchy

```
root
 └── qure_platform_api
      └── qure_platform_api.patients
           └── qure_platform_api.patients.views
```

Loggers inherit from their parent. Setting level on `qure_platform_api` affects all child loggers automatically.

```python
# silence noisy third-party libraries
logging.getLogger("boto3").setLevel(logging.WARNING)
logging.getLogger("urllib3").setLevel(logging.WARNING)
```

---

## Django Setup (settings.py)

```python
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "verbose": {
            "format": "{asctime} | {levelname} | {name} | {message}",
            "style": "{",
        },
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "verbose",
        },
    },
    "loggers": {
        "qure_platform_api": {          # your app namespace
            "handlers": ["console"],
            "level": "DEBUG",
            "propagate": False,
        },
        "django": {
            "handlers": ["console"],
            "level": "WARNING",         # suppress django's noise
        },
    },
}
```

---

## Common Mistakes

```python
# BAD — string formatting before logging (wasteful if level is off)
logger.debug("patient data: " + str(patient))

# GOOD — lazy formatting, only runs if DEBUG is enabled
logger.debug("patient data: %s", patient)

# BAD — losing exception context
except Exception as e:
    logger.error(str(e))              # just the message, no traceback

# GOOD
except Exception as e:
    logger.exception("failed to process study")   # full traceback
```

---

## Quick Reference

```
getLogger(__name__)    →  always use this, never root logger
logger.debug()         →  internal state, dev only
logger.info()          →  flow tracking, happy path
logger.warning()       →  recovered from something unexpected
logger.error()         →  something broke, needs attention
logger.exception()     →  same as error but includes traceback
exc_info=True          →  attach traceback to any level
extra={}               →  structured fields for Datadog/log search
```
