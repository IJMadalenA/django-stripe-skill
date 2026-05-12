# Architecture — Django Stripe System Design

## Project Structure

```
project/
  settings.py              # STRIPE_* settings, INSTALLED_APPS
  urls.py                  # path("stripe/", include("djstripe.urls", namespace="djstripe"))
  myapp/
    models.py              # Your models referencing dj-stripe via FK
    webhooks.py            # @djstripe_receiver handlers
    apps.py                # ready() imports webhooks
    views.py               # Checkout creation, customer portal, etc.
    services.py            # Business logic for Stripe operations
    tests/
      test_webhooks.py     # Webhook handler tests
      test_services.py     # Service layer tests
      test_integration.py  # End-to-end payment flow tests
```

## Architecture Patterns

### Pattern 1: Stripe as Source of Truth

```
┌──────────┐    write     ┌──────────┐    webhook     ┌──────────────┐
│  Django   │ ──────────► │  Stripe   │ ────────────► │  dj-stripe    │
│   App     │             │   API     │               │  syncs to DB  │
└──────────┘              └──────────┘               └──────────────┘
      │                                                     │
      │                  ┌──────────┐                       │
      └── read ──────────│  Local   │◄────── read ──────────┘
                         │    DB     │
                         └──────────┘
```

- **Write path:** Your app → Stripe API (e.g., create customer, charge)
- **Read path:** Your app → Local DB (e.g., list subscriptions, query charges)
- **Sync path:** Stripe → webhook → dj-stripe → Local DB

**Never** write to dj-stripe model instances directly. **Never** modify synced records.

### Pattern 2: Business Logic in Services

```python
# services.py — Business logic layer
from djstripe.models import Customer

class SubscriptionService:
    @staticmethod
    def create_subscription(user, price_id, trial_days=None):
        customer = Customer.get_or_create(subscriber=user)[0]
        kwargs = {"items": [{"price": price_id}]}
        if trial_days:
            kwargs["trial_period_days"] = trial_days
        return customer.subscribe(**kwargs)
        # NOTE: If subscribe() doesn't support all the arguments you need,
        # call the official stripe.Customer.subscribe() directly instead.

    @staticmethod
    def cancel_subscription(user, at_period_end=True):
        customer = Customer.objects.get(subscriber=user)
        subscription = customer.subscription
        if subscription:
            return subscription.cancel(at_period_end=at_period_end)
        return None
```

### Pattern 3: Webhook Handlers in Dedicated Module

```python
# webhooks.py — All Stripe event handlers
from djstripe.event_handlers import djstripe_receiver
from djstripe.models import Event

@djstripe_receiver("customer.subscription.deleted")
def handle_subscription_deleted(sender, event: Event, **kwargs):
    # Fulfill cancellation — deactivate features, notify user
    pass
```

**Register in AppConfig:**
```python
# apps.py
class MyAppConfig(AppConfig):
    name = "myapp"
    def ready(self):
        import myapp.webhooks
```

### Pattern 4: Model Extensions via Foreign Key (or CharField for pre-creation references)

```python
# models.py — Never subclass dj-stripe models
class UserSubscription(models.Model):
    user = models.OneToOneField(AUTH_USER_MODEL, on_delete=models.CASCADE)
    stripe_customer = models.ForeignKey("djstripe.Customer", on_delete=models.SET_NULL, null=True)
    plan_name = models.CharField(max_length=255)
    features_enabled = models.JSONField(default=dict)

class Order(models.Model):
    user = models.ForeignKey(AUTH_USER_MODEL, on_delete=models.CASCADE)
    stripe_checkout_session = models.CharField(max_length=255)  # cs_xxx (CharField used because Order is created before Stripe object exists)
    stripe_payment_intent = models.CharField(max_length=255, blank=True)  # pi_xxx
    status = models.CharField(max_length=50, default="pending")
    fulfilled = models.BooleanField(default=False)
```

### Pattern 5: Metadata Bridge

Use Stripe `metadata` to link Stripe objects to your application records:

```python
import stripe

# Inside a Django view
# When creating a Stripe object, store your app IDs in metadata
stripe.checkout.Session.create(
    metadata={
        "djstripe_subscriber": str(request.user.id),  # Known by dj-stripe
        "order_id": str(order.id),                # Your app's order
        "campaign": "spring_sale",                # Business context
    },
)
```

In webhook handlers, read metadata to find your records:

```python
@djstripe_receiver("checkout.session.completed")
def handle_checkout_completed(sender, event, **kwargs):
    # Idempotency: deduplicate by event ID
    if ProcessedEvent.objects.filter(event_id=event.id).exists():
        return
    metadata = event.data["object"]["metadata"]
    order_id = metadata.get("order_id")
    order = Order.objects.get(id=order_id)
    order.fulfilled = True
    order.stripe_payment_intent = event.data["object"]["payment_intent"]
    order.save()
    ProcessedEvent.objects.create(event_id=event.id)
```

### ProcessedEvent Model (Idempotency)

```python
class ProcessedEvent(models.Model):
    event_id = models.CharField(max_length=255, unique=True)
    processed_at = models.DateTimeField(auto_now_add=True)
```

## Anti-Patterns to Avoid

| Anti-Pattern | Why It's Wrong | Correct Approach |
|---|---|---|
| Modifying synced Stripe records | dj-stripe overwrites on next sync | Only modify your own models |
| Polling Stripe API in views | Slow, rate-limited, unnecessary | Read from local DB |
| Subclassing dj-stripe models | Breaks sync, migration hell | Use FK to your own models |
| Ignoring webhook idempotency | Duplicate fulfillment, double charges | Deduplicate by event ID |
| Hardcoding Stripe IDs in code | IDs change between environments | Store in DB, use fixtures for tests |
| Calling Stripe API in Django model signals (post_save, pre_delete) | Couples ORM to external API; triggers on fixtures/migrations; uncontrolled retries | Use dj-stripe receivers or background tasks |

## Environment Strategy

```
Development:
  STRIPE_LIVE_MODE = False
  STRIPE_TEST_SECRET_KEY = sk_test_...
  Webhooks via: stripe listen --forward-to localhost:8000/stripe/webhook/<uuid>/

Staging:
  STRIPE_LIVE_MODE = False
  STRIPE_TEST_SECRET_KEY = sk_test_...  # Separate test account
  Real webhooks from Stripe test mode

Production:
  STRIPE_LIVE_MODE = True
  STRIPE_LIVE_SECRET_KEY = sk_live_...
  Real webhooks from Stripe live mode
  Restricted API key (rk_live_...) for dj-stripe
```

> **DO NOT override `STRIPE_API_VERSION`** in your settings. dj-stripe pins it to its tested version. Changing it will break sync and webhook processing.

```python
# Required for Stripe FK via Stripe IDs (new installs)
DJSTRIPE_FOREIGN_KEY_TO_FIELD = "id"
```
