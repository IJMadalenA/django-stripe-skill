# Synchronization and Data Management

How dj-stripe syncs Stripe data to your local database, how to query it, and
how to manage the sync lifecycle.

## Sync Architecture

```
Stripe API                     Your Django App
──────────                     ───────────────
                               ┌─────────────────┐
  Products ◄── sync ──────────►│ Product (local)  │
  Prices   ◄── sync ──────────►│ Price (local)    │
  Customers◄── sync ──────────►│ Customer (local) │
  Subscriptions ◄── sync ─────►│ Subscription     │
  Invoices ◄── sync ──────────►│ Invoice (local)  │
  Charges  ◄── sync ──────────►│ Charge (local)   │
                               └─────────────────┘
```

Two sync mechanisms:
1. **Initial/Manual sync:** `djstripe_sync_models` management command or `sync_from_stripe_data()`
2. **Ongoing sync:** Webhooks automatically sync objects when Stripe events arrive

## Initial Data Sync

### Sync All Models

```bash
python manage.py djstripe_sync_models
```

This is required after first installation. It fetches all existing Stripe objects
and creates local database records.

### Sync Specific Models

```bash
python manage.py djstripe_sync_models Product Price
python manage.py djstripe_sync_models Invoice Subscription Customer
```

Use when you add a new model to your app that references a dj-stripe model.

### Sync with Specific API Keys

```bash
python manage.py djstripe_sync_models Customer --api-keys sk_test_XXX sk_test_YYY
```

Useful for multi-account setups or syncing data from specific keys.

## Programmatic Sync

### Syncing Individual Objects

```python
import stripe
from djstripe.models import Product

# Create on Stripe, sync locally
product_data = stripe.Product.create(name="My Product")
product = Product.sync_from_stripe_data(product_data)

# Update on Stripe, sync locally
stripe.Product.modify(product.id, name="Updated Name")
updated = Product.sync_from_stripe_data(stripe.Product.retrieve(product.id))
```

### Retrieving Latest from Stripe

```python
customer = Customer.objects.get(id="cus_xxx")
customer.sync_from_stripe_data(customer.api_retrieve())  # Fetch latest from Stripe and save to DB
```

### Bootstrapping Customers for Existing Subscribers

```bash
python manage.py djstripe_sync_customers
```

Creates Stripe Customer objects (via `get_or_create`) and syncs their full
Stripe data (subscriptions, invoices, cards, charges) for local subscribers
that don't yet have an associated Stripe customer.

### Initializing Customers for Existing Users

```bash
python manage.py djstripe_init_customers
```

Creates Stripe Customer objects for users who don't have them yet (does not
sync subscriptions/invoices — use `djstripe_sync_customers` for that).

## The `stripe_data` JSONField

Since dj-stripe 2.10, most model fields were moved from concrete database columns
to the `stripe_data` JSONField. This stores the complete Stripe API response.

### Reading stripe_data

```python
charge = Charge.objects.get(id="ch_xxx")

# Access via @property (automatic)
amount = charge.amount  # Still works — @property reads from stripe_data

# Direct JSONField access
charge.stripe_data["amount"]  # 2900 (cents)
charge.stripe_data["currency"]  # "usd"
charge.stripe_data["receipt_url"]  # "https://..."
```

### Filtering with stripe_data

```python
# JSON path lookups (PostgreSQL recommended)
charges = Charge.objects.filter(stripe_data__currency="usd")
charges = Charge.objects.filter(stripe_data__receipt_number="1234-5678")

# Nested lookups
charges = Charge.objects.filter(stripe_data__billing_details__name="John")
```

### Aggregations with stripe_data

```python
from django.db.models import Sum, Avg, FloatField
from django.db.models.functions import Cast

# Cast JSONField values for numeric operations
total_revenue = Charge.objects.filter(
    status="succeeded"
).annotate(
    amount_float=Cast("stripe_data__amount", FloatField())
).aggregate(
    total=Sum("amount_float")
)

# Average charge amount
avg_charge = Charge.objects.filter(
    status="succeeded"
).annotate(
    amount_float=Cast("stripe_data__amount", FloatField())
).aggregate(
    avg=Avg("amount_float")
)
```

**Requires PostgreSQL.** SQLite JSONField support is limited.

### Migrations Affecting stripe_data

Since 2.10, model fields that were moved to `stripe_data` require JSON path
lookups instead of direct field access. Check whether a field is concrete or
in `stripe_data` before writing queries:

```python
# For fields in stripe_data (e.g., receipt_number, billing_details):
Charge.objects.filter(stripe_data__receipt_number="1234-5678")

# For fields still concrete (e.g., status, customer FK):
Charge.objects.filter(status="succeeded")
```

## Webhook-Driven Sync

Webhooks keep data in sync automatically:

1. Stripe sends event (e.g., `charge.succeeded`)
2. dj-stripe creates/updates the `Event` model
3. dj-stripe syncs referenced objects (Charge, Customer, PaymentIntent)
4. Your `@djstripe_receiver` handlers fire
5. Stripe receives 2xx response

**No additional sync code needed.** dj-stripe handles object synchronization
before your handlers run.

However, **don't assume dj-stripe has synced a specific object before your handler.**
Always retrieve by ID:

```python
@djstripe_receiver("invoice.payment_succeeded")
def handle(sender, event: Event, **kwargs):
    # Safe: retrieves from DB (dj-stripe synced it)
    customer = Customer.objects.get(id=event.data["object"]["customer"])

    # Also safe: explicit retrieval
    charge_id = event.data["object"]["charge"]
    if charge_id:
        charge = Charge.objects.get(id=charge_id)
```

## Reprocessing Events

### All Events

```bash
python manage.py djstripe_process_events
```

### Failed Events Only

```bash
python manage.py djstripe_process_events --failed
```

### By Event Type

```bash
python manage.py djstripe_process_events --type "payment_intent.*"
python manage.py djstripe_process_events --type "charge.succeeded"
```

### Specific Event IDs

```bash
python manage.py djstripe_process_events --ids evt_xxx evt_yyy
```

**Important:** Events are only guaranteed available in Stripe API for 30 days.
After that, reprocessing from Stripe may fail. Store event data locally if needed
for longer retention.

## Idempotency Key Management

dj-stripe uses Stripe's idempotency keys to prevent duplicate operations. Keys
expire after 24 hours.

```bash
# Clean up expired idempotency keys
python manage.py djstripe_clear_expired_idempotency_keys
```

Keys are stored as `{object_type}:{action}` and are unique per (action, livemode).

## Data Integrity

> **Note:** `djstripe_update_invoiceitem_ids` requires the `--i-understand` flag.
> Without it, the command prints a preview of affected items and exits.

### Never Modify Synced Records

```python
# WRONG — dj-stripe will overwrite on next sync
customer = Customer.objects.get(id="cus_xxx")
customer.email = "new@email.com"
customer.save()

# CORRECT — update via Stripe API
import stripe
stripe.Customer.modify("cus_xxx", email="new@email.com")
# dj-stripe syncs the change automatically via webhook or next api_retrieve()
```

### Deleting Records

Use Stripe API methods, not Django ORM:

```python
# WRONG — local DB only, Stripe not affected
customer.delete()

# CORRECT — Deletes from Stripe, keeps local record (soft delete)
customer.purge()
```

### Foreign Key Integrity

With `DJSTRIPE_FOREIGN_KEY_TO_FIELD = "id"`, Stripe IDs are primary keys:

```python
# dj-stripe model PKs are Stripe IDs
customer = Customer.objects.get(id="cus_xxx")  # Stripe ID lookup (unique field)
charge = Charge.objects.filter(customer_id="cus_xxx")  # FK filter
```

## Multiple Stripe Accounts

dj-stripe supports multiple Stripe accounts via the API key system:

```python
# Keys stored in database, manageable in admin
# Sync with specific keys
python manage.py djstripe_sync_models --api-keys sk_test_XXX sk_test_YYY
```

## Data Migration: From Legacy to Modern Field Access

### Before dj-stripe 2.10 (Concrete Fields)

```python
# Direct field access — some models had all fields concrete
product.name  # Concrete TextField
```

### After dj-stripe 2.10 (JSONField + Properties)

```python
# @property access (same API for most fields)
product.name  # Concrete TextField (still exists for name)
dispute.amount  # @property reads stripe_data["amount"]

# Fields moved to stripe_data require JSON path for filtering
Dispute.objects.filter(stripe_data__amount=1000)

# Aggregation requires Cast for JSONField values
from django.db.models import Sum, FloatField
from django.db.models.functions import Cast
Dispute.objects.annotate(
    amt=Cast("stripe_data__amount", FloatField())
).aggregate(Sum("amt"))
```

### Common Migration Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `FieldError: Cannot resolve keyword 'amount'` | Field moved to stripe_data | Use `stripe_data__amount` |
| `FieldError: Unsupported lookup` | JSONField lookup syntax wrong | Use `stripe_data__field__subfield` |
| TypeError on aggregation | JSONField values are strings | Use `Cast()` to numeric type |
