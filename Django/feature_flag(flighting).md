# Feature Flags / Flighting Service — Notes

---

## What Is It?

A system to turn features on/off without a deploy. Instead of shipping a feature to 100% of users and hoping nothing breaks, you do a controlled rollout.

```mermaid
flowchart TD
    A[Deploy code with flag check] --> B[Flag created — OFF by default]
    B --> C[Enable for 10% of users]
    C --> D{Monitor errors}
    D -->|All good| E[Bump to 50%]
    D -->|Bug found| F[Flip OFF instantly — no deploy]
    E --> G[Bump to 100%]
    G --> H[Remove flag from code]
```

---

## Config File vs DB Config vs Flighting Service

```python
# Config file — needs deploy to change
ENABLE_S3_IMAGES = True

# DB config — no deploy, but basic
FeatureFlag.objects.filter(feature="s3_images", workspace_id=2434)

# Flighting service — no deploy, advanced targeting
waffle.flag_is_active(request, "s3_image_flow")
```

| | Config File | DB Config (DIY) | Flighting Service |
|---|---|---|---|
| Needs deploy | ✅ Yes | ❌ No | ❌ No |
| Per-workspace targeting | ❌ Redeploy each time | ✅ Yes | ✅ Yes |
| Percentage rollout | ❌ | ❌ Build it | ✅ Built in |
| Audit log | ❌ | ❌ Build it | ✅ Built in |
| Kill switch | ❌ Redeploy | ⚠️ Go to admin | ✅ One click |
| Scheduling | ❌ | ❌ Build it | ✅ "Enable at 9am Monday" |
| Analytics | ❌ | ❌ Build it | ✅ "Feature X increased errors by 3%" |
| Cross-service | ❌ One config per service | ❌ One DB per service | ✅ Unified |

**DB config is basically a homegrown flighting service.** The question is: build vs buy.

---

## Common Targeting Strategies

```
By percentage:    "Enable for 10% of all users"
By user:          "Enable only for john_doe"
By workspace:     "Enable for workspace 2434 only"
By group:         "Enable for QA team"
By region:        "Enable for Southeast Asia sites first"
By environment:   "Enable in staging, not production"
```

---

## Django Waffle

Django-specific feature flag library. Stores flags in its own DB tables.

### Setup (3 steps)

```bash
pip install django-waffle
```

```python
# settings.py
INSTALLED_APPS = [..., "waffle"]
MIDDLEWARE = [..., "waffle.middleware.WaffleMiddleware"]
```

```bash
python manage.py migrate
```

### Usage

```python
import waffle

if waffle.flag_is_active(request, "s3_image_flow"):
    new_flow()
else:
    old_flow()
```

### What Waffle Creates in DB

Waffle ships its own Django models. After `migrate`, three tables are created:

```
Django Admin
├── Waffle
│   ├── Flags      ← per-user targeting (percentage, groups, specific users)
│   ├── Switches   ← global on/off (no user logic)
│   └── Samples    ← random percentage per request (not per user)
```

The `Flag` model (simplified):

```python
# waffle/models.py — you don't write this, Waffle ships it
class Flag(models.Model):
    name = models.CharField(max_length=100, unique=True)
    everyone = models.NullBooleanField()       # True/False/None
    percent = models.DecimalField(null=True)    # 0-100
    superusers = models.BooleanField(default=True)
    staff = models.BooleanField(default=False)
    users = models.ManyToManyField(User, blank=True)
    groups = models.ManyToManyField(Group, blank=True)

    class Meta:
        db_table = "waffle_flag"
```

### What `flag_is_active` Does Internally

```python
# waffle/__init__.py (simplified)
def flag_is_active(request, flag_name):
    flag = Flag.objects.get(name=flag_name)  # DB lookup

    if flag.everyone is True:
        return True
    if flag.everyone is False:
        return False
    if request.user in flag.users.all():
        return True
    if flag.percent and flag.percent > 0:
        hash_value = stable_hash(request.user.id) % 100
        return hash_value < flag.percent

    return False
```

```mermaid
flowchart TD
    A["waffle.flag_is_active(request, 'feature_x')"] --> B[Get flag from DB]
    B --> C{everyone = True?}
    C -->|Yes| D[Return True]
    C -->|No| E{User in flag.users?}
    E -->|Yes| D
    E -->|No| F{User's group in flag.groups?}
    F -->|Yes| D
    F -->|No| G{percent set?}
    G -->|Yes| H["hash(user.id) % 100 < percent?"]
    H -->|Yes| D
    H -->|No| I[Return False]
    G -->|No| I
```

It needs `request` only to get `request.user` — that's how it knows which user to hash.

### How Percentage Rollout Works

Hash-based, **per USER** not per request. Same user always gets the same result:

```mermaid
flowchart TD
    A[User A logs in] --> B["hash(user_A.id) % 100 = 23"]
    B --> C{23 < 50?}
    C -->|Yes| D["New flow ✅ — EVERY request"]

    E[User B logs in] --> F["hash(user_B.id) % 100 = 78"]
    F --> G{78 < 50?}
    G -->|No| H["Old flow ❌ — EVERY request"]
```

**Why not random per request?**

```
Random per request:
  User A → request 1 → new flow  ✅
  User A → request 2 → old flow  ❌  ← confusing!
  User A → request 3 → new flow  ✅

Hash per user:
  User A → ALWAYS new flow  ✅ (consistent experience)
```

**Bumping percentage only adds users, never removes:**

```
At 50%:  User A (hash=23) → in    User B (hash=78) → out
At 75%:  User A (hash=23) → still in    User B (hash=78) → still out
At 80%:  User A (hash=23) → still in    User B (hash=78) → NOW in ✅
```

### What's Stored vs Computed

```mermaid
flowchart LR
    subgraph "Stored in DB (survives restart)"
        A[Flag name]
        B[Percent: 50]
        C[Users M2M]
        D[Groups M2M]
        E["Everyone: True/False/Null"]
    end
    subgraph "Computed at Runtime (stateless)"
        F["hash(user.id) % 100"]
        G["< percent? → True/False"]
    end
    A --> F
    B --> G
    F --> G
```

No per-user results stored. Hash is computed on every request. System restart doesn't matter — same input, same output.

**Python's `hash()` changes across restarts** (security feature). So Waffle uses stable hashing (MD5/CRC32):

```python
# Bad — changes every restart
hash("user_42")  # different each time

# Good — always the same
import hashlib
int(hashlib.md5(b"user_42").hexdigest(), 16) % 100  # always 67
```

### Use One Flag Across Multiple Views

```python
# One flag, multiple views — user gets consistent experience everywhere
def upload_image(request):
    if waffle.flag_is_active(request, FeatureFlags.S3_IMAGE_FLOW):
        upload_to_s3()
    else:
        store_as_base64()

def resolve_image(request):
    if waffle.flag_is_active(request, FeatureFlags.S3_IMAGE_FLOW):
        return presigned_url()
    else:
        return base64_data()

def delete_image(request):
    if waffle.flag_is_active(request, FeatureFlags.S3_IMAGE_FLOW):
        delete_from_s3()
    else:
        remove_from_db()
```

No limit on how many views check the same flag. Each call is a cached DB lookup — negligible cost.

### Define Flag Names as Constants

Avoid magic strings scattered across files:

```python
# feature_flags.py
class FeatureFlags:
    S3_IMAGE_FLOW = "s3_image_flow"
    NEW_LETTER_EDITOR = "new_letter_editor"
    AI_SUGGESTIONS = "ai_suggestions"

# Usage
from your_app.feature_flags import FeatureFlags
waffle.flag_is_active(request, FeatureFlags.S3_IMAGE_FLOW)
```

Without this:

```python
# Typo silently fails — always returns False
waffle.flag_is_active(request, "s3_image_flw")  # ← oops
```

### Creating Flags

Three options:

**Option A — Django admin (most common)**

```
/admin/waffle/flag/add/ → type name → set targeting → save
```

**Option B — Data migration (flag must exist on deploy)**

```python
def create_flags(apps, schema_editor):
    Flag = apps.get_model("waffle", "Flag")
    Flag.objects.get_or_create(
        name="s3_image_flow",
        defaults={"everyone": None, "percent": 0},
    )
```

**Option C — Auto-create missing flags**

```python
# settings.py
WAFFLE_CREATE_MISSING_FLAGS = True   # dev — auto-create on first check
WAFFLE_CREATE_MISSING_FLAGS = False  # prod — explicit only
```

First time code checks a non-existent flag → Waffle creates it (disabled by default) → toggle in admin to enable.

### Without Request (Background Tasks)

Waffle needs `request` to get the user. For Celery tasks, management commands, etc:

```python
# In a view — has request
waffle.flag_is_active(request, "feature_x")

# In a background task — no request
from waffle import flag_is_active_for_user
flag_is_active_for_user(user_obj, "feature_x")

# Global on/off — no user needed
from waffle import switch_is_active
switch_is_active("feature_x")
```

---

## DIY Approach (Per-Workspace Targeting)

Waffle is user-centric. If your rollouts are workspace-centric (like at a multi-tenant SaaS), DIY may be simpler:

```python
# models.py
class FeatureFlag(models.Model):
    name = models.CharField(max_length=100)
    workspace = models.ForeignKey(Workspace, on_delete=models.CASCADE, null=True, blank=True)
    enabled = models.BooleanField(default=False)
    enable_globally = models.BooleanField(default=False)

    class Meta:
        unique_together = ["name", "workspace"]
```

```python
# helpers.py
from django.core.cache import cache

def is_enabled(flag_name, workspace_id=None):
    cache_key = f"ff:{flag_name}:{workspace_id}"
    result = cache.get(cache_key)
    if result is not None:
        return result

    try:
        flag = FeatureFlag.objects.get(name=flag_name, workspace_id=workspace_id)
        result = flag.enabled
    except FeatureFlag.DoesNotExist:
        try:
            flag = FeatureFlag.objects.get(name=flag_name, workspace__isnull=True)
            result = flag.enable_globally
        except FeatureFlag.DoesNotExist:
            result = False

    cache.set(cache_key, result, 60 * 5)
    return result
```

```python
# Usage — works everywhere, no request needed
if is_enabled("s3_image_flow", workspace_id=2434):
    new_flow()
```

Django admin rollout:

```
1. name="s3_image_flow", workspace=2434, enabled=True   → only workspace 2434
2. name="s3_image_flow", workspace=2435, enabled=True   → add 2435
3. name="s3_image_flow", workspace=NULL, enable_globally=True → everyone
```

---

## Unleash (Open Source, Self-Hosted)

Language-agnostic feature management platform. Runs as a separate server.

| | Waffle | Unleash |
|---|---|---|
| Scope | Django only | Any language |
| Architecture | Embedded in app | Separate server + SDK |
| UI | Django admin | Own dashboard |
| Database | Your app's DB | Its own PostgreSQL |
| API | None | Full REST API |
| Best for | Single Django app | Multiple services |

```python
# Django service
from UnleashClient import UnleashClient
client = UnleashClient(url="http://unleash.internal:4242/api", ...)
client.initialize_client()

if client.is_enabled("s3_image_flow"):
    new_flow()
```

```javascript
// Node.js service
const unleash = initialize({ url: 'http://unleash.internal:4242/api', ... });
if (unleash.isEnabled('s3_image_flow')) { newFlow(); }
```

One dashboard controls flags across all services.

```bash
# Self-hosting
docker run -p 4242:4242 \
  -e DATABASE_URL=postgres://user:pass@localhost:5432/unleash \
  unleashorg/unleash-server
```

---

## Decision Tree

```mermaid
flowchart TD
    A{How many services need flags?}
    A -->|Single Django app| B{Need percentage rollout?}
    B -->|No| C["DB model + Django admin\n(DIY approach)"]
    B -->|Yes| D{Per-user or per-workspace?}
    D -->|Per-user| E[Django Waffle]
    D -->|Per-workspace| F["DIY with workspace FK\n(Waffle doesn't support this natively)"]
    A -->|Multiple services / languages| G{Budget for hosted service?}
    G -->|Yes| H[LaunchDarkly]
    G -->|No| I[Unleash — self-hosted]
```