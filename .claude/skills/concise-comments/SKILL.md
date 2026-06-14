---
name: concise-comments
description: Comment style for modern Python (3.12/3.13) — code should be self-explanatory first; comments justify the *why*, never restate the *what*; 1–2 lines max, locally scoped, no prose, no changelog. Use when reviewing or writing code that is reaching for a comment.
---

# Concise Comments

Comments compete with code for the reader's attention. Every comment is a small bet that the code below it is not clear enough on its own — and that bet has to keep paying off as the code changes. Most comments lose that bet within a release or two: the code moves, the comment doesn't, and now the file has a lie in it.

This skill is the rule set for when to comment, when to refactor instead, and how to keep the comments that survive small and load-bearing.

## When to Use

- Writing or reviewing any new comment, docstring, or `# TODO`
- Reading code where a comment is longer than the block it describes
- Reviewing diffs that add comments instead of better names or type hints
- Editing a file where existing comments no longer match the code
- Deciding between extracting a function and adding an explanatory comment

---

## Core principle

**The code is the documentation. The comment is the footnote.**

A comment earns its place by saying something the code *cannot* say:

- A constraint that isn't visible at the call site.
- A reason a non-obvious choice was made.
- A warning about a subtle invariant or footgun.
- A pointer to an external spec, ticket, or RFC the reader needs.

If the comment restates what the next line does, delete it and rename the next line. Type hints carry more than any comment can — let `mypy` enforce what a sentence merely claims.

The four pressures to push back on:

1. **Restatement** — comment narrates what the code already shows.
2. **Prose** — paragraph-length explanation where one line would do.
3. **Drift** — comment refers to code, files, or tickets no longer in scope.
4. **Decoration** — banners, dividers, `# constructor`, `# property`.

---

## Rule 1 — Refactor before commenting

If you reach for a comment to explain *what* a line does, the line is too cryptic. Rename or extract.

```python
# BAD — comment papers over a bad name
# check if the user can place an order today
if u.s == 1 and u.l > date.today() - timedelta(days=30):
    ...

# GOOD — the code says what the comment said
if user.can_place_order():
    ...
```

```python
# BAD — comment explains a clever line
# shift then mask to get the high nibble
hi = (b >> 4) & 0x0F

# GOOD — extract; the function name is the explanation
hi = high_nibble(b)
```

**Rule:** if the comment is *what*, the code is wrong. If the comment is *why*, the comment may be right. See `refactoring-python` for extract-function mechanics.

---

## Rule 2 — Comment the *why*, not the *what*

The reader can already see what the code does. They cannot see why it had to be done this way.

```python
# BAD — restates the line
# increment retry count
retries += 1

# GOOD — explains the decision the code can't show
# Stripe's idempotency window is 24h; retrying past that creates a new charge.
if (now - attempt.started_at) >= timedelta(hours=24):
    ...
```

```python
# BAD
# loop over orders
for order in orders:
    ...

# GOOD — only if the *why* is non-obvious
# Process oldest first so partial failures leave the newest intact for retry.
orders.sort(key=lambda o: o.placed_at)
```

**Rule:** if removing the comment wouldn't lose information a reader needs, don't write it.

---

## Rule 3 — One or two lines, never a paragraph

A comment that runs to a paragraph is a design doc that wandered into the source file. Move it to the PR description, the commit message, the ADR, or the ticket. Leave a one-line pointer if the reader needs it.

```python
# BAD — eight lines of prose nobody will keep current
def migrate(customer: Customer) -> None:
    """
    This function handles the migration of legacy customer records into the
    new profile schema. Originally the schema was flat, but in 2023 we split
    out preferences into a sub-document because of the GDPR work that the
    platform team was doing. The reason this function has two passes is that
    during the migration window we needed to support both shapes at once,
    which meant ... [continues]
    """
    ...

# GOOD — the explanation is in the ticket; leave a pointer
def migrate(customer: Customer) -> None:
    """Two-pass migration during the dual-write window. See PLAT-2189."""
    ...
```

**Rule:** if it doesn't fit in two lines, it doesn't belong inline. Link out.

---

## Rule 4 — Local scope only

A comment describes the code beneath it. It does not narrate other modules, recap the history of the package, or reference callers by name. Those things rot the moment something is renamed or moved.

```python
# BAD — references things outside the reader's view
# Called from OrderView.create() and order_batch_job.run(); the latter was
# added in v3.2 when we moved batch off the legacy queue. Used to live in
# order_helper before we extracted this service.
def place(self, request: OrderRequest) -> Order:
    ...

# GOOD — say only what's true here
def place(self, request: OrderRequest) -> Order:
    ...
```

```python
# BAD — restates project state
# This is the new way; the old PaymentGateway class is being deprecated.
def charge(self, amount: Money) -> Charge:
    ...

# GOOD — silent. Deprecation belongs on the deprecated class.
@deprecated("Use PaymentGateway instead")
class LegacyGateway:
    ...
```

**Rule:** if the comment talks about anything outside the function or class it's attached to, it will lie within a quarter. Cut it.

---

## Rule 5 — No noise comments

Some comments add zero information. They exist out of habit.

```python
# BAD — all of these are noise
class OrderService:

    # fields
    def __init__(self, repository: OrderRepository) -> None:
        # constructor
        self._repository = repository

    # property
    @property
    def repository(self) -> OrderRepository:
        return self._repository

    # ====================================================================
    #  PUBLIC API
    # ====================================================================
    def place(self, request: OrderRequest) -> Order:
        ...
```

Delete on sight:

- Banner dividers (`# ===`, `#---`, `# ### Section ###`).
- Section labels that name the next obvious construct (`# fields`, `# methods`, `# constructor`).
- Docstrings that restate the function name (`def place_order(...): """Places an order."""`).
- Boilerplate `:param x: the x` / `:returns: the result`.
- `# end of class` / `# end main`.

---

## Rule 6 — No changelog or dead-code comments

Git tracks history. The source file is not a second copy of it.

```python
# BAD
# 2024-03-12 (alice): added retry logic
# 2024-09-01 (bob): bumped timeout to 30s after PROD-441
# 2025-02-18 (alice): switched to asyncio.TaskGroup
def call(self) -> Response:
    ...

# GOOD — silent. `git log -L` and the commit messages own this.
def call(self) -> Response:
    ...
```

```python
# BAD — commented-out code "in case we need it"
def process(self, event: Event) -> None:
    self._handler.handle(event)
    # legacy path — keep until Q3
    # if event.is_legacy:
    #     self._legacy_handler.handle(event)

# GOOD — delete. Git keeps it findable.
def process(self, event: Event) -> None:
    self._handler.handle(event)
```

**Rule:** never use comments as version control. Delete dead code; let `git` remember. Ruff's `ERA` rule flags commented-out code automatically.

---

## Rule 7 — Docstrings on public API: short and specific

Public API deserves a docstring — but the same rules apply. One sentence of intent beats a paragraph of restated parameters, especially when type hints already carry the types.

```python
# BAD — boilerplate that adds nothing the signature doesn't
def place(self, request: OrderRequest, user_id: int) -> Order:
    """
    Places an order.

    :param request: the request
    :param user_id: the user id
    :returns: the order
    :raises OrderError: if the order cannot be placed
    """
    ...

# GOOD — says what the signature can't
def place(self, request: OrderRequest, user_id: int) -> Order:
    """Place an order and reserve stock atomically.

    Reservation is released if payment authorization fails.
    Idempotent on ``request.client_token``.
    """
    ...
```

**Rule:** a docstring says what the signature alone doesn't — atomicity, idempotency, side effects, thread/async safety, ordering. If you're only restating types and names, the type hints already did it; leave the docstring off.

---

## Rule 8 — TODOs must be actionable, or deleted

A `TODO` with no owner and no condition for resolution is a permanent comment pretending to be temporary.

```python
# BAD — will be there forever
try:
    ...
except OSError:
    # TODO: handle this better
    logger.warning("oops", exc_info=True)

# GOOD — pointer to a tracked item, with the *why*
try:
    ...
except OSError:
    # TODO(PLAT-3120): retry with backoff once the broker exposes idempotency keys.
    logger.warning("oops", exc_info=True)
```

**Rule:** every TODO needs a ticket reference *or* a concrete trigger ("when we drop 3.11"). Otherwise it's clutter — delete it or fix it now. Ruff's `TD` and `FIX` rules can enforce the format.

---

## Quick checklist (use during review)

- [ ] Every comment explains *why*, not *what*.
- [ ] No comment runs longer than two lines — anything bigger goes to the PR/ADR/ticket.
- [ ] No comment references other modules, callers, or history.
- [ ] No section banners, no `# constructor` / `# property`, no `# end of class`.
- [ ] No commented-out code; no date/author changelog blocks (let `ruff` catch them).
- [ ] Docstrings on public API state intent, not types — no `:param x: the x`.
- [ ] Type hints carry the contract; comments don't duplicate what `mypy` checks.
- [ ] Every `TODO` has a ticket or a concrete resolution trigger.
- [ ] The comment would still be true if the next reader renamed the surrounding class.

---

## When a comment is the right answer

Most comments should be deleted. These survive:

- **Non-obvious constraint:** `# Stripe rejects amounts < 50 cents.`
- **Subtle invariant:** `# Caller holds the lock; do not re-acquire.`
- **Workaround for an external bug:** `# httpx 0.27 drops trailing slash on redirect — see encode/httpx#3210.`
- **Performance choice the code can't show:** `# Manual loop: the comprehension materializes ~3x on the hot path.`
- **Pointer to an external spec:** `# RFC 7807 §3.1 — type must be a URI reference.`
- **Warning that prevents a future "fix":** `# Do not sort: order is significant for downstream signing.`
- **Type-checker escape hatch with a reason:** `x = untyped_lib.get()  # type: ignore[no-any-return]  # stub is wrong, see #88`

Each of these tells the reader something the code, the types, and the names cannot. That's the bar.

---

## Related Skills

- **modern-python** — type hints, `X | None`, dataclasses, and pattern matching that make comments unnecessary.
- **refactoring-python** — extract-function and rename moves to apply before reaching for a comment.
- **readable-comprehensions** — when a comprehension needs no comment vs. when it should become a loop.
- **review-changes-python** — using this checklist while reviewing a diff.
- **logging-observability** — what belongs in a log line vs. a comment.
