---
name: django-configuration
description: Django configuration patterns — django-environ / pydantic-settings, per-environment settings modules, type-safe settings objects, feature flags, secrets management, and config validation. Use when externalizing config, reviewing for hardcoded values, or setting up environment-specific behavior.
---

# Django Configuration

Configuration-first mindset for Django 5.x on Python 3.12+ — nothing hardcoded in Python.

## When to Use

- Externalizing hardcoded values to settings or environment
- Setting up typed settings with `django-environ` or `pydantic-settings`
- Managing environment-specific behavior with per-environment settings modules
- Reviewing code for magic numbers, hardcoded URLs, or inline credentials
- Configuring feature flags or conditional behavior

---

## Quick Reference: What Must Be Externalized

| Category | Example | Where |
|----------|---------|-------|
| URLs, ports, hostnames | `https://api.example.com` | `.env` / settings |
| Timeouts, retry counts | `5.0` seconds, `max_retries=3` | `.env` / settings |
| Pool sizes, limits | `CONN_MAX_AGE=600` | `.env` / settings |
| Credentials, API keys | DB password, `SECRET_KEY` | Env vars / secrets manager |
| Feature flags | `ENABLE_NEW_CHECKOUT=true` | `.env` / settings |
| Batch/page sizes | `PAGE_SIZE=20` | settings |
| Cron / schedule expressions | `0 2 * * *` | `.env` / settings |

---

## Core Principle

If it can change between environments, it must be externalized. This is not optional.

```python
# BAD: Hardcoded values
import time

import httpx


class PaymentService:
    API_URL = "http://localhost:8080/api/payments"
    MAX_RETRIES = 3
    TIMEOUT_MS = 5000
    PAGE_SIZE = 20

    def process_payment(self, req: PaymentRequest) -> None:
        time.sleep(5)  # Magic number
        for _ in range(3):  # Magic number
            httpx.post(self.API_URL, json=req.as_dict())
```

```python
# GOOD: All externalized to a typed config object
import time

import httpx
from django.conf import settings


class PaymentService:
    def __init__(self, config: PaymentConfig | None = None) -> None:
        self._config = config or settings.PAYMENT

    def process_payment(self, req: PaymentRequest) -> None:
        time.sleep(self._config.retry_delay_seconds)
        for _ in range(self._config.max_retries):
            httpx.post(self._config.api_url, timeout=self._config.timeout_seconds)
```

---

## Typed Settings — django-environ

`django-environ` parses and casts environment variables with sane defaults. Declare the schema once at the top of your settings module.

```python
# BAD: os.environ scattered with no casting or defaults
import os

DEBUG = os.environ["DEBUG"]            # str "false" is truthy!
ALLOWED_HOSTS = os.environ["HOSTS"]    # a single string, not a list
DB_PORT = os.environ["DB_PORT"]        # str, breaks psycopg
TIMEOUT = os.environ.get("TIMEOUT")    # None if unset, crashes later
```

```python
# GOOD: settings/base.py — declared schema, typed, defaulted
from pathlib import Path

import environ

BASE_DIR = Path(__file__).resolve().parent.parent.parent

env = environ.Env(
    DEBUG=(bool, False),
    ALLOWED_HOSTS=(list, []),
    DB_CONN_MAX_AGE=(int, 600),
    API_TIMEOUT=(float, 30.0),
)
environ.Env.read_env(BASE_DIR / ".env")  # no-op if file is absent

SECRET_KEY = env("SECRET_KEY")           # required: raises if missing
DEBUG = env("DEBUG")                     # cast to bool
ALLOWED_HOSTS = env("ALLOWED_HOSTS")     # cast to list
DATABASES = {"default": env.db("DATABASE_URL")}  # parses a DSN
```

```dotenv
# .env (never committed — see .env.example for the template)
SECRET_KEY=django-insecure-change-me
DEBUG=true
ALLOWED_HOSTS=localhost,127.0.0.1
DATABASE_URL=postgres://app:secret@localhost:5432/app
API_TIMEOUT=30.0
```

---

## Typed Settings — pydantic-settings

For richer validation, group config into `BaseSettings` models. Prefer this when you want fail-fast parsing, nested config, and full type checking under mypy. See `modern-python` for dataclass/model idioms.

```python
# BAD: a grab-bag of module-level constants, no validation
MAIL_HOST = "smtp.example.com"
MAIL_PORT = 587
MAIL_FROM = "noreply@example.com"
MAIL_SSL = True
MAIL_TIMEOUT = 5
# Scattered, no grouping, no validation, silently wrong if mistyped
```

```python
# GOOD: settings/config.py — grouped, validated, type-checked
from pydantic import EmailStr, Field, HttpUrl
from pydantic_settings import BaseSettings, SettingsConfigDict


class MailSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="MAIL_")

    host: str
    port: int = Field(default=587, ge=1, le=65535)
    from_addr: EmailStr = Field(alias="MAIL_FROM")
    ssl: bool = True
    timeout_seconds: float = Field(default=5.0, gt=0)


class ApiSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="API_")

    base_url: HttpUrl
    timeout_seconds: float = Field(default=30.0, gt=0)
    max_retries: int = Field(default=3, ge=0, le=10)
    retry_delay_seconds: float = Field(default=1.0, ge=0)


# Instantiated once at import time — fails fast on bad config
MAIL = MailSettings()  # type: ignore[call-arg]  # values come from env
API = ApiSettings()    # type: ignore[call-arg]
```

### Nested Configuration

```python
from pydantic import BaseModel, Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class CacheConfig(BaseModel):
    ttl_seconds: int = Field(default=600, gt=0)
    max_size: int = Field(default=1000, gt=0)


class SecurityConfig(BaseModel):
    jwt_secret: str
    jwt_expiration_seconds: int = Field(default=3600, gt=0)


class AppSettings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="APP_",
        env_nested_delimiter="__",  # APP_CACHE__TTL_SECONDS=60
    )

    cache: CacheConfig = CacheConfig()
    security: SecurityConfig
```

```dotenv
APP_CACHE__TTL_SECONDS=600
APP_CACHE__MAX_SIZE=1000
APP_SECURITY__JWT_SECRET=${JWT_SECRET}
APP_SECURITY__JWT_EXPIRATION_SECONDS=3600
```

### Lists and Maps

```python
from pydantic import BaseModel
from pydantic_settings import BaseSettings, SettingsConfigDict


class ChannelConfig(BaseModel):
    enabled: bool = False
    endpoint: str
    timeout_seconds: float = 5.0


class NotificationSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="NOTIFY_")

    admin_emails: list[str] = []
    channels: dict[str, ChannelConfig] = {}
```

```dotenv
# JSON for complex values with pydantic-settings
NOTIFY_ADMIN_EMAILS=["admin@example.com", "ops@example.com"]
NOTIFY_CHANNELS={"slack": {"enabled": true, "endpoint": "https://hooks.slack.com/x"}}
```

---

## Per-Environment Settings Modules

Split `settings.py` into a package: a shared `base.py` plus one module per environment. Select with `DJANGO_SETTINGS_MODULE`.

```
config/
  settings/
    __init__.py
    base.py     # shared defaults
    dev.py      # local development
    prod.py     # production
    test.py     # fast, isolated test config
```

```python
# BAD: one settings.py with runtime environment branching
import os

if os.environ.get("ENV") == "prod":
    DEBUG = False
    ALLOWED_HOSTS = ["api.production.com"]
    CACHE_TTL = 3600
else:
    DEBUG = True
    ALLOWED_HOSTS = ["*"]
    CACHE_TTL = 60
# Fragile, untestable, easy to leak prod values into dev
```

```python
# GOOD: config/settings/dev.py
from .base import *  # noqa: F403

DEBUG = True
ALLOWED_HOSTS = ["localhost", "127.0.0.1"]
EMAIL_BACKEND = "django.core.mail.backends.console.EmailBackend"
```

```python
# config/settings/prod.py
from .base import *  # noqa: F403

DEBUG = False
ALLOWED_HOSTS = env("ALLOWED_HOSTS")  # noqa: F405  — required from env
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
```

```dotenv
# Select the module via env (do not hardcode in manage.py)
DJANGO_SETTINGS_MODULE=config.settings.prod
```

---

## Environment-Driven Behavior

```python
# BAD: runtime DEBUG / env-string checks scattered through code
class NotificationService:
    def notify(self, message: str) -> None:
        from django.conf import settings

        if not settings.DEBUG:        # fragile coupling to DEBUG
            self._send_real_email(message)
        else:
            logger.info("Would send: %s", message)
```

```python
# GOOD: choose the implementation via settings, inject it
from typing import Protocol


class Notifier(Protocol):
    def notify(self, message: str) -> None: ...


class EmailNotifier:
    def notify(self, message: str) -> None:
        self._send_real_email(message)


class LoggingNotifier:
    def notify(self, message: str) -> None:
        logger.info("Notification: %s", message)
```

```python
# config/settings/prod.py
NOTIFIER = "myapp.notifications.EmailNotifier"

# config/settings/dev.py
NOTIFIER = "myapp.notifications.LoggingNotifier"
```

```python
# Resolve once, at the wiring layer — see clean-architecture
from django.conf import settings
from django.utils.module_loading import import_string

notifier: Notifier = import_string(settings.NOTIFIER)()
```

---

## Feature Flags

### Settings-Based (Simple)

```python
# BAD: feature gated on a literal that needs a redeploy to change
class CheckoutService:
    def checkout(self, request: CheckoutRequest) -> CheckoutResult:
        if True:  # "we'll flip this when ready"
            return self._new_checkout(request)
        return self._legacy_checkout(request)
```

```python
# GOOD: gated on a typed, externalized flag
from dataclasses import dataclass

from django.conf import settings


@dataclass(frozen=True, slots=True)
class FeatureFlags:
    new_checkout_flow: bool = False
    beta_search: bool = False
    email_notifications: bool = True


class CheckoutService:
    def __init__(self, flags: FeatureFlags | None = None) -> None:
        self._flags = flags or settings.FEATURES

    def checkout(self, request: CheckoutRequest) -> CheckoutResult:
        if self._flags.new_checkout_flow:
            return self._new_checkout(request)
        return self._legacy_checkout(request)
```

```python
# config/settings/base.py
FEATURES = FeatureFlags(
    new_checkout_flow=env.bool("FEATURE_NEW_CHECKOUT", default=False),
    beta_search=env.bool("FEATURE_BETA_SEARCH", default=False),
)
```

### Strategy Selection (Cleaner for diverging code paths)

Rather than branching on a flag everywhere, select the implementation once. See `design-patterns` for the strategy pattern.

```python
from django.utils.module_loading import import_string

_CHECKOUT_IMPLS = {
    True: "myapp.checkout.NewCheckoutService",
    False: "myapp.checkout.LegacyCheckoutService",
}


def get_checkout_service() -> CheckoutService:
    return import_string(_CHECKOUT_IMPLS[settings.FEATURES.new_checkout_flow])()
```

---

## Secrets Management

```python
# BAD: secrets hardcoded in settings, committed to git
SECRET_KEY = "django-insecure-9x!hardcoded-key-visible-in-history"
DATABASES = {
    "default": {
        "PASSWORD": "my-super-secret-password",  # In source control!
    }
}
JWT_SECRET = "hardcoded-jwt-secret"  # Visible in every clone
```

```python
# GOOD: read secrets from the environment, never from source
SECRET_KEY = env("SECRET_KEY")            # raises if unset — fail fast
DATABASES = {"default": env.db("DATABASE_URL")}  # password lives in the DSN
JWT_SECRET = env("JWT_SECRET")
```

### Resolution Hierarchy (highest wins)

1. Process environment variables (`DJANGO_SECRET_KEY=...`)
2. `.env` file loaded by `read_env` (local dev only, git-ignored)
3. Per-environment settings module (`prod.py` overriding `base.py`)
4. Defaults declared in the `environ.Env(...)` / pydantic schema

### Cloud Secrets at Startup

```python
# config/settings/prod.py — pull secrets from a manager into env
import json

import boto3


def _load_aws_secrets(secret_id: str) -> None:
    client = boto3.client("secretsmanager")
    payload = json.loads(client.get_secret_value(SecretId=secret_id)["SecretString"])
    for key, value in payload.items():
        os.environ.setdefault(key, str(value))  # noqa: F405


_load_aws_secrets(env("AWS_SECRET_ID"))  # noqa: F405  — before reading secrets
SECRET_KEY = env("SECRET_KEY")  # noqa: F405
```

---

## Configuration Validation

### Fail-Fast at Import Time

`pydantic-settings` validates on instantiation — a bad value crashes startup, not a request. For cross-field rules use a validator. See `exception-handling` for raising clear, actionable errors.

```python
from pydantic import Field, field_validator, model_validator
from pydantic_settings import BaseSettings, SettingsConfigDict


class ApiSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="API_")

    base_url: str
    max_retries: int = Field(default=3, ge=1, le=10)
    timeout_seconds: float = Field(default=30.0, gt=0)

    @field_validator("base_url")
    @classmethod
    def _must_be_http(cls, value: str) -> str:
        if not value.startswith(("http://", "https://")):
            raise ValueError("base_url must start with http(s)")
        return value

    @model_validator(mode="after")
    def _retries_vs_timeout(self) -> "ApiSettings":
        if self.timeout_seconds < 0.1:
            raise ValueError("timeout_seconds must be at least 0.1")
        return self
```

### Django System Check

For sanity warnings that should not crash but should surface, register a Django check. It runs on `manage.py check` and in CI.

```python
# myapp/checks.py
from django.conf import settings
from django.core.checks import Warning, register


@register()
def config_sanity(app_configs, **kwargs) -> list[Warning]:
    errors: list[Warning] = []
    api = settings.API
    if api.max_retries > 5 and api.timeout_seconds < 5:
        errors.append(
            Warning(
                "High retry count with low timeout may cause cascading failures",
                hint="Lower API_MAX_RETRIES or raise API_TIMEOUT.",
                id="myapp.W001",
            )
        )
    return errors
```

---

## Defaults and Documentation

Commit a `.env.example` documenting every variable; never commit the real `.env`.

```dotenv
# .env.example — copy to .env and fill in
# Django secret key (generate with: python -c "import secrets; print(secrets.token_urlsafe(50))")
SECRET_KEY=
# Enable debug mode — MUST be false in production
DEBUG=false
# Comma-separated allowed hosts
ALLOWED_HOSTS=localhost,127.0.0.1

# Base URL of the external payment API
API_BASE_URL=https://api.default.com
# Maximum retry attempts for failed API calls (1-10)
API_MAX_RETRIES=3
# Timeout per API call, in seconds
API_TIMEOUT=30.0

# Feature flags
FEATURE_NEW_CHECKOUT=false
```

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Hardcoded URLs, timeouts, sizes | Can't change without redeploy | Externalize to env / typed settings |
| Magic numbers in code | Unclear intent, not configurable | Named config attributes |
| `os.environ[...]` sprawl | No casting, no defaults, no validation | `django-environ` / `pydantic-settings` |
| Secrets in `settings.py` | In source control forever | Env vars, secrets manager |
| `str` env value used as bool | `"false"` is truthy | Cast with `env.bool` / pydantic |
| No defaults | App crashes without explicit config | Declare defaults in the schema |
| No validation | Silent misconfiguration | pydantic validators / Django checks |
| `if settings.DEBUG:` for logic | Fragile environment coupling | Per-environment settings + injection |
| Config falls back silently | Wrong behavior unnoticed | Fail-fast: required keys raise |
| One giant settings module | Hard to navigate, env values leak | Settings package per environment |
| Inline cron / schedule strings | Hard to override per env | Externalize: `env("CLEANUP_CRON")` |

---

## Related Skills

- `django-security` — Externalizing security settings and secrets
- `logging-observability` — Log level and handler configuration
- `django-orm` — Database connection and pool configuration
- `modern-python` — Dataclasses, type hints, and model idioms for config
- `review-changes-python` — Hardcoded values review checklist
