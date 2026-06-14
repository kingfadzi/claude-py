---
name: python-concurrency
description: Concurrency patterns for Python/Django — asyncio and the event loop, the GIL, threads vs processes, concurrent.futures, structured concurrency with TaskGroup, anyio, and Django async views. Use when dealing with async I/O, multi-threading, parallel processing, or concurrent data access.
---

# Python Concurrency

Concurrency and parallelism patterns for Django applications on Python 3.12/3.13.

## When to Use

- Choosing between asyncio, threads, and processes for a workload
- Writing async Django views, middleware, or ORM access
- Fanning out I/O-bound work (HTTP calls, DB queries) and joining results
- Parallelizing CPU-bound computation across cores
- Handling concurrent database access and lost updates

---

## Quick Reference

| Need | Approach |
|------|----------|
| I/O-bound parallelism (HTTP, sockets) | `asyncio` + `async`/`await` |
| Fan-out/fan-in async tasks | `asyncio.TaskGroup` (3.11+) |
| CPU-bound parallelism | `ProcessPoolExecutor` (sidestep the GIL) |
| Blocking I/O in async code | `asyncio.to_thread(...)` |
| Blocking I/O without asyncio | `ThreadPoolExecutor` |
| Backend-agnostic async (trio/asyncio) | `anyio` |
| Async DB access in Django | `async`/`await` ORM methods (`aget`, `acreate`) |
| Concurrent DB writes | `select_for_update()` or optimistic version check |

---

## The GIL: Threads vs Processes

The Global Interpreter Lock means only one thread executes Python bytecode at a time. Threads help with **I/O-bound** work (they release the GIL while blocked on I/O); they do **not** speed up **CPU-bound** work.

```python
# BAD: ThreadPoolExecutor for CPU-bound work — GIL serializes it, no speedup
from concurrent.futures import ThreadPoolExecutor

def hash_pricing(rows: list[Row]) -> list[str]:
    with ThreadPoolExecutor(max_workers=8) as pool:
        # All 8 threads contend for the GIL; effectively single-core.
        return list(pool.map(expensive_pure_python_hash, rows))


# GOOD: ProcessPoolExecutor for CPU-bound work — true parallelism across cores
from concurrent.futures import ProcessPoolExecutor

def hash_pricing(rows: list[Row]) -> list[str]:
    with ProcessPoolExecutor(max_workers=8) as pool:
        return list(pool.map(expensive_pure_python_hash, rows))
```

```python
# GOOD: Threads ARE the right tool for blocking I/O (GIL released during I/O)
from concurrent.futures import ThreadPoolExecutor

def fetch_all(urls: list[str]) -> list[Response]:
    with ThreadPoolExecutor(max_workers=16) as pool:
        return list(pool.map(httpx.get, urls))
```

> On Python 3.13+ a free-threaded (no-GIL) build exists, but it is experimental. Assume the GIL is present unless you have explicitly opted in.

---

## asyncio Fundamentals

One event loop per thread runs many coroutines cooperatively. A coroutine must `await` to yield control — a blocking call freezes the whole loop.

```python
# BAD: Blocking call inside a coroutine — stalls the entire event loop
import time
import requests  # synchronous

async def fetch_quote(symbol: str) -> Quote:
    time.sleep(0.2)               # blocks the loop for every caller
    resp = requests.get(url(symbol))  # synchronous I/O blocks the loop too
    return Quote.from_json(resp.json())


# GOOD: Use awaitables; offload unavoidable blocking calls to a thread
import asyncio
import httpx

async def fetch_quote(client: httpx.AsyncClient, symbol: str) -> Quote:
    await asyncio.sleep(0.2)                  # non-blocking
    resp = await client.get(url(symbol))      # non-blocking I/O
    return Quote.from_json(resp.json())

async def legacy_call(arg: str) -> Result:
    # No async variant available? Run it in a thread so the loop stays free.
    return await asyncio.to_thread(blocking_sdk_call, arg)
```

---

## Structured Concurrency with TaskGroup

`asyncio.TaskGroup` (3.11+) is the idiomatic fan-out/fan-in: it awaits all children, and if any raises, it cancels the siblings and propagates an `ExceptionGroup`. Prefer it over bare `asyncio.gather`.

```python
# BAD: gather without cancellation semantics — a failure leaves siblings running
async def build_dashboard(user_id: int) -> Dashboard:
    user, orders, prefs = await asyncio.gather(
        fetch_user(user_id),
        fetch_orders(user_id),
        fetch_prefs(user_id),
    )
    # gather(return_exceptions=False) raises the FIRST error, but the other
    # tasks keep running in the background — wasted work, possible warnings.
    return Dashboard(user, orders, prefs)


# GOOD: TaskGroup — fail-fast, automatic cancellation of siblings
async def build_dashboard(user_id: int) -> Dashboard:
    async with asyncio.TaskGroup() as tg:
        user_t = tg.create_task(fetch_user(user_id))
        orders_t = tg.create_task(fetch_orders(user_id))
        prefs_t = tg.create_task(fetch_prefs(user_id))
    # On exit the block has awaited all tasks. If ANY failed, the rest were
    # cancelled and an ExceptionGroup is raised here.
    return Dashboard(user_t.result(), orders_t.result(), prefs_t.result())
```

### Handling ExceptionGroup

```python
# GOOD: except* matches specific members of the group
try:
    await build_dashboard(user_id)
except* TimeoutError as eg:
    logger.warning("dashboard timed out: %s", eg.exceptions)
except* ValueError as eg:
    logger.error("bad data while building dashboard", exc_info=eg)
```

See `exception-handling` for `except*` and error-propagation patterns.

---

## Cancellation and Timeouts

```python
# BAD: wait_for is fine, but swallowing CancelledError breaks cancellation
async def run() -> None:
    try:
        await long_operation()
    except asyncio.CancelledError:
        pass  # NEVER swallow — the task won't actually stop


# GOOD: asyncio.timeout context manager; always re-raise CancelledError
async def run() -> Result | None:
    try:
        async with asyncio.timeout(5.0):
            return await long_operation()
    except TimeoutError:
        logger.warning("operation exceeded 5s budget")
        return None
    except asyncio.CancelledError:
        await cleanup()
        raise  # propagate cancellation
```

---

## Bounding Concurrency

Unbounded fan-out exhausts sockets, file handles, or the remote service. Gate it with a `Semaphore`.

```python
# BAD: 10_000 simultaneous requests — connection errors, rate-limit bans
async def scrape(urls: list[str]) -> list[Page]:
    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(fetch(u)) for u in urls]
    return [t.result() for t in tasks]


# GOOD: cap in-flight work with a semaphore
async def scrape(urls: list[str], limit: int = 20) -> list[Page]:
    sem = asyncio.Semaphore(limit)

    async def guarded(url: str) -> Page:
        async with sem:
            return await fetch(url)

    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(guarded(u)) for u in urls]
    return [t.result() for t in tasks]
```

---

## anyio — Backend-Agnostic Structured Concurrency

`anyio` runs on asyncio or trio, gives you task groups everywhere (incl. older Pythons), and powers Starlette/FastAPI. Use it for libraries that must not assume a backend.

```python
# GOOD: anyio task group + thread offload — works on asyncio AND trio
import anyio

async def enrich(order_ids: list[int]) -> list[Order]:
    results: list[Order] = []

    async def worker(order_id: int) -> None:
        order = await anyio.to_thread.run_sync(load_order_blocking, order_id)
        results.append(order)

    async with anyio.create_task_group() as tg:
        for oid in order_ids:
            tg.start_soon(worker, oid)
    return results
```

---

## concurrent.futures for Sync Code

When you are not in an event loop, `concurrent.futures` is the clean primitive. Prefer `as_completed` over polling, and a context manager over manual `shutdown()`.

```python
# BAD: manual future polling and no shutdown — leaks the pool
pool = ThreadPoolExecutor(max_workers=8)
futures = [pool.submit(fetch, u) for u in urls]
results = []
while futures:
    for f in list(futures):
        if f.done():
            results.append(f.result())
            futures.remove(f)


# GOOD: context manager + as_completed
from concurrent.futures import ThreadPoolExecutor, as_completed

def fetch_all(urls: list[str]) -> list[Response]:
    with ThreadPoolExecutor(max_workers=8) as pool:
        futures = {pool.submit(fetch, u): u for u in urls}
        results: list[Response] = []
        for future in as_completed(futures):
            url = futures[future]
            try:
                results.append(future.result())
            except httpx.HTTPError:
                logger.exception("fetch failed for %s", url)
        return results
```

---

## Shared State and Thread Safety

Module-level mutable state shared across threads is a race condition. Prefer immutability; if you must share, use a lock or a thread-safe primitive.

```python
# BAD: unsynchronized shared mutable state — lost updates under threads
class MetricsCollector:
    def __init__(self) -> None:
        self.request_count = 0
        self.by_endpoint: dict[str, int] = {}

    def record(self, endpoint: str) -> None:
        self.request_count += 1                       # read-modify-write race
        self.by_endpoint[endpoint] = self.by_endpoint.get(endpoint, 0) + 1


# GOOD: guard the critical section with a lock
import threading

class MetricsCollector:
    def __init__(self) -> None:
        self._lock = threading.Lock()
        self.request_count = 0
        self.by_endpoint: dict[str, int] = {}

    def record(self, endpoint: str) -> None:
        with self._lock:
            self.request_count += 1
            self.by_endpoint[endpoint] = self.by_endpoint.get(endpoint, 0) + 1
```

```python
# BETTER: avoid shared state entirely — return immutable results and fold them
from dataclasses import dataclass
from collections import Counter

@dataclass(frozen=True, slots=True)
class Batch:
    endpoint: str
    count: int

def summarize(batches: list[Batch]) -> Counter[str]:
    totals: Counter[str] = Counter()
    for b in batches:
        totals[b.endpoint] += b.count
    return totals
```

In asyncio, single-threaded cooperative scheduling means you rarely need locks for in-memory state — but you still need `asyncio.Lock` (not `threading.Lock`) to serialize across `await` points.

---

## Django Async Views

Django supports `async def` views end to end. Mixing sync and async incorrectly throws `SynchronousOnlyOperation`. Use the async ORM methods, and wrap unavoidable sync calls in `sync_to_async`.

```python
# BAD: sync ORM call inside an async view — SynchronousOnlyOperation
async def order_detail(request, order_id: int) -> JsonResponse:
    order = Order.objects.get(id=order_id)        # raises in async context
    return JsonResponse(order.as_dict())


# GOOD: async ORM methods + concurrent fan-out with TaskGroup
from django.http import JsonResponse

async def order_detail(request, order_id: int) -> JsonResponse:
    order = await Order.objects.aget(id=order_id)  # async query
    async with asyncio.TaskGroup() as tg:
        items_t = tg.create_task(_load_items(order_id))
        ship_t = tg.create_task(fetch_shipping_quote(order))
    return JsonResponse({
        "order": order.as_dict(),
        "items": [i.as_dict() for i in items_t.result()],
        "shipping": ship_t.result(),
    })

async def _load_items(order_id: int) -> list[Item]:
    return [item async for item in Item.objects.filter(order_id=order_id)]
```

```python
# GOOD: bridge to legacy sync code with sync_to_async
from asgiref.sync import sync_to_async

async def export(request) -> JsonResponse:
    # thread_sensitive=True (default) runs it in a shared thread so DB
    # connections and transactions behave correctly.
    payload = await sync_to_async(build_legacy_report)(request.user.id)
    return JsonResponse(payload)
```

See `django-orm` for query patterns and `django-rest-api` for async view design.

---

## Concurrent Database Access

The ORM does not protect you from two requests overwriting each other. Choose pessimistic or optimistic locking explicitly.

```python
# BAD: read-modify-write race — concurrent requests lose updates
@transaction.atomic
def apply_discount(product_id: int, pct: Decimal) -> None:
    product = Product.objects.get(id=product_id)
    product.price *= (1 - pct)   # another tx may have changed price meanwhile
    product.save()


# GOOD: pessimistic lock — serialize the row for the transaction
@transaction.atomic
def apply_discount(product_id: int, pct: Decimal) -> None:
    product = Product.objects.select_for_update().get(id=product_id)
    product.price *= (1 - pct)
    product.save()
```

```python
# GOOD: optimistic concurrency — a version column, retried on conflict
class Product(models.Model):
    price = models.DecimalField(max_digits=10, decimal_places=2)
    version = models.PositiveIntegerField(default=0)

@transaction.atomic
def apply_discount(product_id: int, pct: Decimal) -> None:
    product = Product.objects.get(id=product_id)
    updated = (
        Product.objects.filter(id=product_id, version=product.version)
        .update(price=product.price * (1 - pct), version=product.version + 1)
    )
    if updated == 0:
        raise ConcurrentModificationError(product_id)  # caller retries
```

For atomic counters, push the work into the database with `F()` instead of read-modify-write:

```python
# GOOD: F() expression — the database performs the increment atomically
Product.objects.filter(id=product_id).update(view_count=F("view_count") + 1)
```

---

## Background Jobs

`@Scheduled`-style cron belongs in a task queue (Celery, Django-Q, RQ), not in a thread you spawn from a request. Never block a request thread on long work.

```python
# BAD: spawning a raw thread from a view — no lifecycle, lost on restart
def trigger_report(request) -> HttpResponse:
    threading.Thread(target=build_huge_report, args=(request.user.id,)).start()
    return HttpResponse(status=202)


# GOOD: hand off to a task queue (Celery shown)
from celery import shared_task

@shared_task
def build_report(user_id: int) -> None:
    ...

def trigger_report(request) -> HttpResponse:
    build_report.delay(request.user.id)
    return HttpResponse(status=202)
```

For periodic jobs use Celery Beat or `cron` invoking a management command — and guard against duplicate runs across workers with a database/Redis lock (the Python analogue of distributed locking).

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| `ThreadPoolExecutor` for CPU-bound work | GIL serializes it — no speedup | `ProcessPoolExecutor` |
| Blocking call (`time.sleep`, `requests`) in a coroutine | Freezes the whole event loop | `await` async APIs or `asyncio.to_thread` |
| Bare `asyncio.gather` for fan-out | Siblings keep running on failure | `asyncio.TaskGroup` |
| Swallowing `asyncio.CancelledError` | Task never actually cancels | Re-raise after cleanup |
| Unbounded fan-out | Socket/FD exhaustion, rate-limit bans | `asyncio.Semaphore` |
| Sync ORM call in async view | `SynchronousOnlyOperation` | Async ORM methods / `sync_to_async` |
| `threading.Lock` across `await` | Wrong lock; blocks the loop | `asyncio.Lock` |
| Read-modify-write on a DB row | Lost updates under concurrency | `select_for_update()` or `F()` |
| Raw `Thread` from a request | No lifecycle, lost on restart | Task queue (Celery/RQ/Django-Q) |
| Unguarded periodic job across workers | Duplicate execution in a cluster | Distributed lock (Redis/DB) |

---

## Related Skills

- `modern-python` — async/await, `except*`, structural pattern matching, dataclasses
- `django-orm` — async ORM methods, `select_for_update`, `F()` expressions, transactions
- `django-rest-api` — designing async views and endpoints
- `exception-handling` — `ExceptionGroup`/`except*` and cancellation handling
- `logging-observability` — correlating logs across threads, tasks, and workers
- `efficient-python` — choosing the right primitive and avoiding wasted work
