# Security — API Keys, PCI Compliance, and Webhook Protection

Security is critical for payment systems. This covers key management, PCI
compliance, webhook protection, and secure Stripe integration patterns.

## API Key Management

### Key Types

| Prefix | Type | Purpose |
|---|---|---|
| `sk_live_` | Live Secret | Server-side, full access to live account |
| `sk_test_` | Test Secret | Server-side, full access to test account |
| `rk_live_` | Restricted Live | Server-side, limited permissions |
| `rk_test_` | Restricted Test | Server-side, limited test permissions |
| `pk_live_` | Live Publishable | Client-side, limited public operations |
| `pk_test_` | Test Publishable | Client-side, limited test operations |
| `whsec_` | Webhook Secret | Webhook signature verification |

### Key Storage

```python
# ✅ CORRECT: Environment variables
STRIPE_LIVE_SECRET_KEY = os.environ.get("STRIPE_LIVE_SECRET_KEY")
STRIPE_TEST_SECRET_KEY = os.environ.get("STRIPE_TEST_SECRET_KEY")

# ❌ WRONG: Hardcoded
STRIPE_LIVE_SECRET_KEY = "sk_live_abc123"

# ❌ WRONG: In settings committed to git
# (settings.py with hardcoded key)
```

### Restricted API Keys (Recommended)

Restricted keys (`rk_`) limit what dj-stripe can do:

```
rk_live_xxx with permissions:
  ✅ Customers: Read, Write
  ✅ Charges: Read, Write
  ✅ Products: Read
  ✅ Prices: Read
  ✅ Webhook Endpoints: Read, Write
  ❌ Transfers: No access
  ❌ Account: No access
```

**Create restricted keys in Stripe Dashboard** → Developers → API Keys → Restricted Keys.

### API Key in Django Admin

dj-stripe stores API keys in the database for multi-account support:

- Admin: `/admin/djstripe/apikey/`
- Auto-detects key type (secret/restricted/publishable, live/test)
- Secret and restricted keys auto-associate with matching Account

> **SECURITY WARNING:** Secret API keys stored in the `APIKey` model are visible as plaintext readonly fields in Django Admin to any staff user with dj-stripe admin access. Restrict Django Admin access severely for payment systems.

### Key Rotation

```python
# 1. Create new key in Stripe Dashboard
# 2. Add to environment variables
# 3. Deploy with both old and new keys
# 4. Verify new key works
# 5. Revoke old key in Stripe Dashboard
# 6. Remove old key from environment
```

## PCI Compliance

### Stripe's Shared Responsibility Model

**Stripe handles:**
- Card data storage
- Card data transmission
- PCI certification

**You handle:**
- Secure API key storage
- TLS for your server
- Not storing raw card data

### PCI DSS Requirements You Must Meet

| Requirement | How to Meet |
|---|---|
| Don't store CVV | Never send CVV to your server — Stripe.js handles it |
| Don't store full PAN | Stripe.js tokenizes cards client-side |
| Encrypt data in transit | Always use HTTPS |
| Secure API keys | Environment variables, never in code |
| Access control | Restrict API key permissions |
| Monitor access | Stripe Dashboard audit logs |

### Client-Side Security

```html
<!-- ✅ CORRECT: Stripe.js loaded from Stripe's CDN -->
<script src="https://js.stripe.com/v3/"></script>

<!-- ❌ WRONG: Self-hosted or modified Stripe.js -->
<script src="/static/js/stripe-custom.js"></script>
```

### Server-Side Security

```python
# ✅ CORRECT: Use Stripe tokens, never raw card data
def charge_view(request):
    token = request.POST["stripe_token"]  # Token from Stripe.js
    customer.charge(amount=Decimal("10.00"), source=token)

# ❌ WRONG: Never do this
def charge_view(request):
    card_number = request.POST["card_number"]  # PCI VIOLATION
    # ...
```

## Webhook Security

### UUID-Secured Endpoints

dj-stripe 2.7+ uses UUIDs in webhook URLs:

```
/stripe/webhook/a1b2c3d4-e5f6-7890-abcd-ef1234567890/
```

This makes endpoints impossible to brute-force. UUID space is 2^122.

### Signature Verification

Tolerance is configured per-endpoint via `WebhookEndpoint.djstripe_tolerance` (default: 300 seconds). Configure it in Django Admin, not as a global setting.

> **IMPORTANT:** `djstripe_tolerance = 0` does NOT disable verification — it requires exact timestamp match (strictest check). To disable verification (dev only), set `djstripe_validation_method` to `none` on the WebhookEndpoint.

Signature verification confirms:
1. The webhook came from Stripe (not an attacker)
2. The payload hasn't been modified in transit
3. The webhook isn't a replay (timestamp check)

### Webhook Secret Storage

**Webhook secrets are stored per-endpoint in the database** on `WebhookEndpoint.secret`. They are NOT configured as Django settings. Manage them via Django Admin at `/admin/djstripe/webhookendpoint/`.

### Webhook Best Practices

1. Always verify signatures in production (`djstripe_validation_method = "verify_signature"`)
2. Use separate webhook secrets per endpoint
3. Rotate secrets if compromised
4. Monitor webhook delivery in Stripe Dashboard

> **Note:** The webhook view at `/stripe/webhook/<uuid>/` is intentionally exempt from CSRF protection (`@csrf_exempt`). Stripe sends webhooks without CSRF tokens. This is required, not a misconfiguration.

## Stripe Connect Security

For platforms/marketplaces:

### Platform Responsibilities

- Verify connected accounts before enabling payments
- Use OAuth for account connections
- Handle connected account webhooks securely
- Implement proper fee structures

### Connected Account Data

```python
# Account data is synced to Account model
from djstripe.models import Account

account = Account.objects.get(id="acct_xxx")
# Account data accessible like any dj-stripe model
```

## Secure Coding Patterns

### Never Log Sensitive Data

```python
import logging

# ❌ WRONG: Logging full event data may expose PII
logger.info(f"Webhook received: {event.data}")

# ✅ CORRECT: Log only safe fields
logger.info(f"Webhook received: type={event.type}, id={event.id}")
```

### Environment-Specific Configuration

```python
# settings.py
STRIPE_LIVE_MODE = os.environ.get("STRIPE_LIVE_MODE", "False").lower() == "true"

if STRIPE_LIVE_MODE:
    STRIPE_SECRET_KEY = os.environ["STRIPE_LIVE_SECRET_KEY"]
else:
    STRIPE_SECRET_KEY = os.environ["STRIPE_TEST_SECRET_KEY"]
```

### Database Encryption

- Use PostgreSQL with encryption at rest
- Keep billing database on separate, secured instance
- Encrypt sensitive metadata stored in your models
- Never store raw card data, CVV, or full PAN

### Django Security Headers

```python
# settings.py
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

## Security Audit Checklist

### Before Going Live

- [ ] All Stripe keys in environment variables (not code)
- [ ] `STRIPE_LIVE_MODE` properly configured per environment
- [ ] Restricted API keys used for dj-stripe (principle of least privilege)
- [ ] Webhook endpoints use UUID URLs
- [ ] Webhook signature verification enabled (`djstripe_validation_method = "verify_signature"`, which is the default)
- [ ] `whsec_` secrets stored securely
- [ ] HTTPS enforced (SECURE_SSL_REDIRECT = True)
- [ ] No card data touches server (Stripe.js only)
- [ ] No Stripe keys in git history
- [ ] `.env` in `.gitignore`
- [ ] Database encryption at rest
- [ ] Secrets never logged
- [ ] Stripe Dashboard restricted to authorized team members
- [ ] 2FA enabled on all Stripe Dashboard accounts

### Ongoing

- [ ] Monitor Stripe Dashboard for suspicious activity
- [ ] Review API key permissions quarterly
- [ ] Rotate secrets if team members leave
- [ ] Keep dj-stripe and stripe-python updated
- [ ] Review Stripe changelog for security updates

## Common Security Mistakes

| Mistake | Risk | Fix |
|---|---|---|
| Keys in source code | Keys leaked in git history | Environment variables only |
| No webhook verification | Fake webhooks accepted | Set `djstripe_validation_method = "verify_signature"` in production |
| Logging full event data | PII exposure in logs | Only log safe fields |
| Using live keys in dev | Accidental real charges | Separate test/live keys per environment |
| Full secret keys (not restricted) | Broader access than needed | Use restricted keys (`rk_`) |
| Ignoring PCI requirements | Legal/financial liability | Use Stripe.js, never touch raw cards |
| No HTTPS in production | Man-in-the-middle attacks | Enforce HTTPS + HSTS |
