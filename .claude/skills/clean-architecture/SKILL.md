---
name: clean-architecture
description: Architecture patterns for modern Python — package-by-feature, hexagonal architecture (ports/adapters with Protocols), DDD building blocks with dataclasses, module boundaries, and import-linter contracts. Use when structuring projects, reviewing package organization, or applying domain-driven design.
---

# Clean Architecture

Architectural patterns for organizing Python applications (Django, FastAPI, or plain packages).

## When to Use

- Starting a new project and choosing package structure
- Refactoring a tangled codebase into clear boundaries
- Applying domain-driven design concepts
- Reviewing code for architectural violations
- Deciding between simple and complex architectures

---

## Quick Reference

| Project Complexity | Architecture | When |
|-------------------|-------------|------|
| Simple CRUD | Package-by-feature (flat) | < 5 entities, straightforward logic |
| Medium business logic | Layered package-by-feature | Most Django/FastAPI apps |
| Complex domain | Hexagonal / Ports & Adapters | Rich domain logic, multiple integrations |
| Microservice ecosystem | Hexagonal + DDD | Bounded contexts, event-driven |

---

## Package-by-Feature vs Package-by-Layer

### Package-by-Layer (Avoid)

```
# BAD: Grouped by technical role — changes touch many packages
app/
├── controllers/    # order.py, user.py, product.py
├── services/       # order.py, user.py, product.py
├── repositories/   # order.py, user.py, product.py
├── models/         # order.py, user.py, product.py
└── schemas/        # order.py, user.py, product.py
```

Problems:
- Adding an "order" feature touches 5 packages
- No encapsulation — everything is imported from everywhere
- Can't tell what the app does from package names
- High coupling between unrelated features

### Package-by-Feature (Preferred)

```
# GOOD: Grouped by business capability
app/
├── order/
│   ├── __init__.py        # Re-exports the public surface only
│   ├── api.py             # views / routes
│   ├── service.py
│   ├── repository.py
│   ├── models.py          # Order entity
│   └── schemas.py         # CreateOrderRequest, OrderResponse
├── user/                  # same shape: __init__, api, service, repository, models
├── product/               # same shape
└── shared/
    ├── config.py
    └── exceptions.py
```

Benefits:
- Adding "order" features touches one package
- Internals stay private via `__all__` and underscore-prefixed names
- Package names describe business capabilities
- Easy to extract into a separate distribution later

### Encapsulation with `__all__` and `__init__.py`

Python has no `package-private`, so the convention is: expose a deliberate public
surface in `__init__.py`, prefix internals with `_`, and let imports flow through the
package root rather than reaching into modules.

```python
# GOOD: app/order/__init__.py — the only sanctioned entry point
from app.order.api import router
from app.order.schemas import CreateOrderRequest, OrderResponse

__all__ = ["router", "CreateOrderRequest", "OrderResponse"]
```

```python
# BAD: another feature reaching into order internals
from app.order.repository import OrderRepository  # crosses a boundary
from app.order.service import OrderService        # imports an unexported name

# GOOD: depend on the published surface (or a shared port), not the guts
from app.order import CreateOrderRequest, OrderResponse
```

---

## Hexagonal Architecture (Ports & Adapters)

For applications with complex domain logic and multiple integration points.

```
app/
├── order/
│   ├── domain/            # Pure business logic — no Django, no I/O, no framework
│   │   ├── order.py
│   │   ├── status.py
│   │   ├── errors.py
│   │   └── pricing.py
│   ├── ports/
│   │   ├── inbound.py     # Driving ports (use cases) — Protocols
│   │   └── outbound.py    # Driven ports (persistence, payment, notification)
│   ├── application/
│   │   └── order_service.py   # Use case implementations (orchestration)
│   └── adapters/
│       ├── inbound/
│       │   └── web.py     # REST view / router + mappers
│       └── outbound/
│           ├── persistence.py   # ORM / SQL adapter
│           ├── payment.py       # Stripe adapter
│           └── notification.py  # Email adapter
```

### Ports (Protocols, not ABCs)

Prefer `typing.Protocol` for ports: adapters satisfy them structurally with zero
import coupling back to the domain, and mypy verifies conformance.

```python
# app/order/ports/inbound.py — driving port: what the application can do
from dataclasses import dataclass
from decimal import Decimal
from typing import Protocol

from app.order.domain.order import LineItem


@dataclass(frozen=True, slots=True)
class CreateOrderCommand:
    customer_id: str
    items: list[LineItem]


@dataclass(frozen=True, slots=True)
class OrderResponse:
    id: int
    customer_id: str
    total: Decimal


class CreateOrderUseCase(Protocol):
    def create(self, command: CreateOrderCommand) -> OrderResponse: ...
```

```python
# app/order/ports/outbound.py — driven ports: what the application needs
from dataclasses import dataclass
from decimal import Decimal
from typing import Protocol

from app.order.domain.order import Order


@dataclass(frozen=True, slots=True)
class PaymentResult:
    transaction_id: str


class OrderPersistencePort(Protocol):
    def save(self, order: Order) -> int: ...
    def find_by_id(self, order_id: int) -> Order | None: ...


class PaymentPort(Protocol):
    def charge(self, customer_id: str, amount: Decimal) -> PaymentResult: ...


class NotificationPort(Protocol):  # send_confirmation(order) -> None
    def send_confirmation(self, order: Order) -> None: ...
```

### Application Service (Use Case Implementation)

```python
# app/order/application/order_service.py
from app.order.domain.order import Order
from app.order.ports.inbound import CreateOrderCommand, OrderResponse
from app.order.ports.outbound import (
    NotificationPort,
    OrderPersistencePort,
    PaymentPort,
)


class OrderApplicationService:
    """Implements CreateOrderUseCase structurally — depends only on ports."""

    def __init__(
        self,
        persistence: OrderPersistencePort,
        payment: PaymentPort,
        notification: NotificationPort,
    ) -> None:
        self._persistence = persistence
        self._payment = payment
        self._notification = notification

    def create(self, command: CreateOrderCommand) -> OrderResponse:
        # Pure domain logic
        order = Order.create(command.customer_id, command.items)
        order.calculate_total()

        # Driven ports
        result = self._payment.charge(command.customer_id, order.total)
        order.mark_as_paid(result.transaction_id)

        order_id = self._persistence.save(order)
        self._notification.send_confirmation(order)

        return OrderResponse(id=order_id, customer_id=order.customer_id, total=order.total)
```

### Adapters

```python
# app/order/adapters/inbound/web.py — driving adapter (FastAPI shown)
from fastapi import APIRouter, status

from app.order.ports.inbound import CreateOrderCommand, CreateOrderUseCase, OrderResponse
from app.order.schemas import CreateOrderRequest


def make_router(use_case: CreateOrderUseCase) -> APIRouter:
    router = APIRouter(prefix="/api/v1/orders")

    @router.post("", status_code=status.HTTP_201_CREATED)
    def create(request: CreateOrderRequest) -> OrderResponse:
        command = CreateOrderCommand(request.customer_id, request.items)
        return use_case.create(command)

    return router
```

```python
# app/order/adapters/outbound/persistence.py — driven adapters (structural typing)
from decimal import Decimal

from app.order.domain.order import Order
from app.order.ports.outbound import PaymentResult


class OrmOrderAdapter:  # structurally satisfies OrderPersistencePort
    def __init__(self, session: "Session") -> None:
        self._session = session

    def save(self, order: Order) -> int:
        row = OrderRow.from_domain(order)
        self._session.add(row)
        self._session.flush()
        return row.id

    def find_by_id(self, order_id: int) -> Order | None:
        row = self._session.get(OrderRow, order_id)
        return row.to_domain() if row is not None else None


class StripePaymentAdapter:  # structurally satisfies PaymentPort
    def __init__(self, client: "StripeClient") -> None:
        self._client = client

    def charge(self, customer_id: str, amount: Decimal) -> PaymentResult:
        charge = self._client.charges.create(
            customer=customer_id, amount=int(amount * 100), currency="usd"
        )
        return PaymentResult(transaction_id=charge.id)
```

---

## DDD Building Blocks

| Concept | What | Python Mapping |
|---------|------|---------------|
| Entity | Object with identity (ID) | Class with `__eq__`/`__hash__` by ID |
| Value Object | Immutable, identity by value | `@dataclass(frozen=True, slots=True)` |
| Aggregate | Cluster of entities with consistency boundary | Root entity + children |
| Domain Service | Logic that fits no single entity | Plain class/function in `domain/` |
| Repository | Persistence abstraction | `Protocol` port |
| Domain Event | Something that happened in the domain | `@dataclass(frozen=True)` event |

```python
# Value Object — immutable, identity by value, validated in __post_init__
from dataclasses import dataclass
from decimal import Decimal


@dataclass(frozen=True, slots=True)
class Money:
    amount: Decimal
    currency: str

    def __post_init__(self) -> None:
        if self.amount < 0:
            raise ValueError("Amount cannot be negative")

    def add(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError("Cannot add different currencies")
        return Money(self.amount + other.amount, self.currency)
```

```python
# Aggregate Root — business logic lives here, not in the service
from dataclasses import dataclass, field

from app.order.domain.errors import OrderError
from app.order.domain.status import OrderStatus


@dataclass(eq=False)  # identity by id, not by value
class Order:
    id: int | None
    customer_id: str
    items: list["LineItem"] = field(default_factory=list)
    status: OrderStatus = OrderStatus.DRAFT
    total: Money = field(default_factory=lambda: Money(Decimal("0"), "USD"))

    def __eq__(self, other: object) -> bool:
        return isinstance(other, Order) and self.id is not None and self.id == other.id

    def __hash__(self) -> int:
        return hash(self.id)

    def add_item(self, item: "LineItem") -> None:
        if self.status is not OrderStatus.DRAFT:
            raise OrderError("Cannot add items to a non-draft order")
        self.items.append(item)
        self._recalculate_total()

    def submit(self) -> None:
        if not self.items:
            raise OrderError("Cannot submit an empty order")
        self.status = OrderStatus.SUBMITTED
```

```python
# Domain Event
@dataclass(frozen=True, slots=True)
class OrderSubmitted:
    order_id: int
    customer_id: str
    total: Money
```

---

## Module Boundaries — import-linter

Enforce architecture rules at test/CI time with [import-linter](https://import-linter.readthedocs.io),
the Python analogue to ArchUnit. Define contracts in `pyproject.toml` (or `.importlinter`)
and run `lint-imports` in CI.

```toml
# pyproject.toml
[tool.importlinter]
root_package = "app"

# The domain layer must not import adapters or frameworks.
[[tool.importlinter.contracts]]
name = "Domain is pure"
type = "forbidden"
source_modules = ["app.order.domain"]
forbidden_modules = [
    "app.order.adapters",
    "django",
    "fastapi",
    "sqlalchemy",
]

# Layered direction: adapters -> application -> ports -> domain (never reversed).
[[tool.importlinter.contracts]]
name = "Layered architecture"
type = "layers"
layers = [
    "app.order.adapters",
    "app.order.application",
    "app.order.ports",
    "app.order.domain",
]

# Features must not reach into each other; they talk through shared only.
[[tool.importlinter.contracts]]
name = "Features are independent"
type = "independence"
modules = [
    "app.order",
    "app.user",
    "app.product",
]
```

```bash
# Run as part of the test gate alongside ruff and mypy
uv run lint-imports && uv run ruff check . && uv run mypy app && uv run pytest
```

---

## When NOT to Use Hexagonal

| Scenario | Recommendation |
|----------|---------------|
| Simple CRUD app | Package-by-feature is enough |
| < 5 domain entities | Hexagonal is over-engineering |
| Prototype / MVP | Keep it simple, refactor later |
| Team is unfamiliar | Start layered, evolve |
| No complex domain logic | Ports/adapters add ceremony without value |
| Django app leaning on the ORM | Fat models + services usually beat ports |

**Rule of thumb:** If your service functions are mostly "validate → save → return", you
don't need hexagonal. If you have complex business rules, state machines, or multiple
integration points, consider it.

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Package-by-layer | Features scattered, no encapsulation | Package-by-feature |
| `from module import *` everywhere | No module boundaries | Curate `__all__`, import the package surface |
| Anemic domain model | All logic in services, entities are plain dicts/dataclasses of data | Push logic into domain objects |
| Leaky abstraction | ORM models or `Session` types in the domain | Depend on `Protocol` ports |
| Over-engineering CRUD | Hexagonal for simple REST-to-DB | Keep it simple |
| Circular imports | Feature A → Feature B → Feature A | Extract a shared port, emit events |
| God module | One `models.py` with 50+ classes | Split by feature/subdomain |
| ABC ports with inheritance everywhere | Forces import coupling to the domain | Use `Protocol` for structural typing |

---

## Related Skills

- `solid-principles` — Principles that guide architectural decisions
- `design-patterns` — Patterns used within architectural layers
- `django-orm` — Repository/persistence adapter implementation
- `modern-python` — Dataclasses, `Protocol`, and `X | None` idioms used above
