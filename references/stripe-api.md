# Stripe API — Integration Patterns, Versioning, and Best Practices

Reference for direct Stripe API integration alongside dj-stripe. Covers API
versioning, integration surface selection, Connect, and platform patterns.

## Stripe API Version

> **Terminology note:** "v2" in dj-stripe contexts can refer to (1) the legacy Stripe Checkout.js v2 (deprecated), (2) the Stripe REST API v2 (new, for Connect accounts). Always clarify which v2 you mean.

**Latest version: 2026-04-22.dahlia**

Stripe uses date-based API versions. Each version has a codename:
- `2026-04-22.dahlia` (latest)
- `2025-08-27.basil`
- `2024-12-18.acacia`

### Version Behavior

| Change Type | Examples | Requires Code Update? |
|---|---|---|
| New resources | New `ProductFeature` model | No |
| New optional params | `trial_period_days` on subscription | No |
| New response fields | New field in webhook payload | No |
| Field renames | `statement_descriptor` → `statement_descriptor_suffix` | **Yes** |
| Field removals | Deprecated field removed | **Yes** |
| Behavioral changes | Different default for `proration_behavior` | **Yes** |

### Version with dj-stripe

**CRITICAL:** dj-stripe pins `STRIPE_API_VERSION` to its tested version
(currently `2020-08-27`). Do NOT override this. dj-stripe re-fetches data
at its pinned version regardless of your account's default.

For direct API calls (not through dj-stripe models), use the latest version:

```python
import stripe
stripe.api_version = "2026-04-22.dahlia"
```

### Idempotency Keys

dj-stripe manages idempotency keys via the `IdempotencyKey` model (`djstripe/models/core.py`). The key is generated as `{object_type}:{action}` with a UUID:

```python
# Automatic (dj-stripe handles this internally)
customer, created = Customer.get_or_create(subscriber=user)

# Manual
from djstripe.models import IdempotencyKey
key, _ = IdempotencyKey.objects.get_or_create(
    action="customer:create", livemode=False
)
```

**Cleanup:** Expired keys can be purged:
```bash
python manage.py djstripe_clear_expired_idempotency_keys
```

## Integration Surface Selection

```
What are you building?
│
├── One-time payments
│   ├── Hosted page → Checkout Sessions (easiest)
│   └── Custom UI → Payment Element + PaymentIntents
│
├── Subscriptions
│   ├── Hosted → Checkout Sessions (mode="subscription")
│   ├── Custom UI → Payment Element + Subscriptions API
│   └── Customer portal → Billing Portal Session
│
├── Marketplace / Platform
│   ├── Standalone accounts → Connect Standard
│   ├── Custom onboarding → Connect Express / Custom
│   └── v2 API access → /v2/core/accounts
│
├── Save payment method for later → Setup Intents
│
├── Invoicing → Invoices API + Customer Portal
│
└── Financial services → Treasury (v2)
```

## Version-Specific API Calls

### Using Latest Version Per-Request

```python
# Per-request override (dynamically typed languages only)
stripe.Customer.create(
    email="customer@example.com",
    stripe_version="2026-04-22.dahlia",
)
```

### Strongly-Typed SDKs

Java, Go, .NET SDKs use a fixed API version matching the SDK release date.
Don't set a different API version. Update the SDK to target a new version.

## Stripe Connect Patterns

### Platform Types

| Platform Type | Description | API |
|---|---|---|
| Standard | Stripe-hosted onboarding | Connect Standard |
| Express | Custom onboarding, Stripe dashboard | Connect Express |
| Custom | Full control, no Stripe dashboard | Connect Custom |

### Account Creation

```python
# Create connected account
account = stripe.Account.create(
    type="express",
    country="US",
    email="merchant@example.com",
    capabilities={
        "card_payments": {"requested": True},
        "transfers": {"requested": True},
    },
)

# Create onboarding link
link = stripe.AccountLink.create(
    account=account.id,
    refresh_url="https://example.com/reauth",
    return_url="https://example.com/return",
    type="account_onboarding",
)
```

### Platform Fees

```python
# Create payment with application fee
intent = stripe.PaymentIntent.create(
    amount=10000,  # $100.00
    currency="usd",
    application_fee_amount=1000,  # $10.00 fee
    transfer_data={"destination": "acct_xxx"},
)
```

### v2 Accounts API

```python
# v2 API (newer, recommended for new integrations)
# Uses /v2/core/accounts endpoint
import stripe
    stripe.api_version = "2026-04-22.dahlia"

account = stripe.Account.retrieve(
    "acct_xxx",
    stripe_version="2026-04-22.dahlia",
)
```

> **Note:** v2 API calls go through the Stripe SDK directly. dj-stripe models use the v1 API exclusively. For v2 features, use `stripe.Account` v2 endpoints directly — data will NOT be synced to dj-stripe models.

## Billing Patterns

### Products and Prices

```python
# Create product
product = stripe.Product.create(
    name="Pro Plan",
    description="Professional tier with advanced features",
)

# Create price
price = stripe.Price.create(
    product=product.id,
    unit_amount=2900,  # $29.00
    currency="usd",
    recurring={"interval": "month"},
)
```

### Checkout Sessions for Subscriptions

```python
session = stripe.checkout.Session.create(
    mode="subscription",
    line_items=[{"price": "price_xxx", "quantity": 1}],
    subscription_data={
        "trial_period_days": 14,
        "metadata": {"plan": "pro"},
    },
    metadata={"djstripe_subscriber": str(request.user.id)},
    success_url="...",
    cancel_url="...",
)
```

### Customer Portal

```python
# Let customers manage their own subscriptions
portal = stripe.billing_portal.Session.create(
    customer="cus_xxx",
    return_url="https://example.com/account",
)
# Redirect to portal.url
```

## Payment Patterns

### Checkout Sessions (Payment Mode)

```python
session = stripe.checkout.Session.create(
    mode="payment",
    line_items=[{
        "price_data": {
            "currency": "usd",
            "product_data": {"name": "Widget"},
            "unit_amount": 2000,  # $20.00
        },
        "quantity": 1,
    }],
    success_url="...",
    cancel_url="...",
)
```

### Payment Element (Custom UI)

```python
# Server-side
intent = stripe.PaymentIntent.create(
    amount=2000,
    currency="usd",
    automatic_payment_methods={"enabled": True},
)
# Send client_secret to frontend
```

```javascript
// Client-side
const stripe = Stripe("pk_test_...");
const elements = stripe.elements();
const paymentElement = elements.create("payment");
paymentElement.mount("#payment-element");

await stripe.confirmPayment({
  elements,
  clientSecret,
  confirmParams: {
    return_url: "https://example.com/complete",
  },
});
```

## Webhook Event Handling

### Essential Events

```python
WEBHOOK_EVENTS = [
    "checkout.session.completed",
    "checkout.session.async_payment_succeeded",
    "customer.subscription.created",
    "customer.subscription.updated",
    "customer.subscription.deleted",
    "invoice.payment_succeeded",
    "invoice.payment_failed",
    "payment_intent.succeeded",
    "payment_intent.payment_failed",
    "charge.refunded",
    "charge.dispute.created",
]
```

### Webhook Best Practices

1. Handle unfamiliar event types gracefully (log, don't crash)
2. Verify signatures in production
3. Make handlers idempotent
4. Return 2xx quickly (delegate slow work to background tasks)
5. Test with Stripe CLI before deploying

When checking what changed in an update webhook, access `previous_attributes` through the nested `data` key:

```python
# Correct path in dj-stripe's Event model:
# Event.data stores the `data` sub-object (not the full event envelope).
# previous_attributes sits alongside `object` inside `data`.
previous = event.data.get("previous_attributes", {})
```

## Treasury / Financial Accounts

> **Note:** v2 API calls go through the Stripe SDK directly. dj-stripe models use the v1 API exclusively. For v2 features, use `stripe.Account` v2 endpoints directly — data will NOT be synced to dj-stripe models.

## SDK Management

### Python SDK

```bash
pip install --upgrade stripe
```

```python
import stripe
stripe.api_key = "sk_test_..."
stripe.api_version = "2026-04-22.dahlia"
```

### Stripe.js

```html
<!-- Load Stripe.js (recommended: always use the latest version) -->
<script src="https://js.stripe.com/v3/"></script>
```

Stripe.js uses a rolling release model — the `v3` URL always serves the latest stable version. API version compatibility is handled server-side by pinning `Stripe-Version` headers, not by loading specific JS versions.

### Mobile SDKs

- iOS: `pod 'Stripe'` or Swift Package Manager
- Android: `implementation 'com.stripe:stripe-android:VERSION'`
- React Native: `@stripe/stripe-react-native`

All mobile SDKs work with any API version on your backend.

### Restricted API Keys (Recommended)

Use restricted API keys (prefix `rk_`) instead of full secret keys (`sk_`) whenever possible. dj-stripe supports all three key types:

| Prefix | Type | Usage |
|--------|------|-------|
| `pk_` | Publishable | Client-side (Stripe.js) |
| `sk_` | Secret | Full server-side access |
| `rk_` | Restricted | Limited permissions (recommended for production) |

Keys are configured via Django settings, not `stripe.api_key`:
```python
STRIPE_LIVE_SECRET_KEY = "rk_live_xxx"  # Restricted key for production
STRIPE_TEST_SECRET_KEY = "rk_test_xxx"
```

### Rate Limits & Retries

Stripe API enforces rate limits. dj-stripe handles retries for webhook events by wrapping processing in `transaction.atomic()`. If any receiver raises, the transaction rolls back, dj-stripe returns 500, and Stripe retries with exponential backoff (up to 3 days).

For invoice retries:
```python
invoice.retry()  # Re-attempt payment on a failed invoice
```

For manual API calls, handle `stripe.error.RateLimitError`:
```python
import time
from stripe.error import RateLimitError

def api_call_with_retry(fn, *args, max_retries=3, **kwargs):
    for attempt in range(max_retries):
        try:
            return fn(*args, **kwargs)
        except RateLimitError:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
```

## Testing Stripe API

### Test Mode

Use `sk_test_` keys for all development:
- No real money changes hands
- Test card numbers: `4242424242424242` (always succeeds)
- Test webhooks: `stripe trigger <event>`
- Test clock: simulate time-based events

### Test Card Numbers

| Card Number | Result |
|---|---|
| `4242424242424242` | Always succeeds |
| `4000000000000002` | Charge declined |
| `4000000000003220` | 3D Secure required |
| `4000000000009995` | Always declined (insufficient funds) |

### Stripe CLI Testing

```bash
# Start test listener
stripe listen --forward-to http://localhost:8000/stripe/webhook/<uuid>/

# Trigger events
stripe trigger payment_intent.succeeded
stripe trigger customer.subscription.created
stripe trigger checkout.session.completed

# Test with specific API version
stripe trigger --api-version 2026-04-22.dahlia payment_intent.created
```

## API Reference

### Key Documentation URLs

- [API Reference](https://docs.stripe.com/api)
- [API Changelog](https://docs.stripe.com/changelog)
- [Upgrade Guide](https://docs.stripe.com/upgrades)
- [Integration Options](https://docs.stripe.com/payments/payment-methods/integration-options)
- [Go-Live Checklist](https://docs.stripe.com/get-started/checklist/go-live)

### Searching Stripe Docs

```python
# Use the search tool for specific questions
stripe_search_stripe_documentation(
    question="How to set up subscription schedules",
    language="python",
)
```

## Common Integration Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Using account default API version | Behavior changes unexpectedly on upgrade | Pin explicit API version |
| Not updating API version before SDK | SDK may not support new features | Update SDK first, then API version |
| Hardcoding test card numbers in code | Tests break if card behavior changes | Use test mode tokens dynamically |
| Not handling `payment_intent.processing` | Payments appear stuck | Handle intermediate processing state |
| Ignoring `previous_attributes` in webhooks | Can't detect what changed | Check `event.data.get("previous_attributes", {})` (sits alongside `object` inside the `data` sub-object) |
| Using v1 API for new Connect features | Missing functionality | Use /v2/core/accounts for new integrations |
