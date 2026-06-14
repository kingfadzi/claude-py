---
name: refactoring-python
description: Python refactoring techniques — size-based triggers, Extract Function/Class, replace conditional with polymorphism or pattern matching, introduce parameter object (dataclasses), simplify with modern Python, and code smell mapping. Use when refactoring large functions/classes, simplifying complex logic, or modernizing legacy code.
---

# Refactoring Python

Systematic refactoring techniques for Python with modern idioms.

## When to Use

- Function or class exceeds size thresholds
- Code review identifies code smells
- Simplifying complex conditional logic
- Modernizing legacy code to Python 3.12/3.13 idioms
- Reducing duplication or improving readability

---

## Quick Reference: Size-Based Triggers

| Metric | Threshold | Refactoring |
|--------|-----------|-------------|
| Function LOC | > 40 lines | Extract Function |
| Class LOC | > 250 lines | Extract Class (check SRP) |
| Parameters | > 3 | Introduce Parameter Object (dataclass) |
| Nesting depth | > 2 levels | Guard clauses / Extract Function |
| Cyclomatic complexity | > 5 branches | Replace conditional with polymorphism |

Let `ruff` flag these for you: enable `C901` (mccabe complexity), `PLR0913` (too many
arguments) and `PLR0915` (too many statements) in your ruff config.

---

## Extract Function

The most common refactoring — break a long function into named steps.

```python
# BAD: 50-line function doing multiple things
def process_order(order: Order) -> OrderResult:
    # Validate order (10 lines)
    if not order.items:
        raise ValidationError("Order must have items")
    for item in order.items:
        if item.quantity <= 0:
            raise ValidationError("Quantity must be positive")
        if item.price <= Decimal(0):
            raise ValidationError("Price must be positive")

    # Calculate totals (10 lines)
    subtotal = Decimal(0)
    for item in order.items:
        subtotal += item.price * item.quantity
    tax = subtotal * TAX_RATE
    shipping = Decimal(0) if subtotal >= FREE_SHIPPING_THRESHOLD else SHIPPING_COST
    total = subtotal + tax + shipping

    # Apply discount (8 lines)
    discount = Decimal(0)
    if order.promo_code is not None:
        promo = promo_service.find_by_code(order.promo_code)
        if promo is not None and promo.is_valid:
            discount = total * promo.rate
    total -= discount

    # Save and notify (8 lines)
    order.total = total
    order.status = OrderStatus.CONFIRMED
    order_repository.save(order)
    event_publisher.publish(OrderConfirmedEvent(order))
    email_service.send_confirmation(order)
    return OrderResult(order.id, total, OrderStatus.CONFIRMED)


# GOOD: Extracted into named, individually testable steps
def process_order(order: Order) -> OrderResult:
    _validate_order(order)
    total = _calculate_total(order)
    total = _apply_discount(total, order.promo_code)
    return _confirm_order(order, total)


def _validate_order(order: Order) -> None:
    if not order.items:
        raise ValidationError("Order must have items")
    for item in order.items:
        _validate_line_item(item)


def _validate_line_item(item: LineItem) -> None:
    if item.quantity <= 0:
        raise ValidationError("Quantity must be positive")
    if item.price <= Decimal(0):
        raise ValidationError("Price must be positive")


def _calculate_total(order: Order) -> Decimal:
    subtotal = sum((item.price * item.quantity for item in order.items), Decimal(0))
    tax = subtotal * TAX_RATE
    shipping = Decimal(0) if subtotal >= FREE_SHIPPING_THRESHOLD else SHIPPING_COST
    return subtotal + tax + shipping


def _apply_discount(total: Decimal, promo_code: str | None) -> Decimal:
    if promo_code is None:
        return total
    promo = promo_service.find_by_code(promo_code)
    if promo is None or not promo.is_valid:
        return total
    return total - total * promo.rate


def _confirm_order(order: Order, total: Decimal) -> OrderResult:
    order.total = total
    order.status = OrderStatus.CONFIRMED
    order_repository.save(order)
    event_publisher.publish(OrderConfirmedEvent(order))
    email_service.send_confirmation(order)
    return OrderResult(order.id, total, OrderStatus.CONFIRMED)
```

See `flatten-nesting` for the guard-clause patterns used above.

---

## Extract Class

When a class has too many responsibilities (> 250 LOC, multiple "reasons to change").

```python
# BAD: UserService handles users, notifications, and audit (300+ lines)
class UserService:
    def create_user(self, req: CreateUserRequest) -> User: ...
    def update_user(self, user_id: int, req: UpdateUserRequest) -> User: ...
    def delete_user(self, user_id: int) -> None: ...
    def send_welcome_email(self, user: User) -> None: ...
    def send_password_reset_email(self, user: User) -> None: ...
    def send_account_locked_email(self, user: User) -> None: ...
    def log_user_created(self, user: User) -> None: ...
    def log_user_updated(self, user: User, changes: dict[str, object]) -> None: ...
    def log_user_deleted(self, user_id: int) -> None: ...


# GOOD: Extracted by responsibility, collaborators injected
class UserService:
    def __init__(
        self,
        repository: UserRepository,
        notifications: UserNotificationService,
        audit: UserAuditService,
    ) -> None:
        self._repository = repository
        self._notifications = notifications
        self._audit = audit

    def create_user(self, req: CreateUserRequest) -> User:
        user = self._repository.save(_to_user(req))
        self._notifications.send_welcome(user)
        self._audit.log_created(user)
        return user


class UserNotificationService:
    def send_welcome(self, user: User) -> None: ...
    def send_password_reset(self, user: User) -> None: ...
    def send_account_locked(self, user: User) -> None: ...


class UserAuditService:
    def log_created(self, user: User) -> None: ...
    def log_updated(self, user: User, changes: dict[str, object]) -> None: ...
    def log_deleted(self, user_id: int) -> None: ...
```

See `solid-principles` (Single Responsibility) and `clean-architecture`.

---

## Introduce Parameter Object

When a function has more than 3 parameters.

```python
# BAD: Too many parameters
def search_products(
    query: str,
    category: str,
    min_price: Decimal,
    max_price: Decimal,
    sort_by: str = "name",
    sort_dir: str = "asc",
    page: int = 0,
    size: int = 20,
) -> SearchResult:
    ...


# GOOD: Parameter object as a frozen dataclass with normalization
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class ProductSearchCriteria:
    query: str
    category: str
    min_price: Decimal
    max_price: Decimal
    sort_by: str = "name"
    sort_dir: str = "asc"
    page: int = 0
    size: int = 20

    def __post_init__(self) -> None:
        # frozen dataclass: assign via object.__setattr__ to clamp values
        object.__setattr__(self, "page", max(self.page, 0))
        object.__setattr__(self, "size", min(max(self.size, 1), 100))


def search_products(criteria: ProductSearchCriteria) -> SearchResult:
    # Clean, extensible, self-documenting
    ...
```

For request bodies in a web API, a `pydantic.BaseModel` plays the same role with
validation built in — see `django-rest-api`.

---

## Replace Conditional with Polymorphism

```python
# BAD: Growing if/elif chain on a type field
def calculate_shipping(order: Order) -> Decimal:
    if order.type == "STANDARD":
        if order.total >= Decimal("50"):
            return Decimal(0)
        return Decimal("5.99")
    elif order.type == "EXPRESS":
        return Decimal("14.99")
    elif order.type == "OVERNIGHT":
        return Decimal("24.99")
    elif order.type == "INTERNATIONAL":
        return Decimal("29.99") + order.weight * Decimal("2.50")
    raise ValueError(f"Unknown type: {order.type}")


# GOOD: Structural pattern matching (when logic is simple)
def calculate_shipping(order: Order) -> Decimal:
    match order.type:
        case OrderType.STANDARD:
            return Decimal(0) if order.total >= Decimal("50") else Decimal("5.99")
        case OrderType.EXPRESS:
            return Decimal("14.99")
        case OrderType.OVERNIGHT:
            return Decimal("24.99")
        case OrderType.INTERNATIONAL:
            return Decimal("29.99") + order.weight * Decimal("2.50")
        case _:
            raise ValueError(f"Unknown type: {order.type}")


# GOOD: Strategy protocol (when logic is complex or extensible)
from typing import Protocol


class ShippingCalculator(Protocol):
    def calculate(self, order: Order) -> Decimal: ...


class StandardShipping:
    def calculate(self, order: Order) -> Decimal:
        return Decimal(0) if order.total >= Decimal("50") else Decimal("5.99")


class InternationalShipping:
    def calculate(self, order: Order) -> Decimal:
        return Decimal("29.99") + order.weight * Decimal("2.50")


# Register implementations by type — no isinstance, no growing branches
_CALCULATORS: dict[OrderType, ShippingCalculator] = {
    OrderType.STANDARD: StandardShipping(),
    OrderType.INTERNATIONAL: InternationalShipping(),
}

def calculate_shipping(order: Order) -> Decimal:
    return _CALCULATORS[order.type].calculate(order)
```

See `design-patterns` for the Strategy pattern in depth.

---

## Replace Inheritance with Composition

```python
# BAD: Deep inheritance for code reuse
class AbstractAuditableService(ABC, Generic[T]):
    @abstractmethod
    def save(self, entity: T) -> T: ...

    def create(self, entity: T) -> T:
        saved = self.save(entity)
        self._audit("CREATE", saved)  # behavior baked into the base class
        return saved


class AbstractValidatableService(AbstractAuditableService[T]):
    @abstractmethod
    def validate(self, entity: T) -> None: ...

    def create(self, entity: T) -> T:
        self.validate(entity)
        return super().create(entity)  # fragile super() chain


class OrderService(AbstractValidatableService[Order]):
    # Locked into a rigid hierarchy, hard to test in isolation
    ...


# GOOD: Composition — flexible, testable, mockable
class OrderService:
    def __init__(
        self,
        validator: OrderValidator,
        repository: OrderRepository,
        audit: AuditService,
    ) -> None:
        self._validator = validator
        self._repository = repository
        self._audit = audit

    def create_order(self, order: Order) -> Order:
        self._validator.validate(order)
        saved = self._repository.save(order)
        self._audit.log("CREATE", saved)
        return saved
```

---

## Simplify with Modern Python

### Dataclasses / NamedTuple Replace Value Classes

```python
# BAD: Manual value class — boilerplate __init__, __eq__, __repr__
class Address:
    def __init__(self, street: str, city: str, zip_code: str) -> None:
        self.street = street
        self.city = city
        self.zip_code = zip_code
    # __eq__, __hash__, __repr__ ... (30+ lines)


# GOOD: Frozen dataclass — equality, repr, hashing for free
@dataclass(frozen=True, slots=True)
class Address:
    street: str
    city: str
    zip_code: str


# GOOD: NamedTuple when you want a lightweight immutable record
class Point(NamedTuple):
    x: float
    y: float
```

### Pattern Matching Replaces isinstance Chains

```python
# BAD: isinstance checks everywhere
def process_result(result: Result) -> str:
    if isinstance(result, SuccessResult):
        return result.data
    elif isinstance(result, ErrorResult):
        return result.message
    elif isinstance(result, PendingResult):
        return "Processing..."
    raise RuntimeError("Unknown result type")


# GOOD: Discriminated union + structural pattern matching
@dataclass(frozen=True)
class Success:
    data: str

@dataclass(frozen=True)
class Error:
    message: str
    code: int

@dataclass(frozen=True)
class Pending:
    estimate: timedelta

Result = Success | Error | Pending


def process_result(result: Result) -> str:
    match result:
        case Success(data):
            return data
        case Error(message, _):
            return message
        case Pending(estimate):
            return f"Processing, ETA: {estimate}"
```

mypy will flag a missing case if you annotate the return and add an
`assert_never(result)` in the default branch — see `modern-python`.

### Comprehensions / Generators Replace Loop + Accumulator

```python
# BAD: Manual accumulation
orders_by_status: dict[str, list[Order]] = {}
for order in orders:
    orders_by_status.setdefault(order.status, []).append(order)

# GOOD: defaultdict for grouping; comprehension replaces filter+map+loop
from collections import defaultdict

orders_by_status: dict[str, list[Order]] = defaultdict(list)
for order in orders:
    orders_by_status[order.status].append(order)

active_totals = [o.total for o in orders if o.status == OrderStatus.ACTIVE]
```

See `readable-comprehensions` and `leverage-libraries` (itertools, collections).

---

## Code Smells -> Refactoring Mapping

| Code Smell | Indicator | Refactoring |
|-----------|-----------|-------------|
| Long Function | > 40 LOC | Extract Function |
| Large Class | > 250 LOC, multiple responsibilities | Extract Class |
| Long Parameter List | > 3 parameters | Introduce Parameter Object (dataclass) |
| Deeply Nested Code | > 2 levels of nesting | Guard Clauses, Extract Function |
| Feature Envy | Function uses another object's data more than its own | Move Method |
| Data Clumps | Same group of arguments appears together | Extract dataclass |
| Type Dispatch | Growing if-elif on a type field | Replace with polymorphism / match |
| Primitive Obsession | Using str/int for domain concepts | Value Objects (dataclasses), NewType |
| Middleman | Class/object that only delegates | Inline |
| Shotgun Surgery | One change requires editing many modules | Move Method, Extract Class |
| Duplicate Code | Same logic in multiple places | Extract Function, share helpers |
| Speculative Generality | Protocols/ABCs with one implementation | Inline, remove indirection |

---

## Refactoring Safety

1. **Have tests first** — never refactor without `pytest` coverage of the behavior
2. **Small steps** — one refactoring at a time, run `pytest` and `mypy` after each
3. **Don't change behavior** — refactoring preserves functionality
4. **Lean on tooling** — `ruff check --fix` and `ruff format` automate mechanical changes
5. **Commit between refactorings** — easy to revert if something breaks

```bash
uv run ruff check . && uv run ruff format . && uv run mypy . && uv run pytest
```

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Refactoring without tests | No safety net | Write tests first |
| Big-bang refactoring | High risk, hard to review | Small, incremental changes |
| Refactoring and adding features | Can't tell what changed behavior | Separate commits |
| Premature abstraction | Complexity without benefit | Wait for the third use case |
| Leaving dead code | Confusion, maintenance burden | Delete unused code |
| Over-using inheritance | Rigid, hard to test | Prefer composition / protocols |
| Refactoring for style only | Churn without value | Only if readability is genuinely poor |

---

## Related Skills

- `solid-principles` — Principles that guide when to refactor
- `design-patterns` — Patterns that emerge from refactoring
- `clean-architecture` — Architectural refactoring
- `flatten-nesting` — Guard clauses and reducing nesting depth
- `review-changes-python` — Identifying refactoring opportunities in review
- `efficient-python` — Performance-focused refactoring (data structures, loops)
