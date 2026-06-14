---
name: efficient-python
description: Efficient Python coding patterns — right data structures, eliminating nested loops, comprehensions and generators, guard clauses, and immutable containers. Use when reviewing code for performance, simplifying complex logic, or replacing imperative patterns.
---

# Efficient Python

No nested loops with embedded conditionals. Choose the right data structure. Let the standard library (`collections`, `itertools`, comprehensions) do the work.

## When to Use

- Reviewing code for O(n²) patterns that should be O(n)
- Replacing imperative loops with comprehensions or generators
- Choosing the right container type for the job
- Simplifying deeply nested conditionals
- Building or reviewing data-processing pipelines

---

## Quick Reference

| Need | Data Structure | Why |
|------|---------------|-----|
| Fast lookup by key | `dict` | O(1) vs O(n) list scan |
| Uniqueness | `set` | O(1) `in` vs O(n) on `list` |
| Sorted access | `sortedcontainers.SortedDict` | O(log n) sorted operations |
| Insertion order | `dict` | Ordered since 3.7 (built in) |
| FIFO queue | `collections.deque` | O(1) `popleft()` vs O(n) on `list` |
| Frequency counting | `collections.Counter` | One-pass tallies, `most_common()` |
| Group-to-list default | `collections.defaultdict(list)` | No `setdefault` boilerplate |
| Immutable snapshot | `tuple` / `frozenset` / `MappingProxyType` | No accidental mutation |

---

## Kill the Nested Loop

The most common performance mistake in Python: scanning a list inside a loop.

```python
# BAD: O(n²) — nested loop with linear scan
def enrich_orders(orders: list[Order], customers: list[Customer]) -> list[OrderDto]:
    result: list[OrderDto] = []
    for order in orders:
        for customer in customers:
            if customer.id == order.customer_id:
                result.append(OrderDto(order, customer.name))
                break
    return result

# GOOD: O(n) — pre-index into a dict, then single-pass lookup
def enrich_orders(orders: list[Order], customers: list[Customer]) -> list[OrderDto]:
    customer_names = {c.id: c.name for c in customers}
    return [OrderDto(order, customer_names[order.customer_id]) for order in orders]
```

### Grouping to Eliminate Nested Loops

```python
from collections import defaultdict

# BAD: O(n²) — find all orders per customer
orders_by_customer: dict[int, list[Order]] = {}
for customer in customers:
    cust_orders: list[Order] = []
    for order in orders:
        if order.customer_id == customer.id:
            cust_orders.append(order)
    orders_by_customer[customer.id] = cust_orders

# GOOD: O(n) — one pass with defaultdict
orders_by_customer: dict[int, list[Order]] = defaultdict(list)
for order in orders:
    orders_by_customer[order.customer_id].append(order)
```

### Set-Based Filtering

```python
# BAD: O(n²) — membership test against a list inside a loop
active_users: list[User] = []
for user in all_users:
    if user.id in active_user_ids:  # active_user_ids is a list!
        active_users.append(user)

# GOOD: O(n) — convert to a set first
active_id_set = set(active_user_ids)
active_users = [user for user in all_users if user.id in active_id_set]
```

---

## Replace Loops with Comprehensions and Generators

```python
# BAD: Manual accumulation with a loop
active_emails: list[str] = []
for user in users:
    if user.active:
        active_emails.append(user.email.lower())
active_emails.sort()

# GOOD: Comprehension — filter and transform, then sort
active_emails = sorted(user.email.lower() for user in users if user.active)
```

### Aggregation

```python
# BAD: Manual counting
count = 0
for order in orders:
    if order.status is Status.COMPLETED:
        count += 1

# GOOD: sum over a generator of booleans
count = sum(order.status is Status.COMPLETED for order in orders)

from decimal import Decimal

# BAD: Manual sum
total = Decimal("0")
for order in orders:
    total += order.amount

# GOOD: sum with a generator (start value keeps the Decimal type)
total = sum((order.amount for order in orders), Decimal("0"))
```

### Flattening Nested Sequences

```python
from itertools import chain

# BAD: Nested loop to flatten
all_items: list[LineItem] = []
for order in orders:
    for item in order.items:
        all_items.append(item)

# GOOD: chain.from_iterable — flatten in one expression
all_items = list(chain.from_iterable(order.items for order in orders))

# GOOD: nested comprehension reads left-to-right like the loop
all_items = [item for order in orders for item in order.items]
```

---

## Prefer Generators for Large or Streamed Data

Comprehensions build the whole list in memory; generators yield lazily.

```python
# BAD: Materializes every line, then sums — high memory on big files
def total_revenue(path: str) -> Decimal:
    rows = [parse(line) for line in open(path)]
    return sum((r.amount for r in rows), Decimal("0"))

# GOOD: Stream line-by-line with a generator and a context manager
def total_revenue(path: str) -> Decimal:
    with open(path) as f:
        return sum((parse(line).amount for line in f), Decimal("0"))
```

---

## Replace if/elif Chains with Dicts or Pattern Matching

```python
from decimal import Decimal

# BAD: Growing if/elif chain
def calculate_shipping(region: str) -> Decimal:
    if region == "US":
        return Decimal("5.99")
    elif region == "EU":
        return Decimal("9.99")
    elif region == "ASIA":
        return Decimal("14.99")
    elif region == "AU":
        return Decimal("19.99")
    else:
        return Decimal("24.99")

# GOOD: Dict-based dispatch
SHIPPING_RATES: dict[str, Decimal] = {
    "US": Decimal("5.99"),
    "EU": Decimal("9.99"),
    "ASIA": Decimal("14.99"),
    "AU": Decimal("19.99"),
}
DEFAULT_RATE = Decimal("24.99")

def calculate_shipping(region: str) -> Decimal:
    return SHIPPING_RATES.get(region, DEFAULT_RATE)
```

### Structural Pattern Matching (for complex dispatch)

```python
# GOOD: When logic varies per case (see modern-python)
def calculate_discount(customer: Customer) -> Decimal:
    match customer:
        case PremiumCustomer(years_active=y, order_total=t) if y > 5:
            return t * Decimal("0.20")
        case PremiumCustomer(order_total=t):
            return t * Decimal("0.15")
        case RegularCustomer(order_count=c, order_total=t) if c > 10:
            return t * Decimal("0.05")
        case RegularCustomer():
            return Decimal("0")
        case _:
            raise ValueError(f"Unknown customer type: {customer!r}")
```

---

## Avoid Unbounded Loops

```python
import time
from itertools import count

# BAD: while True with manual break for pagination
results: list[User] = []
page = 0
while True:
    batch = repo.fetch_page(page, size=100)
    process_batch(batch)
    if len(batch) < 100:
        break
    page += 1

# GOOD: Bounded iteration — iter() with a sentinel stops cleanly
for page in count():
    batch = repo.fetch_page(page, size=100)
    process_batch(batch)
    if len(batch) < 100:
        break  # still explicit, but the loop variable is meaningful

# BEST when an iterator exists: let the producer signal exhaustion
def pages(repo: Repo) -> Iterator[list[User]]:
    for page in count():
        batch = repo.fetch_page(page, size=100)
        if not batch:
            return
        yield batch

for batch in pages(repo):
    process_batch(batch)

# BAD: Polling with while True
while True:
    status = check_status(job_id)
    if status is Status.DONE:
        break
    time.sleep(1)

# GOOD: Bounded polling — raises if it never completes
for _ in range(max_attempts):
    if check_status(job_id) is Status.DONE:
        break
    time.sleep(1)
else:
    raise TimeoutError("Job did not complete")
```

---

## Index-Free Iteration

```python
# BAD: range(len(...)) when the index is unused
for i in range(len(users)):
    send_email(users[i])

# GOOD: iterate the objects directly
for user in users:
    send_email(user)

# GOOD: map() / comprehension when collecting results
sent = [send_email(user) for user in users]

# ONLY use an index when you genuinely need position — use enumerate
for position, item in enumerate(items, start=1):
    item.display_order = position  # position is meaningful here
```

---

## Efficient String Building

```python
# BAD: O(n²) string concatenation in a loop
result = ""
for item in items:
    result += item + ", "

# GOOD: str.join over an iterable
result = ", ".join(items)

# GOOD: join with a transformation
result = "[" + ", ".join(user.name for user in users) + "]"
# [Alice, Bob, Charlie]

# GOOD: join a generator when building is conditional
result = "\n".join(
    record.format() for record in records if record.is_valid()
)
```

---

## Standard-Library Power Tools

Replace nested loops and manual aggregation with `collections` and `itertools`.

```python
from collections import Counter, defaultdict
from itertools import groupby, pairwise, batched

# index by key — dict comprehension
users_by_id = {user.id: user for user in users}

# group into lists — defaultdict
by_dept: dict[Dept, list[Employee]] = defaultdict(list)
for emp in employees:
    by_dept[emp.department].append(emp)

# frequency map — Counter
status_counts = Counter(order.status for order in orders)
top_three = status_counts.most_common(3)

# statistics per group — statistics module + grouped values
from statistics import mean, pstdev
salaries_by_dept: dict[Dept, list[float]] = defaultdict(list)
for emp in employees:
    salaries_by_dept[emp.department].append(emp.salary)
avg_by_dept = {dept: mean(vals) for dept, vals in salaries_by_dept.items()}

# partition into true/false groups — one pass
paid: list[Order] = []
unpaid: list[Order] = []
for order in orders:
    (paid if order.is_paid else unpaid).append(order)

# min and max in one pass — avoid two scans
lo, hi = min(numbers), max(numbers)  # fine for in-memory lists
# for a single-pass generator, fold manually:
it = iter(numbers)
lo = hi = next(it)
for n in it:
    lo, hi = min(lo, n), max(hi, n)

# itertools.groupby — requires the input to be sorted by the key first
records.sort(key=lambda r: r.department)
skills_by_dept = {
    dept: {skill for emp in group for skill in emp.skills}
    for dept, group in groupby(records, key=lambda r: r.department)
}

# pairwise — sliding windows without index arithmetic
deltas = [b - a for a, b in pairwise(timestamps)]

# batched (3.12+) — chunk an iterable for bulk operations
for chunk in batched(ids, 500):
    bulk_delete(chunk)
```

---

## Early Returns and Guard Clauses

```python
from decimal import Decimal

# BAD: Deeply nested conditionals
def process_order(order: Order | None) -> OrderResult:
    if order is not None:
        if order.items:
            if order.customer is not None:
                if order.customer.active:
                    # Actual business logic buried 4 levels deep
                    total = calculate_total(order)
                    apply_discounts(order, total)
                    return OrderResult(total, Status.SUCCESS)
                else:
                    return OrderResult(Decimal("0"), Status.INACTIVE_CUSTOMER)
            else:
                return OrderResult(Decimal("0"), Status.NO_CUSTOMER)
        else:
            return OrderResult(Decimal("0"), Status.EMPTY_ORDER)
    else:
        return OrderResult(Decimal("0"), Status.INVALID)

# GOOD: Guard clauses — flat control flow, business logic at the end
def process_order(order: Order | None) -> OrderResult:
    if order is None:
        return OrderResult(Decimal("0"), Status.INVALID)
    if not order.items:
        return OrderResult(Decimal("0"), Status.EMPTY_ORDER)
    if order.customer is None:
        return OrderResult(Decimal("0"), Status.NO_CUSTOMER)
    if not order.customer.active:
        return OrderResult(Decimal("0"), Status.INACTIVE_CUSTOMER)

    # Business logic — clear and unnested
    total = calculate_total(order)
    apply_discounts(order, total)
    return OrderResult(total, Status.SUCCESS)
```

See `flatten-nesting` for the full guard-clause playbook.

---

## Immutable Containers

Prevent accidental mutation that leads to defensive copies and subtle bugs.

```python
from dataclasses import dataclass, field
from types import MappingProxyType

# BAD: Mutable list exposed as an attribute
class Team:
    def __init__(self, members: list[Member]) -> None:
        self.members = members  # Caller keeps a live reference!

# GOOD: Defensive copy stored as a tuple
@dataclass(frozen=True, slots=True)
class Team:
    members: tuple[Member, ...]

    @classmethod
    def of(cls, members: Iterable[Member]) -> "Team":
        return cls(tuple(members))  # Immutable snapshot

# GOOD: Module-level constants as immutable literals
ROLES: frozenset[str] = frozenset({"ADMIN", "USER"})
LIMITS: Mapping[str, int] = MappingProxyType({"basic": 10, "premium": 100})

# GOOD: NamedTuple for lightweight immutable value objects
from typing import NamedTuple

class Point(NamedTuple):
    x: float
    y: float
```

Prefer `@dataclass(frozen=True, slots=True)` or `NamedTuple` over hand-written
classes for value objects — see `modern-python`.

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| `x in some_list` inside a loop | O(n²) | Convert to `set` first |
| `range(len(seq))` when index unused | Noise, off-by-one risk | Iterate directly or `enumerate` |
| Nested loop when a dict lookup suffices | O(n²) → O(n) | Pre-index with a dict comprehension |
| `while True` for bounded work | Risk of infinite loop | `for ... range`, `count()`, or `for/else` |
| `s += "..."` in a loop | O(n²) string copies | `"".join(...)` |
| Deeply nested `if/else` | Hard to read, error-prone | Guard clauses, early returns |
| Returning mutable internal lists | Caller can corrupt state | Store/return `tuple` or copy |
| Manual aggregation with temp variables | Verbose, error-prone | `Counter`, `defaultdict`, `statistics` |
| `sorted()` then a separate filter loop | Extra pass | Filter inside the generator passed to `sorted()` |
| Building a list to then iterate once | Wasted memory | Pass a generator expression instead |
| Long `if/elif` ladder on one value | Hard to extend | Dict dispatch or `match` |

---

## Tooling

- **uv** — manage dependencies and virtualenvs (`uv add`, `uv run`)
- **ruff** — lint and format; catches `C4` (comprehension) and `PERF` rules that flag many patterns above
- **mypy** — enforce the type hints used throughout these examples
- **pytest** — verify behavior before and after refactoring for performance

---

## Related Skills

- `modern-python` — Dataclasses, pattern matching, type hints, walrus operator
- `readable-comprehensions` — When a comprehension helps vs. when a loop is clearer
- `refactoring-python` — Techniques for restructuring code to use these patterns
- `flatten-nesting` — Guard clauses and early returns in depth
- `review-changes-python` — Flag inefficient patterns during code review
- `leverage-libraries` — When a library does it better than hand-rolled code
