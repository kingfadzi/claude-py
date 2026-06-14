---
name: logging-observability
description: Logging and observability for Python/Django — stdlib logging + structlog, structured JSON logs, contextvars correlation IDs, OpenTelemetry tracing and metrics, and Django LOGGING config. Use when configuring logging, adding metrics or tracing, reviewing log statements, or setting up observability.
---

# Logging & Observability

Structured logging, metrics, and tracing for Python and Django applications.

## When to Use

- Configuring logging for a new service
- Adding metrics or tracing to existing code
- Reviewing log statements for correctness
- Setting up Django `LOGGING` or OpenTelemetry
- Debugging production issues via logs

---

## Quick Reference

| Concern | Tool | Notes |
|---------|------|-------|
| Logging | stdlib `logging` | Always available, framework-agnostic |
| Structured logging | `structlog` + JSON | Key-value event logs |
| Correlation IDs | `contextvars` | Async/thread-safe, not thread-local |
| Metrics | OpenTelemetry Metrics | Vendor-neutral, OTLP export |
| Health checks | Django view / `django-health-check` | `/healthz` endpoint |
| Distributed tracing | OpenTelemetry SDK | `opentelemetry-instrument` auto-instruments |

Install with `uv`:

```bash
uv add structlog opentelemetry-distro opentelemetry-exporter-otlp
uv run opentelemetry-bootstrap -a install  # pulls matching instrumentations
```

---

## stdlib logging

### Lazy Formatting

```python
# BAD: f-string is always evaluated, even if DEBUG is disabled
log.debug(f"Processing user {user.name} with id {user.id}")

# BAD: print() in production code — not configurable, no levels, no handlers
print(f"Order created: {order_id}")

# GOOD: %-style args — interpolation deferred until the record is emitted
log.debug("Processing user %s with id %s", user.name, user.id)

# GOOD: guard genuinely expensive work
if log.isEnabledFor(logging.DEBUG):
    log.debug("Full order details: %s", order.to_detailed_dict())
```

### Log Levels

| Level | When to Use | Example |
|-------|------------|---------|
| `ERROR` | Unexpected failure requiring investigation | Unhandled exception, external service down |
| `WARNING` | Expected but noteworthy issue | Business rule violation, deprecated API called |
| `INFO` | Key business events | Order created, user registered, job completed |
| `DEBUG` | Developer-useful context | Function entry/exit, intermediate state |

Python has no `TRACE`; use `DEBUG` for fine-grained diagnostics, or define a custom level if you must.

```python
# GOOD: appropriate levels, %-args, exc_info for stack traces
log.info("Order created: order_id=%s customer_id=%s", order.id, order.customer_id)
log.warning("Payment retry attempt %s for order %s", attempt, order_id)
log.error("Payment processing failed for order %s", order_id, exc_info=exc)
log.debug("Applying discount: type=%s rate=%s", discount_type, rate)

# GOOD: inside an except block, log.exception captures the active traceback
try:
    charge(order)
except PaymentError:
    log.exception("Payment processing failed for order %s", order_id)
```

### Logger Declaration

```python
# BAD: the root logger loses the source module and ignores per-module config
logging.info("started")

# GOOD: module-level named logger — gives you the __name__ hierarchy
import logging

log = logging.getLogger(__name__)
```

Never call `logging.basicConfig()` from library/app modules — configure once at the entry point (or via Django `LOGGING`).

---

## Sensitive Data

```python
# BAD: logging secrets and PII
log.info("User login: email=%s password=%s", email, password)
log.debug("Payment processed: card_number=%s", card_number)
log.info("API call with token: %s", bearer_token)

# GOOD: omit or mask
log.info("User login: email=%s", email)
log.debug("Payment processed: last4=%s", card_number[-4:])
log.info("API call for user %s", user_id)
```

A reusable filter keeps secrets out even when a careless call slips through:

```python
import logging
import re

_SECRET_RE = re.compile(r"(password|token|api[_-]?key|secret)=\S+", re.IGNORECASE)


class RedactingFilter(logging.Filter):
    """Mask secret-looking key=value pairs in the formatted message."""

    def filter(self, record: logging.LogRecord) -> bool:
        record.msg = _SECRET_RE.sub(r"\1=***", record.getMessage())
        record.args = ()  # already interpolated above
        return True
```

**Never log:** passwords, tokens, API keys, card numbers, SSNs, or PII beyond what debugging requires.

---

## Structured Logging (structlog)

Plain text logs are hard to query in production. Emit JSON with stable keys.

```python
# BAD: hand-built "structured" string — fragile, not machine-parseable
log.info("Order created orderId=" + str(order.id) + " total=" + str(order.total))

# GOOD: structlog event with typed key-value context
import structlog

log = structlog.get_logger()

log.info("order.created", order_id=order.id, total=order.total)
# JSON: {"event": "order.created", "order_id": 123, "total": "99.99", "timestamp": "..."}
```

### Configuring structlog with JSON output

```python
import logging

import structlog


def configure_logging(*, json_logs: bool, level: str = "INFO") -> None:
    shared_processors: list[structlog.types.Processor] = [
        structlog.contextvars.merge_contextvars,  # inject correlation id etc.
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso", utc=True),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
    ]
    renderer = (
        structlog.processors.JSONRenderer()
        if json_logs
        else structlog.dev.ConsoleRenderer()  # pretty colours for local dev
    )
    structlog.configure(
        processors=[*shared_processors, renderer],
        wrapper_class=structlog.make_filtering_bound_logger(
            logging.getLevelNamesMapping()[level]
        ),
        logger_factory=structlog.PrintLoggerFactory(),
        cache_logger_on_first_use=True,
    )
```

Call `configure_logging(json_logs=not settings.DEBUG)` once at startup (e.g. in your Django `AppConfig.ready()` or ASGI/WSGI entry point).

---

## Correlation IDs with contextvars

MDC-style thread-locals break under `asyncio`. Use `contextvars.ContextVar`, which is both thread-safe and task-safe and is automatically merged by `structlog.contextvars`.

### Django middleware

```python
import uuid
from collections.abc import Callable

import structlog
from django.http import HttpRequest, HttpResponse


class CorrelationIdMiddleware:
    """Bind a correlation id for the lifetime of each request."""

    def __init__(self, get_response: Callable[[HttpRequest], HttpResponse]) -> None:
        self.get_response = get_response

    def __call__(self, request: HttpRequest) -> HttpResponse:
        correlation_id = request.headers.get("X-Correlation-Id") or str(uuid.uuid4())
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(
            correlation_id=correlation_id,
            method=request.method,
            path=request.path,
        )
        try:
            response = self.get_response(request)
        finally:
            structlog.contextvars.clear_contextvars()
        response["X-Correlation-Id"] = correlation_id
        return response
```

Every `log.info(...)` within the request now carries `correlation_id` automatically.

Without structlog, expose the id to stdlib via a filter that reads the
`ContextVar` (`record.correlation_id = correlation_id.get()`) and reference
`%(correlation_id)s` in your format string.

### Propagation across tasks and threads

`contextvars` copies automatically into `asyncio.create_task` and `loop.run_in_executor`. For a raw `ThreadPoolExecutor`, copy the context explicitly:

```python
# BAD: worker thread loses the correlation id
executor.submit(process_order, order)

# GOOD: run the callable inside a copy of the current context
import contextvars

ctx = contextvars.copy_context()
executor.submit(ctx.run, process_order, order)
```

See `python-concurrency` for context propagation across executors and tasks.

---

## OpenTelemetry Metrics

```python
# BAD: ad-hoc module global — not exported, not thread-safe, mixed into domain logic
_orders_created = 0  # incremented inline in create_order(); goes nowhere

# GOOD: extract instruments into a dedicated metrics object
from opentelemetry import metrics

meter = metrics.get_meter("orders")


class OrderMetrics:
    def __init__(self) -> None:
        self.created = meter.create_counter(
            "orders.created", unit="1", description="Orders created"
        )
        self.failed = meter.create_counter("orders.failed", unit="1")
        self.duration = meter.create_histogram(
            "orders.create.duration", unit="ms", description="Time to create an order"
        )

    def record_created(self, *, order_type: str) -> None:
        self.created.add(1, {"type": order_type})

    def record_failed(self) -> None:
        self.failed.add(1)
```

### Timing a scope with a context manager

Prefer a context manager over manual `start`/`stop` so the duration is recorded even on exceptions.

```python
import time
from collections.abc import Iterator
from contextlib import contextmanager

from opentelemetry.metrics import Histogram


@contextmanager
def timed(histogram: Histogram, **attrs: str) -> Iterator[None]:
    start = time.perf_counter()
    try:
        yield
    finally:
        histogram.record((time.perf_counter() - start) * 1000, attrs)


with timed(order_metrics.duration, type="standard"):
    process_order(request)
```

### Observable (callback) gauge

```python
from opentelemetry.metrics import CallbackOptions, Observation


def active_orders(options: CallbackOptions) -> list[Observation]:
    return [Observation(OrderRepository.active_count())]


meter.create_observable_gauge("orders.active", callbacks=[active_orders])
```

---

## Health Checks

```python
# BAD: returns 200 even when the database is down — useless to a load balancer
def healthz(request: HttpRequest) -> JsonResponse:
    return JsonResponse({"status": "ok"})

# GOOD: actually probe dependencies and return 503 on failure
from django.db import connection
from django.http import HttpRequest, JsonResponse


def healthz(request: HttpRequest) -> JsonResponse:
    checks: dict[str, str] = {}
    status = 200
    try:
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
        checks["database"] = "up"
    except Exception:
        checks["database"] = "down"
        status = 503
    body = {"status": "ok" if status == 200 else "degraded", **checks}
    return JsonResponse(body, status=status)
```

Keep `/healthz` (liveness) cheap and unauthenticated; gate any detailed `/readyz` behind auth so you don't leak dependency topology. See `django-security` for endpoint protection.

---

## Distributed Tracing (OpenTelemetry)

Zero-code instrumentation wraps Django, the DB driver, and HTTP clients:

```bash
# Wrap your server; reads OTEL_* env vars
OTEL_SERVICE_NAME=order-service \
OTEL_TRACES_SAMPLER=parentbased_traceidratio \
OTEL_TRACES_SAMPLER_ARG=1.0 \
OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4318 \
uv run opentelemetry-instrument gunicorn config.wsgi:application
```

Sample everything in dev (`1.0`); lower the ratio in production. Bridge stdlib logs to traces with `OTEL_PYTHON_LOG_CORRELATION=true` so each record carries `trace_id`/`span_id`.

### Custom spans

```python
# BAD: a slow internal step is invisible — only auto-instrumented edges show up
def create_order(request: CreateOrderRequest) -> OrderResponse:
    return process_order(request)

# GOOD: wrap the meaningful unit of work in an explicit span
from opentelemetry import trace

tracer = trace.get_tracer("orders")


def create_order(request: CreateOrderRequest) -> OrderResponse:
    with tracer.start_as_current_span("order.create") as span:
        span.set_attribute("order.type", request.type)
        return process_order(request)
```

For whole functions the decorator form is cleaner:
`@tracer.start_as_current_span("order.process")` above the `def`.

---

## Django LOGGING Config

```python
# settings.py — single source of truth, configured once at startup
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "filters": {
        "redact": {"()": "myapp.logging.RedactingFilter"},
    },
    "formatters": {
        "json": {
            "()": "structlog.stdlib.ProcessorFormatter",
            "processor": "structlog.processors.JSONRenderer",
        },
        "console": {
            "format": "%(asctime)s [%(levelname)s] %(name)s: %(message)s",
        },
    },
    "handlers": {
        "stdout": {
            "class": "logging.StreamHandler",
            "filters": ["redact"],
            "formatter": "json" if not DEBUG else "console",
        },
    },
    "root": {"handlers": ["stdout"], "level": "INFO"},
    "loggers": {
        # Quiet noisy framework loggers without silencing your own
        "django.db.backends": {"level": "WARNING", "propagate": True},
        "myapp": {"level": "DEBUG" if DEBUG else "INFO", "propagate": True},
    },
}
```

Drive levels from the environment (`LOG_LEVEL`) rather than hard-coding, and emit JSON to stdout so the platform (k8s, systemd) handles shipping. See `django-configuration` for env-driven settings.

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| `print()` for diagnostics | Not configurable, no levels | Named `logging.getLogger(__name__)` |
| f-string in log call | Interpolated even when level disabled | `%s` args, deferred formatting |
| Logging secrets / PII | Security & compliance risk | Mask or omit; `RedactingFilter` |
| `ERROR` for expected cases | Alert fatigue | `WARNING` for business rules |
| `logging.basicConfig` in modules | Clobbers config, double handlers | Configure once at entry point |
| Thread-local correlation ids | Breaks under asyncio | `contextvars` + structlog |
| Missing context cleanup | Leaks ids into the next request | `clear_contextvars()` in `finally` |
| No structured logging | Hard to query in prod | structlog JSON renderer |
| Metrics in domain code | Clutters business logic | Extract to a metrics class |
| Manual span start/stop | Spans leak on exceptions | `start_as_current_span` context manager |
| `except Exception: log.error(e)` | Drops the traceback | `log.exception(...)` / `exc_info=` |

---

## Related Skills

- `exception-handling` — Logging in `except` blocks, `log.exception`, context propagation
- `django-configuration` — Env-driven `LOGGING`, per-environment log levels
- `python-concurrency` — `contextvars` propagation across tasks, threads, and executors
