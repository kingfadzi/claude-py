---
name: modern-python
description: Modern Python idioms and features (Python 3.12-3.13) including PEP 695 type parameters/generics, structural pattern matching, dataclasses and NamedTuple, typing improvements (X | None, Self, override), pathlib, walrus operator, and f-strings. Use when writing new Python code, modernizing legacy code, or reviewing for outdated idioms.
---

# Modern Python (3.12-3.13)

Reference for modern Python idioms that should be preferred in all new code. This library targets Python 3.13 (with 3.12 as the floor). Standard tooling: `uv` (deps/venv), `ruff` (lint + format), `mypy` (types), `pytest` (tests).

## When to Use

- Writing new Python code (default to modern idioms)
- Reviewing code for pre-3.12 patterns that have better alternatives
- Modernizing legacy codebases (Python 2-isms, `typing.Optional`, manual `__init__`)
- Choosing between old and new approaches (dataclasses vs hand-written classes, `match` vs `if/elif` chains)

---

## Quick Reference

| Python Version | Key Features |
|----------------|-------------|
| **3.10** | Structural pattern matching (`match`), `X | Y` union syntax, parenthesized context managers |
| **3.11** | `Self` type, `ExceptionGroup`/`except*`, `tomllib`, `typing.assert_never`, faster CPython |
| **3.12** | PEP 695 type parameter syntax (`def f[T]`, `class C[T]`, `type` aliases), `@override`, f-string grammar cleanup |
| **3.13** | Improved error messages, experimental free-threaded build, `typing.ReadOnly`/`TypeIs`, deprecation of legacy aliases |

---

## Dataclasses

Default choice for DTOs, value objects, configuration holders, and any class whose identity is defined by its data.

```python
# BAD: Hand-written class with boilerplate
class UserDto:
    def __init__(self, name: str, email: str, age: int) -> None:
        self.name = name
        self.email = email
        self.age = age

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, UserDto):
            return NotImplemented
        return (
            self.name == other.name
            and self.email == other.email
            and self.age == other.age
        )

    def __hash__(self) -> int:
        return hash((self.name, self.email, self.age))

    def __repr__(self) -> str:
        return f"UserDto(name={self.name!r}, email={self.email!r}, age={self.age!r})"


# GOOD: dataclass — all boilerplate generated
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class UserDto:
    name: str
    email: str
    age: int
```

### Dataclasses with Validation (`__post_init__`)

```python
# GOOD: Validate and normalize in __post_init__
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class Email:
    value: str

    def __post_init__(self) -> None:
        if "@" not in self.value:
            raise ValueError(f"Invalid email: {self.value}")
        # frozen instances need object.__setattr__ to normalize
        object.__setattr__(self, "value", self.value.lower().strip())
```

### Dataclasses with Factory Methods and Derived Values

```python
# GOOD: Classmethod factories and computed properties
from dataclasses import dataclass
from datetime import date, timedelta

@dataclass(frozen=True, slots=True)
class DateRange:
    start: date
    end: date

    def __post_init__(self) -> None:
        if self.end < self.start:
            raise ValueError("end must be on or after start")

    @classmethod
    def of_days(cls, start: date, days: int) -> "DateRange":
        return cls(start, start + timedelta(days=days))

    @property
    def days(self) -> int:
        return (self.end - self.start).days
```

### Defaults and `field()` for Mutable Fields

```python
# BAD: Mutable default shared across all instances
from dataclasses import dataclass

@dataclass
class SearchCriteria:
    query: str = ""
    filters: list[str] = []  # ALL instances share one list — bug!


# GOOD: field(default_factory=...) for fresh mutable defaults
from dataclasses import dataclass, field
from enum import Enum

class SortDirection(Enum):
    ASC = "asc"
    DESC = "desc"

@dataclass(slots=True)
class SearchCriteria:
    query: str = ""
    page: int = 0
    size: int = 20
    sort_by: str = "id"
    direction: SortDirection = SortDirection.ASC
    filters: list[str] = field(default_factory=list)
```

### Django: Dataclasses as DTOs / Serializer Inputs

```python
# GOOD: Use dataclasses for service-layer inputs, keep them out of the ORM
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class CreateUserRequest:
    name: str
    email: str
    age: int


@dataclass(frozen=True, slots=True)
class UserResponse:
    id: int
    name: str
    email: str

    @classmethod
    def from_model(cls, user: "User") -> "UserResponse":
        return cls(id=user.pk, name=user.name, email=user.email)
```

See `django-rest-api` for DRF serializer patterns and `django-orm` for model design.

---

## NamedTuple vs dataclass

```python
# GOOD: NamedTuple for lightweight, immutable, positional/iterable records
from typing import NamedTuple

class Point(NamedTuple):
    x: float
    y: float

p = Point(1.0, 2.0)
x, y = p          # unpacks like a tuple
assert p.x == 1.0 # and has named access
```

Use `NamedTuple` when you want tuple semantics (iteration, unpacking, hashable, lightweight). Use `@dataclass` when you want methods, mutability options, `slots`, or richer behavior.

---

## Type Parameters (PEP 695)

Define generics with the inline `[T]` syntax — no more `TypeVar` boilerplate.

```python
# BAD (pre-3.12): explicit TypeVar + Generic declarations
from typing import TypeVar, Generic

T = TypeVar("T")

class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []
    def push(self, item: T) -> None:
        self._items.append(item)
    def pop(self) -> T:
        return self._items.pop()

def first(items: list[T]) -> T:
    return items[0]


# GOOD (3.12+): inline type parameters, no TypeVar needed
class Stack[T]:
    def __init__(self) -> None:
        self._items: list[T] = []
    def push(self, item: T) -> None:
        self._items.append(item)
    def pop(self) -> T:
        return self._items.pop()

def first[T](items: list[T]) -> T:
    return items[0]
```

### Type Aliases (PEP 695 `type` statement)

```python
# BAD: implicit alias, no special typing support
from typing import Union
Json = Union[dict, list, str, int, float, bool, None]

# GOOD: explicit `type` statement, supports generics and lazy evaluation
type Json = dict[str, "Json"] | list["Json"] | str | int | float | bool | None
type Vector[T] = list[tuple[T, T]]
```

### Bounded and Constrained Type Parameters

```python
# GOOD: bound expresses "any subtype of"
from typing import Protocol

class Comparable(Protocol):
    def __lt__(self, other: object, /) -> bool: ...

def maximum[T: Comparable](items: list[T]) -> T:
    return max(items)
```

---

## Structural Pattern Matching

```python
# BAD: if/elif chains with manual type checks
def describe(obj: object) -> str:
    if isinstance(obj, str):
        return f"String of length {len(obj)}"
    elif isinstance(obj, int):
        return f"Integer: {obj}"
    elif isinstance(obj, list):
        return f"List of size {len(obj)}"
    else:
        return f"Unknown: {obj}"


# GOOD: match statement — concise, with guards and capture
def describe(obj: object) -> str:
    match obj:
        case None:
            return "null"
        case str() as s if not s:
            return "Empty string"
        case str() as s:
            return f"String of length {len(s)}"
        case int() as i if i < 0:
            return f"Negative integer: {i}"
        case int() as i:
            return f"Integer: {i}"
        case list():
            return f"List of size {len(obj)}"
        case _:
            return f"Unknown: {obj}"
```

### Closed Hierarchies with Pattern Matching

Python has no `sealed` keyword, but a union type plus `assert_never` gives the compiler-checked exhaustiveness equivalent.

```python
# GOOD: union of dataclasses == sealed hierarchy; mypy enforces exhaustiveness
from dataclasses import dataclass
from decimal import Decimal
from datetime import timedelta
from typing import assert_never

@dataclass(frozen=True, slots=True)
class PaymentSuccess:
    transaction_id: str
    amount: Decimal

@dataclass(frozen=True, slots=True)
class PaymentFailure:
    error_code: str
    message: str

@dataclass(frozen=True, slots=True)
class PaymentPending:
    reference_id: str
    timeout: timedelta

type PaymentResult = PaymentSuccess | PaymentFailure | PaymentPending


def calculate_fee(result: PaymentResult) -> Decimal:
    match result:
        case PaymentSuccess(amount=amount):
            return amount * Decimal("0.02")
        case PaymentFailure():
            return Decimal("0")
        case PaymentPending():
            return Decimal("1.00")
        case _:
            assert_never(result)  # mypy errors if a case is missing
```

### Mapping and Sequence Patterns

```python
# GOOD: destructure dicts and sequences directly (replaces nested key checks)
def parse_command(payload: dict[str, object]) -> str:
    match payload:
        case {"action": "move", "x": int(x), "y": int(y)}:
            return f"move to ({x}, {y})"
        case {"action": "quit"}:
            return "quit"
        case {"action": str(action)}:
            return f"unknown action: {action}"
        case _:
            return "malformed payload"


def head_tail[T](items: list[T]) -> tuple[T, list[T]]:
    match items:
        case [first, *rest]:
            return first, rest
        case []:
            raise ValueError("empty list")
```

---

## Modern Type Hints

```python
# BAD: legacy typing imports and Optional
from typing import Optional, List, Dict, Union

def find(ids: List[int]) -> Optional[Dict[str, Union[int, str]]]:
    ...

# GOOD: built-in generics and X | None
def find(ids: list[int]) -> dict[str, int | str] | None:
    ...
```

### `Self`, `@override`, and `@final`

```python
# GOOD: Self for fluent APIs and factories (3.11+)
from typing import Self, final, override

class QueryBuilder:
    def where(self, clause: str) -> Self:
        ...
        return self

    @classmethod
    def empty(cls) -> Self:
        return cls()


class Base:
    def handle(self) -> None: ...

@final
class Handler(Base):
    @override  # mypy errors if Base.handle is renamed/removed
    def handle(self) -> None:
        ...
```

---

## Walrus Operator (`:=`)

Assign and test in a single expression — avoids recomputation and tightens scope.

```python
# BAD: compute twice or leak a variable
data = expensive_query()
if data:
    process(data)

results = []
chunk = stream.read()
while chunk:
    results.append(chunk)
    chunk = stream.read()


# GOOD: assign inside the condition
if data := expensive_query():
    process(data)

results = []
while chunk := stream.read():
    results.append(chunk)

# GOOD: avoid recomputing in a comprehension
filtered = [y for x in xs if (y := transform(x)) is not None]
```

---

## f-strings

Formatting, interpolation, and debugging output.

```python
# BAD: % formatting and str.format
name, age = "Ada", 36
msg = "User %s is %d" % (name, age)
msg = "User {} is {}".format(name, age)

# GOOD: f-strings
msg = f"User {name} is {age}"

# GOOD: format specs and alignment
price = 1234.5
print(f"{price:>10,.2f}")   # "  1,234.50"

# GOOD: self-documenting debug expressions (= suffix)
print(f"{name=} {age=}")    # name='Ada' age=36
```

For multi-line SQL/JSON/templates, use triple-quoted strings (the Python text-block equivalent):

```python
# GOOD: triple-quoted string for SQL
sql = """
    SELECT u.id, u.name, u.email
    FROM users u
    JOIN orders o ON u.id = o.user_id
    WHERE o.status = %s
    ORDER BY u.name
"""
```

---

## pathlib over os.path

```python
# BAD: string-based path manipulation
import os
config_dir = os.path.join(os.path.expanduser("~"), ".config", "app")
os.makedirs(config_dir, exist_ok=True)
config_file = os.path.join(config_dir, "settings.toml")
if os.path.exists(config_file):
    with open(config_file) as f:
        text = f.read()


# GOOD: pathlib — object-oriented, OS-agnostic
from pathlib import Path
config_dir = Path.home() / ".config" / "app"
config_dir.mkdir(parents=True, exist_ok=True)
config_file = config_dir / "settings.toml"
if config_file.exists():
    text = config_file.read_text(encoding="utf-8")

# GOOD: globbing, suffixes, and iteration
for py_file in Path("src").rglob("*.py"):
    print(py_file.stem, py_file.suffix)
```

---

## Comprehensions and Generators over Loops

```python
# BAD: manual accumulation loop
result = []
for user in users:
    if user.active:
        result.append(user.email)

# GOOD: comprehension
result = [user.email for user in users if user.active]

# GOOD: generator expression for lazy, memory-efficient pipelines
total = sum(line_total(item) for item in cart.items)

# GOOD: dict / set comprehensions
by_id = {user.id: user for user in users}
unique_domains = {email.split("@")[1] for email in emails}
```

Keep comprehensions shallow and readable — see `readable-comprehensions` for when to switch back to a loop.

---

## Context Managers over Manual Cleanup

```python
# BAD: manual try/finally for resource cleanup
f = open("data.txt")
try:
    data = f.read()
finally:
    f.close()


# GOOD: with statement — guaranteed cleanup
with open("data.txt", encoding="utf-8") as f:
    data = f.read()

# GOOD: parenthesized multiple context managers (3.10+)
with (
    open("in.txt", encoding="utf-8") as src,
    open("out.txt", "w", encoding="utf-8") as dst,
):
    dst.write(src.read())

# GOOD: write your own with contextlib
from contextlib import contextmanager
from collections.abc import Iterator

@contextmanager
def timer(label: str) -> Iterator[None]:
    import time
    start = time.perf_counter()
    try:
        yield
    finally:
        print(f"{label}: {time.perf_counter() - start:.3f}s")
```

See `exception-handling` for `ExceptionGroup`/`except*` and resource-cleanup patterns.

---

## Enums

```python
# BAD: magic string/int constants
STATUS_ACTIVE = "active"
STATUS_INACTIVE = "inactive"

# GOOD: Enum / StrEnum (3.11+) for named, type-safe constants
from enum import StrEnum, auto

class Status(StrEnum):
    ACTIVE = auto()      # "active"
    INACTIVE = auto()    # "inactive"
    PENDING = auto()     # "pending"

# Works directly as a string AND is type-checked
label = {
    Status.ACTIVE: "Active",
    Status.INACTIVE: "Inactive",
    Status.PENDING: "Pending Review",
}[status]
```

---

## Anti-Patterns

| Old Pattern | Modern Alternative |
|------------|-------------------|
| Hand-written `__init__`/`__eq__`/`__repr__` | `@dataclass` (or `NamedTuple`) |
| `isinstance` + `if/elif` chains | `match` statement |
| Open class hierarchy, no exhaustiveness | Union type + `assert_never` |
| `typing.Optional[X]` | `X | None` |
| `typing.List`, `typing.Dict`, `typing.Union` | `list`, `dict`, `X | Y` |
| `TypeVar("T")` + `Generic[T]` | PEP 695 `def f[T]` / `class C[T]` |
| Implicit `Alias = Union[...]` | `type Alias = ...` statement |
| `os.path.join` / `os.path.exists` | `pathlib.Path` |
| `%`-formatting / `str.format` | f-strings |
| Manual `try/finally` cleanup | `with` (context managers) |
| Mutable default arg / dataclass field | `field(default_factory=...)` |
| Magic string/int constants | `Enum` / `StrEnum` |
| `datetime.utcnow()` | `datetime.now(UTC)` |
| `dict.get(k)` then recompute | walrus `if (v := dict.get(k))` |

---

## Related Skills

- `python-concurrency` — `asyncio`, `TaskGroup`, threads vs processes, the GIL and free-threaded builds
- `refactoring-python` — Techniques for modernizing legacy code to use these idioms
- `exception-handling` — `ExceptionGroup`/`except*`, custom exception hierarchies, pattern matching in handlers
- `django-rest-api` — Dataclasses as DTOs, serializer and view patterns
- `efficient-python` — Comprehensions, generators, and data-structure selection
- `readable-comprehensions` — When comprehensions help readability and when to use a loop
