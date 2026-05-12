# Migrations and Version Upgrades

Complete guide to upgrading dj-stripe between major versions without breaking
your application or losing data.

## Golden Rules

1. **NEVER skip versions.** Go through each major version sequentially.
2. **Always run `python manage.py migrate djstripe` after each upgrade.**
3. **Always run `python manage.py djstripe_sync_models` after migration.**
4. **Test in a staging environment first.**

## Version Compatibility Matrix

| dj-stripe | Django | Python | Stripe API | Key Changes |
|---|---|---|---|---|
| 2.11.x | >= 5.1 | >= 3.11 | 2020-08-27 | Migrations reset, paused subs, stripe-python 15+ compat |
| 2.10.x | >= 5.0 | >= 3.11 | 2020-08-27 | Massive field removal → stripe_data JSONField |
| 2.9.x | >= 4.2 | >= 3.10 | 2020-08-27 | Legacy webhooks removed, stripe_data JSONField introduced |
| 2.8.x | >= 3.2 | >= 3.9 | 2020-08-27 | Plan model deprecated, Price model preferred |
| 2.7.x | >= 3.2 | >= 3.9 | 2020-08-27 | UUID webhook endpoints in DB |
| 2.6.x | >= 3.2 | >= 3.8 | 2020-08-27 | Price and Customer models introduced |
| 2.5.x | >= 2.2 | >= 3.6 | 2020-08-27 | Base version for upgrades to 2.8+ |
| 2.4.x | >= 2.2 | >= 3.6 | 2020-08-27 | Multi-item subscriptions |
| 0.x - 1.x | — | — | 2017-06-05 | Legacy versions |

**Critical:** `STRIPE_API_VERSION` is set by dj-stripe to its tested version
(currently `2020-08-27`). Do NOT change this setting.

## Upgrade Path: 2.10.x → 2.11.x

> ⚠️ **2.11.x is UNRELEASED** as of May 2026. The changes below are projected based on deprecation notices in 2.10.x and upstream Stripe API changes. Verify against the actual 2.11.x changelog when it ships.

```python
# 1. Update requirements
# dj-stripe>=2.11,<3.0

# 2. Run migrations (ALL migrations reset in 2.11)
python manage.py migrate djstripe

# 3. Sync Stripe data
python manage.py djstripe_sync_models
```

### Breaking Changes

- Django >= 5.1 required
- Python >= 3.11 required
- stripe-python >= 15.0 required
- `InvoiceOrLineItemForeignKey` removed
- `PlanTiersMode`, `PlanUsageType` enums removed
- `Customer.retry_unpaid_invoices` removed
- `Subscription.status` now includes `paused`
- stripe-python 15 compatibility: `StripeObject` no longer inherits from `dict`

**If stripe-python 15 compat:** `.get()`, `.items()`, `.keys()` no longer work
on StripeObject. Use attribute access or convert to dict explicitly.

## Upgrade Path: 2.9.x → 2.10.x

This is the **most impactful upgrade** in dj-stripe history.

### Breaking Changes

- All migrations fully reset (new migration files from scratch)

```python
# 1. Upgrade to latest 2.9.x first
# dj-stripe>=2.9,<2.10

# 2. Then upgrade to 2.10
# dj-stripe>=2.10,<2.11

# 3. Run migrations
python manage.py migrate djstripe

# 4. Check for FieldError on queries
```

### Massive Field Removal

Most concrete model fields were removed. Data moved to `stripe_data` JSONField.
Access via `@property` accessors (transparent for reads, different for queries).

### Query Migration

```python
# BEFORE 2.10 (concrete field)
charges = Charge.objects.filter(amount=1000)
charge.amount  # Returns 1000

# AFTER 2.10 (JSONField + @property)
charge.amount  # Still returns 1000 (via @property)

# But queries must use JSONField path lookups
charges = Charge.objects.filter(stripe_data__amount=1000)

# Aggregations require Cast
from django.db.models import Sum, FloatField
from django.db.models.functions import Cast

Charge.objects.annotate(
    amount_float=Cast("stripe_data__amount", FloatField())
).aggregate(Sum("amount_float"))
```

### Common Migration Errors (2.10)

```
FieldError: Cannot resolve keyword 'amount' into field.
Choices are: stripe_data, metadata, created, ...
```
**Fix:** Use `stripe_data__amount` instead of `amount`.

```
TypeError: unsupported operand type(s) for +: 'str' and 'str'
```
**Fix:** Aggregation over JSONField paths may not infer numeric types automatically.
Use `Cast()` to numeric type (e.g., `FloatField`, `DecimalField`).

### Fields Most Affected

Nearly all model fields moved to `stripe_data`. Key fields still as concrete
columns: `id`, `created`, `livemode`, `metadata`, `djstripe_created`,
`djstripe_updated`, `stripe_data` (the JSONField itself).

## Upgrade Path: 2.8.x → 2.9.x

```python
# 1. Upgrade to 2.9
# dj-stripe>=2.9,<2.10

# 2. Run migrations
python manage.py migrate djstripe
```

### Breaking Changes

- `DJSTRIPE_WEBHOOK_SECRET`, `DJSTRIPE_WEBHOOK_TOLERANCE` settings **removed**
- Legacy webhooks (non-DB) **removed** — all webhooks now in DB
- `djstripe.webhooks` module **removed** — use `@djstripe_receiver`
- `DJSTRIPE_WEBHOOK_VALIDATION` **deprecated** — use per-endpoint `djstripe_validation_method`

### Action Required

```python
# BEFORE (2.8 and earlier)
from djstripe.webhooks import handler

# AFTER (2.9+)
from djstripe.event_handlers import djstripe_receiver
```

If using legacy webhook decorators, migrate to `@djstripe_receiver`.

## Upgrade Path: < 2.5 → 2.8+

```
MUST go through 2.5 first.
Cannot jump from < 2.5 directly to 2.8+.
```

### Required Path

```
2.4 → 2.5 → 2.6 → 2.7 → 2.8 → 2.9 → 2.10 → 2.11
```

At each step:
1. Update requirements to next version
2. `python manage.py migrate djstripe`
3. Fix any FieldError/ImportError
4. Test webhooks and sync
5. Commit and proceed to next version

## Foreign Key Field Migration

### Legacy vs Modern

| Setting | Value | When |
|---|---|---|
| `DJSTRIPE_FOREIGN_KEY_TO_FIELD` | `"id"` | **New installs** (recommended) |
| `DJSTRIPE_FOREIGN_KEY_TO_FIELD` | `"djstripe_id"` | **Legacy** installations |

In 2.0, `stripe_id` was renamed to `id`. The setting controls FK resolution.

If you have legacy data with `djstripe_id`, keep the old setting. For new
installs, always use `"id"`.

## JSONField Migration Issues

### ImportError: JSONField

```
ImportError: cannot import name 'JSONField' from 'jsonfield'
```

**Fix:** Remove both `jsonfield` and `jsonfield2` packages, reinstall dj-stripe.

dj-stripe switched from `jsonfield` to Django's native `JSONField`. If the old
package is installed, it conflicts.

## Management Commands for Migrations

### Update InvoiceItem IDs

```bash
python manage.py djstripe_update_invoiceitem_ids
```

Only needed when upgrading from very old versions where invoice item ID format
changed.

### Clear Expired Idempotency Keys

```bash
python manage.py djstripe_clear_expired_idempotency_keys
```

Cleans up expired idempotency keys. Safe to run anytime.

## Migration Safety Protocol

### Before Upgrading

- [ ] Read the changelog for ALL versions between current and target
- [ ] Back up your database
- [ ] Note all deprecated settings you're using
- [ ] Check Django and Python version requirements
- [ ] Review breaking changes in your codebase

### During Upgrade

- [ ] Upgrade one version at a time
- [ ] Run `python manage.py migrate djstripe` after each version
- [ ] Run `python manage.py djstripe_sync_models` after migration
- [ ] Check for FieldError on your queries
- [ ] Run your test suite
- [ ] Test webhook processing

### After Upgrade

- [ ] Verify Stripe data synced correctly
- [ ] Check admin for any migration warnings
- [ ] Monitor for webhook errors in production
- [ ] Update `requirements.txt` with pinned version

## Rollback Strategy

dj-stripe migrations may not be reversible between major versions due to
schema changes. Always:

1. Back up the database before upgrading
2. Test the upgrade in staging first
3. Have a rollback plan (DB restore from backup)
4. Keep the old requirements.txt as backup

## Common Upgrade Issues

| Issue | Version | Fix |
|---|---|---|
| FieldError on `amount` | 2.10 | Use `stripe_data__amount` |
| ImportError on `webhooks` | 2.9 | Use `event_handlers.djstripe_receiver` |
| Settings not found | 2.9 | Move webhook config to DB endpoints |
| `Plan` model deprecated | 2.6+ | Use `Price` model instead |
| `customer.can_charge()` missing | 2.8 | Use `customer.charge()` directly |
| `stripe_id` vs `id` | 2.0+ | Set `DJSTRIPE_FOREIGN_KEY_TO_FIELD` |
| JSONField ImportError | 2.9+ | Remove `jsonfield`/`jsonfield2` packages |
| stripe-python compat | 2.11 | Don't use `.get()` on StripeObject |
