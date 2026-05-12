# Payments — Checkout, PaymentIntents, and Payment Methods

Guide to processing payments with dj-stripe. Covers Checkout Sessions (hosted),
Payment Intents (custom UI), and payment method management.

## Integration Decision Flow

```
Need to accept payments?
├── Hosted checkout page? ──────► Stripe Checkout Sessions
├── Custom UI with embeddable form? ─► Payment Element + PaymentIntents
├── Subscriptions / recurring billing? → Use Checkout Sessions (mode="subscription") or Stripe Billing API directly. See `subscriptions.md`.
├── Save card for later? ───────► Setup Intents
└── Marketplace/platform? ──────► Stripe Connect
```

## Stripe Checkout (Hosted)

Stripe hosts the payment page. Your app redirects, Stripe handles the rest.

### Creating a Checkout Session

```python
import stripe
from django.urls import reverse
from djstripe.settings import djstripe_settings

checkout_session = stripe.checkout.Session.create(
    mode="payment",  # payment | subscription | setup
    line_items=[{
        "price": price.id,
        "quantity": 1,
    }],
    success_url=request.build_absolute_uri(reverse("checkout_success")),
    cancel_url=request.build_absolute_uri(reverse("checkout_cancel")),
    customer_email=request.user.email,
    metadata={
        djstripe_settings.SUBSCRIBER_CUSTOMER_KEY: str(request.user.id),
        "order_id": str(order.id),
    },
)

return redirect(checkout_session.url)
```

> **Note:** `djstripe_settings.SUBSCRIBER_CUSTOMER_KEY` defaults to `"djstripe_subscriber"`. Use the setting rather than hardcoding the string so your code respects any configuration override.

### Subscription Mode

```python
checkout_session = stripe.checkout.Session.create(
    mode="subscription",
    line_items=[{
        "price": monthly_price.id,
        "quantity": 1,
    }],
    subscription_data={
        "trial_period_days": 14,
        "metadata": {"source": "pricing_page"},
    },
    metadata={djstripe_settings.SUBSCRIBER_CUSTOMER_KEY: str(request.user.id)},
    success_url=...,
    cancel_url=...,
)
```

### Multi-Item Checkout

```python
checkout_session = stripe.checkout.Session.create(
    mode="payment",
    line_items=[
        {"price": main_product.id, "quantity": 1},
        {"price": addon_product.id, "quantity": 2},
        {"price": shipping_price.id, "quantity": 1},
    ],
    metadata={djstripe_settings.SUBSCRIBER_CUSTOMER_KEY: str(request.user.id)},
    ...
)
```

### Handling Checkout Webhooks

```python
from djstripe.event_handlers import djstripe_receiver
from djstripe.models import Event, Customer

@djstripe_receiver("checkout.session.completed")
def handle_checkout_completed(sender, event: Event, **kwargs):
    session_data = event.data["object"]
    metadata = session_data.get("metadata", {})

    # Link to your user
    subscriber_key = djstripe_settings.SUBSCRIBER_CUSTOMER_KEY
    subscriber_id = metadata.get(subscriber_key)
    user = get_user_model().objects.get(id=subscriber_id)

    # Guard: only fulfill if payment is complete
    if session_data.get("payment_status") != "paid":
        return

    # Fulfill order
    order_id = metadata.get("order_id")
    order = Order.objects.get(id=order_id)

    mode = session_data["mode"]
    if mode == "payment":
        order.fulfilled = True
        order.stripe_payment_intent = session_data.get("payment_intent")
        order.save()
        grant_access(user, order.product)
    elif mode == "subscription":
        # Subscription is auto-created and synced by dj-stripe
        grant_subscription_access(user)
    elif mode == "setup":
        # Handle setup mode sessions (e.g., save payment method for future use)
        customer = Customer.objects.get(id=event.data["object"]["customer"])
        logger.info(f"Setup completed for customer {customer.id}")

@djstripe_receiver("checkout.session.async_payment_succeeded")
def handle_async_payment(sender, event: Event, **kwargs):
    # Some payment methods (bank transfers) are asynchronous
    session_data = event.data["object"]
    # Fulfill the order now that payment confirmed
    fulfill_order(session_data)

@djstripe_receiver("checkout.session.expired")
def handle_checkout_expired(sender, event: Event, **kwargs):
    # Session expired without completion
    session_data = event.data["object"]
    order_id = session_data.get("metadata", {}).get("order_id")
    if order_id:
        Order.objects.filter(id=order_id).update(status="expired")
```

## Payment Intents (Custom UI)

For custom payment forms with Stripe Elements.

### Creating a PaymentIntent (Server-Side)

```python
import stripe

intent = stripe.PaymentIntent.create(
    amount=2900,  # $29.00 in cents
    currency="usd",
    customer=customer.id,
    automatic_payment_methods={"enabled": True},
    metadata={"order_id": str(order.id)},
)

# Send client_secret to frontend
return JsonResponse({"client_secret": intent.client_secret})
```

> **Important:** Omit `payment_method_types` entirely — dynamic payment methods are the default behavior on API versions 2023-08-16+. The `automatic_payment_methods={"enabled": True}` parameter ensures dynamic methods work on older API versions. Never hardcode `payment_method_types: ["card"]`.

### Confirming Payment (Client-Side)

```javascript
// Frontend with Stripe.js
const stripe = Stripe("pk_test_...");
const elements = stripe.elements();
const paymentElement = elements.create("payment");
paymentElement.mount("#payment-element");

// Extract client_secret from server response
const response = await fetch("/api/create-payment-intent/");
const { client_secret } = await response.json();

const { error } = await stripe.confirmPayment({
  elements,
  clientSecret: client_secret,
  confirmParams: {
    return_url: window.location.origin + "/payment/complete",
  },
});
```

### Handling PaymentIntent Webhooks

```python
@djstripe_receiver("payment_intent.succeeded")
def handle_payment_succeeded(sender, event: Event, **kwargs):
    intent_data = event.data["object"]
    payment_intent_id = intent_data["id"]
    metadata = intent_data.get("metadata", {})

    order_id = metadata.get("order_id")
    order = Order.objects.get(id=order_id)
    order.stripe_payment_intent = payment_intent_id
    order.status = "paid"
    order.save()

@djstripe_receiver("payment_intent.payment_failed")
def handle_payment_failed(sender, event: Event, **kwargs):
    intent_data = event.data["object"]
    last_error = intent_data.get("last_payment_error", {})
    error_message = last_error.get("message", "Unknown error")

    # Notify user of failure
    metadata = intent_data.get("metadata", {})
    order_id = metadata.get("order_id")
    Order.objects.filter(id=order_id).update(status="payment_failed")
```

## Individual Charges (Legacy)

> **Warning:** The Charges API is deprecated by Stripe. Prefer Checkout Sessions or PaymentIntents for all new integrations. The `customer.charge()` method is provided only for legacy compatibility and cards that do not require SCA authentication. See the [Charges API migration guide](https://docs.stripe.com/payments/payment-intents/migration/charges).

> **Unit warning:** Raw Stripe API calls (`stripe.PaymentIntent.create(amount=2900)`) use **cents** (minor units). dj-stripe model helpers (`customer.charge(amount=Decimal("10.00"))`) use **dollars** (major units). Always verify which unit your code path expects.

```python
from decimal import Decimal
from djstripe.models import Customer

customer = Customer.objects.get(subscriber=user)

# Legacy charge — use PaymentIntents for new code
charge = customer.charge(
    amount=Decimal("10.00"),
    currency="usd",
    description="One-time purchase: Widget",
    metadata={"product_id": str(product.id)},
)
```

## Payment Methods

### Adding a Payment Method

```python
from djstripe.models import Customer

customer = Customer.objects.get(subscriber=user)

# Add and set as default
customer.add_payment_method("pm_card_visa", set_default=True)

# Add without setting default
customer.add_payment_method("pm_card_visa", set_default=False)
```

**Security:** Never send raw credit card info through your server. Use Stripe.js
or Stripe Elements to tokenize on the client side first.

### Client-Side Payment Method Collection (via Setup Intents)

Prefer the Setup Intents flow over `createPaymentMethod` or the legacy Card Element.
Use the Payment Element with `stripe.confirmSetup`:

```javascript
const stripe = Stripe("pk_test_...");
const elements = stripe.elements();
const paymentElement = elements.create("payment");
paymentElement.mount("#payment-element");

// Create SetupIntent on server, get client_secret
const response = await fetch("/api/create-setup-intent/");
const { client_secret } = await response.json();

const { error, setupIntent } = await stripe.confirmSetup({
  elements,
  clientSecret: client_secret,
  confirmParams: {
    return_url: window.location.origin + "/payment/setup-complete",
  },
});
```

After setup, the payment method is automatically synced by dj-stripe's webhook handlers.

### Listing Payment Methods

```python
customer = Customer.objects.get(subscriber=user)

# All payment methods for a customer
payment_methods = PaymentMethod.objects.filter(customer=customer)

# Default payment method
default_pm = customer.default_payment_method
```

### Detaching Payment Methods

```python
import stripe

# Via Stripe API (preferred — Stripe handles cleanup)
stripe.PaymentMethod.detach("pm_xxx")

# dj-stripe syncs the detachment via webhooks
```

## Refunds

### Full Refund

```python
charge = Charge.objects.get(id="ch_xxx")
refund = charge.refund()
```

### Partial Refund

```python
charge = Charge.objects.get(id="ch_xxx")
refund = charge.refund(amount=Decimal("5.00"))
```

### With Reason

```python
charge.refund(
    amount=Decimal("10.00"),
    reason="requested_by_customer",  # duplicate | fraudulent | requested_by_customer
)
```

## Setup Intents

For saving payment methods without immediate payment:

```python
import stripe

setup_intent = stripe.SetupIntent.create(
    customer=customer.id,
    automatic_payment_methods={"enabled": True},
    metadata={djstripe_settings.SUBSCRIBER_CUSTOMER_KEY: str(request.user.id)},
)

# Send client_secret to frontend for confirmation
return JsonResponse({"client_secret": setup_intent.client_secret})
```

> **Note:** Omit `payment_method_types`. Dynamic payment methods are the default. Never hardcode `payment_method_types: ["card"]`.

## Stripe Customer Portal

Redirect customers to Stripe-hosted billing management:

```python
import stripe

session = stripe.billing_portal.Session.create(
    customer=customer.id,
    return_url=request.build_absolute_uri(reverse("account")),
)
return redirect(session.url)
```

Customers can manage payment methods, view invoices, cancel subscriptions.

## Payment Method Types

| Type | Stripe ID | Use Case |
|---|---|---|
| Card | `card` | Credit/debit cards |
| ACH Direct Debit | `us_bank_account` | US bank accounts |
| SEPA Direct Debit | `sepa_debit` | EU bank accounts |
| iDEAL | `ideal` | Netherlands |
| Bancontact | `bancontact` | Belgium |
| Sofort | `sofort` | Germany/Austria |
| Giropay | `giropay` | Germany |
| EPS | `eps` | Austria |
| P24 | `p24` | Poland |
| Alipay | `alipay` | China |
| WeChat Pay | `wechat_pay` | China |
| Link | `link` | Stripe's one-click checkout |
| Bacs Direct Debit | `bacs_debit` | UK bank accounts |
| ACSS Direct Debit | `acss_debit` | Canada bank accounts |
| Boleto | `boleto` | Brazil |
| Klarna | `klarna` | Buy now, pay later |
| Afterpay / Clearpay | `afterpay_clearpay` | Buy now, pay later |
| Affirm | `affirm` | Buy now, pay later |
| Customer Balance | `customer_balance` | Credit balance |

> **Note:** `sofort` is being deprecated by Stripe in favor of Klarna. Use `klarna` for new integrations.

## Querying Payment Data

### Charges

```python
# All charges for a customer
charges = Charge.objects.filter(customer=customer)

# Successful charges
successful = Charge.objects.filter(customer=customer, status="succeeded")

# Charges with stripe_data filtering (2.10+)
refunded = Charge.objects.filter(stripe_data__refunded=True)

# Amount aggregation (requires PostgreSQL + Cast)
from django.db.models import Sum, FloatField
from django.db.models.functions import Cast

total = Charge.objects.filter(
    customer=customer, status="succeeded"
).annotate(
    amount_float=Cast("stripe_data__amount", FloatField())
).aggregate(Sum("amount_float"))
```

### Payment Intents

```python
# Payment intents for a customer
intents = PaymentIntent.objects.filter(customer=customer)

# By status
succeeded = PaymentIntent.objects.filter(customer=customer, status="succeeded")
processing = PaymentIntent.objects.filter(customer=customer, status="processing")
```

## Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Sending raw card numbers to server | PCI violation | Use Stripe.js/Elements client-side |
| Not handling async payment methods | Order stuck unfulfilled | Handle `async_payment_succeeded` event |
| Reusing PaymentIntent | Errors, incorrect amounts | Create new PaymentIntent per payment |
| Not checking payment status in webhook | Fulfilling unpaid orders | Check `payment_status == "paid"` |
| Hardcoding prices in code | Price changes require deploy | Use Price model from Stripe dashboard |
| Hardcoding `payment_method_types` | Blocks dynamic payment methods, lowers conversion | Omit the parameter entirely |
| Using legacy Card Element | Missing payment method types, poor UX | Use Payment Element |
