---
name: testing-python
description: Testing patterns for Python/Django — pytest, pytest-django, fixtures, factory_boy, parametrize, mocking (unittest.mock / pytest-mock), Testcontainers/pytest-docker, and responses/httpx mocking. Use when writing tests, reviewing test code, or choosing test strategies.
---

# Testing Python/Django

Patterns and best practices for testing Django applications with pytest, pytest-django, factory_boy, and Testcontainers.

## When to Use

- Writing new tests for features or bug fixes
- Reviewing test code for correctness and maintainability
- Choosing the right test strategy (unit vs integration vs db-backed)
- Setting up Testcontainers for integration tests

---

## Quick Reference: Test Types

| Type | Marker/Tool | Speed | Scope | When |
|------|-------------|-------|-------|------|
| Unit test | none | Fast | Single function/class | Business logic, utilities |
| API test | `APIClient` (DRF) | Fast | View + serializer | Request/response mapping |
| DB test | `@pytest.mark.django_db` | Medium | Model + queryset | Query correctness |
| Serializer test | direct instantiation | Fast | (De)serialization | DTO mapping |
| Integration | Testcontainers | Slow | Full stack + real DB | End-to-end workflows |

---

## pytest Essentials

### Classes — Organize by Behavior

```python
# BAD: Flat module with unclear organization
def test_create_order(): ...
def test_create_order_with_invalid_customer(): ...
def test_create_order_with_empty_items(): ...
def test_cancel_order(): ...
def test_cancel_already_cancelled_order(): ...

# GOOD: Group by behavior with classes (no inheritance needed)
class TestCreateOrder:
    def test_creates_order_with_valid_request(self): ...

    def test_raises_when_customer_not_found(self): ...

    def test_raises_when_items_empty(self): ...


class TestCancelOrder:
    def test_cancels_active_order(self): ...

    def test_raises_when_already_cancelled(self): ...
```

### parametrize — Data-Driven Tests

```python
# BAD: Duplicate test functions for the same logic
def test_valid_email_1(): assert is_valid("user@example.com")
def test_valid_email_2(): assert is_valid("a@b.co")
def test_invalid_email_1(): assert not is_valid("not-email")
def test_invalid_email_2(): assert not is_valid("")

# GOOD: parametrized test
import pytest


@pytest.mark.parametrize(
    "email",
    ["user@example.com", "a@b.co", "test+tag@domain.org"],
)
def test_accepts_valid_emails(email: str) -> None:
    assert validator.is_valid(email)


@pytest.mark.parametrize(
    "email",
    ["not-email", "", "  ", "@no-local.com", "no-domain@", None],
)
def test_rejects_invalid_emails(email: str | None) -> None:
    assert not validator.is_valid(email)


# Multiple arguments — readable with ids
@pytest.mark.parametrize(
    ("total", "quantity", "expected"),
    [
        (Decimal("100"), 10, Decimal("10.00")),
        (Decimal("200"), 20, Decimal("10.00")),
        (Decimal("50"), 5, Decimal("10.00")),
        (Decimal("0"), 1, Decimal("0.00")),
    ],
)
def test_calculates_unit_price(total: Decimal, quantity: int, expected: Decimal) -> None:
    assert calculator.unit_price(total, quantity) == expected


# Complex objects via a factory inside the param list
@pytest.mark.parametrize(
    ("order", "expected_total"),
    [
        (order_with(quantity=2, price="10.00"), Decimal("20.00")),
        (order_with(quantity=0, price="10.00"), Decimal("0.00")),
    ],
    ids=["two-items", "no-items"],
)
def test_calculates_order_total(order: Order, expected_total: Decimal) -> None:
    assert calculator.calculate_total(order) == expected_total
```

### Grouped Assertions — Report All Failures

```python
# BAD: First failure hides the others
def test_creates_user():
    user = service.create_user(request)
    assert user.name == "John"        # if this fails...
    assert user.email == "j@e.com"    # ...this never runs
    assert user.id is not None

# GOOD: assert against one structure so the diff shows every mismatch
def test_creates_user():
    user = service.create_user(request)
    assert (user.name, user.email, user.id is not None) == ("John", "j@e.com", True)

# Or use pytest-check for soft assertions when you truly need independence
from pytest_check import check


def test_creates_user_soft():
    user = service.create_user(request)
    with check:
        assert user.name == "John"
    with check:
        assert user.email == "j@e.com"
    with check:
        assert user.id is not None
```

---

## Test Naming

```python
# BAD: Unclear names
def test_1(): ...
def test_create_order(): ...
def test_create_order_fail(): ...

# GOOD: Behavior-focused names
def test_creates_order_with_valid_request(): ...
def test_raises_when_customer_not_found(): ...
def test_returns_empty_page_when_no_orders_exist(): ...
def test_applies_discount_for_vip_customers(): ...
```

Pattern: `verb` + `what happens` + `under what conditions`

---

## Mocking — unittest.mock / pytest-mock

```python
# BAD: Testing implementation details
def test_create_order(mocker):
    repo = mocker.Mock()
    repo.save.return_value = order
    service = OrderService(repo, publisher)
    service.create_order(request)
    repo.save.assert_called_once()       # asserting "it was called", not the result
    publisher.publish.assert_called_once()

# GOOD: focus on behavior, arrange/act/assert
def test_creates_order_with_calculated_total(mocker) -> None:
    # given
    request = CreateOrderRequest(
        customer_id="cust-1",
        items=[LineItem("prod-1", quantity=2, price=Decimal("10.00"))],
    )
    repo = mocker.Mock(spec=OrderRepository)
    repo.save.side_effect = lambda order: replace(order, id=1)
    service = OrderService(repo, mocker.Mock(spec=EventPublisher))

    # when
    response = service.create_order(request)

    # then
    assert response.total == Decimal("20.00")
    assert response.id == 1
```

Use `spec=` so the mock rejects calls to attributes the real object doesn't have.

### Asserting Call Arguments

```python
def test_publishes_order_created_event(mocker) -> None:
    # given
    request = CreateOrderRequest(customer_id="cust-1", items=[item])
    repo = mocker.Mock(spec=OrderRepository)
    repo.save.return_value = saved_order
    publisher = mocker.Mock(spec=EventPublisher)
    service = OrderService(repo, publisher)

    # when
    service.create_order(request)

    # then — inspect the captured argument
    publisher.publish.assert_called_once()
    event = publisher.publish.call_args.args[0]
    assert event.order_id == saved_order.id
    assert event.customer_id == "cust-1"
```

### Patching — Patch Where It's Used

```python
# BAD: patching the definition site has no effect on the caller
mocker.patch("myapp.clients.payment.charge")

# GOOD: patch the name in the module that looks it up
def test_charges_on_checkout(mocker) -> None:
    charge = mocker.patch("myapp.services.checkout.charge", return_value="txn-1")
    result = checkout(cart)
    charge.assert_called_once_with(cart.total)
    assert result.transaction_id == "txn-1"
```

---

## Fixtures over setUp

```python
# BAD: unittest-style setUp shares mutable state and hides dependencies
class TestOrderService(unittest.TestCase):
    def setUp(self):
        self.repo = FakeRepo()
        self.service = OrderService(self.repo)

# GOOD: fixtures are explicit, composable, and scoped
import pytest


@pytest.fixture
def repo() -> FakeRepo:
    return FakeRepo()


@pytest.fixture
def service(repo: FakeRepo) -> OrderService:
    return OrderService(repo, NullPublisher())


def test_creates_order(service: OrderService) -> None:
    response = service.create_order(a_request())
    assert response.id is not None


# Fixtures that need teardown use yield (context-manager style)
@pytest.fixture
def temp_dir() -> Iterator[Path]:
    with tempfile.TemporaryDirectory() as d:
        yield Path(d)
```

Share fixtures across a package by putting them in `conftest.py`. Pick the
narrowest scope that works (`function` default; `session` for expensive setup).

---

## pytest-django

### Database Access

```python
# Any test that touches the ORM must opt in to the db
import pytest


@pytest.mark.django_db
def test_saves_and_finds_order() -> None:
    order = Order.objects.create(customer_id="cust-1", total=Decimal("100.00"))

    found = Order.objects.get(pk=order.pk)

    assert found.customer_id == "cust-1"
    assert found.total == Decimal("100.00")


# Apply to a whole module when most tests hit the db
pytestmark = pytest.mark.django_db
```

### DRF API Tests

```python
import pytest
from rest_framework import status
from rest_framework.test import APIClient


@pytest.fixture
def api() -> APIClient:
    return APIClient()


@pytest.mark.django_db
class TestOrderApi:
    def test_returns_order_by_id(self, api: APIClient) -> None:
        order = OrderFactory(customer_id="cust-1", total=Decimal("100.00"))

        resp = api.get(f"/api/v1/orders/{order.pk}/")

        assert resp.status_code == status.HTTP_200_OK
        assert resp.json() == {
            "id": order.pk,
            "customer_id": "cust-1",
            "total": "100.00",
        }

    def test_returns_404_when_not_found(self, api: APIClient) -> None:
        resp = api.get("/api/v1/orders/99/")
        assert resp.status_code == status.HTTP_404_NOT_FOUND

    def test_returns_400_for_invalid_request(self, api: APIClient) -> None:
        resp = api.post("/api/v1/orders/", {"customer_id": "", "items": []}, format="json")
        assert resp.status_code == status.HTTP_400_BAD_REQUEST

    def test_returns_201_with_location_header(self, api: APIClient) -> None:
        payload = {
            "customer_id": "cust-1",
            "items": [{"product_id": "p1", "quantity": 2, "price": "10.00"}],
        }
        resp = api.post("/api/v1/orders/", payload, format="json")

        assert resp.status_code == status.HTTP_201_CREATED
        assert resp.json()["total"] == "20.00"
        assert "/api/v1/orders/" in resp["Location"]
```

### Serializer Tests

```python
@pytest.mark.django_db
class TestOrderSerializer:
    def test_serializes(self) -> None:
        order = OrderFactory(customer_id="cust-1", total=Decimal("99.99"))

        data = OrderSerializer(order).data

        assert data["id"] == order.pk
        assert data["customer_id"] == "cust-1"
        assert data["total"] == "99.99"

    def test_deserializes_and_validates(self) -> None:
        serializer = OrderSerializer(data={"customer_id": "cust-1", "total": "99.99"})

        assert serializer.is_valid(), serializer.errors
        assert serializer.validated_data["total"] == Decimal("99.99")
```

---

## Testcontainers

```python
# Integration test with a real PostgreSQL — no fakes, no in-memory SQLite drift
import pytest
from testcontainers.postgres import PostgresContainer


@pytest.fixture(scope="session")
def postgres() -> Iterator[PostgresContainer]:
    with PostgresContainer("postgres:16") as pg:
        yield pg


@pytest.fixture(scope="session")
def django_db_setup(postgres: PostgresContainer, django_db_blocker):  # noqa: ANN001
    from django.conf import settings

    settings.DATABASES["default"] = {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": postgres.dbname,
        "USER": postgres.username,
        "PASSWORD": postgres.password,
        "HOST": postgres.get_container_host_ip(),
        "PORT": postgres.get_exposed_port(5432),
    }
    with django_db_blocker.unblock():
        yield


@pytest.mark.django_db
def test_creates_and_retrieves_order() -> None:
    request = CreateOrderRequest(
        customer_id="cust-1",
        items=[LineItem("prod-1", quantity=2, price=Decimal("10.00"))],
    )

    created = order_service.create_order(request)
    retrieved = order_service.get_order(created.id)

    assert retrieved.customer_id == "cust-1"
    assert retrieved.total == Decimal("20.00")
```

The `session`-scoped container starts once and is shared across the whole run.
`pytest-docker` is an alternative when you need a `docker-compose.yml` of
services rather than a single container.

---

## Mocking HTTP — responses / httpx

```python
# BAD: hitting the real network — slow, flaky, fails in CI
def test_fetches_rate():
    rate = client.fetch_rate("USD")  # real HTTP call
    assert rate > 0

# GOOD (requests): stub the endpoint with `responses`
import responses


@responses.activate
def test_fetches_rate() -> None:
    responses.get(
        "https://api.example.com/rates/USD",
        json={"currency": "USD", "rate": "1.10"},
        status=200,
    )

    rate = client.fetch_rate("USD")

    assert rate == Decimal("1.10")
    assert responses.calls[0].request.url.endswith("/rates/USD")


# GOOD (httpx): use a MockTransport — type-safe and no monkeypatching
import httpx


def test_fetches_rate_httpx() -> None:
    def handler(request: httpx.Request) -> httpx.Response:
        assert request.url.path == "/rates/USD"
        return httpx.Response(200, json={"currency": "USD", "rate": "1.10"})

    client = RateClient(transport=httpx.MockTransport(handler))

    assert client.fetch_rate("USD") == Decimal("1.10")
```

---

## Test Data Builders — factory_boy

```python
# BAD: long constructors repeated in every test
def test_calculates_discount():
    customer = Customer(
        id=1, name="John", email="john@example.com", type=CustomerType.VIP,
        joined=date(2020, 1, 1), active=True, country="US", phone=None, notes=None,
    )
    order = Order(
        id=1, customer_id=customer.id, status=Status.PENDING,
        total=Decimal("0"), created_at=datetime.now(UTC),
    )
    # Hard to read — what actually matters for this test?

# GOOD: factory_boy — sensible defaults, override only what matters
import factory


class CustomerFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Customer

    name = factory.Faker("name")
    email = factory.Faker("email")
    type = CustomerType.STANDARD
    active = True
    country = "US"


class OrderFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Order

    customer = factory.SubFactory(CustomerFactory)
    status = Status.PENDING
    total = Decimal("100.00")
    created_at = factory.LazyFunction(lambda: datetime.now(UTC))


# Test — specify only the relevant fields
@pytest.mark.django_db
def test_applies_vip_discount() -> None:
    order = OrderFactory(total=Decimal("200.00"))
    customer = CustomerFactory(type=CustomerType.VIP)

    discount = calculator.calculate(order, customer)

    assert discount == Decimal("40.00")
```

For plain dataclasses (no ORM), a small builder function or `factory.Factory`
base class does the same job without a database.

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|--------------|---------|-----|
| Testing implementation, not behavior | Brittle tests | Assert outcomes, not `mock.assert_called_once()` |
| Real DB/network for unit tests | Slow, flaky suite | Mock at boundaries; reserve real DB for integration |
| Mock without `spec=` | Tests pass on typos | `Mock(spec=Repo)` rejects unknown attributes |
| Patching the definition site | Patch has no effect | Patch where the name is looked up |
| No assertions (just "doesn't raise") | False confidence | Assert specific outcomes |
| Testing properties/getters | Wasted effort | Test behavior that uses them |
| Shared mutable state between tests | Flaky tests | Use fixtures; avoid module-level mutables |
| Hardcoded test data everywhere | Duplication | factory_boy / builder functions |
| `time.sleep()` in tests | Slow, flaky | Poll with a timeout helper or freeze time |
| `@pytest.mark.skip` on failing tests | Rot | Fix or delete |

---

## Related Skills

- `django-rest-api` — API/view patterns being tested
- `django-orm` — Model and queryset testing with `@pytest.mark.django_db`
- `review-changes-python` — Testing checklist for code review
- `modern-python` — Type hints and dataclasses used throughout test code
