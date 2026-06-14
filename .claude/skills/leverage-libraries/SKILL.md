---
name: leverage-libraries
description: Battle-tested Python libraries and stdlib to replace hand-rolled code — pydantic, httpx, attrs/dataclasses, tenacity, cachetools, dateutil/zoneinfo, structlog, pytest, respx, and more. Use when reviewing for reinvented wheels, choosing libraries, or replacing custom utilities.
---

# Leverage Libraries

Don't hand-roll non-differentiating logic when a battle-tested library or stdlib module exists.

## When to Use

- Reviewing code that reimplements common functionality
- Choosing between building vs using a library
- Replacing custom utilities with standard solutions
- Evaluating new dependencies for a project

---

## Quick Reference: Hand-Rolled → Library

| Hand-Rolled Pattern | Use Instead |
|---------------------|-------------|
| Custom string/list utilities | stdlib `str`, `itertools`, `functools` |
| Manual `None` checks everywhere | `X \| None` + walrus, `dict.get(...)` |
| Manual dict ↔ object mapping | `pydantic` / `dataclasses.asdict` |
| Hand-written `__init__`/`__eq__`/`__repr__` | `@dataclass`, `NamedTuple`, `attrs` |
| Manual validation logic | `pydantic` models / validators |
| Hand-rolled HTTP client | `httpx` (sync + async) |
| Manual JSON building | `json` / `pydantic.model_dump_json` |
| `datetime.utcnow()` + string parsing | `zoneinfo`, `dateutil`, aware `datetime` |
| Hand-rolled `dict` cache | `functools.lru_cache` / `cachetools` |
| Manual retry loops | `tenacity` |
| Custom `Timer`/`threading` scheduling | `APScheduler` / cron + entrypoint |
| Custom assertion helpers | plain `assert` + `pytest` rich diffs |
| Manual JSON comparison in tests | `assert a == b` on parsed dicts |
| Hand-rolled HTTP mock server | `respx` (httpx) / `responses` (requests) |
| `os.path` string juggling | `pathlib.Path` |
| Manual CLI arg parsing | `argparse` / `typer` / `click` |
| Manual env/config parsing | `pydantic-settings` |

Tooling baseline for any project: **uv** (deps/venv), **ruff** (lint + format),
**mypy** (types), **pytest** (tests). Don't hand-roll what these already do.

---

## Stdlib First: itertools / functools / collections

```python
# BAD: hand-rolled grouping and order-preserving dedupe
def unique_preserving_order(items):
    seen = []
    for x in items:
        if x not in seen:
            seen.append(x)  # O(n^2)
    return seen

# GOOD: stdlib does it, faster and clearer
from collections import Counter, defaultdict
from itertools import batched
from operator import attrgetter

def group_by_status(orders: list[Order]) -> dict[str, list[Order]]:
    grouped: dict[str, list[Order]] = defaultdict(list)
    for order in orders:
        grouped[order.status].append(order)
    return grouped

unique = list(dict.fromkeys(items))                  # O(n), insertion-ordered
status_counts = Counter(o.status for o in orders)    # frequency map
for chunk in batched(rows, 500):                     # 3.12+ paging
    bulk_insert(chunk)
```

See `efficient-python` for hot-path data-structure choices and
`readable-comprehensions` for when a comprehension beats a loop.

---

## Data Classes & Value Objects

```python
# BAD: hand-written __init__/__eq__/__repr__ for a simple value object
class User:
    def __init__(self, id, name, email):
        self.id = id
        self.name = name
        self.email = email

    def __eq__(self, other):
        return (isinstance(other, User)
                and (self.id, self.name, self.email)
                == (other.id, other.name, other.email))

    def __repr__(self):
        return f"User(id={self.id!r}, name={self.name!r}, email={self.email!r})"
    # + __hash__, copy helpers, ...

# GOOD: dataclass — __init__/__eq__/__repr__ generated
from dataclasses import dataclass, field

@dataclass(slots=True)
class User:
    id: int
    name: str
    email: str
    roles: list[str] = field(default_factory=list)

# GOOD: immutable value object (hashable, usable as dict key / set member)
@dataclass(frozen=True, slots=True)
class Money:
    amount: int          # store minor units, never float
    currency: str = "USD"

# GOOD: lightweight, indexable record
from typing import NamedTuple

class Point(NamedTuple):
    x: float
    y: float
```

**Prefer stdlib `dataclasses`/`NamedTuple`** for plain value objects. Reach for
**`attrs`** only when you need its extras (converters, `__slots__` ergonomics on
older targets, validators without pydantic). Reach for **`pydantic`** when the
data crosses a trust boundary and needs parsing/validation — see below.

See `modern-python` for the full dataclass/typing toolbox.

---

## Validation & Parsing — pydantic

```python
# BAD: hand-rolled validation logic
def create_user(data: dict) -> User:
    name = data.get("name")
    if not name or not name.strip():
        raise ValueError("Name is required")
    email = data.get("email", "")
    if "@" not in email or "." not in email.split("@")[-1]:
        raise ValueError("Invalid email")
    if not (0 <= data.get("age", 0) <= 150):
        raise ValueError("Age must be between 0 and 150")
    return User(name=name, email=email, age=data["age"])

# GOOD: declarative, parses-and-validates at the boundary
from pydantic import BaseModel, EmailStr, Field, field_validator

class CreateUserRequest(BaseModel):
    name: str = Field(min_length=1)
    email: EmailStr
    age: int = Field(ge=0, le=150)

    @field_validator("name")
    @classmethod
    def strip_name(cls, v: str) -> str:
        v = v.strip()
        if not v:
            raise ValueError("Name is required")
        return v

# Raises pydantic.ValidationError with a precise, structured report
request = CreateUserRequest.model_validate(payload)

# GOOD: typed settings instead of scattered os.environ reads
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="APP_")
    database_url: str
    request_timeout: float = 10.0
```

---

## Mapping Between Models

```python
# BAD: manual field-by-field copying (drifts as fields are added)
def to_response(user: User) -> dict:
    return {
        "id": user.id,
        "name": user.name,
        "email": user.email,
        "created_at": user.created_at.isoformat(),
    }

# GOOD: pydantic reads attributes off ORM/dataclass instances
from pydantic import BaseModel, ConfigDict

class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    name: str
    email: str
    created_at: datetime

response = UserResponse.model_validate(user)         # one line, type-checked
payload = response.model_dump(mode="json")           # ready for JSON

# GOOD: dataclass round-trips via stdlib (immutable-friendly copy)
from dataclasses import asdict, replace
updated = replace(user, email="new@example.com")
```

---

## HTTP Clients — httpx

```python
# BAD: manual urllib plumbing
import json
from urllib.request import Request, urlopen

req = Request("https://api.example.com/users")
req.add_header("Authorization", f"Bearer {token}")
with urlopen(req) as resp:                # no timeout, no retries, no status check
    data = json.loads(resp.read().decode())

# GOOD: httpx — connection pooling, timeouts, status checks, sync + async
import httpx

class UserApiClient:
    def __init__(self, base_url: str, token: str) -> None:
        self._client = httpx.Client(
            base_url=base_url,
            headers={"Authorization": f"Bearer {token}"},
            timeout=10.0,
        )

    def get_user(self, user_id: int) -> UserDto:
        resp = self._client.get(f"/users/{user_id}")
        resp.raise_for_status()
        return UserDto.model_validate(resp.json())

    def create_user(self, request: CreateUserRequest) -> UserDto:
        resp = self._client.post("/users", json=request.model_dump())
        resp.raise_for_status()
        return UserDto.model_validate(resp.json())

    def close(self) -> None:
        self._client.close()

# GOOD: async variant shares the same API; reuse one pooled client
async def fetch_user(client: httpx.AsyncClient, user_id: int) -> UserDto:
    resp = await client.get(f"/users/{user_id}")
    resp.raise_for_status()
    return UserDto.model_validate(resp.json())
```

See `django-rest-api` for building the server side and `python-concurrency`
for driving many `httpx.AsyncClient` calls concurrently.

---

## JSON Processing

```python
# BAD: manual JSON string building (injection-prone, breaks on quotes/None)
body = '{"name":"' + name + '","email":"' + email + '"}'

# GOOD: stdlib json / pydantic handle escaping, types, dates
import json

body = json.dumps({"name": name, "email": email})

# pydantic serializes nested models, datetimes, enums correctly
json_str = user_response.model_dump_json()
user = UserResponse.model_validate_json(raw)

# Need speed on large payloads? orjson is a drop-in, same shape
import orjson
data = orjson.loads(raw)                  # bytes-in, fast
```

---

## Date / Time

```python
# BAD: naive datetimes and ad-hoc arithmetic
from datetime import datetime

now = datetime.utcnow()                    # naive, deprecated in 3.12
next_week = now.replace(day=now.day + 7)   # crashes near month end

# GOOD: timezone-aware datetimes with zoneinfo
from datetime import UTC, datetime, timedelta
from zoneinfo import ZoneInfo

now = datetime.now(UTC)                     # aware, correct
local = now.astimezone(ZoneInfo("Europe/Berlin"))
next_week = now + timedelta(weeks=1)

# GOOD: lenient parsing of arbitrary inputs
from dateutil import parser
when = parser.isoparse("2026-06-13T08:30:00+02:00")
```

---

## Caching

```python
# BAD: hand-rolled cache — no TTL, no eviction, unbounded growth
_product_cache: dict[int, Product] = {}

def get_product(product_id: int) -> Product:
    if product_id not in _product_cache:
        _product_cache[product_id] = _fetch_from_db(product_id)  # memory leak
    return _product_cache[product_id]

# GOOD: stdlib lru_cache for pure functions with bounded keyspace
from functools import lru_cache

@lru_cache(maxsize=1024)
def get_country_name(code: str) -> str:
    return _load_country(code)            # .cache_clear() to invalidate

# GOOD: cachetools when you need TTL / size eviction / shared cache object
from cachetools import TTLCache, cached

product_cache: TTLCache[int, Product] = TTLCache(maxsize=1000, ttl=600)

@cached(product_cache)
def get_product(product_id: int) -> Product:
    return _fetch_from_db(product_id)
```

Don't cache mutable instance methods with `lru_cache` (it pins `self` and leaks).
See `efficient-python` for memoization trade-offs.

---

## Retry / Resilience — tenacity

```python
# BAD: hand-rolled retry loop with manual backoff
import time

for attempt in range(3):
    try:
        return external_service.call(request)
    except ServiceError:
        if attempt == 2:
            raise
        time.sleep(2 ** attempt)           # no jitter, no logging

# GOOD: tenacity — declarative retry, backoff, jitter
from tenacity import (
    retry,
    retry_if_exception_type,
    stop_after_attempt,
    wait_exponential_jitter,
)

@retry(
    retry=retry_if_exception_type(ServiceError),
    stop=stop_after_attempt(3),
    wait=wait_exponential_jitter(initial=1, max=10),
    reraise=True,
)
def call_external_service(request: Request) -> Response:
    return external_service.call(request)

# The same @retry decorator works on async functions too.
```

See `exception-handling` for what to catch (and what to let propagate).

---

## Scheduling

```python
# BAD: custom threading.Timer loop — fragile lifecycle, no overlap protection
import threading

def schedule_cleanup() -> None:
    cleanup_expired()
    threading.Timer(300, schedule_cleanup).start()  # drifts, leaks threads

# GOOD: APScheduler for in-process jobs
from apscheduler.schedulers.background import BackgroundScheduler

scheduler = BackgroundScheduler()
scheduler.add_job(cleanup_expired, "interval", minutes=5)
scheduler.add_job(generate_report, "cron", hour=2, minute=0)
scheduler.start()

# For containerized apps, prefer an external cron / k8s CronJob invoking a
# console entrypoint (uv run myapp cleanup-expired) — nothing to babysit.
```

---

## Testing Utilities — pytest & friends

```python
# BAD: custom assertion helper that hides the real diff
def assert_user_equals(expected, actual):
    assert expected.name == actual.name
    assert expected.email == actual.email

# GOOD: plain assert — pytest rewrites it into a rich diff
assert actual == expected
assert actual.model_dump() == {"id": 1, "name": "John", "email": "john@x.com"}

# GOOD: parametrize instead of copy-pasted test bodies
import pytest

@pytest.mark.parametrize(("value", "ok"), [("", False), ("  ", False), ("x", True)])
def test_is_present(value: str, ok: bool) -> None:
    assert bool(value.strip()) is ok

# GOOD: respx — mock httpx without a real server
import httpx
import respx

@respx.mock
def test_get_user() -> None:
    respx.get("https://api.example.com/users/1").mock(
        return_value=httpx.Response(200, json={"id": 1, "name": "John"})
    )
    user = UserApiClient("https://api.example.com", "tok").get_user(1)
    assert user.name == "John"

# GOOD: freezegun for deterministic time; pytest's tmp_path for the filesystem
from freezegun import freeze_time

@freeze_time("2026-06-13")
def test_report_date() -> None:
    assert build_report().date == "2026-06-13"
```

Don't hand-roll fixtures, temp dirs, or assertion diffs — pytest provides them.

See `testing-python` for fixtures, factories, and coverage strategy.

---

## Decision Framework: Build vs Use Library

| Factor | Build | Use Library |
|--------|-------|-------------|
| Differentiating logic | Yes — it's your competitive edge | No |
| Well-solved problem | No | Yes — use the standard |
| In the stdlib already | No — don't vendor it | Use the stdlib (zero deps) |
| Maintenance burden | You own it forever | Community maintained |
| Learning curve | Team knows it | Acceptable ramp-up |
| Dependency weight | N/A | Don't add a heavy lib for one helper |
| License / activity | N/A | Must be compatible and maintained |

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| NIH syndrome | Reinventing solved problems | Check stdlib, then PyPI, before writing |
| Ignoring the stdlib | Adding deps for `itertools`/`pathlib` work | Learn the stdlib first |
| Wrapping libraries needlessly | Extra indirection, drift | Use the library directly unless you need a seam |
| Heavy lib for one function | Bloated, slow installs | Use a focused package or copy the one function |
| Unpinned / outdated deps | CVEs, irreproducible builds | `uv lock`; update on a cadence |
| No dependency management | Version conflicts | `uv` + `pyproject.toml` lockfile |
| Naive `datetime` everywhere | tz bugs, DST errors | Always store/compute aware UTC |

---

## Related Skills

- `review-changes-python` — Flag hand-rolled code during review
- `django-rest-api` — httpx/pydantic at the API boundary
- `testing-python` — pytest, respx, parametrize, fixtures
- `efficient-python` — When stdlib data structures and caching pay off
- `modern-python` — dataclasses, typing, and 3.12/3.13 language features
