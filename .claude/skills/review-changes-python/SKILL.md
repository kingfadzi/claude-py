---
name: review-changes-python
description: Systematic Python/Django code review workflow — covers correctness, modern Python, performance, security, REST API design, error handling, logging, concurrency, code size, and naming/readability. Use when reviewing PRs, commits, or diffs.
---

# Review Changes (Python/Django)

Systematic workflow for reviewing Python/Django code changes.

## When to Use

- Reviewing pull requests or commits
- Evaluating diffs for correctness and best practices
- Pre-merge quality gate checks

---

## Quick Reference: Severity Levels

| Severity | Meaning | Blocking? |
|----------|---------|-----------|
| **CRITICAL** | Security vulnerability, data loss risk, correctness bug | Yes |
| **MAJOR** | Significant design flaw, maintainability concern, missing test | Yes |
| **MINOR** | Style, naming, minor improvement opportunity | No |
| **PRAISE** | Highlight good practice worth recognizing | No |

---

## Step 0: Read the Full Diff First

Before commenting on anything:

1. Read the **entire** diff — understand the full scope of the change
2. Identify the **purpose** — what problem does this solve?
3. Check the **scope of impact** — what could break? (migrations, signals, settings)
4. Note files added, modified, deleted — is the scope appropriate?
5. Only then begin category-by-category review

---

## 1. Correctness and Robustness

### Resource Management

```python
# BAD: Resource leak — file never closed if an exception is thrown
f = open(path, encoding="utf-8")
content = f.read()

# GOOD: context manager closes the file deterministically
with open(path, encoding="utf-8") as f:
    content = f.read()
```

### None Handling

```python
# BAD: defensive ladder that buries intent
user = find_user(user_id)
if user is not None:
    return user.name
return "unknown"

# GOOD: explicit default, single expression
user = find_user(user_id)
return user.name if user else "unknown"
```

### Mutable Default Arguments

```python
# BAD: shared mutable default — accumulates across calls
def add_tag(tag: str, tags: list[str] = []) -> list[str]:
    tags.append(tag)
    return tags

# GOOD: sentinel default
def add_tag(tag: str, tags: list[str] | None = None) -> list[str]:
    tags = list(tags) if tags is not None else []
    tags.append(tag)
    return tags
```

### Contracts

- If `__eq__` is overridden, is `__hash__` defined (or set to `None` for mutable types)?
- Are public functions and methods fully type-hinted (params and return)?
- Are mutable internals leaked? Return a copy or expose a read-only view.

---

## 2. Modern Python

Flag pre-3.12 idioms when modern alternatives exist:

| Old Pattern | Flag | Modern Alternative |
|------------|------|-------------------|
| `Optional[X]` / `Union[X, Y]` | MINOR | `X \| None` / `X \| Y` |
| `List[str]`, `Dict[str, int]` (typing) | MINOR | `list[str]`, `dict[str, int]` builtins |
| `if/elif isinstance` chain | MAJOR | `match`/`case` structural pattern matching |
| Boilerplate `__init__`/`__eq__`/`__repr__` | MINOR | `@dataclass` or `NamedTuple` |
| `os.path.join` / string paths | MINOR | `pathlib.Path` |
| `%`-format / `.format()` in code | MINOR | f-strings |
| Manual index loop `range(len(x))` | MINOR | `enumerate(x)` |
| `dict.keys()` membership check | MINOR | `key in dict` |
| C-style accumulation loop | MINOR | comprehension / generator |

See `modern-python` for full reference. See `readable-comprehensions` for comprehension limits.

---

## 3. Code Size

| Metric | Threshold | Action |
|--------|-----------|--------|
| Function LOC | > 20 lines | Flag for Extract Function |
| Module LOC | > 400 lines | Flag for SRP violation / split module |
| Function parameters | > 3 | Flag for parameter object (`dataclass`) |
| Nesting depth | > 2 levels | Flag for guard clauses / extraction |
| Cyclomatic complexity | > 5 branches | Flag for simplification (ruff `C901`) |

```python
# BAD: too many positional parameters
def create_order(
    customer_id, product_id, quantity, price,
    currency, shipping_address, billing_address,
): ...

# GOOD: parameter object
from dataclasses import dataclass
from decimal import Decimal

@dataclass(frozen=True, slots=True)
class CreateOrderRequest:
    customer_id: str
    product_id: str
    quantity: int
    price: Decimal
    currency: str
    shipping_address: str
    billing_address: str

def create_order(request: CreateOrderRequest) -> Order: ...
```

See `flatten-nesting` for guard-clause patterns and `refactoring-python` for extraction.

---

## 4. Django Patterns

### Dependencies and Services

```python
# BAD: fat view doing ORM, validation, and business logic inline
class OrderView(APIView):
    def post(self, request):
        if int(request.data["quantity"]) <= 0:
            return Response({"error": "bad quantity"}, status=400)
        total = Decimal(request.data["price"]) * int(request.data["quantity"])
        order = Order.objects.create(customer_id=request.data["customer_id"], total=total)
        return Response({"id": order.id})

# GOOD: thin view; validation in serializer, logic in service
class OrderView(APIView):
    def post(self, request):
        serializer = OrderRequestSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        order = order_service.create_order(serializer.validated_data)
        return Response(OrderSerializer(order).data, status=201)
```

### ORM Discipline

```python
# BAD: query in a loop -> N+1
for order in Order.objects.all():
    print(order.customer.name)  # one extra query per row

# GOOD: select_related pulls the FK in one join
for order in Order.objects.select_related("customer"):
    print(order.customer.name)
```

- Using `select_related` / `prefetch_related` for related access?
- `update_fields=` on partial `.save()` to avoid clobbering columns?
- Heavy work wrapped in `transaction.atomic()`?

See `django-orm` for query optimization patterns.

---

## 5. Performance

| Issue | Severity | What to Look For |
|-------|----------|-----------------|
| N+1 queries | CRITICAL | Related access in a loop; missing `select_related`/`prefetch_related` |
| Missing pagination | MAJOR | `Model.objects.all()` returned without limit/`Paginator` |
| Loading full rows | MINOR | Use `.only()` / `.values()` / `.exists()` instead of fetching objects |
| O(n²) collection ops | MAJOR | `x in some_list` inside a loop — use a `set` |
| Repeated expensive lookup | MINOR | Cache static/slow-changing data (`functools.cache`, Django cache) |
| `len(queryset)` to test emptiness | MINOR | `.exists()` / `.count()` instead of materializing |

See `efficient-python` for data structure and loop optimization patterns.

---

## 6. API Design

| Check | Severity |
|-------|----------|
| Wrong HTTP method (POST for read, GET for mutation) | MAJOR |
| Wrong status code (200 for creation instead of 201) | MINOR |
| Model returned directly (no serializer / DTO separation) | MAJOR |
| Missing serializer validation on input | MAJOR |
| No pagination on list endpoints | MAJOR |
| Inconsistent URL naming (mixed singular/plural, verbs in paths) | MINOR |

See `django-rest-api` for full API design reference.

---

## 7. Error Handling

| Issue | Severity |
|-------|----------|
| Bare `except:` or broad `except Exception` swallowing errors | CRITICAL |
| Swallowed exception (`except ...: pass`) | CRITICAL |
| Returning internal details to client (tracebacks, SQL, settings) | CRITICAL |
| Ad-hoc error handling per view (no custom exception handler) | MAJOR |
| Catching an exception only to re-raise it unchanged | MINOR |

```python
# BAD: swallowed exception — order looks successful but payment failed
try:
    process_payment(order)
except Exception:
    pass

# GOOD: log with context and chain into a domain exception
try:
    process_payment(order)
except PaymentError as exc:
    logger.error("Payment failed for order %s", order.id, exc_info=exc)
    raise OrderProcessingError("Payment failed") from exc
```

See `exception-handling` for centralized error handling patterns.

---

## 8. Logging

| Issue | Severity |
|-------|----------|
| `print()` in production code | MAJOR |
| Sensitive data in logs (passwords, tokens, PII) | CRITICAL |
| f-string / `%`-concatenation built before level check | MINOR |
| Missing correlation/request ID context | MINOR |
| Wrong log level (ERROR for non-errors, DEBUG for important events) | MINOR |

```python
# BAD: f-string is always evaluated, even when DEBUG is disabled
logger.debug(f"Processing user {user.name} with id {user.id}")

# GOOD: %-style args — formatted lazily only if the record is emitted
logger.debug("Processing user %s with id %s", user.name, user.id)
```

See `logging-observability` for structured logging and context patterns.

---

## 9. Concurrency

| Issue | Severity |
|-------|----------|
| Module-level mutable state shared across requests/threads | CRITICAL |
| Multiple dependent DB writes without `transaction.atomic()` | MAJOR |
| Read-modify-write race (no `select_for_update()` / `F()` expression) | MAJOR |
| Blocking I/O inside an `async def` view/coroutine | MAJOR |
| Mixing sync ORM calls in async context without `sync_to_async` | CRITICAL |

```python
# BAD: read-modify-write race — two requests can lose an increment
account = Account.objects.get(pk=pk)
account.balance += amount
account.save()

# GOOD: atomic update pushed into the database
Account.objects.filter(pk=pk).update(balance=F("balance") + amount)
```

See `python-concurrency` for thread/async safety patterns.

---

## 10. Reinventing the Wheel

Flag hand-rolled logic when a battle-tested library exists:

| Hand-Rolled | Use Instead |
|------------|-------------|
| Manual request validation | DRF serializers / `pydantic` |
| Hand-written retry loop | `tenacity` |
| Custom date parsing | `datetime` / `dateutil` / `django.utils.timezone` |
| Manual JSON building | `json` / DRF renderers |
| Custom HTTP client wrapper | `httpx` / `requests` |
| Hand-rolled env parsing | `django-environ` / `pydantic-settings` |

See `leverage-libraries` for the full library reference.

---

## 11. Hardcoded Values

Any value that could change between environments must be externalized:

```python
# BAD: hardcoded values scattered in code
API_URL = "http://localhost:8080/api"
MAX_RETRIES = 3
PAGE_SIZE = 20
time.sleep(5)

# GOOD: typed settings sourced from the environment
from pydantic_settings import BaseSettings

class ExternalApiSettings(BaseSettings):
    api_url: str
    max_retries: int = 3
    page_size: int = 20
    retry_delay_seconds: float = 5.0

    model_config = {"env_prefix": "APP_EXTERNAL_"}
```

Flag: URLs, ports, timeouts, retry counts, pool sizes, page sizes, cron schedules, credentials, magic numbers. Secrets belong in the environment, never in `settings.py` or source.

See `django-configuration` for configuration patterns and `django-security` for secret handling.

---

## 12. Testing

| Check | Severity |
|-------|----------|
| No tests for new functionality | MAJOR |
| Test asserts implementation detail, not behavior | MINOR |
| Heavyweight fixture where a unit test suffices (full DB for pure logic) | MINOR |
| No test exercising the query/migration against the DB | MAJOR |
| Missing edge cases (None, empty, boundary values) | MAJOR |
| `time.sleep`/real network in tests instead of fakes/`freezegun`/`responses` | MINOR |

See `testing-python` for pytest patterns and fixtures.

---

## 13. Naming & Readability

Code must be self-documenting — a junior developer should be able to follow the logic without decoding it. Identifiers reveal intent; logic flow stays linear; cleverness loses to clarity. Comprehensions and generators are fine, but only as far as the next reader can still trace what the code does.

| Issue | Severity |
|-------|----------|
| Single-letter variable outside tight scope (`p`, `c`, `o`, `x`) | MAJOR |
| Cryptic abbreviation (`usr`, `prc`, `addr`, `cust`) | MAJOR |
| Type-only name (`s = ""`, `lst = []`, `d = {}`) | MAJOR |
| Hungarian/type-prefixed name (`str_name`, `i_count`, `b_active`) | MINOR |
| Misleading name (variable named `count` holding a sum) | CRITICAL |
| Boolean without `is`/`has`/`should` prefix (`active` vs `is_active`) | MINOR |

```python
# BAD: cryptic
for p in ps:
    if p.s > t:
        r.append(p.v)

# GOOD: intent-revealing
for payment in payments:
    if payment.status > threshold:
        results.append(payment.value)
```

Acceptable single-letter names: loop indices in tight loops (`i`, `j`), throwaway lambda params over short data (`key=lambda p: p.value`) — but prefer meaningful names once scope grows beyond a few lines.

### Readability of logic

Once names are clear, the logic itself must stay readable. The test is whether a junior developer can read top-to-bottom and follow what happens — without running it, without a debugger, without asking the author.

| Issue | Severity |
|-------|----------|
| Comprehension with 2+ `for` clauses plus a condition, no intermediate name | MAJOR |
| Multi-expression logic crammed into a lambda (extract to a function) | MAJOR |
| Nested ternary (`a if b else c if d else e`) | MAJOR |
| Dense one-liner that hides a non-trivial decision | MAJOR |
| Method/queryset chain spanning many transformations with no named step | MINOR |
| Boolean expression with 3+ operators and no extraction (`a and (b or c) and not d`) | MINOR |
| Lambda/comprehension var named `x`/`p` when the body is more than one accessor | MINOR |
| "Clever" trick that needs a comment to explain *what* it does | MAJOR |

```python
# BAD: cryptic — reader must decode the comprehension and the predicate
return [
    R(o.id, sum(li.q for li in o.l))
    for o in os
    if o.s == S.A and o.t > date.today() - timedelta(days=30) and any(li.q > 0 for li in o.l)
]

# GOOD: intent-revealing names, extracted predicate, named step
def is_recent_active_order(order: Order) -> bool:
    thirty_days_ago = date.today() - timedelta(days=30)
    return (
        order.status == Status.ACTIVE
        and order.placed_at > thirty_days_ago
        and any(item.quantity > 0 for item in order.line_items)
    )

recent_active = [
    OrderSummary.from_order(order)
    for order in orders
    if is_recent_active_order(order)
]
```

Rule of thumb: if explaining the line requires a comment about *what it does* (not *why*), the line is too cryptic — extract a function whose name is the explanation. Comments justify decisions; function names document logic.

For comment style itself — when a comment earns its place, length limits, no-changelog/no-banner rules, docstrings on public API — use the **concise-comments** skill.

---

## Tooling Gate

Before approving, confirm the change passes the standard toolchain:

- `uv sync` — dependencies resolve and lockfile is current
- `ruff check` / `ruff format --check` — lint and format clean
- `mypy` — type checks pass with no new ignores
- `pytest` — tests pass, including new coverage for the change

---

## Feedback Format

### CRITICAL (blocking)
> Security vulnerabilities, data loss, correctness bugs, swallowed exceptions, shared mutable state, ORM races

### MAJOR (blocking)
> Design flaws, missing tests, fat views, N+1 queries, missing pagination, isinstance chains over `match`

### MINOR (non-blocking)
> Style improvements, naming, modern Python sugar, documentation

### PRAISE
> Highlight good practices — proper dataclasses, good test coverage, clean error chaining, lean querysets

---

## Related Skills

- `modern-python` — Modern Python idioms reference
- `efficient-python` — Data structure and loop optimization
- `exception-handling` — Centralized error handling patterns
- `django-rest-api` — API design standards
- `testing-python` — Testing patterns and fixtures
- `logging-observability` — Structured logging and context
- `django-orm` — Database access and query patterns
- `django-configuration` — Configuration externalization
- `django-security` — Secrets and security hardening
- `leverage-libraries` — Avoid reinventing the wheel
- `readable-comprehensions` — Keeping comprehensions legible
- `flatten-nesting` — Guard clauses and reduced nesting
