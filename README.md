# claude-py — Python/Django Claude Code Skills

A library of [Claude Code **Skills**](https://docs.claude.com/en/docs/claude-code/skills) for writing and reviewing idiomatic, modern Python (3.12/3.13) and Django applications. Each skill is a focused, single-responsibility reference that Claude auto-invokes when relevant — covering language idioms, code quality, architecture, and the Django web stack.

Ported from and structured identically to [`claude-spring`](https://gitlab01.butterflycluster.com/staging/claude-spring) (its Java/Spring counterpart): frontmatter with explicit "Use when" triggers, BAD/GOOD code examples, severity tables, and cross-references between skills.

## Install

Copy the skills into a project (or your user scope):

```bash
cp -r .claude/skills/* <your-project>/.claude/skills/
# or for all projects:
cp -r .claude/skills/* ~/.claude/skills/
```

Claude Code discovers each `SKILL.md` via its frontmatter and invokes it automatically when the work matches the skill's "Use when" description. No slash command needed.

## Skills

### Language & code quality
- **modern-python** — Python 3.12/3.13 idioms: type params, `match`, dataclasses, pathlib, walrus
- **efficient-python** — right data structures, eliminating nested loops, generators, guard clauses
- **flatten-nesting** — kill arrow code: guard clauses, early returns, lookup maps, extract-method
- **readable-comprehensions** — comprehension/lambda readability; when to extract a named function
- **concise-comments** — comments justify *why*, never restate *what*
- **exception-handling** — exception hierarchies, narrow catches, error responses, logging strategy
- **logging-observability** — stdlib logging + structlog, JSON logs, correlation IDs, OpenTelemetry
- **python-concurrency** — asyncio, the GIL, threads vs processes, `concurrent.futures`, `TaskGroup`
- **leverage-libraries** — battle-tested libraries over hand-rolled code
- **design-patterns** — common patterns in idiomatic Python
- **solid-principles** — SOLID with Python examples
- **clean-architecture** — package-by-feature, hexagonal/ports-and-adapters, DDD, import-linter
- **refactoring-python** — size-based triggers, extract method/class, parameter objects, modernization
- **review-changes-python** — systematic Python/Django code-review workflow
- **testing-python** — pytest, pytest-django, fixtures, factory_boy, mocking, Testcontainers

### Django web stack
- **django-configuration** — settings per environment, django-environ / pydantic-settings, secrets, feature flags
- **django-orm** — models, QuerySets, avoiding N+1 (`select_related`/`prefetch_related`), migrations, transactions
- **django-rest-api** — DRF serializers, viewsets, routers, pagination, versioning, thin views
- **django-security** — auth backends, permissions, JWT/OAuth2, CSRF, CORS, password hashing, hardening

## Conventions

Standard tooling referenced throughout: **uv** (deps), **ruff** (lint + format), **mypy** (types), **pytest** (tests).

## Credits

Structure and approach ported from `claude-spring`. The original orchestration-style command prompts this repo started from were replaced wholesale by this skills library.
