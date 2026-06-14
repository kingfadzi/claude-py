---
name: flatten-nesting
description: Eliminate nested loops, nested conditionals, and mixed loop/conditional nesting in Python. Guard clauses, early returns, dict/set lookups over inner loops, structural pattern matching over if-elif cascades, comprehensions/generators, and extract-function to flatten "arrow code". Use when reviewing or writing functions with > 2 levels of indentation, nested for-loops, deeply nested if/elif, or any combination.
---

# Flatten Nesting

Nesting is a readability tax compounded by performance cost. A function with 3+ levels of indent forces the reader to hold a stack of "what conditions apply here" while reading the body. This skill is about flattening — collapsing nesting back to one or two levels using guard clauses, dict/set lookups, structural pattern matching, comprehensions, and extract-function.

Not the same as `efficient-python` (which focuses on the *performance* of nested loops) — this skill is about the *structure*: even O(1) nested code reads badly. Python makes nesting especially painful because indentation *is* syntax: every extra level is four more columns of drift and a real `IndentationError` risk on edit.

## When to Use

- Any function with > 2 levels of indentation in its body
- Nested `for` / `while` loops, even short ones
- Cascading `if / elif / elif / ...` with > 3 branches
- Mixed `for: if: for: if:` patterns
- "Arrow code" — code that drifts steeply to the right
- Reviewing a function and finding yourself scrolling sideways
- A comprehension with two `for` clauses *and* a body too complex to read

---

## Core principle

**Two levels of nesting maximum** in any function body. The body itself counts as level 0; one `if` or one `for` puts you at level 1; one `if` inside a `for` puts you at level 2; anything beyond that is a refactor.

The four moves to flatten:

1. **Guard clauses** — `return` / `raise` / `continue` early to remove an `else`.
2. **Lookup tables** — replace an inner loop with `dict` / `set` access.
3. **Pattern matching over cascade** — collapse `if / elif` into `match` (Python 3.10+).
4. **Extract function** — pull the deep block into a named function that starts fresh at level 0.

Lint rules to enforce it: ruff's `PLR0915` (too many statements), `C901` (mccabe complexity), and `PLR1702` (too many nested blocks) all flag the symptom. Turn them on.

---

## Rule 1 — Guard clauses kill `else`

If an `if` body returns, raises, or continues, the `else` is dead weight. Invert and flatten.

```python
# BAD — happy path is buried at level 3
def process(payload: Input | None) -> Result:
    if payload is not None:
        if payload.is_valid():
            if payload.has_body():
                return do_work(payload)
            else:
                raise BadRequestError("missing body")
        else:
            raise BadRequestError("invalid input")
    else:
        raise BadRequestError("null input")


# GOOD — guards up front, happy path at level 0
def process(payload: Input | None) -> Result:
    if payload is None:
        raise BadRequestError("null input")
    if not payload.is_valid():
        raise BadRequestError("invalid input")
    if not payload.has_body():
        raise BadRequestError("missing body")
    return do_work(payload)
```

Same shape works inside loops with `continue`:

```python
# BAD
for item in items:
    if item.is_active:
        if item.owner is not None:
            process(item)

# GOOD
for item in items:
    if not item.is_active:
        continue
    if item.owner is None:
        continue
    process(item)

# BETTER — if it's pure filtering, lift to a generator expression
for item in (i for i in items if i.is_active and i.owner is not None):
    process(item)
```

See `exception-handling` for *which* exception to raise in a guard; see `readable-comprehensions` for when the generator form earns its keep.

---

## Rule 2 — Replace inner loop with a lookup

A `for` inside another `for` to find a match is almost always a `dict`/`set` lookup waiting to happen. Even when N is small, the nested form reads worse and is accidentally O(n²).

```python
# BAD — nested loop to "find the matching guild"
for eng in engagements:
    for guild in guilds:
        if guild.id == eng.guild_id:
            eng.guild_name = guild.name
            break

# GOOD — index once, look up
guild_by_id: dict[str, Guild] = {g.id: g for g in guilds}

for eng in engagements:
    guild = guild_by_id.get(eng.guild_id)
    if guild is not None:
        eng.guild_name = guild.name
```

The `set` membership form is also flatter than `for + if`:

```python
# BAD
for name in mandatory:
    found = False
    for assigned in assigned_names:
        if assigned.casefold() == name.casefold():
            found = True
            break
    if not found:
        missing.append(name)

# GOOD
assigned_folded = {a.casefold() for a in assigned_names}
missing = [name for name in mandatory if name.casefold() not in assigned_folded]
```

**Rule:** if the inner loop's only job is to find one matching element, you wanted a `dict`/`set`. Build the index *outside* the outer loop. For grouped indexes, reach for `collections.defaultdict(list)` or `itertools.groupby` (see `leverage-libraries`).

---

## Rule 3 — `match` over cascading `if / elif`

A chain of `if x == "A": ... elif x == "B": ...` reads as a flat dispatch with extra ceremony. Python 3.10+ `match` handles literals, classes, and destructuring.

```python
# BAD — cascading if/elif on a discriminator
if event.type == EventType.CREATED:
    return f"created at {event.timestamp}"
elif event.type == EventType.UPDATED:
    return f"updated by {event.actor}"
elif event.type == EventType.DELETED:
    return f"deleted reason: {event.reason}"
else:
    return "unknown"


# GOOD — match statement, explicit default
match event.type:
    case EventType.CREATED:
        return f"created at {event.timestamp}"
    case EventType.UPDATED:
        return f"updated by {event.actor}"
    case EventType.DELETED:
        return f"deleted reason: {event.reason}"
    case _:
        return "unknown"
```

For a tagged-union hierarchy, structural pattern matching with capture is the move:

```python
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class Circle:
    radius: float


@dataclass(frozen=True, slots=True)
class Rectangle:
    width: float
    height: float


@dataclass(frozen=True, slots=True)
class Triangle:
    base: float
    height: float


Shape = Circle | Rectangle | Triangle


# BAD — isinstance chain
def area(shape: Shape) -> float:
    if isinstance(shape, Circle):
        return math.pi * shape.radius**2
    elif isinstance(shape, Rectangle):
        return shape.width * shape.height
    elif isinstance(shape, Triangle):
        return 0.5 * shape.base * shape.height
    raise TypeError(f"unknown shape: {shape!r}")


# GOOD — class patterns with destructuring
def area(shape: Shape) -> float:
    match shape:
        case Circle(radius=r):
            return math.pi * r**2
        case Rectangle(width=w, height=h):
            return w * h
        case Triangle(base=b, height=h):
            return 0.5 * b * h
```

A pure value-to-value mapping is flatter still as a dict — no branching at all:

```python
# GOOD — dict dispatch when each branch is a constant or callable
HANDLERS: dict[EventType, Callable[[Event], str]] = {
    EventType.CREATED: lambda e: f"created at {e.timestamp}",
    EventType.UPDATED: lambda e: f"updated by {e.actor}",
    EventType.DELETED: lambda e: f"deleted reason: {e.reason}",
}

return HANDLERS.get(event.type, lambda _: "unknown")(event)
```

See `design-patterns` for dict-dispatch vs. `match` trade-offs and `modern-python` for `match` depth.

---

## Rule 4 — Extract the deep block

When the inner block is meaningful work, give it a name. The outer function goes back to flat; the extracted function starts fresh at level 0.

```python
# BAD — outer logic and inner work tangled at level 3
def process_batch(batches: list[Batch]) -> list[Result]:
    results: list[Result] = []
    for batch in batches:
        for item in batch.items:
            if item.is_ready:
                try:
                    normalized = item.payload.strip().lower()
                    score = score_for(normalized)
                    results.append(Result(item.id, normalized, score))
                except ValueError:
                    logger.warning("failed item %s", item.id, exc_info=True)
    return results


# GOOD — outer is iteration only; inner work is named
def process_batch(batches: list[Batch]) -> list[Result]:
    items = (item for batch in batches for item in batch.items if item.is_ready)
    return [r for item in items if (r := _process_item_safely(item)) is not None]


def _process_item_safely(item: Item) -> Result | None:
    try:
        normalized = item.payload.strip().lower()
        return Result(item.id, normalized, score_for(normalized))
    except ValueError:
        logger.warning("failed item %s", item.id, exc_info=True)
        return None
```

Note `logger.warning(..., exc_info=True)` instead of swallowing the trace — see `logging-observability`. The flattening here also separates "iterate and collect" from "process one item, handle its failure," which is the real win (`refactoring-python` calls this Extract Function).

---

## Rule 5 — `for: if: for:` is two functions

Mixed nesting (loop containing condition containing loop) is the strongest signal that two distinct concerns are interleaved. Separate them.

```python
# BAD — loop, conditional, loop — three concerns mashed together
for order in orders:
    if order.is_shippable:
        for line in order.lines:
            shipping_service.schedule(order, line)


# GOOD — outer iterates, inner is a named operation
for order in (o for o in orders if o.is_shippable):
    _schedule_shipping(order)


def _schedule_shipping(order: Order) -> None:
    for line in order.lines:
        shipping_service.schedule(order, line)
```

---

## Rule 6 — Inverted boolean conditions

When an `if` reads as "if (negative thing is false)", invert it. Double negatives break flow, and Python's `not` makes them easy to write by accident.

```python
# BAD
if not user.is_inactive:
    if user.email is not None:
        send(user.email)

# GOOD
if user.is_inactive:
    return
if user.email is None:
    return
send(user.email)
```

Prefer a positively-named predicate (`is_active`) over the negated form (`not is_inactive`) so the guard reads cleanly.

---

## Rule 7 — Don't over-nest comprehensions

Comprehensions flatten loops, but a comprehension with two `for` clauses and an `if` and a complex body just moves the arrow code onto one unreadable line. The 2-level rule applies here too.

```python
# BAD — a comprehension nobody can read
results = [
    transform(x, y)
    for group in groups
    for x in group.items
    for y in x.children
    if y.active and x.enabled and group.live
]

# GOOD — name the flattening, keep the body simple
def _live_children(groups: list[Group]) -> Iterator[tuple[Item, Child]]:
    for group in groups:
        if not group.live:
            continue
        for x in (i for i in group.items if i.enabled):
            yield from ((x, y) for y in x.children if y.active)


results = [transform(x, y) for x, y in _live_children(groups)]
```

A generator function reads top-to-bottom with guard clauses; a triple-`for` comprehension reads inside-out. See `readable-comprehensions` for the cutoff.

---

## When nesting is correct

Some structures are inherently nested and flattening would hurt:

- **Building a nested data structure** — `dict[str, dict[str, list[X]]]` aggregation. Use `collections.defaultdict` or `itertools.groupby` so the *code* is flat even though the *data* is nested.
- **Genuine nested iteration** — e.g. a 2D grid where every `(row, col)` pair really does need work. Keep the loop, but don't put a 4-line `if` body inside it; consider `itertools.product(rows, cols)` to collapse two `for`s into one.
- **Context managers that nest** — multiple `with` blocks are legitimate. Use a single parenthesized `with` (3.10+) so they read flat:

```python
with (
    open(src) as fin,
    open(dst, "w") as fout,
):
    fout.write(fin.read())
```

The rule isn't "no nesting" — it's "no *gratuitous* nesting." If the data shape forces it, fine. If the *code* shape is forcing it, refactor.

---

## Quick checklist (use during review)

- [ ] No function body has more than 2 levels of indentation.
- [ ] No `else` branch where the `if` body returns / raises / continues — invert into a guard.
- [ ] No nested `for` loops where the inner loop is just a search — index into a `dict`/`set` instead.
- [ ] No `if / elif` chain over a discriminator — use `match` or dict dispatch.
- [ ] No `isinstance` chain over a known type union — use `match` with class patterns.
- [ ] No mixed `for: if: for:` — extract the inner work as a function.
- [ ] No double-negative conditions (`not is_inactive`, `not is_invalid`).
- [ ] No comprehension with > 2 `for`/`if` clauses *and* a non-trivial body.
- [ ] If the data is genuinely nested, the *code* is still flat (defaultdict, helper generators).
- [ ] ruff `C901` / `PLR1702` pass without `# noqa`.

---

## Companion skills

- `efficient-python` — performance angle on nested loops (O(n²) → O(n) via `dict`/`set`).
- `readable-comprehensions` — when a flattening refactor pushes work into a comprehension or generator.
- `refactoring-python` — Extract Function / Replace Conditional with dispatch.
- `modern-python` — structural pattern matching, `match` over `isinstance`.
- `exception-handling` — choosing the right exception for a guard clause.
