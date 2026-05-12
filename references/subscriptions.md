# Subscriptions and Billing

Complete guide to subscription management, invoicing, and recurring billing
with dj-stripe and Django.

## Subscription Lifecycle

```
                       ┌──────────────┐
      ┌────────────────►│    active     │◄──────────────┐
      │                 └──────┬───────┘               │
      │                        │                        │
      │       ┌────────────────┼────────────────┐       │
      │       │                │                │       │
      │  ┌────▼─────┐   ┌──────▼──────┐  ┌──────▼──────┐
      │  │  past_due  │   │   paused    │  │  trialing   │
      │  └────┬─────┘   └──────┬──────┘  └──────┬──────┘
      │       │                │                │
      │  ┌────▼─────┐         │                │
      │  │   unpaid   │        │                │
      │  └────┬─────┘         │                │
      │       │                │                │
      │  ┌────▼─────┐         │                │
      └──┤ canceled  │◄────────┘────────────────┘
         └──────────┘

  incomplete ──► incomplete_expired (initial payment failed)
```

The full `SubscriptionStatus` enum includes 8 states: `incomplete`,
`incomplete_expired`, `trialing`, `active`, `paused`, `past_due`, `canceled`, `unpaid`.
Stripe manages subscription state. dj-stripe syncs it. Your app reacts to state
changes via webhooks.

## Creating Subscriptions

### Single Price Subscription

```python
from djstripe.models import Customer, Price

customer = Customer.objects.get(subscriber=user)
price = Price.objects.get(nickname="Pro Plan")

subscription = customer.subscribe(
    price=price,
    trial_period_days=14,
)
```

### Multi-Item Subscription

```python
base_price = Price.objects.get(lookup_key="base_monthly")
seat_price = Price.objects.get(lookup_key="seat_monthly")
storage_price = Price.objects.get(lookup_key="storage_gb")

subscription = customer.subscribe(items=[
    {"price": base_price},
    {"price": seat_price, "quantity": 5},
    {"price": storage_price},  # Metered — quantity is null
])
```

> **WARNING:** `Customer.subscribe()` only processes the `"price"` and `"plan"` keys from each item dict. `quantity`, `tax_rates`, and `metadata` are silently ignored. For multi-quantity or taxed subscriptions, use `stripe.Subscription.create()` directly.

### With Trial Period

```python
subscription = customer.subscribe(
    items=[{"price": price}],
    trial_period_days=30,
)
```

### With Coupon/Discount

```python
from djstripe.models import Coupon

coupon = Coupon.objects.get(id="summer2026")
subscription = customer.subscribe(
    items=[{"price": price}],
    coupon=coupon,
)
```

### Arbitrary Stripe Parameters

```python
subscription = customer.subscribe(
    items=[{"price": price}],
    metadata={"plan_source": "upgrade_modal", "campaign": "spring2026"},
    payment_behavior="default_incomplete",
    collection_method="send_invoice",
    days_until_due=30,
)
```

## Managing Subscriptions

### Updating Subscriptions

```python
# Change plan
new_price = Price.objects.get(lookup_key="enterprise_monthly")

subscription.update(
    items=[{
        "id": subscription.items.first().id,  # Existing item to modify
        "price": new_price,
        "deleted": True,  # price on deleted items is ignored by Stripe
    }, {
        "price": new_price,  # Add new item
    }],
    proration_behavior="create_prorations",  # always_invoice | none | create_prorations
)
```

### Canceling Subscriptions

```python
# Cancel immediately
subscription.cancel(at_period_end=False)

# Cancel at end of billing period
subscription.cancel(at_period_end=True)

# With custom parameters
subscription.cancel(
    at_period_end=True,
    cancellation_details={"comment": "Customer requested via support"},
)
```

### Reactivating Subscriptions

```python
# Reactivate a canceled (but not yet ended) subscription
subscription.reactivate()
```

### Extending Trials

```python
from datetime import timedelta

# Extend trial by 30 days
subscription.extend(timedelta(days=30))
```

**Warning:** Uses `trial_end` manipulation. Subscription status may change to
`trialing`. This is a workaround — prefer using trial periods at creation.

### Checking Subscription Status

```python
customer = Customer.objects.get(subscriber=user)
subscription = customer.subscription

if subscription and subscription.status == "active":
    # Active subscriber
    pass

# Check specific product
has_access = SubscriptionItem.objects.filter(
    subscription__customer=customer,
    subscription__status="active",
    price__product=product,
).exists()
if has_access:
    # Has access to this product
    pass
```

## Webhooks for Subscriptions

```python
from djstripe.event_handlers import djstripe_receiver
from djstripe.models import Event, Customer

@djstripe_receiver([
    "customer.subscription.created",
    "customer.subscription.updated",
    "customer.subscription.deleted",
    "customer.subscription.paused",
    "customer.subscription.resumed",
    "customer.subscription.trial_will_end",
])
def handle_subscription_change(sender, event: Event, **kwargs):
    subscription_data = event.data["object"]
    subscription_id = subscription_data["id"]
    customer_id = subscription_data["customer"]
    status = subscription_data["status"]

    customer = Customer.objects.get(id=customer_id)
    user = customer.subscriber

    if status == "active":
        activate_features(user)
    elif status == "past_due":
        notify_payment_issue(user)
    elif status in ("canceled", "unpaid"):
        deactivate_features(user)
```

## Invoices

### Invoice Lifecycle

```
draft ──► open ──► paid
  │         │         ▲
  │         │         │
  ├── void   ├── uncollectible ──┘
  │          │         │
  │          └── void  │
  └────────────────────┘
```

Transitions:
- `draft → open → paid` (happy path)
- `draft → void` (canceled before sending)
- `open → uncollectible` (recovery failed)
- `open → void` (canceled while open)
- `uncollectible → paid` (payment received after marked)
- `uncollectible → void` (canceled after marked uncollectible)

### Handling Invoice Webhooks

```python
@djstripe_receiver("invoice.payment_succeeded")
def handle_invoice_paid(sender, event: Event, **kwargs):
    invoice_data = event.data["object"]
    customer_id = invoice_data["customer"]
    amount_paid = invoice_data["amount_paid"]
    invoice_number = invoice_data["number"]

    customer = Customer.objects.get(id=customer_id)

    # Record payment in your app
    PaymentRecord.objects.create(
        customer=customer,
        amount=amount_paid,
        invoice_number=invoice_number,
    )

@djstripe_receiver("invoice.payment_failed")
def handle_invoice_failed(sender, event: Event, **kwargs):
    invoice_data = event.data["object"]
    customer_id = invoice_data["customer"]

    customer = Customer.objects.get(id=customer_id)
    user = customer.subscriber

    # Notify user to update payment method
    send_payment_failed_email(user, invoice_data["hosted_invoice_url"])
```

### Creating Manual Invoices

```python
import stripe

# Create invoice via Stripe API
invoice = stripe.Invoice.create(
    customer=customer.id,
    collection_method="send_invoice",
    days_until_due=30,
)

# Add line items
stripe.InvoiceItem.create(
    customer=customer.id,
    invoice=invoice.id,
    price=price.id,
    quantity=1,
)

# Finalize and send
invoice.finalize_invoice()
```

## Pricing Models

### Fixed Price

```python
stripe.Price.create(product=product.id, unit_amount=2900, currency="usd", recurring={"interval": "month"})
```

### Per-Seat (Quantity-Based)

```python
stripe.Price.create(
    product=product.id,
    unit_amount=1000,  # $10.00 per seat
    currency="usd",
    recurring={"interval": "month"},
)
# Quantity set on SubscriptionItem
```

### Metered Usage

```python
stripe.Price.create(
    product=product.id,
    unit_amount=5,  # $0.05 per unit
    currency="usd",
    recurring={"interval": "month", "usage_type": "metered"},
)
# Quantity is null — Stripe tracks usage
```

### Tiered Pricing

```python
stripe.Price.create(
    product=product.id,
    currency="usd",
    recurring={"interval": "month"},
    billing_scheme="tiered",
    tiers_mode="graduated",  # or "volume"
    tiers=[
        {"up_to": 10, "unit_amount": 1000},    # $10 each for first 10
        {"up_to": 100, "unit_amount": 800},    # $8 each for next 90
        {"up_to": "inf", "unit_amount": 500},  # $5 each beyond 100
    ],
)
```

## Querying Subscriptions

### From Customer Object

```python
customer = Customer.objects.get(subscriber=user)

# Current active subscription
subscription = customer.subscription

# All subscriptions (active, past, canceled)
all_subs = Subscription.objects.filter(customer=customer)
```

### Filtering by Status

```python
active_subs = Subscription.objects.filter(status="active")
canceled_subs = Subscription.objects.filter(status="canceled")
past_due_subs = Subscription.objects.filter(status="past_due")
```

### Checking Product Access

```python
# Check via SubscriptionItem
has_access = SubscriptionItem.objects.filter(
    subscription__customer=customer,
    subscription__status="active",
    price__product=product,
).exists()
```

## Invoice Querying

```python
# All invoices for a customer
invoices = Invoice.objects.filter(customer=customer)

# Paid invoices
paid = Invoice.objects.filter(customer=customer, status="paid")

# Unpaid invoices
unpaid = Invoice.objects.filter(customer=customer, status="open")

# Recent invoices
from django.utils import timezone
recent = Invoice.objects.filter(
    customer=customer,
    created__gte=timezone.now() - timedelta(days=30),
)
```

## Upcoming Invoice

```python
from djstripe.models import Invoice

upcoming = Invoice.upcoming(customer=customer)
if upcoming:
    print(f"Next charge: {upcoming.total} on {upcoming.period_end}")
```

## Tax Support

### Tax Rates

```python
# Tax rates are synced from Stripe
from djstripe.models import TaxRate, TaxCode

# Apply tax to subscription (use Stripe API directly since Customer.subscribe()
# silently ignores tax_rates/metadata/quantity in items dict)
import stripe
subscription_obj = stripe.Subscription.create(
    customer=customer.id,
    items=[{"price": price.id, "tax_rates": [tax_rate.id]}],
)
```

### Customer Tax IDs

Tax IDs are synced to `TaxId` model and linked to `Customer`.

### Tax-Exempt Customers

```python
# On Customer model
if customer.tax_exempt == "exempt":
    # Don't charge tax
    pass
```

## Subscription Schedules

For planned subscription changes (e.g., upgrade at end of billing period):

```python
import stripe

schedule = stripe.SubscriptionSchedule.create(
    customer=customer.id,
    start_date="now",
    end_behavior="release",
    phases=[
        {
            "items": [{"price": monthly_price.id, "quantity": 1}],
            "iterations": 3,  # 3 months
        },
        {
            "items": [{"price": annual_price.id, "quantity": 1}],
            "iterations": 1,  # then switch to annual
        },
    ],
)
```

## Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Polling subscription status in views | Slow, hits Stripe API limits | Read from local DB, react to webhooks |
| Not handling `past_due` → `active` transitions | User stuck without access after fixing payment | Handle all status transitions in webhook |
| Hard-deleting user data on cancel | Can't reactivate subscription | Soft-delete: deactivate features, keep record |
| Extending trial via `trial_end` for active subs | Status changes to `trialing` | Use subscription schedules for planned changes |
| Not checking for null subscription | `customer.subscription` can be None | Always check before accessing |
