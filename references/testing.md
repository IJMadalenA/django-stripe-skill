# Testing — Unit, Integration, and Webhook Testing

Complete testing strategies for Django applications using dj-stripe.

## Testing Philosophy

Stripe integrations require testing at multiple levels:

1. **Unit tests:** Test webhook handlers, service logic, model methods
2. **Integration tests:** Test full payment flows with Stripe test mode
3. **Webhook tests:** Test event processing and idempotency
4. **End-to-end tests:** Test complete user journeys

## Test Database Setup

### Database Configuration

```python
# settings_test.py or environment variables
DJSTRIPE_TEST_DB_VENDOR = os.environ.get("DJSTRIPE_TEST_DB_VENDOR", "sqlite")

if DJSTRIPE_TEST_DB_VENDOR == "postgres":
    DATABASES["default"].update({
        "ENGINE": "django.db.backends.postgresql",
        "HOST": os.environ.get("DJSTRIPE_TEST_DB_HOST", "localhost"),
        "PORT": os.environ.get("DJSTRIPE_TEST_DB_PORT", "5432"),
    })
```

**Recommendation:** Use PostgreSQL for tests to match production. SQLite
JSONField behavior differs from PostgreSQL.

### Test Dependencies

```bash
pip install pytest pytest-django pytest-dotenv
```

## Unit Testing

### Testing Webhook Handlers

```python
import json
from django.test import TestCase
from unittest.mock import patch
from djstripe.models import Event

class WebhookHandlerTests(TestCase):
    fixtures = ["test_events.json"]  # Pre-loaded event fixtures

    def test_checkout_completed_fulfills_order(self):
        event = Event.objects.filter(type="checkout.session.completed").first()

        # Create test data your handler expects
        metadata = event.data["object"]["metadata"]
        user = User.objects.create(username="testuser")
        order = Order.objects.create(
            user=user,
            id=metadata["order_id"],
            status="pending",
        )

        # Manually trigger the handler
        from myapp.webhooks import handle_checkout_completed
        handle_checkout_completed(sender=Event, event=event)

        order.refresh_from_db()
        self.assertTrue(order.fulfilled)

    def test_handler_is_idempotent(self):
        event = Event.objects.filter(type="checkout.session.completed").first()
        metadata = event.data["object"]["metadata"]
        order = Order.objects.create(
            id=metadata["order_id"],
            status="pending",
        )

        # Process twice
        from myapp.webhooks import handle_checkout_completed
        handle_checkout_completed(sender=Event, event=event)
        handle_checkout_completed(sender=Event, event=event)

        # Should not create duplicate fulfillment
        self.assertEqual(Fulfillment.objects.filter(order=order).count(), 1)

    @patch("stripe.PaymentIntent.create")
    def test_create_payment_intent_service(self, mock_create):
        mock_create.return_value = {"client_secret": "pi_test_secret"}

        result = PaymentService.create_intent(
            amount=Decimal("29.00"),
            customer_id="cus_test123",
        )

        mock_create.assert_called_once_with(
            amount=2900,
            currency="usd",
            customer="cus_test123",
        )
        self.assertEqual(result["client_secret"], "pi_test_secret")
```

### Testing Service Layer

```python
class SubscriptionServiceTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(username="testuser")
        self.product = stripe.Product.create(name="Test Product")
        self.price = stripe.Price.create(
            product=self.product.id,
            unit_amount=2900,  # $29.00 in cents
            currency="usd",
        )
        Price.sync_from_stripe_data(self.price)

    @patch("stripe.Subscription.create")
    def test_create_subscription(self, mock_create):
        mock_create.return_value = {"id": "sub_test123", "status": "active"}

        result = SubscriptionService.create_subscription(
            user=self.user,
            price_id=self.price.id,
            trial_days=14,
        )

        mock_create.assert_called_once()
        self.assertEqual(result["status"], "active")

    def test_cancel_subscription_no_subscription(self):
        """Should handle users without subscriptions gracefully."""
        result = SubscriptionService.cancel_subscription(self.user)
        self.assertIsNone(result)
```

### Testing Models

```python
class CustomerModelTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(username="testuser")
        self.customer = Customer.objects.create(
            id="cus_test123",
            subscriber=self.user,
            email="test@example.com",
        )
        self.product = Product.sync_from_stripe_data(
            stripe.Product.create(name="Test Product")
        )
        self.price = Price.sync_from_stripe_data(
            stripe.Price.create(
                product=self.product.id,
                unit_amount=2900,  # $29.00 in cents
                currency="usd",
            )
        )

    def test_customer_has_no_subscription(self):
        self.assertIsNone(self.customer.subscription)

    def test_customer_has_active_subscription(self):
        sub = Subscription.objects.create(
            id="sub_test123",
            customer=self.customer,
            status="active",
        )
        SubscriptionItem.objects.create(
            id="si_test123",
            subscription=sub,
            price=self.price,
        )

        has_access = SubscriptionItem.objects.filter(
            subscription__customer=self.customer,
            subscription__status="active",
            price__product=self.product,
        ).exists()
        self.assertTrue(has_access)
```

## Building Test Fixtures

### Creating Event Fixtures

```python
# Generate fixture from a real event
python manage.py shell

from djstripe.models import Event
import json

event = Event.objects.get(id="evt_test123")
fixture = [{
    "model": "djstripe.event",
    "pk": event.id,
    "fields": {
        "type": event.type,
        "data": event.data,
        "created": event.created,
        "api_version": event.api_version,
    }
}]

with open("myapp/fixtures/test_events.json", "w") as f:
    json.dump(fixture, f, indent=2)
```

### Fixture Best Practices

```json
[
  {
    "model": "djstripe.event",
    "pk": "evt_test_001",
    "fields": {
      "type": "checkout.session.completed",
      "data": {
        "object": {
          "id": "cs_test_001",
          "mode": "payment",
          "payment_status": "paid",
          "metadata": {
            "djstripe_subscriber": "1",
            "order_id": "100"
          }
        }
      },
      "created": 1700000000,
      "api_version": "2020-08-27",
      "livemode": false
    }
  }
]
```

- Use test-mode IDs (contain `_test_`)
- Include only fields your handlers use
- Store in `myapp/fixtures/` directory
- Keep fixtures minimal and focused

### Regenerating Test Fixtures

dj-stripe provides a management command to regenerate test fixtures from a real Stripe account:

```bash
python manage.py regenerate_test_fixtures
```

This updates fixture JSON files to match current Stripe API schemas. Use a dedicated "dj-stripe scratch" Stripe account for this.

## Integration Testing

### Testing with Stripe Test Mode

```python
import stripe
from django.test import TestCase, override_settings

@override_settings(STRIPE_LIVE_MODE=False)
class StripeIntegrationTests(TestCase):
    def setUp(self):
        stripe.api_key = settings.STRIPE_TEST_SECRET_KEY

    def test_create_customer_in_stripe(self):
        customer_data = stripe.Customer.create(
            email="test@example.com",
            name="Test User",
        )

        self.assertIsNotNone(customer_data["id"])
        self.assertTrue(customer_data["id"].startswith("cus_"))

        # Clean up
        stripe.Customer.delete(customer_data["id"])

    def test_full_payment_flow(self):
        # Create product and price
        product = stripe.Product.create(name="Test Product")
        price = stripe.Price.create(
            product=product.id,
            unit_amount=1000,
            currency="usd",
        )

        # Create customer
        customer = stripe.Customer.create(email="test@example.com")

        # Create Checkout Session
        session = stripe.checkout.Session.create(
            mode="payment",
            line_items=[{"price": price.id, "quantity": 1}],
            success_url="https://example.com/success",
            cancel_url="https://example.com/cancel",
            customer=customer.id,
        )

        self.assertEqual(session.mode, "payment")
        self.assertEqual(session.payment_status, "unpaid")

        # Clean up
        stripe.Product.modify(product.id, active=False)
        stripe.Customer.delete(customer.id)
```

### Testing Webhook Delivery

```bash
# Trigger real Stripe events in test mode
stripe trigger customer.created
stripe trigger checkout.session.completed
stripe trigger invoice.payment_succeeded
stripe trigger payment_intent.succeeded

# Forward to test server (include webhook secret header for signature verification)
# dj-stripe looks for X-Djstripe-Webhook-Secret header when the Stripe CLI is used
stripe listen \
  --forward-to http://localhost:8000/stripe/webhook/<uuid>/ \
  -H "x-djstripe-webhook-secret: $(stripe listen --print-secret)"
```

## Webhook Testing Patterns

### Testing Webhook View (HTTP Level)

```python
from django.test import TestCase, Client
from django.urls import reverse

class WebhookViewTests(TestCase):
    def setUp(self):
        self.client = Client()
        self.endpoint = WebhookEndpoint.objects.create(
            id="we_test123",
            url="https://example.com/stripe/webhook/test-uuid/",
            secret="whsec_test",
            djstripe_tolerance=300,
        )

    def test_webhook_signature_verification(self):
        url = reverse("djstripe:djstripe_webhook_by_uuid", kwargs={"uuid": self.endpoint.djstripe_uuid})

        # Missing signature header
        response = self.client.post(
            url,
            data=json.dumps({"type": "test"}),
            content_type="application/json",
        )
        self.assertEqual(response.status_code, 400)

    def test_webhook_processes_event(self):
        url = reverse("djstripe:djstripe_webhook_by_uuid", kwargs={"uuid": self.endpoint.djstripe_uuid})

        payload = json.dumps({
            "id": "evt_test_001",
            "type": "checkout.session.completed",
            "data": {"object": {"id": "cs_test_001", "mode": "payment"}},
        })

        # With valid signature (computed from secret)
        import hmac, hashlib, time
        timestamp = str(int(time.time()))
        signed = hmac.new(
            self.endpoint.secret.encode(),
            f"{timestamp}.{payload}".encode(),
            hashlib.sha256,
        ).hexdigest()

        response = self.client.post(
            url,
            data=payload,
            content_type="application/json",
            HTTP_STRIPE_SIGNATURE=f"t={timestamp},v1={signed}",
        )
        self.assertEqual(response.status_code, 200)
```

## Testing with Mocks and Stubs

### Mocking Stripe API

```python
from unittest.mock import patch, Mock

class PaymentServiceTests(TestCase):
    @patch("stripe.PaymentIntent.create")
    def test_create_payment_intent(self, mock_create):
        mock_create.return_value = Mock(
            id="pi_test123",
            client_secret="pi_test123_secret",
            status="requires_payment_method",
        )

        result = PaymentService.create_payment_intent(
            amount=Decimal("49.99"),
            currency="eur",
            customer_id="cus_test123",
        )

        self.assertEqual(result.id, "pi_test123")

    @patch("stripe.Subscription.modify")
    def test_cancel_subscription(self, mock_modify):
        mock_modify.return_value = Mock(
            id="sub_test123",
            status="canceled",
            cancel_at_period_end=True,
        )

        result = SubscriptionService.cancel_subscription(
            subscription_id="sub_test123",
            at_period_end=True,
        )

        mock_modify.assert_called_once_with(
            "sub_test123",
            cancel_at_period_end=True,
        )
```

### Mocking dj-stripe Models

```python
class OrderFulfillmentTests(TestCase):
    @patch("djstripe.models.Customer.objects.get")
    def test_fulfill_order_from_webhook(self, mock_get):
        mock_customer = Mock()
        mock_customer.subscriber = self.user
        mock_get.return_value = mock_customer

        event_data = {
            "object": {
                "id": "cs_test_001",
                "metadata": {"order_id": "123"},
            }
        }

        fulfill_order(event_data)

        order = Order.objects.get(id="123")
        self.assertTrue(order.fulfilled)
```

## Test Runner Configuration

### pytest Configuration

```ini
# pytest.ini
[pytest]
DJANGO_SETTINGS_MODULE = myproject.settings_test
python_files = test_*.py
addopts = --reuse-db --create-db -v
```

### Running Tests

```bash
# Quick test run
DJSTRIPE_TEST_DB_VENDOR=sqlite pytest --reuse-db

# Full test with PostgreSQL
DJSTRIPE_TEST_DB_VENDOR=postgres pytest --reuse-db

# Specific test
pytest myapp/tests/test_webhooks.py::WebhookHandlerTests::test_idempotency

# With coverage
pytest --cov=myapp --cov-report=html
```

## CI/CD Testing

### GitHub Actions Example

```yaml
name: Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_pass
        ports:
          - 5432:5432
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: poetry install
      - run: tox
      - run: |
          DJSTRIPE_TEST_DB_VENDOR=postgres \
          DJSTRIPE_TEST_DB_HOST=localhost \
          DJSTRIPE_TEST_DB_PORT=5432 \
          pytest --reuse-db
```

## Testing Checklist

### Before Deploying

- [ ] All webhook handlers have unit tests
- [ ] Idempotency tests for every handler
- [ ] Service layer methods tested
- [ ] Edge cases covered (no subscription, failed payment, etc.)
- [ ] Integration tests pass with Stripe test mode
- [ ] Tests use PostgreSQL (matching production)
- [ ] No real customer data in test fixtures
- [ ] All mocked Stripe calls match real API signatures
- [ ] CI pipeline runs full test suite

### Common Test Gaps

| Gap | Why It Matters | How to Fill |
|---|---|---|
| No idempotency tests | Double fulfillment in production | Test each handler called twice |
| No `past_due` tests | Subscription status transitions missed | Test all status transition webhooks |
| No `subscription=None` tests | Null pointer on user without sub | Test customer with no subscription |
| SQLite-only testing | JSONField behavior differs from PG | Run CI with PostgreSQL |
| No webhook signature tests | Fake webhooks accepted in prod | Test signature verification |
| No integration tests | Stripe API changes break silently | Run periodic integration tests |
