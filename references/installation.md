# Installation and Configuration

## Requirements

| Dependency | Minimum Version | Recommended |
|---|---|---|
| Python | 3.8 | 3.12+ |
| Django | 4.2 | 5.1+ |
| dj-stripe | 2.9.x | latest 2.x |
| PostgreSQL | 12 | 16+ |
| stripe-python | >= 11.0 | latest |

- MariaDB >= 10.5 or MySQL >= 8.0 also supported
- SQLite for development only (not production; JSONField queries are limited)

## Installation

```bash
pip install dj-stripe
```

> **Note:** Do NOT install `jsonfield` or `jsonfield2` packages alongside dj-stripe 2.10+. These are incompatible and will cause `ImportError`. dj-stripe uses Django's native JSONField.

## Setup Steps

### 1. Add to INSTALLED_APPS

```python
INSTALLED_APPS = [
    # ...
    "djstripe",
]
```

### 2. Add URL Configuration

```python
# urls.py
urlpatterns = [
    # ...
    path("stripe/", include("djstripe.urls", namespace="djstripe")),
]
```

This provides:
- `/stripe/webhook/<uuid:uuid>/` — Webhook endpoint
- Admin views for managing Stripe objects

### 3. Configure Settings

**Required:**
```python
STRIPE_LIVE_SECRET_KEY = os.environ.get("STRIPE_LIVE_SECRET_KEY", "")
STRIPE_TEST_SECRET_KEY = os.environ.get("STRIPE_TEST_SECRET_KEY", "")
STRIPE_LIVE_MODE = False  # True in production
DJSTRIPE_FOREIGN_KEY_TO_FIELD = "id"  # Use Stripe IDs as FKs
```

**Important — Do NOT set `STRIPE_API_VERSION`:**
dj-stripe pins it to its internally tested version. Changing it causes data
mismatches between webhook payloads and model schemas.

### 4. Add API Keys

Add your Stripe API keys via Django Admin at `/admin/djstripe/apikey/`. dj-stripe requires keys to exist in the database before syncing.

### 5. Run Migrations

```bash
python manage.py migrate
```

### 6. Initial Data Sync

```bash
python manage.py djstripe_sync_models
```

This fetches all existing Stripe objects (products, prices, customers, etc.)
and creates local database records.

### 7. Configure Webhooks

1. In Django Admin, go to `/admin/djstripe/webhookendpoint/add/`
2. Choose the Stripe account (or leave blank for default API key)
3. Select test mode or live mode
4. Set a real domain as the base URL (Stripe rejects `localhost`)
5. Save — dj-stripe creates the endpoint with a UUID-based URL in Stripe automatically
6. For local testing, use the Stripe CLI (see step 8)

### 8. Local Testing Setup

```bash
# Forward Stripe events to your local server
stripe listen --forward-to http://localhost:8000/stripe/webhook/<uuid>/

# Trigger test events
stripe trigger customer.created
stripe trigger checkout.session.completed
stripe trigger invoice.payment_succeeded
```

For signature verification in local dev:
```bash
stripe listen \
  --forward-to http://localhost:8000/stripe/webhook/<uuid>/ \
  -H "x-djstripe-webhook-secret: $(stripe listen --print-secret)"
```

## Complete Settings Reference

### Required

| Setting | Type | Description |
|---|---|---|
| `STRIPE_LIVE_SECRET_KEY` | str | Live mode secret key (`sk_live_`) |
| `STRIPE_TEST_SECRET_KEY` | str | Test mode secret key (`sk_test_`) |
| `STRIPE_LIVE_MODE` | bool | `True` in production, `False` otherwise |
| `DJSTRIPE_FOREIGN_KEY_TO_FIELD` | str | `"id"` for new installs, `"djstripe_id"` for legacy |

> **Note:** `STRIPE_LIVE_MODE` must be a Python boolean. When set via environment variables, use `bool(int(os.environ.get("STRIPE_LIVE_MODE", "0")))` or similar conversion, as `os.environ.get()` returns strings.

### Optional

| Setting | Default | Description |
|---|---|---|
| `DJSTRIPE_SUBSCRIBER_CUSTOMER_KEY` | `"djstripe_subscriber"` | Metadata key linking Checkout customers to local users |
| `DJSTRIPE_SUBSCRIBER_MODEL` | `AUTH_USER_MODEL` | Custom subscriber model (must have `email` field) |
| `DJSTRIPE_WEBHOOK_EVENT_CALLBACK` | `None` | Callback for custom event handling (e.g., push to Celery queue) |
| `DJSTRIPE_WEBHOOK_VALIDATION` | `"verify_signature"` | Webhook signature validation; per-endpoint `djstripe_validation_method` preferred |
| `STRIPE_API_VERSION` | `"2020-08-27"` (set by dj-stripe) | **DO NOT CHANGE** |

> **WARNING:** `STRIPE_API_VERSION` is a dj-stripe internal setting. DO NOT override it in your Django settings. dj-stripe pins it to its tested version and changing it will break sync and webhook processing.

### Deprecated / Removed

**Deprecated (still functional but replaced):**
- `DJSTRIPE_WEBHOOK_URL` — deprecated since 2.7. Use database-stored WebhookEndpoint instead.
- `DJSTRIPE_WEBHOOK_VALIDATION` — deprecated since 2.9. Use per-endpoint `djstripe_validation_method` on WebhookEndpoint instead.

**Removed:**
- `DJSTRIPE_WEBHOOK_SECRET` — removed. Webhook secrets are per-endpoint in the database.
- `DJSTRIPE_WEBHOOK_TOLERANCE` — removed.
- `DJSTRIPE_SUBSCRIPTION_REDIRECT` — removed in 2.7.
- `DJSTRIPE_PRORATION_POLICY` — deprecated in 2.6, removed in 2.8.

## Environment Variables Template

```bash
# .env (never commit to version control)
# Standard secret keys:
# STRIPE_LIVE_SECRET_KEY=sk_live_xxxxxxxxxxxxxxxxxxxx
# STRIPE_TEST_SECRET_KEY=sk_test_xxxxxxxxxxxxxxxxxxxx

# Or restricted keys (recommended for production):
STRIPE_LIVE_SECRET_KEY=rk_live_xxxxxxxxxxxxxxxxxxxx
STRIPE_TEST_SECRET_KEY=rk_test_xxxxxxxxxxxxxxxxxxxx
```

## Database Considerations

### PostgreSQL (Recommended)

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "mydb",
        "USER": "mydbuser",
        "PASSWORD": "mypassword",
        "HOST": "localhost",
        "PORT": "5432",
    }
}
```

PostgreSQL is required for:
- `stripe_data` JSONField path queries (`stripe_data__field__subfield`)
- `Cast()` annotations for numeric JSONField field aggregations
- Reliable webhook transaction handling

### SQLite (Development Only)

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": BASE_DIR / "db.sqlite3",
    }
}
```

SQLite >= 3.26 required. JSONField queries work but are limited.

## Verification Commands

```bash
# Verify installation
python manage.py check

# Verify Stripe connectivity
python manage.py shell -c "
import stripe
stripe.api_key = 'sk_test_...'
print(stripe.Account.retrieve().email)
"

# Sync and verify models
python manage.py djstripe_sync_models Product Price
python manage.py shell -c "
from djstripe.models import Product
print(Product.objects.count())
"
```
