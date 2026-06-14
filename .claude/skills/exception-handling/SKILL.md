---
name: exception-handling
description: Exception handling patterns for modern Python — custom exception hierarchies, exception groups, structural pattern matching, FastAPI/Django error responses (RFC 7807 problem+json), validation errors, context managers, and logging strategy. Use when designing error handling, reviewing except blocks, or implementing API error responses.
---

# Exception Handling

Centralized, consistent error handling for modern Python (3.12/3.13) applications.

## When to Use

- Designing an error handling strategy for a new service
- Implementing a centralized exception handler for API error responses
- Reviewing `except` blocks and exception propagation
- Creating custom exception hierarchies

---

## Quick Reference

| Concern | Approach |
|---------|----------|
| Domain exceptions | Custom hierarchy rooted at one base `Exception` |
| API error responses | Single exception handler returning `application/problem+json` |
| Validation errors | Pydantic / framework validators + structured handler |
| Recoverable vs programmer error | Catch specific exceptions; let bugs propagate |
| Cleanup | Context managers (`with`), not `try/finally` boilerplate |
| Concurrent failures | `ExceptionGroup` + `except*` |
| Logging | `ERROR` for unexpected, `WARNING` for business rule violations |
| Correlation | `contextvars` with request/trace IDs |

---

## Custom Exception Hierarchy

Root every domain error at one base class so callers can catch broadly or narrowly. Carry structured data on the instance, not in the message string.

```python
# BAD: Generic exceptions with no type safety
raise Exception("Order not found")
raise ValueError("Insufficient stock")
# Caller can't distinguish error types without parsing strings,
# and a bare `except Exception` here also swallows real bugs.
```

```python
# GOOD: A typed hierarchy with structured attributes
from dataclasses import dataclass


class OrderError(Exception):
    """Base class for all order-domain errors."""

    error_code: str = "ORDER_ERROR"


@dataclass
class OrderNotFoundError(OrderError):
    order_id: int
    error_code: str = "ORDER_NOT_FOUND"

    def __str__(self) -> str:
        return f"Order not found: {self.order_id}"


@dataclass
class InsufficientStockError(OrderError):
    product_id: str
    requested: int
    available: int
    error_code: str = "INSUFFICIENT_STOCK"

    def __str__(self) -> str:
        return (
            f"Insufficient stock for {self.product_id}: "
            f"requested {self.requested}, available {self.available}"
        )


@dataclass
class OrderAlreadyExistsError(OrderError):
    reference_id: str
    error_code: str = "ORDER_DUPLICATE"


@dataclass
class PaymentFailedError(OrderError):
    reason: str
    error_code: str = "PAYMENT_FAILED"
```

### Mapping with Structural Pattern Matching

Python has no compiler exhaustiveness check, but `match` on the type keeps the
mapping in one readable place. Add a `case _` so a new subclass degrades safely.

```python
# GOOD: One mapping function, easy to extend
import http


def map_to_status(exc: OrderError) -> http.HTTPStatus:
    match exc:
        case OrderNotFoundError():
            return http.HTTPStatus.NOT_FOUND
        case InsufficientStockError() | OrderAlreadyExistsError():
            return http.HTTPStatus.CONFLICT
        case PaymentFailedError():
            return http.HTTPStatus.PAYMENT_REQUIRED
        case _:
            return http.HTTPStatus.INTERNAL_SERVER_ERROR
```

See `modern-python` for pattern matching and dataclass idioms.

---

## Centralized Error Handling (FastAPI)

```python
# BAD: try/except in every endpoint
@app.post("/orders")
def create_order(req: OrderRequest):
    try:
        order = order_service.create_order(req)
        return order
    except OrderNotFoundError:
        return JSONResponse(status_code=404, content={})
    except InsufficientStockError as e:
        return JSONResponse(status_code=409, content={"error": str(e)})
    except Exception:  # hides bugs, leaks nothing useful
        return JSONResponse(status_code=500, content={"error": "Internal error"})
```

```python
# GOOD: Centralized handlers — endpoints stay clean
import logging

from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

log = logging.getLogger(__name__)
app = FastAPI()

PROBLEM_JSON = "application/problem+json"


def _problem(status: int, title: str, exc: OrderError, **extra: object) -> JSONResponse:
    body = {
        "type": "about:blank",
        "title": title,
        "status": status,
        "detail": str(exc),
        "errorCode": exc.error_code,
        **extra,
    }
    return JSONResponse(status_code=status, content=body, media_type=PROBLEM_JSON)


@app.exception_handler(OrderNotFoundError)
def handle_not_found(request: Request, exc: OrderNotFoundError) -> JSONResponse:
    log.warning("Order not found: %s [path=%s]", exc.order_id, request.url.path)
    return _problem(404, "Order Not Found", exc, orderId=exc.order_id)


@app.exception_handler(InsufficientStockError)
def handle_stock(request: Request, exc: InsufficientStockError) -> JSONResponse:
    return _problem(409, "Insufficient Stock", exc, available=exc.available)


@app.exception_handler(OrderAlreadyExistsError)
def handle_duplicate(request: Request, exc: OrderAlreadyExistsError) -> JSONResponse:
    return _problem(409, "Duplicate Order", exc)


@app.exception_handler(Exception)
def handle_unexpected(request: Request, exc: Exception) -> JSONResponse:
    # ERROR — unexpected, needs investigation. exc_info captures the traceback.
    log.error("Unexpected error [path=%s]", request.url.path, exc_info=exc)
    # NEVER expose the traceback or internal details to the client.
    return JSONResponse(
        status_code=500,
        content={"type": "about:blank", "title": "Internal Server Error", "status": 500},
        media_type=PROBLEM_JSON,
    )
```

The endpoint stays clean — exceptions propagate to the registered handlers:

```python
# GOOD: No try/except — let the handlers do their job
@app.post("/orders", status_code=201)
def create_order(req: OrderRequest) -> OrderResponse:
    return order_service.create_order(req)
```

See `django-rest-api` for the Django REST Framework equivalent.

For Django REST Framework, wire one custom handler via
`REST_FRAMEWORK["EXCEPTION_HANDLER"]` that maps `OrderError` to the same shape
and falls back to DRF's default for everything else — see `django-rest-api`.

---

## RFC 7807 — problem+json

Return a consistent, machine-readable error shape with the
`application/problem+json` media type. Custom members (`errorCode`, `orderId`)
sit alongside the standard fields.

```json
{
    "type": "about:blank",
    "title": "Order Not Found",
    "status": 404,
    "detail": "Order not found: 12345",
    "instance": "/api/orders/12345",
    "errorCode": "ORDER_NOT_FOUND",
    "orderId": 12345
}
```

A small frozen `dataclass` (`Problem`) with a `to_dict()` that drops `None`
fields and renames `error_code -> errorCode` keeps every handler consistent.

---

## Validation Error Handling

Pydantic raises `ValidationError` with a structured `.errors()` list — map it to
a field-level response instead of leaking the raw exception.

```python
# GOOD: Structured response from a Pydantic ValidationError
from pydantic import ValidationError


def validation_problem(exc: ValidationError) -> dict[str, object]:
    errors = [
        {
            "field": ".".join(str(loc) for loc in err["loc"]),
            "message": err["msg"],
            "rejected": err.get("input"),
        }
        for err in exc.errors()
    ]
    return {
        "type": "about:blank",
        "title": "Validation Failed",
        "status": 422,
        "errors": errors,
    }
```

Response:
```json
{
    "type": "about:blank",
    "title": "Validation Failed",
    "status": 422,
    "errors": [
        { "field": "email", "message": "value is not a valid email address", "rejected": "not-an-email" },
        { "field": "name", "message": "String should have at least 1 character", "rejected": "" }
    ]
}
```

---

## Context Managers Over try/finally

Reach for `with` (and `contextlib`) instead of hand-rolled `try/finally` cleanup.

```python
# BAD: Manual cleanup, easy to leak on early return / exception
conn = pool.acquire()
try:
    do_work(conn)
finally:
    conn.close()
```

```python
# GOOD: Context manager guarantees release; intent is obvious
with pool.acquire() as conn:
    do_work(conn)


# GOOD: Build your own with @contextmanager
from contextlib import contextmanager
from collections.abc import Iterator


@contextmanager
def acquired(pool: ConnectionPool) -> Iterator[Connection]:
    conn = pool.acquire()
    try:
        yield conn
    finally:
        conn.close()
```

Use `contextlib.suppress(SomeError)` instead of an empty `except` when you
*genuinely* intend to ignore a specific error:

```python
# GOOD: Intentional, narrow, and self-documenting
from contextlib import suppress

with suppress(FileNotFoundError):
    cache_path.unlink()
```

See `efficient-python` and `flatten-nesting` for more on cutting boilerplate.

---

## Exception Groups and `except*`

When fanning out concurrent work, multiple tasks can fail at once. `ExceptionGroup`
(3.11+) preserves them all, and `except*` lets you handle by type.

```python
# GOOD: Collect concurrent failures without losing any
import asyncio


async def fetch_all(order_ids: list[int]) -> list[Order]:
    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(fetch_order(oid)) for oid in order_ids]
    return [t.result() for t in tasks]


try:
    orders = await fetch_all(ids)
except* OrderNotFoundError as eg:
    for exc in eg.exceptions:
        log.warning("Missing order: %s", exc)
except* PaymentFailedError as eg:
    log.error("Payment failures: %d", len(eg.exceptions))
```

See `python-concurrency` for `TaskGroup` and structured concurrency details.

---

## Logging in Exception Handlers

```python
# GOOD: Log level matches severity; use exc_info for tracebacks
import logging

log = logging.getLogger(__name__)


def handle_not_found(exc: OrderNotFoundError, path: str) -> Problem:
    # WARNING — a business rule, not a bug
    log.warning("Order not found: %s [path=%s]", exc.order_id, path)
    return Problem("Order Not Found", 404, str(exc), exc.error_code)


def handle_unexpected(exc: Exception, path: str) -> Problem:
    # ERROR — unexpected; exc_info attaches the full traceback
    log.error("Unexpected error [path=%s]", path, exc_info=exc)
    return Problem("Internal Server Error", 500, "An unexpected error occurred", "INTERNAL")
```

Never use bare `print()` or f-strings inside the log call — pass args so the
message is only formatted when the level is enabled.

### Correlation with contextvars

```python
# GOOD: A trace ID flows through all logs without threading it manually
import uuid
from contextvars import ContextVar

trace_id_var: ContextVar[str] = ContextVar("trace_id", default="")


async def correlation_middleware(request, call_next):
    trace_id = request.headers.get("X-Trace-Id") or str(uuid.uuid4())
    token = trace_id_var.set(trace_id)
    try:
        response = await call_next(request)
        response.headers["X-Trace-Id"] = trace_id
        return response
    finally:
        trace_id_var.reset(token)


class TraceFilter(logging.Filter):
    def filter(self, record: logging.LogRecord) -> bool:
        record.trace_id = trace_id_var.get()
        return True
```

`contextvars` is async- and thread-safe, unlike a thread-local. See
`logging-observability` for structured logging and correlation patterns.

---

## Recoverable vs Programmer Error

| Kind | What to Do |
|------|------------|
| **Recoverable** (not-found, validation, transient I/O) | Raise a specific domain exception; the caller catches and reacts |
| **Programmer error** (bad argument, broken invariant) | Let it propagate; don't catch `AssertionError`/`TypeError` to "handle" a bug |

**Rule of thumb:** catch the *narrowest* exception you can actually do something
about. Everything else should bubble up to the centralized handler, which logs it
and returns a safe 500.

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Bare `except:` | Catches `KeyboardInterrupt`/`SystemExit`, hides bugs | Catch the specific exception type |
| `except Exception: pass` | Silent failure | Log, re-raise, or use `contextlib.suppress` deliberately |
| `except` and `return None` | Pushes `AttributeError` downstream | Raise, or return a typed result |
| Traceback in API response | Information leak | Return a `problem+json` with a safe message |
| Error info packed in the message string | Re-parsing, no structure | Put data on dataclass attributes |
| `raise Exception("...")` | Untyped, uncatchable narrowly | Define a specific subclass |
| `raise NewError()` losing the cause | Drops the original traceback | `raise NewError() from exc` |
| Log-and-re-raise at every level | Duplicate log entries | Log once, at the handler boundary |
| `try/finally` for cleanup | Boilerplate, leak-prone | Use a context manager (`with`) |

---

## Tooling

- **ruff** flags bare `except`, blind `except Exception`, and `except ... pass`
  via the `BLE`, `TRY`, and `E722` rule sets — enable them.
- **mypy** catches handlers that return the wrong type and exceptions raised with
  missing dataclass fields.
- **pytest** — assert on exceptions with `pytest.raises(OrderNotFoundError)` and
  `excinfo.value.order_id`; see `testing-python`.

---

## Related Skills

- `django-rest-api` — API design patterns that work with centralized error handling
- `logging-observability` — Structured logging, contextvars, and correlation IDs
- `python-concurrency` — `TaskGroup`, `ExceptionGroup`, and `except*`
- `review-changes-python` — Error handling review checklist
- `modern-python` — Dataclasses and pattern matching for exception hierarchies
