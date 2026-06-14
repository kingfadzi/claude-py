---
name: readable-comprehensions
description: Comprehension, generator-expression, and lambda readability — when a comprehension or lambda is too cryptic, when to extract to a named function, single-letter variables, nested comprehensions, side effects in comprehensions, walrus abuse, and comprehension length limits. Use when reviewing or writing comprehensions, generator pipelines, lambda arguments to map/filter/sorted, or anywhere a comprehension or lambda is non-trivial.
---

# Readable Comprehensions

A comprehension exists to make code clearer than the imperative loop. When it stops doing that — when a reader has to pause and decode it — the comprehension is wrong. This skill lists the patterns that turn comprehensions and lambdas cryptic and the refactors that fix them.

## When to Use

- Reviewing list/dict/set comprehensions and generator expressions
- Reviewing `lambda` arguments passed to `map`, `filter`, `sorted`, `min`, `max`, `key=...`
- Writing or modifying any comprehension with more than one `for` or one `if`
- Replacing an imperative loop with a comprehension — checking if it's actually clearer
- Reading code where a `for` loop became "more Pythonic" via a comprehension but harder to follow

---

## Core principle

**A comprehension must be readable in one pass.** If a teammate reads it and has to retrace variables, count the nested `for` clauses, or mentally substitute values to understand the flow, refactor.

The four pressures to push back on:

1. **Density** — too much logic in one expression.
2. **Anonymity** — `lambda` or single-letter names with no anchor.
3. **Side effects** — doing work for effect inside a comprehension.
4. **Cleverness** — using comprehension plumbing where a `for` loop would be plainer.

---

## Rule 1 — Single-letter variables

Acceptable for a one-clause comprehension where the type is obvious from the source:

```python
# OK — iterating users, the variable is obviously a User
active = [u for u in users if u.is_active]

# OK — short and the source name carries the meaning
emails = [u.email for u in users]
```

Not acceptable when the comprehension has more than one clause, or when there's nesting:

```python
# BAD — what is `e`? what is the shape of e value?
labels = [
    f"{e[0]}={e[1][0].name}"
    for e in mapping.items()
    if e[1] and not e[0].startswith("_")
]

# GOOD — name the variables; unpack the tuple
labels = [
    first_value_label(key, values)
    for key, values in mapping.items()
    if is_active_bucket(key, values)
]
```

**Rule:** name the variable the moment the body needs more than one navigation step. Unpack tuples in the `for` clause (`for key, values in ...`) instead of indexing `e[0]` / `e[1]`.

---

## Rule 2 — Comprehensions with control flow or nesting

A comprehension that grows a conditional expression, a `try`, or a second `for` is a function that's been forced into one line. Extract.

```python
# BAD — branching + parsing crammed into the expression
results = [
    Result(i.id, normalize(i.payload), Status.OK)
    if i.type is Type.A
    else parse_with(parser, i)
    for i in inputs
]

# GOOD — extract; the comprehension becomes scannable
results = [to_result(i) for i in inputs]


def to_result(item: Input) -> Result:
    if item.type is Type.A:
        return Result(item.id, normalize(item.payload), Status.OK)
    try:
        return parse_with(parser, item)
    except ParseError:
        logger.warning("parse failed for %s", item.id, exc_info=True)
        return Result.failed(item.id)
```

You cannot put `try`/`except` or `match` inside a comprehension at all — that on its own is the signal to extract a named function.

---

## Rule 3 — Lambdas should clarify, not obscure

Use a `lambda` only when it's a trivial one-expression `key=` or callback. Reach for `operator.attrgetter` / `itemgetter` when they read like the operation, and a named function when the body is non-trivial.

```python
# GOOD — lambda reads like the action
risks_by_age = sorted(risks, key=lambda r: r.opened_at)

# BETTER — operator names the intent and skips the lambda
from operator import attrgetter

risks_by_age = sorted(risks, key=attrgetter("opened_at"))

# BAD — multi-step logic wedged into a lambda key
ranked = sorted(
    hits,
    key=lambda h: (h.score * weight(h.source)) - penalty(h.age_days),
)

# GOOD — name it; the sort line now states what it sorts by
def rank(hit: SearchHit) -> float:
    return hit.score * weight(hit.source) - penalty(hit.age_days)


ranked = sorted(hits, key=rank)
```

**Rule:** if the `lambda` forces the reader to compute, write a named function. Never bind a `lambda` to a name (`f = lambda x: ...`) — that is just a worse `def`; ruff flags it as `E731`.

---

## Rule 4 — Prefer comprehensions over map/filter with lambdas

`map`/`filter` taking a `lambda` is almost always a comprehension wearing a disguise, and the comprehension reads better.

```python
# BAD — map + filter + lambda noise
names = list(map(lambda u: u.name, filter(lambda u: u.is_active, users)))

# GOOD — one comprehension says it plainly
names = [u.name for u in users if u.is_active]
```

`map`/`filter` are fine with an existing named function or builtin (`map(str.strip, lines)`, `filter(None, values)`); they cost readability only when paired with a `lambda`.

---

## Rule 5 — No side effects in comprehensions

A comprehension is for building a collection. Doing work for its side effect — calling `print`, `append`, a repo write — and throwing the result list away is a `for` loop pretending not to be one.

```python
# BAD — comprehension built only for its side effects, result discarded
[repo.upsert(item) for item in items]

# GOOD — a for loop says what's happening
for item in items:
    repo.upsert(item)

# BAD — appending to an outside list from inside a comprehension
errors: list[str] = []
[errors.extend(validate(item).messages) for item in items if validate(item).has_errors]

# GOOD — build the result with the comprehension itself
errors = [
    message
    for item in items
    for message in validate(item).messages
]
```

**Rule:** if you discard the comprehension's result, or mutate an outside `list`/`dict` from inside it, you wanted a `for` loop.

---

## Rule 6 — Don't recompute inside a comprehension

Calling the same function twice — once in the `if`, once in the expression — is a readability *and* performance smell. Bind it once with a walrus, or extract.

```python
# BAD — validate(item) runs twice per item
errors = [
    validate(item).messages
    for item in items
    if validate(item).has_errors
]

# GOOD — walrus binds the result once, named clearly
errors = [
    result.messages
    for item in items
    if (result := validate(item)).has_errors
]

# ALSO GOOD — a generator helper when the walrus gets dense
def failed_validations(items: list[Item]) -> Iterator[Validation]:
    for item in items:
        result = validate(item)
        if result.has_errors:
            yield result


errors = [r.messages for r in failed_validations(items)]
```

**Rule:** the walrus (`:=`) earns its place by removing a duplicate call. If it makes the line harder to parse, extract a generator function instead.

---

## Rule 7 — Comprehension length and nesting

A comprehension wider than ~80 columns, or with more than one `for` clause plus one `if`, becomes a wall. Either:

1. Extract the predicate / transform to a named function.
2. Use a `for` loop if the operations are imperative anyway.

```python
# BAD — three filters and a transform, all inline and non-obvious
bodies = [
    event.payload.body.strip()
    for event in events
    if event.timestamp > cutoff
    if not event.is_internal
    if event.payload is not None
    if event.payload.body.strip()
]

# GOOD — name what the predicate and the transform do
def is_reportable_external(event: Event) -> bool:
    return event.timestamp > cutoff and not event.is_internal


def extract_body(event: Event) -> str:
    payload = event.payload
    return payload.body.strip() if payload else ""


bodies = [
    body
    for event in events
    if is_reportable_external(event)
    if (body := extract_body(event))
]
```

Nested comprehensions (`[x for row in grid for x in row]`) are fine one level deep for flattening; two levels of nesting with conditionals is a `for` loop.

---

## Rule 8 — Use generators for pipelines and large data

A list comprehension materializes the whole list. When you only iterate once, or the input is large or unbounded, a generator expression streams instead — and reads the same.

```python
# BAD — builds a full list just to sum it; doubles peak memory
total = sum([line_total(row) for row in huge_rows])

# GOOD — generator expression, no intermediate list
total = sum(line_total(row) for row in huge_rows)

# BAD — list comprehension over a file means loading every line first
matches = [parse(line) for line in open(path) if line.startswith("ERROR")]

# GOOD — stream lines through a context-managed generator pipeline
def error_records(path: Path) -> Iterator[Record]:
    with path.open() as handle:
        for line in handle:
            if line.startswith("ERROR"):
                yield parse(line)


for record in error_records(path):
    handle(record)
```

**Rule:** if the result feeds straight into `sum`/`any`/`all`/`min`/`max`/`join` or another loop, drop the brackets and use a generator expression. Reach for a named generator function when the pipeline needs a context manager or branching.

---

## Quick checklist (use during review)

- [ ] Every multi-clause comprehension uses named variables (no bare `e`/`x`/`i`).
- [ ] Tuples are unpacked in the `for` clause, not indexed (`for k, v in ...`).
- [ ] No comprehension nests more than one `for` plus one `if`.
- [ ] No conditional-expression branching that wants a `match` or `if/else` — extract.
- [ ] No `lambda` bound to a name; no `lambda` doing multi-step work.
- [ ] No `map`/`filter` with a `lambda` where a comprehension reads better.
- [ ] No comprehension built purely for side effects (result discarded).
- [ ] No outside `list`/`dict` mutated from inside a comprehension.
- [ ] No function called twice per element — bind with `:=` or extract.
- [ ] Generator expression (not list) when feeding `sum`/`any`/`all`/`join` or one-shot iteration.

---

## When `for` beats a comprehension

Comprehensions are great for transformations and filtering. They lose to a `for` loop when:

- The body has side effects (`repo.upsert`, `logger.info`, `audit.record`).
- A `try`/`except` or `match` is needed per element.
- Multiple collections must be built at once and a single comprehension would obscure intent.
- Early termination is needed and it's not a simple `next(...)` / `any(...)`.
- The reader needs to step through with a debugger.

The point of comprehensions is clarity, not brevity for its own sake. Pick the form that reads better.

---

## Related Skills

- **modern-python** — type hints, dataclasses, `match`, `X | None`, walrus usage in context.
- **flatten-nesting** — when extraction from a comprehension should also flatten the surrounding code.
- **efficient-python** — generator vs list trade-offs, avoiding redundant passes and intermediate allocations.
- **leverage-libraries** — `operator`, `itertools`, and `functools` instead of hand-rolled lambdas.
- **refactoring-python** — mechanics of extracting a named function or generator from a dense expression.
- **concise-comments** — naming the extracted function so it needs no comment.
- **review-changes-python** — apply the checklist above during code review.

See **efficient-python** for the memory and performance side of generators vs lists, and **modern-python** for where the walrus operator and structural pattern matching fit.
