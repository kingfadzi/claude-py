---
name: solid-principles
description: SOLID principles checklist with idiomatic Python examples (Protocols, dataclasses, dependency injection). Use when reviewing classes, refactoring code, or when user asks about Single Responsibility, Open/Closed, Liskov, Interface Segregation, or Dependency Inversion.
---

# SOLID Principles Skill

Review and apply SOLID principles in Python code.

## When to Use
- User says "check SOLID" / "SOLID review" / "is this class doing too much?"
- Reviewing class design
- Refactoring large classes
- Code review focusing on design

---

## Quick Reference

| Letter | Principle | One-liner |
|--------|-----------|-----------|
| **S** | Single Responsibility | One class = one reason to change |
| **O** | Open/Closed | Open for extension, closed for modification |
| **L** | Liskov Substitution | Subtypes must be substitutable for base types |
| **I** | Interface Segregation | Many specific protocols > one general protocol |
| **D** | Dependency Inversion | Depend on abstractions, not concretions |

> In Python, "abstractions" usually means `typing.Protocol` (structural) or `abc.ABC`
> (nominal). Prefer `Protocol` — it gives duck-typing with mypy-checked contracts and
> no inheritance coupling. See `modern-python`.

---

## S - Single Responsibility Principle (SRP)

> "A class should have only one reason to change."

### Violation

```python
# BAD: UserService does too much
class UserService:
    def __init__(self, session: Session, email_client: EmailClient) -> None:
        self._session = session
        self._email_client = email_client

    def create_user(self, name: str, email: str) -> User:
        # validation logic
        if "@" not in email:
            raise ValueError("Invalid email")

        # persistence logic
        user = User(name=name, email=email)
        self._session.add(user)
        self._session.commit()

        # notification logic
        self._email_client.send(email, "Welcome!", f"Hello {name}")

        # audit logic
        logging.getLogger("audit").info("User created: %s", email)

        return user
```

**Problems:**
- Validation changes? Modify `UserService`
- Email template changes? Modify `UserService`
- Audit format changes? Modify `UserService`
- Hard to test each concern separately

### Refactored

```python
# GOOD: Each class has one responsibility
from dataclasses import dataclass


class UserValidator:
    def validate(self, name: str, email: str) -> None:
        if "@" not in email:
            raise ValidationError("Invalid email")


class UserRepository:
    def __init__(self, session: Session) -> None:
        self._session = session

    def save(self, user: User) -> User:
        self._session.add(user)
        self._session.commit()
        return user


class WelcomeEmailSender:
    def __init__(self, email_client: EmailClient) -> None:
        self._email_client = email_client

    def send_welcome(self, user: User) -> None:
        self._email_client.send(user.email, "Welcome!", f"Hello {user.name}")


class UserAuditLogger:
    def __init__(self) -> None:
        self._log = logging.getLogger("audit")

    def log_creation(self, user: User) -> None:
        self._log.info("User created: %s", user.email)


@dataclass
class UserService:
    validator: UserValidator
    repository: UserRepository
    email_sender: WelcomeEmailSender
    audit_logger: UserAuditLogger

    def create_user(self, name: str, email: str) -> User:
        self.validator.validate(name, email)
        user = self.repository.save(User(name=name, email=email))
        self.email_sender.send_welcome(user)
        self.audit_logger.log_creation(user)
        return user
```

### How to Detect SRP Violations

- Module mixes `import` statements from unrelated domains (db driver + smtplib + jinja2)
- Class name contains "And" or "Manager" or "Handler" (often)
- Methods operate on unrelated data
- Changes in one area require touching unrelated methods
- Hard to name the class concisely

### Python Example: Centralized error handling for SRP

```python
# GOOD: Exception-to-response mapping is a separate responsibility.
# Don't put try/except in every view — centralize it (DRF exception handler).
from rest_framework.views import exception_handler
from rest_framework.response import Response


def api_exception_handler(exc: Exception, context: dict) -> Response | None:
    if isinstance(exc, EntityNotFound):
        return Response({"detail": str(exc)}, status=404)
    return exception_handler(exc, context)
```

See `django-rest-api` and `exception-handling`.

### When NOT to Apply SRP

- Don't split a class just because it has multiple methods — they may all serve one responsibility
- A 50-line service with validate + save + notify for one entity is fine
- Over-splitting leads to "ravioli code" — dozens of tiny classes that are hard to follow

### Quick Check Questions

1. Can you describe the class purpose in one sentence without "and"?
2. Would different stakeholders request changes to this class?
3. Are there methods that don't use any of `self`'s attributes (candidate for a module function)?

---

## O - Open/Closed Principle (OCP)

> "Software entities should be open for extension, but closed for modification."

### Violation

```python
# BAD: Must modify function to add new discount type
class DiscountCalculator:
    def calculate(self, order: Order, discount_type: str) -> float:
        if discount_type == "PERCENTAGE":
            return order.total * 0.1
        elif discount_type == "FIXED":
            return 50.0
        elif discount_type == "LOYALTY":
            return order.total * order.customer.loyalty_rate
        # Every new discount type = modify this method
        return 0.0
```

### Refactored

```python
# GOOD: Add new discounts without modifying existing code
from typing import Protocol


class DiscountStrategy(Protocol):
    def supports(self, discount_type: str) -> bool: ...
    def calculate(self, order: Order) -> float: ...


class PercentageDiscount:
    def supports(self, discount_type: str) -> bool:
        return discount_type == "PERCENTAGE"

    def calculate(self, order: Order) -> float:
        return order.total * 0.1


class FixedDiscount:
    def supports(self, discount_type: str) -> bool:
        return discount_type == "FIXED"

    def calculate(self, order: Order) -> float:
        return 50.0


class LoyaltyDiscount:
    def supports(self, discount_type: str) -> bool:
        return discount_type == "LOYALTY"

    def calculate(self, order: Order) -> float:
        return order.total * order.customer.loyalty_rate


# New discount? Just add a new class — nothing else changes.
class SeasonalDiscount:
    def supports(self, discount_type: str) -> bool:
        return discount_type == "SEASONAL"

    def calculate(self, order: Order) -> float:
        return order.total * 0.2


class DiscountCalculator:
    def __init__(self, strategies: list[DiscountStrategy]) -> None:
        self._strategies = strategies

    def calculate(self, order: Order, discount_type: str) -> float:
        strategy = next(
            (s for s in self._strategies if s.supports(discount_type)),
            None,
        )
        return strategy.calculate(order) if strategy else 0.0
```

Note the generator-expression-with-`next` idiom replaces Java's `stream().filter().findFirst()`. See `readable-comprehensions`.

### Pythonic Alternative: dict dispatch / pattern matching

```python
# GOOD: For a small, stable set, a registry or match is plenty — no Protocol needed.
from collections.abc import Callable

DISCOUNTS: dict[str, Callable[[Order], float]] = {
    "PERCENTAGE": lambda o: o.total * 0.1,
    "FIXED": lambda _: 50.0,
    "LOYALTY": lambda o: o.total * o.customer.loyalty_rate,
}


def calculate(order: Order, discount_type: str) -> float:
    return DISCOUNTS.get(discount_type, lambda _: 0.0)(order)
```

### When NOT to Apply OCP

- Don't create a `Protocol` for logic that has only one implementation and is unlikely to change
- A simple `match` on 3 stable variants doesn't need a strategy pattern
- Over-applying OCP leads to premature abstraction — wait for the second or third variation

### How to Detect OCP Violations

- `if/elif` chain or `match` on a `type`/`status` string that grows over time
- Enum-based dispatching with frequent new values
- Changes require modifying core classes

### Common OCP Patterns

| Pattern | Use When |
|---------|----------|
| Strategy (Protocol) | Multiple algorithms for same operation |
| Registry / dict dispatch | Map a key to a behavior, extend by registering |
| Decorator (`functools.wraps`) | Add behavior dynamically |
| Plugin via entry points | Third parties extend without forking |

See `design-patterns`.

---

## L - Liskov Substitution Principle (LSP)

> "Subtypes must be substitutable for their base types."

### Violation

```python
# BAD: Square violates Rectangle's contract
class Rectangle:
    def __init__(self, width: int, height: int) -> None:
        self._width = width
        self._height = height

    def set_width(self, width: int) -> None:
        self._width = width

    def set_height(self, height: int) -> None:
        self._height = height

    @property
    def area(self) -> int:
        return self._width * self._height


class Square(Rectangle):
    def set_width(self, width: int) -> None:
        self._width = width
        self._height = width  # Violates expected behavior!

    def set_height(self, height: int) -> None:
        self._width = height  # Violates expected behavior!
        self._height = height


# This test fails for Square!
def test_rectangle(r: Rectangle) -> None:
    r.set_width(5)
    r.set_height(4)
    assert r.area == 20  # Square returns 16!
```

### Refactored

```python
# GOOD: Separate, immutable abstractions
from dataclasses import dataclass
from typing import Protocol


class Shape(Protocol):
    @property
    def area(self) -> int: ...


@dataclass(frozen=True)
class Rectangle:
    width: int
    height: int

    @property
    def area(self) -> int:
        return self.width * self.height


@dataclass(frozen=True)
class Square:
    side: int

    @property
    def area(self) -> int:
        return self.side * self.side
```

`frozen=True` removes the mutating setters entirely, so the LSP trap can't exist.

### LSP Rules

| Rule | Meaning |
|------|---------|
| Preconditions | Subtype cannot strengthen (require more) |
| Postconditions | Subtype cannot weaken (promise less) |
| Invariants | Subtype must maintain the base's invariants |
| Signatures | Overrides keep compatible parameter/return types (mypy enforces this) |

### How to Detect LSP Violations

- Subclass raises an exception the base does not
- Subclass returns `None` where the base returns a value
- Subclass narrows an accepted argument type (mypy reports `incompatible override`)
- `isinstance` checks before calling methods
- Empty overrides or `raise NotImplementedError` in a concrete subclass

### When NOT to Apply LSP

- Not every related set of classes needs to be substitutable — use a tagged union instead of inheritance
- Don't force inheritance just to reuse code — prefer composition

### Quick Check

```python
# If you see this, LSP might be violated
if isinstance(bird, Penguin):
    pass  # don't call fly()
else:
    bird.fly()

# Pythonic fix: a tagged union + structural pattern matching, no fragile base class.
from dataclasses import dataclass


@dataclass(frozen=True)
class FlyingBird:
    name: str


@dataclass(frozen=True)
class FlightlessBird:
    name: str


type Bird = FlyingBird | FlightlessBird


def move(bird: Bird) -> str:
    match bird:
        case FlyingBird(name):
            return f"{name} flies"
        case FlightlessBird(name):
            return f"{name} swims"
```

See `flatten-nesting` for replacing `isinstance` ladders with `match`.

---

## I - Interface Segregation Principle (ISP)

> "Clients should not be forced to depend on interfaces they do not use."

### Violation

```python
# BAD: Fat protocol forces unnecessary implementations
from typing import Protocol


class Worker(Protocol):
    def work(self) -> None: ...
    def eat(self) -> None: ...
    def sleep(self) -> None: ...
    def attend_meeting(self) -> None: ...
    def write_report(self) -> None: ...


# Robot can't eat or sleep!
class Robot:
    def work(self) -> None: ...
    def eat(self) -> None:
        raise NotImplementedError("Robots don't eat")  # smell
    def sleep(self) -> None:
        raise NotImplementedError("Robots don't sleep")  # smell
    def attend_meeting(self) -> None: ...
    def write_report(self) -> None: ...
```

### Refactored

```python
# GOOD: Segregated protocols — compose only what a client needs
from typing import Protocol


class Workable(Protocol):
    def work(self) -> None: ...


class Feedable(Protocol):
    def eat(self) -> None: ...
    def sleep(self) -> None: ...


class Manageable(Protocol):
    def attend_meeting(self) -> None: ...
    def write_report(self) -> None: ...


# Structural typing: Employee just has all the methods — no explicit "implements".
class Employee:
    def work(self) -> None: ...
    def eat(self) -> None: ...
    def sleep(self) -> None: ...
    def attend_meeting(self) -> None: ...
    def write_report(self) -> None: ...


class Robot:
    def work(self) -> None: ...
    # No unnecessary methods!


class Intern:
    def work(self) -> None: ...
    def eat(self) -> None: ...
    def sleep(self) -> None: ...
    # No meeting/report methods!


def run_shift(w: Workable) -> None:  # accepts Robot, Intern, Employee
    w.work()
```

Because `Protocol` is structural, `Robot` satisfies `Workable` without inheriting from it — mypy verifies the match at the call site.

### How to Detect ISP Violations

- Implementations with empty methods or `raise NotImplementedError`
- A protocol/ABC with 10+ methods
- Different callers use completely different subsets of methods
- Changes to one method's signature ripple into unrelated implementations

### Python Example: read-only vs read-write protocols for ISP

```python
# GOOD: Separate read and write contracts
from typing import Protocol


class ReadOnlyRepository[T, ID](Protocol):
    def find_by_id(self, id_: ID) -> T | None: ...
    def find_all(self) -> list[T]: ...
    def count(self) -> int: ...


class WriteRepository[T, ID](ReadOnlyRepository[T, ID], Protocol):
    def save(self, entity: T) -> T: ...
    def delete_by_id(self, id_: ID) -> None: ...


# A read-only service literally cannot call save()/delete() — wrong type.
class ReportService:
    def __init__(self, orders: ReadOnlyRepository[Order, int]) -> None:
        self._orders = orders
```

PEP 695 generic syntax (`class Foo[T]`) is the modern form. See `modern-python`.

### When NOT to Apply ISP

- Don't split a 3-method protocol into three 1-method protocols — that's over-segregation
- A cohesive protocol where most clients use most methods is fine
- Only split when you see implementations with `NotImplementedError` or empty methods

---

## D - Dependency Inversion Principle (DIP)

> "High-level modules should not depend on low-level modules. Both should depend on abstractions."

### Violation

```python
# BAD: High-level service constructs its own low-level dependencies
class OrderService:
    def __init__(self) -> None:
        self._repository = PostgresOrderRepository()  # hard dependency
        self._email = SmtpEmailSender()               # hard dependency

    def create_order(self, order: Order) -> None:
        self._repository.save(order)
        self._email.send(order.customer_email, "Order confirmed")
```

**Problems:**
- Cannot test without a real Postgres database
- Cannot swap the email provider
- `OrderService` knows about Postgres and SMTP details

### Refactored

```python
# GOOD: Depend on abstractions (Protocols), inject the concretions
from typing import Protocol


# Abstractions
class OrderRepository(Protocol):
    def save(self, order: Order) -> None: ...
    def find_by_id(self, id_: int) -> Order | None: ...


class NotificationSender(Protocol):
    def send(self, recipient: str, message: str) -> None: ...


# High-level module depends only on the abstractions
class OrderService:
    def __init__(
        self,
        repository: OrderRepository,
        notifier: NotificationSender,
    ) -> None:
        self._repository = repository
        self._notifier = notifier

    def create_order(self, order: Order) -> None:
        self._repository.save(order)
        self._notifier.send(order.customer_email, "Order confirmed")


# Low-level modules implement the abstractions
class PostgresOrderRepository:
    def save(self, order: Order) -> None: ...           # Postgres-specific
    def find_by_id(self, id_: int) -> Order | None: ...  # Postgres-specific


class SmtpEmailSender:
    def send(self, recipient: str, message: str) -> None: ...  # SMTP-specific


# Easy to test with a fake — no mocking framework required
class InMemoryOrderRepository:
    def __init__(self) -> None:
        self._orders: dict[int, Order] = {}

    def save(self, order: Order) -> None:
        self._orders[order.id] = order

    def find_by_id(self, id_: int) -> Order | None:
        return self._orders.get(id_)
```

### DIP in tests

```python
# GOOD: Inject the in-memory fake — fast, deterministic, no I/O.
import pytest


def test_create_order_persists_and_notifies() -> None:
    repo = InMemoryOrderRepository()
    notifier = RecordingNotifier()
    service = OrderService(repo, notifier)

    service.create_order(Order(id=1, customer_email="a@b.com"))

    assert repo.find_by_id(1) is not None
    assert notifier.sent == [("a@b.com", "Order confirmed")]
```

A hand-written fake that satisfies the `Protocol` is usually clearer than
`unittest.mock.Mock`, and mypy still type-checks it. See `testing-python`.

### When NOT to Apply DIP

- Don't create a `Protocol` for a class that will only ever have one implementation
- Dataclasses, `NamedTuple`s, value objects, and pure utility functions don't need abstraction — use them directly
- Only abstract at boundaries (repositories, external services, notification channels)

### How to Detect DIP Violations

- Constructing a concrete class (`PostgresRepository()`) inside business logic
- Imports pull in implementation packages (`psycopg`, `smtplib`, `requests`) at the high-level layer
- Cannot easily swap implementations
- Tests require real infrastructure (database, network)

See `clean-architecture` for how DIP shapes layer boundaries.

---

## SOLID Review Checklist

When reviewing code, check:

| Principle | Question |
|-----------|----------|
| **SRP** | Does this class have more than one reason to change? |
| **OCP** | Will adding a new type/feature require modifying this class? |
| **LSP** | Can subtypes be used wherever the base is expected (mypy clean)? |
| **ISP** | Are there empty or `NotImplementedError` method bodies? |
| **DIP** | Does high-level code construct or import concrete implementations? |

---

## Common Refactoring Patterns

| Violation | Refactoring |
|-----------|-------------|
| SRP - God class | Extract Class, Move Method to module function |
| OCP - Type switching | Strategy via Protocol, dict dispatch |
| LSP - Broken inheritance | Composition over inheritance, frozen dataclass, tagged union |
| ISP - Fat protocol | Split into role Protocols |
| DIP - Hard dependencies | Constructor injection, depend on Protocols |

See `refactoring-python`.

---

## Related Skills

- `design-patterns` — Patterns that implement SOLID principles (Strategy for OCP, Decorator for OCP/SRP)
- `refactoring-python` — Refactoring techniques to fix SOLID violations
- `clean-architecture` — Architectural patterns built on SOLID foundations
- `modern-python` — Protocols, PEP 695 generics, `type` aliases, and pattern matching used throughout
- `testing-python` — Fakes and dependency injection that DIP enables
