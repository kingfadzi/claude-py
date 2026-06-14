---
name: django-orm
description: Django ORM patterns — models, QuerySets, select_related/prefetch_related to avoid N+1, indexes, migrations, atomic transactions, bulk operations, the raw SQL escape hatch, and connection pooling/persistent connections. Use when building or reviewing ORM-based data access, writing queries, or configuring database access.
---

# Django ORM

ORM-based data access patterns for Django 5.x using models, QuerySets, and managers.
Tooling baseline: `uv` for deps, `ruff` for lint+format, `mypy` (with `django-stubs`) for types, `pytest` (+ `pytest-django`) for tests.

## When to Use

- Building the data-access layer with models and managers
- Writing or reviewing QuerySets and avoiding N+1 queries
- Configuring transactions, connection pooling, or migrations
- Deciding between ORM, `.raw()`, and `connection.cursor()`

---

## Quick Reference

| Need | Use |
|------|-----|
| Simple queries/updates | QuerySet methods (`.filter()`, `.get()`, `.update()`) |
| Avoid N+1 on FK/O2O | `select_related(...)` (SQL JOIN) |
| Avoid N+1 on reverse/M2M | `prefetch_related(...)` (second query) |
| Bulk insert/update | `bulk_create()` / `bulk_update()` |
| Atomicity | `transaction.atomic()` (decorator or context manager) |
| Computed columns/filters | `annotate()` + `F()` / `Q()` expressions |
| Escape hatch | `Model.objects.raw(...)` or `connection.cursor()` |
| Schema management | Migrations (`makemigrations` / `migrate`) |
| Connection reuse | `CONN_MAX_AGE` or an external pooler (PgBouncer) |

---

## Models

Models are the single source of truth — fields, constraints, and indexes live here, not scattered across migrations.

```python
# BAD: stringly-typed fields, no constraints, no indexes, no type hints
class User(models.Model):
    name = models.CharField(max_length=255)
    email = models.CharField(max_length=255)   # no uniqueness, no index
    status = models.CharField(max_length=50)   # any string accepted
    created_at = models.DateTimeField()        # caller must remember to set it


# GOOD: typed choices, constraints, indexes, sensible defaults
from django.db import models
from django.utils import timezone


class UserStatus(models.TextChoices):
    ACTIVE = "ACTIVE", "Active"
    SUSPENDED = "SUSPENDED", "Suspended"
    CLOSED = "CLOSED", "Closed"


class User(models.Model):
    name: models.CharField = models.CharField(max_length=255)
    email: models.EmailField = models.EmailField(unique=True)
    status: models.CharField = models.CharField(
        max_length=50, choices=UserStatus.choices, default=UserStatus.ACTIVE
    )
    created_at: models.DateTimeField = models.DateTimeField(default=timezone.now)

    class Meta:
        indexes = [
            models.Index(fields=["status"]),
        ]

    def __str__(self) -> str:
        return self.name
```

For data passed across layers (not persisted), prefer a `dataclass` or `NamedTuple` over a model instance.

```python
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class UserSummary:
    id: int
    name: str
```

---

## QuerySets

QuerySets are lazy and composable — build them up, then evaluate once.

```python
# BAD: evaluating eagerly, filtering in Python, fetching whole objects for a count
def active_user_names(min_orders: int) -> list[str]:
    users = list(User.objects.all())                      # loads every row
    names = []
    for user in users:
        if user.status == "ACTIVE":                       # filtering in Python
            if user.order_set.count() >= min_orders:      # N+1 COUNT queries
                names.append(user.name)
    return names


# GOOD: push work to the database; one query
from django.db.models import Count


def active_user_names(min_orders: int) -> list[str]:
    return list(
        User.objects.filter(status=UserStatus.ACTIVE)
        .annotate(order_count=Count("order"))
        .filter(order_count__gte=min_orders)
        .values_list("name", flat=True)
    )
```

### Common Patterns

```python
# Single result — get() raises DoesNotExist / MultipleObjectsReturned
def find_by_id(user_id: int) -> User | None:
    try:
        return User.objects.get(pk=user_id)
    except User.DoesNotExist:
        return None

# List result, ordered
def find_by_status(status: UserStatus) -> list[User]:
    return list(User.objects.filter(status=status).order_by("name"))

# Single value
def count() -> int:
    return User.objects.count()

# Existence check — cheaper than count() > 0
def exists_by_email(email: str) -> bool:
    return User.objects.filter(email=email).exists()

# Only the columns you need — avoids hydrating full model instances
def email_index() -> list[UserSummary]:
    rows = User.objects.values("id", "name")          # returns dicts
    return [UserSummary(**row) for row in rows]
```

### F() and Q() expressions

```python
from django.db.models import F, Q

# BAD: read-modify-write race condition
account = Account.objects.get(pk=account_id)
account.balance -= amount          # stale if another tx ran in between
account.save()

# GOOD: atomic in-database update, no race
Account.objects.filter(pk=account_id).update(balance=F("balance") - amount)

# Complex boolean logic with Q
Order.objects.filter(
    Q(status="PENDING") | Q(status="PROCESSING"),
    customer_id=customer_id,
)
```

---

## Avoiding N+1 Queries

The single most common ORM performance bug. Accessing a related object inside a loop fires one query per row.

```python
# BAD: 1 query for orders + 1 per order for customer + 1 per order for items = N+1
def order_lines() -> list[str]:
    lines = []
    for order in Order.objects.all():           # query 1
        customer = order.customer               # query per order (FK)
        for item in order.items.all():          # query per order (reverse FK)
            lines.append(f"{customer.name}: {item.product_id} x{item.quantity}")
    return lines


# GOOD: select_related for FK/O2O (JOIN), prefetch_related for reverse/M2M
def order_lines() -> list[str]:
    orders = (
        Order.objects.select_related("customer")    # JOIN, no extra queries
        .prefetch_related("items")                  # 1 extra query, batched
    )
    return [
        f"{order.customer.name}: {item.product_id} x{item.quantity}"
        for order in orders
        for item in order.items.all()
    ]
```

| Relation | Use | Mechanism |
|----------|-----|-----------|
| Forward `ForeignKey` / `OneToOne` | `select_related` | SQL JOIN, single query |
| Reverse FK / `ManyToMany` | `prefetch_related` | Second query, joined in Python |
| Nested control / custom filter | `prefetch_related(Prefetch(...))` | Per-prefetch QuerySet |

```python
from django.db.models import Prefetch

# Prefetch with a filtered/ordered inner QuerySet
orders = Order.objects.prefetch_related(
    Prefetch("items", queryset=LineItem.objects.order_by("-price"))
)
```

Catch regressions in tests with `assertNumQueries` (see `testing-python`).

```python
def test_order_lines_is_constant_queries(django_assert_num_queries):
    with django_assert_num_queries(2):   # orders + prefetched items
        order_lines()
```

---

## Bulk Operations

```python
# BAD: one INSERT per row
def import_users(rows: list[dict[str, str]]) -> None:
    for row in rows:
        User.objects.create(name=row["name"], email=row["email"])

# GOOD: single round trip
def import_users(rows: list[dict[str, str]]) -> None:
    User.objects.bulk_create(
        [User(name=row["name"], email=row["email"]) for row in rows],
        batch_size=500,
    )

# GOOD: bulk update of changed instances
def deactivate(users: list[User]) -> None:
    for user in users:
        user.status = UserStatus.SUSPENDED
    User.objects.bulk_update(users, ["status"], batch_size=500)

# GOOD: set-based update without loading rows
def deactivate_stale(cutoff: datetime) -> int:
    return User.objects.filter(created_at__lt=cutoff).update(
        status=UserStatus.SUSPENDED
    )
```

Note: `bulk_create` / `bulk_update` skip `save()`, signals, and `auto_now` — by design, for speed.

---

## Managers and the Repository Boundary

Encapsulate query logic on a custom manager/QuerySet rather than scattering filters across views.

```python
# GOOD: reusable, chainable query methods
class OrderQuerySet(models.QuerySet["Order"]):
    def completed(self) -> "OrderQuerySet":
        return self.filter(status="COMPLETED")

    def for_customer(self, customer_id: int) -> "OrderQuerySet":
        return self.filter(customer_id=customer_id)

    def with_customer(self) -> "OrderQuerySet":
        return self.select_related("customer")


class Order(models.Model):
    customer = models.ForeignKey("User", on_delete=models.PROTECT)
    total = models.DecimalField(max_digits=12, decimal_places=2)
    status = models.CharField(max_length=50)
    created_at = models.DateTimeField(default=timezone.now)

    objects = OrderQuerySet.as_manager()


# Usage reads like the domain:
Order.objects.completed().for_customer(customer_id).with_customer()
```

Keep cross-aggregate orchestration in a service function, not in the manager — see `clean-architecture`.

---

## Transaction Management

### atomic — decorator or context manager

```python
from django.db import transaction

# BAD: partial write — order is saved even if stock reduction fails
def create_order(request: CreateOrderRequest) -> Order:
    order = Order.objects.create(customer_id=request.customer_id, total=request.total)
    for item in request.items:
        reduce_stock(item.product_id, item.quantity)   # may raise mid-loop
    return order


# GOOD: all-or-nothing
@transaction.atomic
def create_order(request: CreateOrderRequest) -> Order:
    order = Order.objects.create(customer_id=request.customer_id, total=request.total)
    for item in request.items:
        reduce_stock(item.product_id, item.quantity)
    return order


# GOOD: context manager for a narrower scope
def process_payment(req: PaymentRequest) -> PaymentResult:
    try:
        with transaction.atomic():
            debit_account(req)
            credit_merchant(req)
    except InsufficientFundsError as exc:
        return PaymentResult.failed(str(exc))   # block rolled back on exception
    return PaymentResult.success()
```

### Locking and isolation

```python
# GOOD: SELECT ... FOR UPDATE to serialize concurrent updates
@transaction.atomic
def reduce_stock(product_id: int, quantity: int) -> None:
    item = Inventory.objects.select_for_update().get(product_id=product_id)
    if item.quantity < quantity:
        raise InsufficientFundsError(product_id)
    item.quantity -= quantity
    item.save(update_fields=["quantity"])
```

| Concern | Tool | Use When |
|---------|------|----------|
| All-or-nothing block | `transaction.atomic()` | Any multi-statement write |
| Nested rollback point | `atomic()` inside `atomic()` | Savepoint within larger tx |
| Run after commit | `transaction.on_commit(fn)` | Enqueue task / send email |
| Row-level lock | `select_for_update()` | Concurrent updates to same row |

Use `on_commit` so side effects never fire on a rolled-back transaction — see `logging-observability`.

```python
transaction.on_commit(lambda: notify_shipping.delay(order.id))
```

---

## Raw SQL Escape Hatch

Reach for raw SQL only when the ORM genuinely can't express the query. Always parameterize.

```python
# BAD: f-string SQL — SQL injection
def search(term: str) -> list[User]:
    with connection.cursor() as cursor:
        cursor.execute(f"SELECT * FROM app_user WHERE name LIKE '%{term}%'")
        return cursor.fetchall()


# GOOD: Model.raw() — maps rows back to model instances, parameterized
def search(term: str) -> list[User]:
    return list(
        User.objects.raw(
            "SELECT * FROM app_user WHERE name LIKE %s",
            [f"%{term}%"],
        )
    )


# GOOD: connection.cursor() for aggregates that don't map to a model
def monthly_revenue(start: date, end: date) -> list[MonthlyRevenue]:
    sql = """
        SELECT date_trunc('month', created_at) AS month,
               SUM(total) AS revenue,
               COUNT(*) AS order_count
        FROM app_order
        WHERE status = 'COMPLETED'
          AND created_at BETWEEN %s AND %s
        GROUP BY 1
        ORDER BY 1
    """
    with connection.cursor() as cursor:        # context manager closes cursor
        cursor.execute(sql, [start, end])
        return [MonthlyRevenue(*row) for row in cursor.fetchall()]
```

Prefer ORM aggregates (`annotate` + `TruncMonth` + `Sum`) before dropping to raw — they stay type-checked and portable.

---

## Migrations

Migrations are generated from model changes and live in version control. Never edit the database by hand.

```
app/migrations/
├── 0001_initial.py
├── 0002_order_status_index.py
└── 0003_alter_user_email_unique.py
```

```bash
uv run python manage.py makemigrations        # generate from model diff
uv run python manage.py migrate               # apply
uv run python manage.py makemigrations --check --dry-run   # CI guard: fail if models drift from migrations
uv run python manage.py sqlmigrate app 0002   # preview the SQL
```

```python
# GOOD: data migration with a reversible operation
from django.db import migrations


def backfill_status(apps, schema_editor):
    User = apps.get_model("app", "User")     # historical model, not the live import
    User.objects.filter(status="").update(status="ACTIVE")


class Migration(migrations.Migration):
    dependencies = [("app", "0002_order_status_index")]
    operations = [
        migrations.RunPython(backfill_status, migrations.RunPython.noop),
    ]
```

Guidelines: keep schema and data migrations separate; always supply a reverse (`noop` if truly irreversible); add indexes/constraints in their own migration for large tables.

---

## Testing

Use `pytest-django`; the `db` fixture wraps each test in a transaction and rolls back.

```python
# GOOD: fast, isolated, query-count assertions
import pytest
from decimal import Decimal


@pytest.mark.django_db
def test_saves_and_finds_by_id() -> None:
    customer = User.objects.create(name="Ada", email="ada@example.com")
    order = Order.objects.create(
        customer=customer, total=Decimal("100.00"), status="PENDING"
    )

    found = Order.objects.get(pk=order.pk)

    assert found.customer.name == "Ada"
    assert found.total == Decimal("100.00")


@pytest.mark.django_db
def test_for_customer_returns_only_theirs() -> None:
    a = User.objects.create(name="A", email="a@example.com")
    b = User.objects.create(name="B", email="b@example.com")
    Order.objects.create(customer=a, total=Decimal("50"), status="PENDING")
    Order.objects.create(customer=a, total=Decimal("100"), status="PENDING")
    Order.objects.create(customer=b, total=Decimal("75"), status="PENDING")

    orders = Order.objects.for_customer(a.pk)

    assert orders.count() == 2


@pytest.mark.django_db
def test_no_n_plus_one(django_assert_num_queries) -> None:
    customer = User.objects.create(name="A", email="a@example.com")
    for _ in range(5):
        Order.objects.create(customer=customer, total=Decimal("1"), status="X")

    with django_assert_num_queries(1):
        names = [o.customer.name for o in Order.objects.select_related("customer")]

    assert len(names) == 5
```

For integration tests against real PostgreSQL, point the test database at a containerized Postgres (Docker / `testcontainers-python`) instead of SQLite, so JSONB, constraints, and `select_for_update` behave as in production. See `testing-python`.

---

## Connection Pooling

Django opens a fresh connection per request by default. Two levers:

```python
# settings.py — persistent connections reused across requests
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "app",
        "CONN_MAX_AGE": 60,          # seconds; reuse connection for 60s (0 = per-request)
        "CONN_HEALTH_CHECKS": True,  # ping before reuse, drop dead connections
        "OPTIONS": {"pool": True},   # Django 5.1+ psycopg3 built-in pool
    }
}
```

| Setting | Effect | Rule of thumb |
|---------|--------|---------------|
| `CONN_MAX_AGE = 0` | New connection per request | Safe default; serverless |
| `CONN_MAX_AGE = 60` | Reuse for 60s | Long-lived workers |
| `OPTIONS["pool"]` | psycopg3 in-process pool | Django 5.1+, single process |
| External PgBouncer | Process-shared pool | Many workers / high concurrency |

With PgBouncer in transaction mode, set `CONN_MAX_AGE = 0` and disable server-side cursors (`DISABLE_SERVER_SIDE_CURSORS = True`) — see `django-configuration`.

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| f-string / `%`-format SQL | SQL injection | Parameterize (`%s` placeholders, ORM filters) |
| Missing `transaction.atomic` | Partial writes on failure | Wrap multi-statement writes |
| N+1 in a loop | Performance disaster | `select_related` / `prefetch_related` |
| Filtering in Python | Loads whole table | Push to `.filter()` / `annotate()` |
| `len(qs)` for a count | Hydrates every row | `qs.count()` / `qs.exists()` |
| Read-modify-write | Lost-update race | `F()` expressions / `select_for_update` |
| `save()` in a loop | One INSERT per row | `bulk_create` / `bulk_update` |
| Editing the DB by hand | Env drift | Migrations, checked into VCS |
| `.all()` then slice in Python | Over-fetching | Slice the QuerySet (`qs[:10]` → `LIMIT`) |
| Side effects before commit | Fire on rolled-back tx | `transaction.on_commit` |

---

## Related Skills

- `testing-python` — `pytest-django`, `assertNumQueries`, containerized Postgres
- `django-rest-api` — View → service → model layer integration
- `django-configuration` — `DATABASES`, pooling, environment settings
- `django-security` — Parameterized queries, mass-assignment, ORM injection
- `review-changes-python` — Database access review checklist
- `efficient-python` — Query-count and allocation hot paths
- `python-concurrency` — Transaction isolation and concurrent access
