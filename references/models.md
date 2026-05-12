# Models Reference

Complete catalog of dj-stripe models with key fields, relationships, and Stripe API mappings.

> **Important:** dj-stripe stores most Stripe fields in a `stripe_data` JSONField and exposes them via `@property` accessors. Only FK relationships and a few critical fields are concrete database columns. Querying by a property requires JSON field lookups (e.g., `Customer.objects.filter(stripe_data__name="Foo")`).

## Model Hierarchy

All Stripe models inherit from `StripeBaseModel` → `StripeModel` (`djstripe/models/base.py`):

```python
# StripeBaseModel (abstract) — holds sync metadata + raw data
class StripeBaseModel(models.Model):
    stripe_data = JSONField(default=dict)       # Full Stripe API response
    djstripe_created = models.DateTimeField(auto_now_add=True)
    djstripe_updated = models.DateTimeField(auto_now=True)

# StripeModel (abstract) — adds Stripe ID + livemode tracking
class StripeModel(StripeBaseModel):
    djstripe_id = models.BigAutoField(primary_key=True)  # Internal PK
    id = StripeIdField(unique=True)                      # Stripe ID (e.g., cus_xxx)
    created = StripeDateTimeField(null=True, blank=True)
    livemode = models.BooleanField(null=True, default=None, blank=True)
    metadata = JSONField(null=True, blank=True)
    djstripe_owner_account = StripeForeignKey("Account", ...)

    @classmethod
    def sync_from_stripe_data(cls, data):
        """Create or update local record from Stripe API data."""
        ...

    @classmethod
    def _get_or_retrieve(cls, id, stripe_account=None, **kwargs):
        """Get from DB or fetch from Stripe API."""
        ...
```

## Core Models

### Customer (`djstripe.models.Customer`)

Stripe object: `cus_xxx` | Key: `djstripe_id` (BigAutoField PK) | Stripe ID: `id` (StripeIdField)

**Concrete fields** (in DB):
| Field | Type | Description |
|---|---|---|
| `djstripe_id` | BigAutoField(PK) | Internal PK |
| `id` | StripeIdField(unique) | Stripe customer ID |
| `email` | TextField | Customer email |
| `default_payment_method` | FK(PaymentMethod), SET_NULL | Default payment method |
| `subscriber` | FK(AUTH_USER_MODEL), SET_NULL | Link to your user model |
| `date_purged` | DateTimeField | When locally purged |

**Properties** (via stripe_data):
| Property | Returns | Description |
|---|---|---|
| `name` | str | Customer name |
| `balance` | int | Account balance in cents |
| `currency` | str | Default currency code |
| `delinquent` | bool | Past due status |
| `coupon` | dict | Coupon discount applied |
| `invoice_prefix` | str | Invoice number prefix |
| `tax_exempt` | str | Tax exemption status |
| `address` | dict | Customer address |
| `phone` | str | Phone number |
| `shipping` | dict | Shipping info |
| `discount` | dict | Active discount |
| `deleted` | bool | Whether deleted upstream |
| `default_source` | str/dict | Default source ID |
| `preferred_locales` | list | Language preferences |

**Key Methods:**
```python
customer, created = Customer.get_or_create(subscriber=user)
customer.subscribe(items=[{"price": price}], trial_period_days=30)
customer.charge(amount=Decimal("10.00"), currency="usd")
customer.add_payment_method("pm_card_visa", set_default=True)
customer.purge()  # Delete from Stripe, keep local record

# Check subscription access:
customer.is_subscribed_to(product)         # Active sub to specific product
customer.has_any_active_subscription()     # Any active sub
customer.active_subscriptions              # Active subs ending in future
customer.valid_subscriptions               # Non-canceled/non-expired subs
```

**Properties (additional):**
- `subscription` — Current active subscription (or None; raises if multiple)
- `pending_charges` — `max(self.balance, 0)`
- `credits` — `abs(min(self.balance, 0))`

### Charge (`djstripe.models.Charge`)

Stripe object: `ch_xxx`

**Concrete fields** (in DB):
| Field | Type | Description |
|---|---|---|
| `djstripe_id` | BigAutoField(PK) | Internal PK |
| `id` | StripeIdField(unique) | Stripe charge ID |
| `amount` | StripeDecimalCurrencyAmountField | Amount in **dollars** (e.g., 10.00 = $10.00) |
| `currency` | StripeCurrencyCodeField | Three-letter ISO code |
| `customer` | FK(Customer), SET_NULL | Customer charged |
| `invoice` | FK(Invoice), CASCADE, null | Invoice (nullable) |
| `payment_intent` | FK(PaymentIntent), SET_NULL | Associated PaymentIntent |
| `payment_method` | FK(PaymentMethod), SET_NULL | Payment method used |
| `source` | PaymentMethodForeignKey, SET_NULL | Polymorphic source |
| `balance_transaction` | FK(BalanceTransaction), SET_NULL | Balance entry |
| `status` | StripeEnumField(ChargeStatus) | pending/succeeded/failed |

**Properties** (via stripe_data):
| Property | Returns | Description |
|---|---|---|
| `amount_captured` | int | Amount captured in cents |
| `amount_refunded` | int | Amount refunded in cents |
| `captured` | bool | Was charge captured? |
| `paid` | bool | Was charge paid? |
| `refunded` | bool | Fully refunded? |
| `disputed` | bool | Under dispute? |
| `outcome` | dict | Stripe Radar outcome |
| `receipt_url` | str | Receipt URL |
| `receipt_email` | str | Receipt email |
| `receipt_number` | str | Receipt number |
| `billing_details` | dict | Billing details |
| `dispute` | dict | Dispute data |
| `payment_method_details` | dict | Card/bank details |
| `shipping` | dict | Shipping info |
| `statement_descriptor` | str | Statement descriptor |
| `transfer` | str | Transfer ID |

**Key Methods:**
```python
charge.capture(**kwargs)  # Capture uncaptured charge
charge.refund(amount=None, reason=None)  # Refund charge
charge.fraudulent  # True if fraud_details mark it as fraudulent
```

### Product (`djstripe.models.Product`)

Stripe object: `prod_xxx`

**Concrete fields:**
| Field | Type | Description |
|---|---|---|
| `name` | TextField | Product name |
| `active` | BooleanField(null=True) | Is product active? |

**Properties** (via stripe_data):
| Property | Returns | Description |
|---|---|---|
| `type` | str | good/service |
| `url` | str | Product URL |
| `unit_label` | str | Label for unit (e.g., "seat") |
| `default_price` | Price\|None | Default price object |
| `description` | str | Description (inherited from StripeModel) |

### Price (`djstripe.models.Price`)

Stripe object: `price_xxx`

**Concrete fields:**
| Field | Type | Description |
|---|---|---|
| `active` | BooleanField | Is price active? |
| `currency` | StripeCurrencyCodeField | Three-letter ISO code |
| `nickname` | CharField | Display name |
| `product` | FK(Product) | Parent product |
| `lookup_key` | CharField(null=True) | Programmatic key |

**Properties** (via stripe_data):
| Property | Returns | Description |
|---|---|---|
| `unit_amount` | int | Amount in cents |
| `recurring` | dict | Recurring config (interval, count, etc.) |
| `type` | str | one_time/recurring |
| `tiers` | list | Tiered pricing config |
| `tiers_mode` | str | graduated/volume |
| `billing_scheme` | str | per_unit/tiered |
| `transform_quantity` | dict | Division/multiplication config |

### PaymentIntent (`djstripe.models.PaymentIntent`)

Stripe object: `pi_xxx`

**Concrete fields:**
| Field | Type | Description |
|---|---|---|
| `customer` | FK(Customer), CASCADE, null | Customer |
| `on_behalf_of` | FK(Account), CASCADE, null | Connect account |
| `payment_method` | FK(PaymentMethod), SET_NULL, null | Payment method |

**Properties** (via stripe_data): `amount`, `currency`, `status`, `client_secret`, `setup_future_usage`, `capture_method`, `confirmation_method`, `last_payment_error` — all accessed from `stripe_data`.

**Key Methods:**
```python
payment_intent.update(**kwargs)
payment_intent._api_cancel(**kwargs)
payment_intent._api_confirm(**kwargs)
```

### PaymentMethod (`djstripe.models.PaymentMethod`)

Stripe object: `pm_xxx`

**Concrete fields:**
| Field | Type | Description |
|---|---|---|
| `customer` | FK(Customer), SET_NULL, null | Owner customer |

**Properties** (via stripe_data):
| Property | Returns | Description |
|---|---|---|
| `type` | str | card/us_bank_account/sepa_debit/... |
| `billing_details` | dict | Name, email, address |

Card/bank-specific details (`card`, `us_bank_account`, etc.) are nested in `stripe_data`.

**Key Methods:**
```python
PaymentMethod.attach(payment_method, customer)
payment_method.detach()
```

## Billing Models

### Subscription (`djstripe.models.Subscription`)

Stripe object: `sub_xxx`

**Concrete fields:**
| Field | Type | Description |
|---|---|---|
| `customer` | FK(Customer), CASCADE | Subscriber |

**Properties** (via stripe_data):
| Property | Returns | Description |
|---|---|---|
| `status` | str | active/past_due/unpaid/canceled/incomplete/incomplete_expired/trialing/paused |
| `current_period_start` | datetime | Current billing period start |
| `current_period_end` | datetime | Current billing period end |
| `cancel_at_period_end` | bool | Cancel at end of period? |
| `canceled_at` | datetime | When canceled |
| `ended_at` | datetime | When ended |
| `trial_start` | datetime | Trial start |
| `trial_end` | datetime | Trial end |
| `schedule` | str/dict | Associated schedule (ID from stripe_data) |
| `collection_method` | str | charge_automatically/send_invoice |
| `days_until_due` | int | Days before invoice due |
| `pending_update` | dict | Pending subscription update |
| `items` | dict | Subscription items data |
| `plan` | dict | Plan (if single plan) |

**Key Methods:**
```python
subscription.cancel(at_period_end=False, **kwargs)
subscription.reactivate()
subscription.update(plan=None, **kwargs)
subscription.extend(timedelta(days=30))  # Uses trial_end manipulation
subscription.is_valid()                  # Status current AND period current
subscription.is_status_temporarily_current()  # Canceled at_period_end, still active
```

### SubscriptionItem (`djstripe.models.SubscriptionItem`)

Stripe object: `si_xxx`

**Concrete fields:**
| Field | Type | Description |
|---|---|---|
| `subscription` | FK(Subscription), CASCADE | Parent subscription |
| `price` | FK(Price), CASCADE, null | Price for this item |
| `plan` | FK(Plan), CASCADE | Plan (legacy) |
| `tax_rates` | M2M(TaxRate) | Tax rates |

**Properties** (via stripe_data):
| Property | Returns | Description |
|---|---|---|
| `quantity` | int | Quantity (null = metered) |
| `billing_thresholds` | dict | Usage thresholds |

### Invoice (`djstripe.models.Invoice`)

Stripe object: `in_xxx`

**Concrete fields** (on BaseInvoice):
| Field | Type | Description |
|---|---|---|
| `customer` | FK(Customer), CASCADE | Billed customer |
| `subscription` | FK(Subscription), SET_NULL | Subscription (if recurring) |
| `charge` | OneToOneField(Charge), CASCADE, null | Payment charge |
| `payment_intent` | OneToOneField(PaymentIntent), CASCADE, null | PaymentIntent |
| `default_payment_method` | FK(PaymentMethod), SET_NULL | Default PM |

**Invoice-only concrete field:** `default_tax_rates` — M2M(TaxRate)

**Properties** (via stripe_data):
| Property | Returns | Description |
|---|---|---|
| `amount_due` | int | Amount due in cents |
| `amount_remaining` | int | Amount unpaid in cents |
| `currency` | str | Three-letter ISO code |
| `status` | str | draft/open/paid/uncollectible/void |
| `billing_reason` | str | Why invoice was created |
| `due_date` | int | Payment due date (Unix timestamp) |
| `period_start` | int | Billing period start |
| `period_end` | int | Billing period end |
| `number` | str | Invoice number |
| `receipt_number` | str | Receipt number |
| `attempt_count` | int | Payment attempts |
| `paid` | bool | Fully paid? |
| `tax` | int | Tax amount in cents |
| `total` | int | Total amount in cents |
| `subtotal` | int | Subtotal in cents |
| `hosted_invoice_url` | str | Hosted invoice URL |
| `invoice_pdf` | str | Invoice PDF URL |

### InvoiceItem (`djstripe.models.InvoiceItem`)

Stripe object: `ii_xxx`

**Concrete fields:**
| Field | Type | Description |
|---|---|---|
| `customer` | FK(Customer), CASCADE | Customer |
| `invoice` | FK(Invoice), CASCADE, null | Parent invoice |
| `price` | FK(Price), SET_NULL, null | Price |
| `plan` | FK(Plan), SET_NULL, null | Plan (legacy) |
| `subscription` | FK(Subscription), SET_NULL, null | Subscription |
| `tax_rates` | M2M(TaxRate) | Tax rates |

**Properties** (via stripe_data):
| Property | Returns | Description |
|---|---|---|
| `amount` | int | Amount in cents |
| `currency` | str | Currency code |
| `quantity` | int | Quantity |
| `description` | str | Line item description |
| `date` | int | Date (Unix timestamp) |
| `period` | dict | Period info |
| `proration` | bool | Proration adjustment? |
| `discountable` | bool | Discounts apply? |
| `discounts` | list | Applied discounts |

### Coupon (`djstripe.models.Coupon`)

Stripe object: `coupon_xxx`

**Concrete fields:**
| Field | Type | Description |
|---|---|---|
| `id` | StripeIdField(max_length=500) | Stripe coupon ID |

**Properties** (via stripe_data):
| Property | Returns | Description |
|---|---|---|
| `name` | str | Display name |
| `amount_off` | int | Fixed discount in cents |
| `percent_off` | float | Percentage discount |
| `currency` | str | Currency of amount_off |
| `duration` | str | once/repeating/forever |
| `duration_in_months` | int | Months if repeating |
| `max_redemptions` | int | Max uses |
| `times_redeemed` | int | Times used |
| `redeem_by` | int | Expiry timestamp |

## Checkout Models

### Session (`djstripe.models.Session`)

Stripe object: `cs_xxx`

**Concrete fields:**
| Field | Type | Description |
|---|---|---|
| `customer` | FK(Customer), SET_NULL | Customer |
| `payment_intent` | FK(PaymentIntent), SET_NULL | PaymentIntent (mode=payment) |
| `subscription` | FK(Subscription), SET_NULL | Subscription (mode=subscription) |
| `setup_intent` | FK(SetupIntent), SET_NULL | SetupIntent (mode=setup) |

**Properties** (via stripe_data):
| Property | Returns | Description |
|---|---|---|
| `mode` | str | payment/setup/subscription |
| `payment_status` | str | paid/unpaid/no_payment_required |
| `status` | str | open/complete/expired |
| `client_reference_id` | str | Your app's reference |
| `success_url` | str | Redirect on success |
| `cancel_url` | str | Redirect on cancel |
| `amount_total` | int | Total after discounts and taxes |
| `amount_subtotal` | int | Total before discounts and taxes |
| `currency` | str | Currency code |
| `customer_email` | str | Customer email |
| `url` | str | Checkout page URL |
| `line_items` | list | Purchased line items |
| `total_details` | dict | Tax/discount breakdown |

## Webhook Models

### Event (`djstripe.models.Event`)

Stripe object: `evt_xxx` | (defined in `models/core.py`)

**Concrete fields:**
| Field | Type | Description |
|---|---|---|
| `id` | StripeIdField(unique) | Stripe event ID |
| `type` | CharField(250) | Event type (e.g., "charge.succeeded") |
| `data` | JSONField | Event payload |
| `api_version` | CharField(64) | Stripe API version |
| `request_id` | CharField(50) | Request trace ID |
| `idempotency_key` | TextField | Idempotency key |

**Properties:** `category`, `verb`, `customer`, `parts`

### WebhookEndpoint (`djstripe.models.WebhookEndpoint`)

Stripe object: `we_xxx`

**Concrete fields:**
| Field | Type | Description |
|---|---|---|
| `url` | URLField(max_length=2048) | Webhook URL |
| `secret` | CharField(256) | Signing secret |
| `djstripe_tolerance` | PositiveSmallIntegerField | Signature tolerance (ms) |
| `djstripe_validation_method` | StripeEnumField | verify_signature/retrieve_event/none |
| `enabled_events` | JSONField | Selected event types |
| `status` | StripeEnumField | enabled/disabled |
| `api_version` | CharField(64) | API version |
| `application` | CharField(255) | Connect app ID |
| `djstripe_uuid` | UUIDField | Local UUID |

**Also:** `WebhookEventTrigger` stores incoming webhook payloads with validation, processing, and traceback fields.

## Connect Models

### Account (`djstripe.models.Account`)

Stripe object: `acct_xxx` — Used for Stripe Connect platforms. Links to API keys stored in the database.

### ApplicationFee (`djstripe.models.ApplicationFee`)

Stripe object: `fee_xxx` — Platform fees for Connect transactions. Concrete FKs: `account`, `balance_transaction`, `charge`. Properties: `amount`, `amount_refunded`, `currency`, `refunded`.

## Identity Models

- **VerificationSession** — Identity verification sessions
- **VerificationReport** — Identity verification results

## Relationship Map

```
AUTH_USER_MODEL
  │
  └── Customer (subscriber FK)
        ├── Charge (customer FK, reverse: customer.charges)
        │     ├── PaymentIntent (FK)
        │     ├── PaymentMethod (FK)
        │     ├── BalanceTransaction (FK)
        │     └── Refund (charge FK, reverse: charge.refunds)
        ├── Subscription (customer FK, reverse: customer.subscriptions)
        │     ├── SubscriptionItem (subscription FK, reverse: subscription.items)
        │     │     └── Price (FK)
        │     │           └── Product (FK)
        │     ├── SubscriptionSchedule (subscription FK)
        │     └── Invoice (subscription FK + customer FK)
        │           ├── Charge (FK, reverse: invoice.charges)
        │           ├── InvoiceItem (invoice FK)
        │           └── PaymentIntent (OneToOne via payment_intent)
        ├── PaymentMethod (customer FK, reverse: customer.payment_methods)
        ├── PaymentIntent (customer FK)
        ├── Session (customer FK) → Checkout
        │     ├── PaymentIntent (FK)
        │     ├── Subscription (FK)
        │     └── SetupIntent (FK)
        ├── Source (customer FK, reverse: customer.sources) — deprecated
        ├── Card (customer FK, reverse: customer.legacy_cards) — legacy
        ├── BankAccount (customer FK) — legacy
        ├── TaxId (reverse FK)
        │
        └── (coupon, discount accessed via stripe_data @property, NOT FK)
```

> **Note:** `Customer.coupon` and `Customer.discount` are `@property` accessors reading from `stripe_data`. They are NOT ForeignKey relationships. Coupon and Discount models exist independently and are linked via JSON data in the stripe_data blob.

## Enum Reference

Key enum fields and their values:

| Enum Class | Values |
|---|---|
| `ChargeStatus` | pending, succeeded, failed |
| `RefundStatus` | pending, succeeded, failed, canceled |
| `SubscriptionStatus` | active, past_due, unpaid, canceled, incomplete, incomplete_expired, trialing, **paused** |
| `InvoiceStatus` | draft, open, paid, uncollectible, void |
| `PaymentMethodType` | card, us_bank_account, sepa_debit, ... (many) |
| `SessionMode` | payment, setup, subscription |
| `SessionPaymentStatus` | paid, unpaid, no_payment_required |
| `SessionStatus` | open, complete, expired |
| `PaymentIntentStatus` | requires_payment_method, requires_confirmation, requires_action, processing, requires_capture, canceled, succeeded |

All enums use `StripeEnumField(max_length=255)` for forward-compatibility with new Stripe enum values.

## Custom Field Types

| Field | Storage | Usage |
|---|---|---|
| `StripeQuantumCurrencyAmountField` | BigInteger | Amount in currency minor units (e.g., cents for USD) |
| `StripeDecimalCurrencyAmountField` | Decimal(14,2) | Amounts in dollars (10.00 = $10.00). **Legacy** — prefer StripeQuantumCurrencyAmountField for new fields |
| `StripeDateTimeField` | DateTime | Converts Unix timestamps to Django DateTime for storage |
| `StripeEnumField` | CharField(255) | Stripe enum values with database choices |
| `StripeIdField` | CharField(255) | Stripe object IDs (e.g., cus_xxx, ch_xxx) |
| `StripeForeignKey` | FK | ForeignKey with configurable `to_field` |
| `StripeCurrencyCodeField` | CharField(3) | Three-letter ISO currency code |
| `PaymentMethodForeignKey` | FK | Polymorphic FK to DjstripePaymentMethod |
| `StripePercentField` | DecimalField | Validated percentage field |
| `JSONField` | JSONField | JSON storage for structured data |
