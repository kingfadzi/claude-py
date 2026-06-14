---
name: design-patterns
description: Common design patterns with idiomatic Python 3.12/3.13 examples (Factory, Builder, Strategy, Observer, Decorator, Chain of Responsibility, Command, Facade, Proxy) using dataclasses, protocols, structural pattern matching, and context managers. Use when implementing patterns, designing extensible components, or reviewing architecture.
---

# Design Patterns

Practical design patterns reference for modern Python (3.12/3.13). Many "patterns" from Java collapse into a single Python idiom — prefer the language feature over ceremony.

## When to Use

- User asks to implement a specific pattern
- Designing extensible/flexible components
- Refactoring rigid code structures
- Code review suggests pattern usage

---

## Quick Reference: When to Use What

| Problem | Pattern / Pythonic answer |
|---------|---------------------------|
| Complex object construction | **Builder** (or dataclass with defaults) |
| Create objects without specifying class | **Factory** (often a dict dispatch) |
| Multiple algorithms, swap at runtime | **Strategy** (often a plain callable) |
| Add behavior without changing class | **Decorator** (the `@decorator` syntax) |
| Notify multiple objects of changes | **Observer** (callbacks / `blinker`) |
| Ensure single instance | **Singleton** (a module is one) |
| Convert incompatible interfaces | **Adapter** |
| Define algorithm skeleton | **Template Method** (or function + hooks) |
| Sequential processing pipeline | **Chain of Responsibility** |
| Encapsulate action as object | **Command** (often a closure) |
| Simplify complex subsystem | **Facade** |
| Control access / add cross-cutting behavior | **Proxy** (or a decorator) |
| Scoped acquire/release of a resource | **Context manager** (`with`) |

> Python is dynamically typed with first-class functions. Before reaching for a
> class hierarchy, ask: would a function, a closure, or a dict do? See
> `efficient-python` and `modern-python`.

---

## Creational Patterns

### Builder

**Use when:** Object has many parameters, some optional.

In Python, keyword arguments with defaults usually replace the Builder entirely.
Reach for a true builder only when construction is multi-step or validated.

```python
# BAD: Telescoping constructor antipattern. Python has no overloading, so you
# end up with a pile of None defaults and branching validation in one __init__.
class User:
    def __init__(self, name, email=None, age=None, phone=None, address=None): ...

# GOOD: Frozen dataclass with keyword-only fields and defaults
from dataclasses import dataclass


@dataclass(frozen=True, kw_only=True, slots=True)
class User:
    name: str
    email: str
    age: int = 0
    phone: str = ""
    address: str = ""


user = User(name="John", email="john@example.com", age=30, phone="+1234567890")
```

When construction is genuinely multi-step or validated, a fluent builder still pays off:

```python
# GOOD: Builder for incremental, validated construction
from dataclasses import dataclass
from enum import Enum
from typing import Self


class SortDirection(Enum):
    ASC = "asc"
    DESC = "desc"


@dataclass(frozen=True, slots=True)
class SearchCriteria:
    query: str = ""
    page: int = 0
    size: int = 20
    sort_by: str = "id"
    direction: SortDirection = SortDirection.ASC


class SearchCriteriaBuilder:
    def __init__(self) -> None:
        self._kwargs: dict[str, object] = {}

    def query(self, value: str) -> Self:
        self._kwargs["query"] = value
        return self

    def page(self, value: int) -> Self:
        self._kwargs["page"] = max(value, 0)
        return self

    def size(self, value: int) -> Self:
        self._kwargs["size"] = value if value > 0 else 20
        return self

    def sort_by(self, value: str) -> Self:
        self._kwargs["sort_by"] = value
        return self

    def build(self) -> SearchCriteria:
        return SearchCriteria(**self._kwargs)  # type: ignore[arg-type]


criteria = SearchCriteriaBuilder().query("python typing").size(50).sort_by("date").build()
```

> For config-shaped objects, prefer a `pydantic.BaseModel` or a `@dataclass`
> with validation in `__post_init__` over a hand-rolled builder.

---

### Factory

**Use when:** Need to create objects without naming the exact class at the call site.

```python
# BAD: Client must know every concrete type; branching grows forever
def make_notification(kind: str, target: str):
    if kind == "EMAIL":
        return EmailNotification(target)
    elif kind == "SMS":
        return SmsNotification(target)
    # ... a new elif for every type

# GOOD: Protocol + structural pattern matching
from dataclasses import dataclass
from typing import Protocol


class Notification(Protocol):
    def send(self, message: str) -> None: ...


@dataclass(frozen=True, slots=True)
class EmailNotification:
    recipient: str

    def send(self, message: str) -> None: ...  # send email


@dataclass(frozen=True, slots=True)
class SmsNotification:
    phone_number: str

    def send(self, message: str) -> None: ...  # send SMS


def create_notification(kind: str, target: str) -> Notification:
    match kind.upper():
        case "EMAIL":
            return EmailNotification(target)
        case "SMS":
            return SmsNotification(target)
        case _:
            raise ValueError(f"Unknown notification type: {kind}")
```

**Registry factory (preferred for open extension) — register without editing the factory:**

```python
from collections.abc import Callable

_REGISTRY: dict[str, Callable[[str], Notification]] = {}


def register(kind: str) -> Callable[[type], type]:
    def decorator(cls: type) -> type:
        _REGISTRY[kind] = cls
        return cls
    return decorator


@register("EMAIL")
@dataclass(frozen=True, slots=True)
class EmailNotification:
    recipient: str

    def send(self, message: str) -> None: ...


def create_notification(kind: str, target: str) -> Notification:
    try:
        return _REGISTRY[kind.upper()](target)
    except KeyError:
        raise ValueError(f"Unknown notification type: {kind}") from None
```

---

### Singleton

**Use when:** Exactly one instance needed (use sparingly!).

In Python, **a module is already a singleton** — imported once and cached in
`sys.modules`. Put shared state at module level rather than building a class.

```python
# GOOD: Module-level state IS the singleton (app_registry.py)
_registry: dict[str, object] = {}


def register(key: str, value: object) -> None:
    _registry[key] = value


def get(key: str) -> object | None:
    return _registry.get(key)


# If you truly need an object, cache the factory:
from functools import lru_cache


@lru_cache(maxsize=1)
def get_settings() -> "Settings":
    return Settings.from_env()
```

**Warning:** Singletons are global mutable state — hard to test, hidden
dependencies. Prefer passing the dependency explicitly. See `solid-principles`
and `testing-python`.

---

## Behavioral Patterns

### Strategy

**Use when:** Multiple algorithms for the same operation, swappable at runtime.

```python
# BAD: if/elif chain that grows with every new payment type
def pay(kind: str, amount: Decimal) -> None:
    if kind == "CREDIT_CARD":
        ...
    elif kind == "PAYPAL":
        ...
    # Every new payment method = edit this function

# GOOD: Strategy as a Protocol
from dataclasses import dataclass
from decimal import Decimal
from typing import Protocol


class PaymentStrategy(Protocol):
    def pay(self, amount: Decimal) -> None: ...


@dataclass(frozen=True, slots=True)
class CreditCardPayment:
    card_number: str

    def pay(self, amount: Decimal) -> None:
        print(f"Paid {amount} with card {self.card_number}")


class ShoppingCart:
    def __init__(self, strategy: PaymentStrategy) -> None:
        self._strategy = strategy

    def checkout(self, total: Decimal) -> None:
        self._strategy.pay(total)
```

**Functional style (preferred when the strategy is one method) — just pass a callable:**

```python
from collections.abc import Callable
from decimal import Decimal

PaymentStrategy = Callable[[Decimal], None]

credit_card: PaymentStrategy = lambda amt: print(f"Card: {amt}")
paypal: PaymentStrategy = lambda amt: print(f"PayPal: {amt}")


def checkout(total: Decimal, pay: PaymentStrategy) -> None:
    pay(total)
```

**Selection across strategies — pick the first that supports the input:**

```python
from collections.abc import Sequence


def calculate_price(
    strategies: Sequence["PricingStrategy"],
    customer_type: "CustomerType",
    product: "Product",
    quantity: int,
) -> Decimal:
    strategy = next((s for s in strategies if s.supports(customer_type)), None)
    if strategy is None:
        raise ValueError(f"No pricing strategy for {customer_type}")
    return strategy.calculate(product, quantity)
```

---

### Observer

**Use when:** Objects need to be notified of changes in another object.

```python
# BAD: Direct coupling — the service knows every collaborator
class OrderService:
    def __init__(self, inventory, email, analytics) -> None:
        self._inventory, self._email, self._analytics = inventory, email, analytics

    def place_order(self, order: "Order") -> None:
        self._save(order)
        self._inventory.reduce_stock(order)   # tightly coupled
        self._email.send_confirmation(order)  # tightly coupled
        self._analytics.track(order)          # tightly coupled

# GOOD: A subscriber list decouples publisher from listeners
from collections.abc import Callable

OrderListener = Callable[["Order"], None]


class OrderService:
    def __init__(self) -> None:
        self._listeners: list[OrderListener] = []

    def subscribe(self, listener: OrderListener) -> None:
        self._listeners.append(listener)

    def place_order(self, order: "Order") -> None:
        self._save(order)
        for notify in self._listeners:
            notify(order)


service = OrderService()
service.subscribe(lambda o: inventory.reduce_stock(o))
service.subscribe(lambda o: email.send_confirmation(o))
```

> For a richer signal/slot system, use the `blinker` library (named signals,
> weak refs) instead of hand-rolling; in Django prefer `django.dispatch.Signal`.
> See `leverage-libraries`.

---

### Template Method

**Use when:** Define an algorithm skeleton, let subclasses fill in steps.

```python
# GOOD: ABC with abstract steps and a concrete hook
from abc import ABC, abstractmethod
from pathlib import Path


class DataExporter(ABC):
    def export(self) -> None:  # template method — fixed algorithm
        data = self.fetch_data()
        transformed = self.transform(data)
        self.write(transformed)
        if self.should_notify():
            self.notify_completion()

    @abstractmethod
    def fetch_data(self) -> list[dict]: ...

    @abstractmethod
    def transform(self, data: list[dict]) -> list[str]: ...

    @abstractmethod
    def write(self, output: list[str]) -> None: ...

    def should_notify(self) -> bool:  # hook with a default
        return True

    def notify_completion(self) -> None:
        logger.info("Export completed")


class CsvExporter(DataExporter):
    def fetch_data(self) -> list[dict]:
        return repository.find_all()

    def transform(self, data: list[dict]) -> list[str]:
        return [to_csv_row(row) for row in data]

    def write(self, output: list[str]) -> None:
        Path("export.csv").write_text("\n".join(output))
```

**Functional template (preferred when steps are stateless) — inject the steps:**

```python
from collections.abc import Callable


def export(
    fetch: Callable[[], list[dict]],
    transform: Callable[[list[dict]], list[str]],
    write: Callable[[list[str]], None],
) -> None:
    write(transform(fetch()))
```

---

### Chain of Responsibility

**Use when:** Pass a request along a chain of handlers; each decides to process or pass along.

```python
# BAD: Monolithic validation with stacked ifs
def validate(order: "Order") -> None:
    if not order.items:
        raise ValidationError("No items")
    if order.total <= 0:
        raise ValidationError("Invalid total")
    if fraud_service.is_suspicious(order):
        raise ValidationError("Fraud detected")
    # ... grows endlessly

# GOOD: Chain as an ordered list of validators
from typing import Protocol


class OrderValidator(Protocol):
    def validate(self, order: "Order") -> None: ...


class ItemValidator:
    def validate(self, order: "Order") -> None:
        if not order.items:
            raise ValidationError("Order must have items")


class FraudValidator:
    def __init__(self, fraud_service: "FraudService") -> None:
        self._fraud = fraud_service

    def validate(self, order: "Order") -> None:
        if self._fraud.is_suspicious(order):
            raise ValidationError("Order flagged for fraud")


class OrderValidationChain:
    def __init__(self, validators: list[OrderValidator]) -> None:
        self._validators = validators  # ordering is explicit in the list

    def validate(self, order: "Order") -> None:
        for validator in self._validators:
            validator.validate(order)


chain = OrderValidationChain([ItemValidator(), FraudValidator(fraud)])
```

> In a web app the framework gives you ready-made chains: WSGI/ASGI middleware
> or Django's `MIDDLEWARE` list. See `django-rest-api` and `django-security`.

---

### Command

**Use when:** Encapsulate a request as an object — enables queuing, logging, undo.

```python
# GOOD: Commands as frozen dataclasses + match dispatch
from dataclasses import dataclass
from decimal import Decimal


@dataclass(frozen=True, slots=True)
class CreateOrder:
    customer_id: str
    items: list["LineItem"]


@dataclass(frozen=True, slots=True)
class CancelOrder:
    order_id: int
    reason: str


@dataclass(frozen=True, slots=True)
class RefundOrder:
    order_id: int
    amount: Decimal


type Command = CreateOrder | CancelOrder | RefundOrder


def dispatch(command: Command) -> None:
    match command:
        case CreateOrder(customer_id=cid, items=items):
            logger.info("Creating order for %s (%d items)", cid, len(items))
        case CancelOrder(order_id=oid, reason=reason):
            logger.info("Cancelling order %d: %s", oid, reason)
        case RefundOrder(order_id=oid, amount=amount):
            logger.info("Refunding %s on order %d", amount, oid)
```

**Often a closure is enough — a "command" is just a deferred call:**

```python
from collections.abc import Callable

queue: list[Callable[[], None]] = []
queue.append(lambda: service.create_order(customer_id, items))
queue.append(lambda: service.cancel_order(order_id, reason))

for command in queue:
    command()
```

---

### Facade

**Use when:** Simplify a complex subsystem behind a single entry point.

```python
# BAD: The view orchestrates many services directly
def checkout_view(request):
    cart = cart_service.get_cart(request.cart_id)
    inventory_service.reserve(cart.items)
    total = pricing_service.calculate(cart)
    final = promotion_service.apply(request.promo_code, total)
    payment = payment_service.charge(request.payment_method, final)
    order = order_service.create(cart, payment)
    notification_service.send_confirmation(order)
    return order

# GOOD: A facade encapsulates the workflow; the view stays thin
from dataclasses import dataclass


@dataclass(slots=True)
class CheckoutFacade:
    cart_service: "CartService"
    inventory_service: "InventoryService"
    pricing_service: "PricingService"
    promotion_service: "PromotionService"
    payment_service: "PaymentService"
    order_service: "OrderService"
    notification_service: "NotificationService"

    def checkout(self, request: "CheckoutRequest") -> "Order":
        cart = self.cart_service.get_cart(request.cart_id)
        self.inventory_service.reserve(cart.items)
        total = self.pricing_service.calculate(cart)
        final = self.promotion_service.apply(request.promo_code, total)
        payment = self.payment_service.charge(request.payment_method, final)
        order = self.order_service.create(cart, payment)
        self.notification_service.send_confirmation(order)
        return order


def checkout_view(request):  # now trivial
    return checkout_facade.checkout(parse_request(request))
```

> Wrap the whole workflow in `with transaction.atomic():` so a mid-flow failure
> rolls back. See `django-orm`.

---

### Proxy

**Use when:** Control access to an object or add cross-cutting behavior.

In Python the natural proxy is a **decorator** — it wraps a callable to add
behavior transparently.

```python
# GOOD: Caching and timing proxies via decorators
import time
from collections.abc import Callable
from functools import lru_cache, wraps
from typing import ParamSpec, TypeVar

P = ParamSpec("P")
R = TypeVar("R")


def timed(func: Callable[P, R]) -> Callable[P, R]:
    @wraps(func)  # preserves __name__/__doc__/signature for tooling
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        start = time.perf_counter()
        try:
            return func(*args, **kwargs)
        finally:
            logger.info("%s took %.3fs", func.__name__, time.perf_counter() - start)
    return wrapper


class ProductService:
    @lru_cache(maxsize=128)  # proxy: returns cached result
    def find_by_id(self, product_id: int) -> "Product":
        return repository.get(product_id)

    @timed  # proxy: logs duration around the real method
    def generate_report(self, product_id: int) -> "Report": ...
```

**`__getattr__` proxy — forward everything to a wrapped (lazily built) object:**

```python
class LazyProxy:
    def __init__(self, factory: Callable[[], object]) -> None:
        self._factory = factory
        self._target: object | None = None

    def __getattr__(self, name: str) -> object:
        if self._target is None:
            self._target = self._factory()
        return getattr(self._target, name)
```

---

## Structural Patterns

### Decorator

**Use when:** Add behavior dynamically without modifying existing classes.

The classic GoF Decorator (wrapping an object to extend it) is clean with Protocols:

```python
# GOOD: Object decorator chain via a Protocol
import json
from dataclasses import dataclass
from datetime import UTC, datetime
from typing import Protocol


class Logger(Protocol):
    def log(self, message: str) -> None: ...


@dataclass(frozen=True, slots=True)
class ConsoleLogger:
    def log(self, message: str) -> None:
        print(message)


@dataclass(frozen=True, slots=True)
class TimestampLogger:
    delegate: Logger

    def log(self, message: str) -> None:
        self.delegate.log(f"[{datetime.now(UTC).isoformat()}] {message}")


@dataclass(frozen=True, slots=True)
class JsonLogger:
    delegate: Logger

    def log(self, message: str) -> None:
        self.delegate.log(json.dumps({"message": message}))


logger: Logger = TimestampLogger(JsonLogger(ConsoleLogger()))  # compose
logger.log("Hello")
```

**Function decorator (the everyday case) — wrap behavior around a call:**

```python
from functools import wraps


def retry(times: int) -> Callable[[Callable[P, R]], Callable[P, R]]:
    def decorator(func: Callable[P, R]) -> Callable[P, R]:
        @wraps(func)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except Exception:
                    if attempt == times:
                        raise
            raise AssertionError("unreachable")
        return wrapper
    return decorator


@retry(times=3)
def fetch(url: str) -> bytes: ...
```

---

### Adapter

**Use when:** Make incompatible interfaces work together.

```python
from decimal import Decimal
from typing import Protocol


class PaymentGateway(Protocol):  # interface our code expects
    def charge(self, amount: Decimal, currency: str) -> "PaymentResult": ...


class StripeClient:  # third-party SDK: cents, different method name
    def create_charge(self, amount_in_cents: int, cur: str) -> "StripeCharge": ...


class StripePaymentAdapter:  # adapter bridges the gap
    def __init__(self, stripe: StripeClient) -> None:
        self._stripe = stripe

    def charge(self, amount: Decimal, currency: str) -> "PaymentResult":
        charge = self._stripe.create_charge(int(amount * 100), currency)
        return PaymentResult(id=charge.id, status=charge.status)
```

---

### Context Manager (Python's scoped-resource pattern)

**Use when:** A resource must be acquired and released as a pair — replaces
Java's try-with-resources. This is the idiomatic Python structural pattern.

```python
# BAD: Manual try/finally at every call site
conn = pool.acquire()
try:
    conn.execute(query)
finally:
    pool.release(conn)

# GOOD: contextlib.contextmanager
from collections.abc import Iterator
from contextlib import contextmanager


@contextmanager
def borrow_connection(pool: "Pool") -> Iterator["Connection"]:
    conn = pool.acquire()
    try:
        yield conn
    finally:
        pool.release(conn)


with borrow_connection(pool) as conn:
    conn.execute(query)
```

See `exception-handling` for cleanup semantics and `python-concurrency` for `async with`.

---

## Pattern Selection Guide

| Situation | Consider |
|-----------|----------|
| Object creation is complex | dataclass defaults, Builder, Factory |
| Need to add features dynamically | Decorator (function or object) |
| Multiple implementations of an algorithm | Strategy (often a callable) |
| React to state changes | Observer (callbacks / `blinker`) |
| Integrate with legacy/third-party code | Adapter |
| Common algorithm, varying steps | Template Method (or inject callables) |
| Need single instance | Module-level state, `@lru_cache` |
| Sequential processing/validation pipeline | Chain of Responsibility |
| Encapsulate action as replayable object | Command (often a closure) |
| Simplify a complex multi-service workflow | Facade |
| Scoped resource acquire/release | Context manager (`with`) |
| Cross-cutting concerns (cache, timing, retry) | Decorator / Proxy |

---

## Anti-Patterns to Avoid

| Anti-Pattern | Problem | Better Approach |
|--------------|---------|-----------------|
| Class for everything | Java-in-Python; ceremony | Use functions, closures, dicts |
| Singleton class | Global mutable state, hard to test | Module state, pass dependencies in |
| Factory everywhere | Over-engineering | Direct construction when type is known |
| Deep decorator chains | Hard-to-read stack traces | Keep chains short; use `functools.wraps` |
| Hand-rolled signal system | Reinventing the wheel | `blinker` / Django signals |
| Forgetting `functools.wraps` | Breaks introspection & docs | Always `@wraps(func)` in decorators |
| God facade | Facade becomes a God class | One facade per workflow |
| Command without benefit | Indirection for plain CRUD | Use commands only for queue/audit/undo |

---

## Related Skills

- `solid-principles` — Design principles that patterns help implement
- `refactoring-python` — Refactoring techniques that often lead to patterns
- `modern-python` — Dataclasses, protocols, pattern matching, `type` aliases used throughout
- `efficient-python` — When a function/closure beats a class hierarchy
- `leverage-libraries` — Prefer `blinker`, `pydantic`, stdlib over hand-rolled patterns
- `flatten-nesting` — Replacing the if/elif chains these patterns target
