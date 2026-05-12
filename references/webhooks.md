# Webhooks — Design, Processing, and Security

Webhooks are the backbone of a reliable Stripe integration. dj-stripe uses a
signal-based architecture where Stripe events trigger `@djstripe_receiver`
handlers in your Django application.

## Event Processing Flow

```
Stripe API
  │
  ├──► POST /stripe/webhook/<uuid>/
  │      │
  │      ├── 1. Signature verification (per-endpoint tolerance)
  │      │      └── Signal: webhook_pre_validate
  │      │      └── Signal: webhook_post_validate(valid=bool)
  │      │
  │      ├── 2. Event saved to Event model
  │      │
  │      ├── 3. dj-stripe syncs referenced objects to DB
  │      │      └── (creates/updates Customer, Charge, Invoice, etc.)
  │      │
  │      ├── 4. Your @djstripe_receiver handlers fire
  │      │      └── Signal: webhook_pre_process
  │      │      └── Your handlers run (synchronous, single DB transaction)
  │      │      └── Signal: webhook_post_process
  │      │
  │      └── 5. Returns 200 (success) or 500 (error → Stripe retries)
  │             └── Signal: webhook_processing_error (on exception)
```

**Critical:** All handlers run in a SINGLE database transaction. If any handler
raises, the entire transaction rolls back. Stripe sees a non-2xx response and
redelivers the event.

## Writing Webhook Handlers

### Basic Handler

```python
from djstripe.event_handlers import djstripe_receiver
from djstripe.models import Event

@djstripe_receiver("charge.succeeded")
def handle_charge_succeeded(sender, event: Event, **kwargs):
    charge_id = event.data["object"]["id"]
    # event.data["object"] contains the full Stripe object
    # But you can also query the synced model:
    from djstripe.models import Charge
    charge = Charge.objects.get(id=charge_id)
    # Fulfill the purchase, send email, etc.
```

### Multi-Event Handlers

```python
@djstripe_receiver([
    "customer.subscription.created",
    "customer.subscription.updated",
    "customer.subscription.deleted",
])
def handle_subscription_change(sender, event: Event, **kwargs):
    subscription_id = event.data["object"]["id"]
    customer_id = event.data["object"]["customer"]
    # Handle subscription change
```

### Accessing Event Data

```python
@djstripe_receiver("invoice.payment_succeeded")
def handle_invoice_paid(sender, event: Event, **kwargs):
    # event.data has structure: {"object": {...}, "previous_attributes": {...}}

    invoice_data = event.data["object"]
    invoice_id = invoice_data["id"]
    customer_id = invoice_data["customer"]
    amount_paid = invoice_data["amount_paid"]

    # previous_attributes exists on .updated events
    previous = event.data.get("previous_attributes", {})
```

## Idempotency — The Golden Rule

**Every handler MUST be idempotent.** Stripe may redeliver events. Common patterns:

### Pattern 1: Event ID Deduplication

```python
class ProcessedEvent(models.Model):
    event_id = models.CharField(max_length=255, unique=True)
    processed_at = models.DateTimeField(auto_now_add=True)

@djstripe_receiver("checkout.session.completed")
def handle_checkout(sender, event: Event, **kwargs):
    _, created = ProcessedEvent.objects.get_or_create(event_id=event.id)
    if not created:
        return  # Already processed

    # Process the event...
    fulfill_order(event.data["object"])
```

### Pattern 2: State Check

```python
@djstripe_receiver("invoice.payment_succeeded")
def handle_invoice_paid(sender, event: Event, **kwargs):
    invoice_id = event.data["object"]["id"]
    # Atomic conditional update prevents race conditions
    updated = Order.objects.filter(
        stripe_invoice_id=invoice_id, fulfilled=False
    ).update(fulfilled=True)
    if not updated:
        return  # Already fulfilled by another worker
    fulfill_order(invoice_id)
```

## Handler Registration

Handlers must be imported before they can receive events. Register in `AppConfig.ready()`:

```python
# myapp/apps.py
from django.apps import AppConfig

class MyAppConfig(AppConfig):
    name = "myapp"

    def ready(self):
        import myapp.webhooks  # noqa: F401
```

Without this, your `@djstripe_receiver` decorators will never be connected.

## Webhook Lifecycle Signals

Use these for cross-cutting concerns (logging, metrics, alerting):

```python
from django.dispatch import receiver
from djstripe.signals import (
    webhook_pre_validate,
    webhook_post_validate,
    webhook_pre_process,
    webhook_post_process,
    webhook_processing_error,
)

@receiver(webhook_post_validate)
def log_validation_result(sender, instance, api_key, valid, **kwargs):
    if not valid:
        logger.warning(f"Invalid webhook signature for endpoint {instance.id}")

@receiver(webhook_processing_error)
def alert_on_error(sender, instance, api_key, exception, data, **kwargs):
    logger.error(f"Webhook processing failed: {exception}")
    # Send to Sentry, PagerDuty, etc.
```

## Webhook Endpoint Management

### Creating Endpoints in Admin

1. Go to `/admin/djstripe/webhookendpoint/add/`
2. Fill in the URL domain (UUID is auto-generated)
3. Select events to listen for
4. Save — this creates the endpoint on Stripe AND in your DB

### Multiple Endpoints

Since 2.7, dj-stripe supports multiple webhook endpoints. This is useful for:
- Different event sets for different parts of your app
- Separate development/staging webhooks
- Gradually migrating event types

### UUID Security

Webhook URLs use UUIDs: `/stripe/webhook/<uuid>/`. This makes them impossible
to brute-force, eliminating the need for additional URL-based security.

## Webhook Signature Verification

### Configuration

Verification is per-endpoint, configured via two fields in admin:

**`djstripe_validation_method`** — controls HOW verification works:
- `verify_signature` (default): Uses Stripe's signature header verification
- `none`: Disables all verification (dev only — DO NOT use in production)
- `retrieve_event`: Fetches the event from Stripe API and compares payloads

**`djstripe_tolerance`** — replay-attack window in seconds (only relevant with `verify_signature`):
- `300` (default): 5-minute tolerance (recommended minimum)
- `600`: 10-minute tolerance (looser, for clock skew)
- `0`: Zero tolerance for clock skew (still verifies signature)

### Global Setting

`DJSTRIPE_WEBHOOK_VALIDATION` (`settings.py`) provides a global default for older
setups. Per-endpoint `djstripe_validation_method` takes precedence when set.

### How It Works

1. Stripe signs each webhook with the endpoint's `whsec_` secret
2. Signature includes timestamp and payload hash
3. dj-stripe recomputes the signature and compares
4. If timestamp is outside `djstripe_tolerance` seconds, verification fails

## Common Event Types

### Essential Events to Handle

| Event | Purpose |
|---|---|
| `checkout.session.completed` | Fulfill orders, grant access |
| `checkout.session.async_payment_succeeded` | Delayed payment confirmed |
| `customer.subscription.created` | New subscription started |
| `customer.subscription.updated` | Subscription changed (upgrade, downgrade) |
| `customer.subscription.deleted` | Subscription ended (cancel, expire) |
| `invoice.payment_succeeded` | Invoice paid successfully |
| `invoice.payment_failed` | Payment failed — notify customer |
| `payment_intent.succeeded` | Payment confirmed (custom UI) |
| `payment_intent.payment_failed` | Payment failed (custom UI) |
| `charge.refunded` | Refund issued |
| `charge.dispute.created` | Dispute filed — urgent |
| `customer.updated` | Customer data changed |

### Event Types by Domain

Use Stripe Dashboard or `stripe trigger --help` to see all available events.

**Payments:** `payment_intent.*`, `charge.*`, `refund.*`, `dispute.*`
**Checkout:** `checkout.session.*`
**Billing:** `customer.subscription.*`, `invoice.*`, `invoiceitem.*`
**Payment Methods:** `payment_method.*`, `setup_intent.*`
**Connect:** `account.*`, `transfer.*`

## Local Webhook Testing

### Stripe CLI

```bash
# Forward events to local server
stripe listen --forward-to http://localhost:8000/stripe/webhook/<uuid>/

# Trigger specific events
stripe trigger customer.created
stripe trigger checkout.session.completed
stripe trigger invoice.payment_succeeded
stripe trigger customer.subscription.deleted
```

### Stripe CLI Secret Handling

The Stripe CLI signs events with its own secret. Pass it via header:

```bash
stripe listen \
  --forward-to http://localhost:8000/stripe/webhook/<uuid>/ \
  -H "x-djstripe-webhook-secret: $(stripe listen --print-secret)"
```

### Legacy: Management Command (dj-stripe < 2.10)

```bash
python manage.py stripe_listen
```

### Docker Setup

```yaml
# docker-compose.yml
services:
  stripe-cli:
    image: stripe/stripe-cli
    command: listen --forward-to http://web:8000/stripe/webhook/<uuid>/ --api-key ${STRIPE_TEST_SECRET_KEY}
    environment:
      - STRIPE_API_KEY=${STRIPE_TEST_SECRET_KEY}
```

## Error Handling and Retries

### Stripe Retry Behavior

- If your webhook returns 2xx: event is marked as delivered
- If your webhook returns non-2xx: Stripe retries with exponential backoff
- Stripe retries for up to 3 days

### Handling Errors in Handlers

```python
@djstripe_receiver("checkout.session.completed")
def handle_checkout(sender, event: Event, **kwargs):
    try:
        order = fulfill_order(event.data["object"])
    except OrderNotFoundError:
        logger.error(f"Order not found for event {event.id}")
        return  # Returning without raising → 200 response → Stripe stops retrying
    except TemporaryError:
        # Retryable — let exception propagate for Stripe to retry
        raise
```

### Reprocessing Failed Events

```bash
# Reprocess all events
python manage.py djstripe_process_events

# Only failed events
python manage.py djstripe_process_events --failed

# Events of specific types
python manage.py djstripe_process_events --type payment_intent.*

# Specific event IDs
python manage.py djstripe_process_events --ids evt_xxx evt_yyy
```

**Note:** Events are only available in Stripe API for 30 days.

## Security Checklist

- [ ] Webhook endpoints use UUID URLs (not guessable)
- [ ] `djstripe_validation_method` set to `verify_signature` in production
- [ ] `djstripe_tolerance` set to 300 (5 min) minimum in production
- [ ] `whsec_` secret stored in environment variables, not code
- [ ] All handlers are idempotent (event dedup or state check)
- [ ] Webhook retry behavior understood and handled
- [ ] No secrets logged in webhook processing
- [ ] Monitoring/alerting on webhook errors (via `webhook_processing_error` signal)

## Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Non-idempotent handler | Double fulfillment, duplicate emails | Dedup by event ID or object state |
| Not importing webhooks in AppConfig | Handlers silently ignored | `import myapp.webhooks` in `ready()` |
| Depending on handler ordering | Works in dev, fails in prod | Retrieve objects by ID, don't rely on dj-stripe sync order |
| Assuming webhook data is latest | Stripe payload may be slightly stale | Use `api_retrieve()` for critical checks |
| Swallowing all exceptions | Stripe stops retrying, event lost | Only catch non-retryable errors |
| Long-running handlers | Timeout, blocking other webhooks | Delegate to async tasks if processing is slow |
| Disabling verification via `djstripe_tolerance = 0` | Verification still happens but rejects all due to clock skew | Use `djstripe_validation_method = "none"` to disable |
