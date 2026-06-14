---
name: django-security
description: Django + DRF security patterns — authentication backends, DRF permission classes, JWT via djangorestframework-simplejwt, OAuth2 (django-oauth-toolkit), CSRF, CORS (django-cors-headers), Argon2 password hashing, and production settings hardening. Use when securing endpoints, implementing authentication/authorization, or reviewing security configuration.
---

# Django Security

Security patterns for Django 5.x with Django REST Framework 3.15+.

## When to Use

- Securing DRF API endpoints
- Implementing JWT or OAuth2 authentication
- Configuring object- and view-level authorization
- Setting up CORS or CSRF policies
- Hardening `settings.py` for production
- Reviewing security configuration

---

## Quick Reference

| Need | Approach |
|------|----------|
| Auth strategy | `REST_FRAMEWORK["DEFAULT_AUTHENTICATION_CLASSES"]` |
| JWT tokens | `djangorestframework-simplejwt` |
| Endpoint authorization | DRF permission classes (`IsAuthenticated`, custom) |
| Object-level authorization | `has_object_permission` |
| Password storage | Argon2 (`PASSWORD_HASHERS`) |
| CORS | `django-cors-headers` (centralized in settings) |
| CSRF | Session auth: enabled; stateless JWT: not used |
| External IdP | `django-oauth-toolkit` (OAuth2 / OIDC) |

---

## Authentication & Permission Defaults

Configure globally; override per-view only when needed.

```python
# BAD: AllowAny implied by leaving DEFAULT_PERMISSION_CLASSES unset,
# then sprinkling auth on each view (easy to forget one -> open endpoint).
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.SessionAuthentication",
    ],
    # No DEFAULT_PERMISSION_CLASSES -> defaults to AllowAny everywhere!
}
```

```python
# GOOD: Secure-by-default. Authenticated unless a view explicitly opts out.
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
        "rest_framework.authentication.SessionAuthentication",  # admin/browsable API
    ],
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.ScopedRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {
        "auth": "5/min",  # brute-force protection on login/refresh
    },
}
```

Opt a single view out of the default explicitly:

```python
class HealthView(APIView):
    authentication_classes: list = []
    permission_classes = [AllowAny]

    def get(self, request) -> Response:
        return Response({"status": "ok"})
```

---

## JWT Authentication (simplejwt)

### Settings

```python
# settings.py
from datetime import timedelta

SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=15),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=7),
    "ROTATE_REFRESH_TOKENS": True,
    "BLACKLIST_AFTER_ROTATION": True,
    "ALGORITHM": "HS256",
    "SIGNING_KEY": env("JWT_SIGNING_KEY"),  # from env, never hardcoded
    "AUTH_HEADER_TYPES": ("Bearer",),
}

INSTALLED_APPS = [
    # ...
    "rest_framework_simplejwt.token_blacklist",
]
```

### Token endpoints

```python
# urls.py
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
)

urlpatterns = [
    path("api/v1/auth/token/", TokenObtainPairView.as_view(), name="token_obtain"),
    path("api/v1/auth/token/refresh/", TokenRefreshView.as_view(), name="token_refresh"),
]
```

### Custom claims

Add roles/tenant info to the token so the resource server avoids extra DB hits.

```python
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer


class RolesTokenSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user) -> "Token":
        token = super().get_token(user)
        token["roles"] = list(user.groups.values_list("name", flat=True))
        token["tenant_id"] = user.tenant_id
        return token


class RolesTokenView(TokenObtainPairView):
    serializer_class = RolesTokenSerializer
```

Throttle the auth endpoints by setting `throttle_scope = "auth"` on these views.

---

## OAuth2 / External Identity Provider

Use `django-oauth-toolkit` when an external IdP (Keycloak, Auth0, Okta) issues tokens,
or when you act as the authorization server.

```python
# settings.py
INSTALLED_APPS = ["oauth2_provider", *INSTALLED_APPS]

REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "oauth2_provider.contrib.rest_framework.OAuth2Authentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
}

OAUTH2_PROVIDER = {
    "SCOPES": {"read": "Read scope", "write": "Write scope"},
    "ACCESS_TOKEN_EXPIRE_SECONDS": 3600,
    "PKCE_REQUIRED": True,  # required for public/SPA clients
}
```

```python
# Enforce scopes per view, like Spring's @PreAuthorize on scope.
from oauth2_provider.contrib.rest_framework import TokenHasScope
from rest_framework.viewsets import ModelViewSet


class OrderViewSet(ModelViewSet):
    permission_classes = [TokenHasScope]
    required_scopes = ["write"]
```

---

## Permission Classes (view + object level)

DRF runs view-level `has_permission` first, then `has_object_permission` for
detail actions. Both must pass.

```python
# BAD: authorization check buried in the view, easy to bypass on other actions.
class OrderViewSet(ModelViewSet):
    def get_object(self):
        obj = super().get_object()
        if obj.customer_id != self.request.user.id:
            raise PermissionDenied  # only protects get_object()
        return obj
```

```python
# GOOD: reusable permission class, enforced consistently across actions.
from rest_framework.permissions import BasePermission
from rest_framework.request import Request


class IsOwnerOrAdmin(BasePermission):
    """Owners read/write their own objects; staff can do anything."""

    def has_object_permission(self, request: Request, view, obj) -> bool:
        return request.user.is_staff or obj.customer_id == request.user.id


class IsAdminRole(BasePermission):
    def has_permission(self, request: Request, view) -> bool:
        return request.user.groups.filter(name="ADMIN").exists()


class OrderViewSet(ModelViewSet):
    permission_classes = [IsAuthenticated, IsOwnerOrAdmin]

    def get_permissions(self) -> list[BasePermission]:
        # Match-style action-based authorization.
        match self.action:
            case "destroy":
                return [IsAuthenticated(), IsAdminRole()]
            case _:
                return super().get_permissions()
```

### Always scope querysets too

Object permissions guard single objects; querysets guard *list* endpoints.

```python
def get_queryset(self):
    qs = Order.objects.all()
    if self.request.user.is_staff:
        return qs
    return qs.filter(customer_id=self.request.user.id)
```

See `django-rest-api` for serializer/viewset structure and `django-orm` for query scoping.

---

## Authentication Backends

Custom backends extend *how* a user is authenticated (e.g. email login, SSO).

```python
# BAD: comparing a raw password string -> timing attacks + plaintext assumptions.
class EmailBackend:
    def authenticate(self, request, username=None, password=None):
        user = User.objects.filter(email=username).first()
        if user and user.password == password:  # WRONG: no hashing
            return user
        return None
```

```python
# GOOD: use check_password (constant-time, hash-aware) and a constant-time
# dummy hash to avoid leaking which emails exist.
from django.contrib.auth import get_user_model
from django.contrib.auth.backends import BaseBackend
from django.contrib.auth.hashers import check_password

User = get_user_model()


class EmailBackend(BaseBackend):
    def authenticate(
        self,
        request,
        username: str | None = None,
        password: str | None = None,
        **kwargs,
    ) -> "User | None":
        if username is None or password is None:
            return None
        try:
            user = User.objects.get(email__iexact=username)
        except User.DoesNotExist:
            User().set_password(password)  # equalize timing for unknown emails
            return None
        if user.check_password(password) and user.is_active:
            return user
        return None

    def get_user(self, user_id: int) -> "User | None":
        return User.objects.filter(pk=user_id).first()
```

```python
# settings.py — order matters; first match wins.
AUTHENTICATION_BACKENDS = [
    "myapp.auth.EmailBackend",
    "django.contrib.auth.backends.ModelBackend",
]
```

---

## CORS Configuration

```python
# BAD: wildcard origin with credentials -> any site can call your API as the user.
CORS_ALLOW_ALL_ORIGINS = True
CORS_ALLOW_CREDENTIALS = True  # invalid + dangerous combination
```

```python
# GOOD: centralized allowlist via django-cors-headers.
# Middleware must sit high, above CommonMiddleware.
MIDDLEWARE = [
    "corsheaders.middleware.CorsMiddleware",
    "django.middleware.common.CommonMiddleware",
    # ...
]

CORS_ALLOWED_ORIGINS = [
    "https://app.example.com",
    *(["http://localhost:3000"] if DEBUG else []),  # dev only
]
CORS_ALLOW_CREDENTIALS = True
CORS_ALLOW_METHODS = ["GET", "POST", "PUT", "PATCH", "DELETE"]
CORS_ALLOW_HEADERS = ["authorization", "content-type"]
CORS_EXPOSE_HEADERS = ["X-Total-Count"]
CORS_PREFLIGHT_MAX_AGE = 3600
```

---

## CSRF

| App Type | CSRF | Why |
|----------|------|-----|
| Stateless API (Bearer JWT) | Not applicable | No cookies sent => no CSRF vector |
| Session-cookie auth (SPA/browser) | Enabled (required) | Cookies are auto-sent by the browser |
| Mixed | Per-auth-class | `SessionAuthentication` enforces it; `JWTAuthentication` skips it |

DRF's `SessionAuthentication` enforces CSRF automatically. For a cookie-auth SPA,
expose the token and require it:

```python
# settings.py
CSRF_COOKIE_HTTPONLY = False  # SPA JS must read it to set X-CSRFToken
CSRF_COOKIE_SECURE = True
CSRF_TRUSTED_ORIGINS = ["https://app.example.com"]
CSRF_COOKIE_SAMESITE = "Lax"
```

Never disable CSRF globally with `@csrf_exempt` to "make it work" — that removes the
protection for session-based clients. If the endpoint is truly stateless, use
JWT/Bearer auth instead, which does not rely on cookies.

---

## Password Hashing (Argon2)

Django's default is PBKDF2; Argon2 is stronger and recommended (`uv add argon2-cffi`).

```python
# BAD: storing or "encoding" passwords without a real hash.
user.password = raw_password                 # plaintext!
user.password = base64.b64encode(raw)        # encoding, NOT hashing!
```

```python
# GOOD: put Argon2 first; the rest stay for verifying legacy hashes,
# which Django auto-upgrades on next successful login.
# settings.py
PASSWORD_HASHERS = [
    "django.contrib.auth.hashers.Argon2PasswordHasher",
    "django.contrib.auth.hashers.PBKDF2PasswordHasher",
    "django.contrib.auth.hashers.BCryptSHA256PasswordHasher",
]

AUTH_PASSWORD_VALIDATORS = [
    {"NAME": "django.contrib.auth.password_validation.MinimumLengthValidator",
     "OPTIONS": {"min_length": 12}},
    {"NAME": "django.contrib.auth.password_validation.CommonPasswordValidator"},
]
```

```python
# Registration — always go through set_password (applies the hasher chain).
def register_user(data: "RegistrationRequest") -> User:
    user = User(username=data.username, email=data.email)
    user.set_password(data.password)  # hashed with Argon2
    user.full_clean()                 # runs password + field validators
    user.save()
    return user
```

---

## Settings Hardening (production)

```python
# Driven by environment, never committed.
DEBUG = env.bool("DJANGO_DEBUG", default=False)
SECRET_KEY = env("DJANGO_SECRET_KEY")            # rotate-able, from secrets manager
ALLOWED_HOSTS = env.list("DJANGO_ALLOWED_HOSTS")

# HTTPS / transport
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31_536_000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")  # behind a proxy

# Cookies
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True

# Misc headers
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = "DENY"
```

Run `python manage.py check --deploy` in CI to catch missing flags.
See `django-configuration` for the env-driven settings layout.

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Unset `DEFAULT_PERMISSION_CLASSES` | Defaults to `AllowAny` — open API | Set `IsAuthenticated` default |
| Per-view auth checks only | Inconsistent; missed on some actions | Reusable `BasePermission` classes |
| Plaintext / base64 passwords | Trivially compromised | Argon2 via `set_password` |
| `SIGNING_KEY` / `SECRET_KEY` in source | Secret in version control | Env var / secrets manager |
| `CORS_ALLOW_ALL_ORIGINS = True` + credentials | Any site acts as the user | Explicit `CORS_ALLOWED_ORIGINS` |
| `@csrf_exempt` on session endpoints | CSRF vulnerability | Keep CSRF; use JWT if truly stateless |
| Long-lived access tokens | Stolen token usable for hours | Short access + refresh rotation + blacklist |
| List endpoint not queryset-scoped | Object perms miss list leakage | Filter `get_queryset` by owner |
| No throttle on auth routes | Brute-force / credential stuffing | `ScopedRateThrottle` on token views |
| `DEBUG = True` in prod | Leaks settings, stack traces | Env-driven `DEBUG`, `check --deploy` |

---

## Related Skills

- `django-rest-api` — Securing viewsets, serializers, and API endpoints
- `django-configuration` — Env-driven secrets and settings hardening
- `django-orm` — Scoping querysets for object-level access control
- `exception-handling` — Handling `PermissionDenied` and `AuthenticationFailed`
- `testing-python` — Testing permission classes and auth flows with pytest
