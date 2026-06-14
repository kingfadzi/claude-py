---
name: django-rest-api
description: Django REST Framework API design — thin views / service layer, serializers with validation, generic views/viewsets, routers, pagination, versioning, HTTP method/status conventions, and error handling. Use when building REST endpoints, reviewing API design, or structuring view/serializer/service layers.
---

# Django REST API

Conventions and patterns for building well-designed REST APIs with Django REST Framework (DRF) on Django 5.x / Python 3.12+.

## When to Use

- Building new REST endpoints
- Reviewing API design for consistency
- Structuring view/serializer/service layers
- Adding pagination, validation, or error handling to endpoints

---

## Quick Reference: HTTP Methods and Status Codes

| Method | Purpose | Success Code | Request Body | Idempotent |
|--------|---------|-------------|-------------|------------|
| `GET` | Read resource(s) | 200 OK | No | Yes |
| `POST` | Create resource | 201 Created | Yes | No |
| `PUT` | Full replace | 200 OK | Yes | Yes |
| `PATCH` | Partial update | 200 OK | Yes | No* |
| `DELETE` | Remove resource | 204 No Content | No | Yes |

### Common Error Codes

| Code | When |
|------|------|
| 400 Bad Request | Validation failure, malformed request |
| 401 Unauthorized | Missing or invalid authentication |
| 403 Forbidden | Authenticated but not authorized |
| 404 Not Found | Resource doesn't exist |
| 409 Conflict | Duplicate, state conflict |
| 422 Unprocessable Entity | Valid syntax but semantic error |
| 500 Internal Server Error | Unexpected server error |

Use `rest_framework.status` constants (`status.HTTP_201_CREATED`) instead of bare integers.

---

## View Structure — Thin Views

Views should only: accept input, delegate to a serializer/service, return a `Response`.

```python
# BAD: Business logic in the view
from decimal import Decimal
from rest_framework.decorators import api_view
from rest_framework.response import Response


@api_view(["POST"])
def create_order(request):
    body = request.data
    customer_id = body.get("customerId")
    items = body.get("items")

    if not customer_id or not items:
        return Response({"error": "Invalid order"}, status=400)

    total = Decimal("0")
    for item in items:
        qty = item["quantity"]
        price = Decimal(str(item["price"]))
        total += price * qty

    tax = total * Decimal("0.1")
    order = Order.objects.create(customer_id=customer_id, total=total + tax)
    return Response({"id": order.id, "total": str(order.total)})
```

```python
# GOOD: Thin viewset, typed serializers, service does the work
from rest_framework import status, viewsets
from rest_framework.response import Response

from orders.serializers import CreateOrderSerializer, OrderSerializer
from orders.services import OrderService


class OrderViewSet(viewsets.ViewSet):
    def __init__(self, *args: object, **kwargs: object) -> None:
        super().__init__(*args, **kwargs)
        self.service = OrderService()

    def create(self, request) -> Response:
        serializer = CreateOrderSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        order = self.service.create_order(serializer.validated_data)
        body = OrderSerializer(order).data
        headers = {"Location": f"/api/v1/orders/{order.id}"}
        return Response(body, status=status.HTTP_201_CREATED, headers=headers)

    def retrieve(self, request, pk: str) -> Response:
        order = self.service.get_order(int(pk))
        return Response(OrderSerializer(order).data)

    def destroy(self, request, pk: str) -> Response:
        self.service.delete_order(int(pk))
        return Response(status=status.HTTP_204_NO_CONTENT)
```

For standard CRUD, prefer `ModelViewSet` + a service layer for non-trivial logic. See `clean-architecture` and `solid-principles` for layering.

---

## Generic Views and ViewSets

```python
# BAD: Re-implementing CRUD by hand in APIView
class UserList(APIView):
    def get(self, request):
        users = User.objects.all()
        return Response([{"id": u.id, "name": u.name} for u in users])

    def post(self, request):
        user = User.objects.create(**request.data)  # no validation!
        return Response({"id": user.id}, status=201)
```

```python
# GOOD: Lean on generics — they handle status codes, pagination, 404s
from rest_framework import generics

from users.models import User
from users.serializers import UserSerializer


class UserListCreateView(generics.ListCreateAPIView):
    queryset = User.objects.all().order_by("name")
    serializer_class = UserSerializer


class UserDetailView(generics.RetrieveUpdateDestroyAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    lookup_field = "pk"
```

| Need | Generic |
|------|---------|
| List + create | `ListCreateAPIView` |
| Retrieve + update + delete | `RetrieveUpdateDestroyAPIView` |
| Full CRUD, one class | `ModelViewSet` |
| Read-only CRUD | `ReadOnlyModelViewSet` |

---

## Routers — Wiring ViewSets to URLs

```python
# BAD: Hand-writing every URL for a viewset
urlpatterns = [
    path("orders/", OrderViewSet.as_view({"get": "list", "post": "create"})),
    path("orders/<int:pk>/", OrderViewSet.as_view({"get": "retrieve"})),
    # ...easy to forget a method or mistype a path
]
```

```python
# GOOD: DefaultRouter generates consistent, named routes
from rest_framework.routers import DefaultRouter

from orders.views import OrderViewSet
from users.views import UserViewSet

router = DefaultRouter()
router.register("orders", OrderViewSet, basename="order")
router.register("users", UserViewSet, basename="user")

urlpatterns = router.urls
# Generates: /orders/, /orders/{pk}/, /users/, /users/{pk}/ with named routes
```

---

## Request/Response Serializers

Serializers are DRF's DTO + validation layer. Keep input and output shapes separate.

```python
# BAD: Using ModelSerializer with "__all__" for both in and out
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = "__all__"
# Problems: exposes internal fields, client can set id/created_at,
# no control over what's writable vs read-only
```

```python
# GOOD: Explicit fields, read-only outputs, dedicated input serializer
from rest_framework import serializers


class UserSerializer(serializers.ModelSerializer):
    """Response shape — what clients read."""

    class Meta:
        model = User
        fields = ("id", "name", "email", "age", "created_at")
        read_only_fields = ("id", "created_at")


class CreateUserSerializer(serializers.Serializer):
    """Request shape — what clients may write."""

    name = serializers.CharField(max_length=120)
    email = serializers.EmailField()
    age = serializers.IntegerField(min_value=0, max_value=150)


class UpdateUserSerializer(serializers.Serializer):
    """Partial update — all fields optional."""

    email = serializers.EmailField(required=False)
    age = serializers.IntegerField(min_value=0, max_value=150, required=False)
```

When passing data into the service layer, prefer a typed structure over a bare
dict. A frozen `dataclass` documents the contract and gives mypy something to
check:

```python
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class NewUser:
    name: str
    email: str
    age: int
```

### Why Separate Serializers Matter

| Concern | `fields="__all__"` | Explicit serializers |
|---------|--------------------|----------------------|
| Internal fields (id, audit cols) | Exposed | Controlled via `read_only_fields` |
| Validation | Implicit, coupled to model | Clean, per-operation |
| API evolution | Breaking changes | Independent versioning |
| Serialization control | Leaks structure | Intentional shape |

---

## Validation

### Field, Object, and Custom Validators

```python
# BAD: Validating inside the view after the fact
@api_view(["POST"])
def create_contact(request):
    phone = request.data.get("phone", "")
    if not re.match(r"^\+?[1-9]\d{7,14}$", phone):
        return Response({"error": "bad phone"}, status=400)
    ...
```

```python
# GOOD: Validation lives in the serializer; the view just raises
import re

from rest_framework import serializers

PHONE_RE = re.compile(r"^\+?[1-9]\d{7,14}$")


def validate_phone_number(value: str) -> str:
    if not PHONE_RE.match(value):
        raise serializers.ValidationError("Invalid phone number.")
    return value


class CreateContactSerializer(serializers.Serializer):
    name = serializers.CharField(max_length=120)
    phone = serializers.CharField(validators=[validate_phone_number])
    start_date = serializers.DateField()
    end_date = serializers.DateField()

    def validate_name(self, value: str) -> str:
        # Field-level: hook named validate_<field>
        return value.strip()

    def validate(self, attrs: dict[str, object]) -> dict[str, object]:
        # Object-level: cross-field checks
        if attrs["end_date"] < attrs["start_date"]:
            raise serializers.ValidationError(
                {"end_date": "Must be on or after start_date."}
            )
        return attrs
```

In the view, call `serializer.is_valid(raise_exception=True)` — DRF turns the
`ValidationError` into a 400 with a structured body automatically. See
`exception-handling` for centralizing other error responses.

---

## Pagination

```python
# BAD: Return the entire collection
@api_view(["GET"])
def list_users(request):
    users = User.objects.all()
    return Response(UserSerializer(users, many=True).data)
    # Loads ALL users into memory — breaks on large datasets
```

```python
# GOOD: Global default pagination in settings
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS": (
        "rest_framework.pagination.PageNumberPagination"
    ),
    "PAGE_SIZE": 20,
}

# Generic views and viewsets paginate automatically.
class UserListView(generics.ListAPIView):
    queryset = User.objects.all().order_by("name")
    serializer_class = UserSerializer

# Client: GET /api/v1/users/?page=2
```

Always `order_by(...)` a paginated queryset — unordered pagination yields
unstable results across pages.

### Custom Pagination (per-view tuning)

```python
from rest_framework.pagination import PageNumberPagination


class StandardResultsPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = "size"  # client may request ?size=50
    max_page_size = 100


class OrderListView(generics.ListAPIView):
    queryset = Order.objects.all().order_by("-created_at")
    serializer_class = OrderSerializer
    pagination_class = StandardResultsPagination
```

For large tables, prefer `CursorPagination` — it is O(1) regardless of offset
and stable under inserts. See `django-orm` for keyset/cursor query patterns.

---

## Versioning

Prefer URL path versioning — most explicit and cacheable.

```python
# GOOD: URL path versioning (recommended)
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_VERSIONING_CLASS": (
        "rest_framework.versioning.URLPathVersioning"
    ),
    "DEFAULT_VERSION": "v1",
    "ALLOWED_VERSIONS": ["v1", "v2"],
}

# urls.py
urlpatterns = [
    path("api/<version>/", include("orders.urls")),
]


# In a view, branch on request.version
class OrderViewSet(viewsets.ModelViewSet):
    def get_serializer_class(self):
        match self.request.version:
            case "v2":
                return OrderSerializerV2
            case _:
                return OrderSerializer
```

| Strategy | Class | Pros | Cons |
|----------|-------|------|------|
| URL path | `URLPathVersioning` | Simple, cacheable | URL changes |
| Header | `AcceptHeaderVersioning` | Clean URLs | Hidden, harder to test |
| Query param | `QueryParameterVersioning` | Easy to try | Pollutes query string |

---

## URL Naming Conventions

```
# Resources are nouns, plural, trailing slash (Django convention)
GET    /api/v1/users/           # List
POST   /api/v1/users/           # Create
GET    /api/v1/users/{id}/      # Get one
PUT    /api/v1/users/{id}/      # Replace
PATCH  /api/v1/users/{id}/      # Partial update
DELETE /api/v1/users/{id}/      # Delete

# Sub-resources
GET    /api/v1/users/{id}/orders/
POST   /api/v1/users/{id}/orders/

# Actions (when CRUD doesn't fit) — use @action on a viewset
POST   /api/v1/orders/{id}/cancel/
POST   /api/v1/users/{id}/activate/

# Filtering/searching (query params, not paths)
GET    /api/v1/orders/?status=PENDING&customer_id=123
GET    /api/v1/products/?search=laptop&min_price=500
```

```python
# Custom actions via @action — router wires the URL automatically
from rest_framework.decorators import action
from rest_framework.response import Response


class OrderViewSet(viewsets.ModelViewSet):
    queryset = Order.objects.all()
    serializer_class = OrderSerializer

    @action(detail=True, methods=["post"])
    def cancel(self, request, pk: str | None = None) -> Response:
        order = self.get_object()
        OrderService().cancel(order)
        return Response(OrderSerializer(order).data)
```

---

## Error Handling Integration

Let the service raise domain exceptions; map them to DRF exceptions (or a custom
handler) instead of try/except in every view.

```python
# Service raises domain exceptions
from rest_framework.exceptions import NotFound

from orders.models import Order


class OrderService:
    def get_order(self, order_id: int) -> Order:
        try:
            return Order.objects.get(pk=order_id)
        except Order.DoesNotExist as exc:
            raise NotFound(f"Order {order_id} not found.") from exc


# View stays clean — no try/except, DRF renders the 404 body
class OrderDetailView(generics.RetrieveAPIView):
    serializer_class = OrderSerializer

    def get_object(self) -> Order:
        return OrderService().get_order(self.kwargs["pk"])
```

For app-specific exceptions, register a `EXCEPTION_HANDLER` in settings to shape
every error response consistently. See `exception-handling` for the full
pattern.

---

## Response Patterns

```python
from rest_framework import status
from rest_framework.response import Response

# 201 Created with Location header
def create(self, request) -> Response:
    serializer = CreateUserSerializer(data=request.data)
    serializer.is_valid(raise_exception=True)
    user = self.service.create_user(serializer.validated_data)
    body = UserSerializer(user).data
    return Response(
        body,
        status=status.HTTP_201_CREATED,
        headers={"Location": f"/api/v1/users/{user.id}/"},
    )


# 204 No Content for delete
def destroy(self, request, pk: str) -> Response:
    self.service.delete_user(int(pk))
    return Response(status=status.HTTP_204_NO_CONTENT)


# 200 with body for updates (partial=True for PATCH)
def partial_update(self, request, pk: str) -> Response:
    serializer = UpdateUserSerializer(data=request.data, partial=True)
    serializer.is_valid(raise_exception=True)
    user = self.service.update_user(int(pk), serializer.validated_data)
    return Response(UserSerializer(user).data)
```

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Business logic in the view | Hard to test, duplicated | Move to a service module |
| `fields = "__all__"` | Leaks internals, no control | List fields explicitly |
| Model instance as response | Bypasses serializer shaping | Serialize through DRF |
| No `is_valid(raise_exception=True)` | Invalid data reaches service | Always validate first |
| `Model.objects.all()` unpaginated | OOM on large data | Set `DEFAULT_PAGINATION_CLASS` |
| Reading `request.data` ad hoc | No type safety, no validation | Use a serializer |
| Bare integer status codes | Magic numbers, unclear intent | Use `status.HTTP_*` |
| Hand-written viewset URLs | Inconsistent, error-prone | Use a `DefaultRouter` |
| Returning 200 for not-found | Lying about the contract | Raise `NotFound` -> 404 |
| Unordered paginated queryset | Unstable pages | `order_by(...)` always |

---

## Tooling

- `uv` — manage deps (`uv add djangorestframework`), run tasks (`uv run pytest`)
- `ruff` — lint + format the API package
- `mypy` with `django-stubs` + `djangorestframework-stubs` — type-check views and serializers
- `pytest` + `pytest-django` and DRF's `APIClient` — test endpoints. See `testing-python`

---

## Related Skills

- `exception-handling` — custom `EXCEPTION_HANDLER` and structured error bodies
- `django-orm` — queryset patterns, cursor pagination, avoiding N+1
- `django-security` — securing endpoints, authentication, permissions, throttling
- `django-configuration` — `REST_FRAMEWORK` settings, environment-driven config
- `review-changes-python` — API design review checklist
- `modern-python` — dataclasses as DTOs, pattern matching, `X | None` typing
