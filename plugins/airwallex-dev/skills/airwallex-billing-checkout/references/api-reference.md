# Airwallex Billing Hosted Checkout: API Reference

This document supplements SKILL.md with complete API schemas, Webhook event listings, and Subscription lifecycle details.

> **Canonical source:** the Billing content here distills the official Airwallex docs: [Hosted Billing Checkout](https://www.airwallex.com/docs/billing/billing-components/checkout/hosted-billing-checkout), [Products](https://www.airwallex.com/docs/billing/billing-components/products/products-via-api), [Prices](https://www.airwallex.com/docs/billing/billing-components/prices/prices-via-api), [Coupons](https://www.airwallex.com/docs/billing/billing-components/coupons/coupons-via-api), [Subscriptions](https://www.airwallex.com/docs/billing/subscriptions/subscriptions-via-api), and [Invoices](https://www.airwallex.com/docs/billing/invoicing/invoices-via-api) via API. (The pure `/docs/api/billing/...` endpoint pages come from a separate API-docs system.) Those pages are authoritative. Prefer them if anything here drifts.

---

## Table of Contents

- [1. Authentication](#1-authentication)
- [2. API Versioning](#2-api-versioning)
- [3. Core Billing Resources](#3-core-billing-resources)
- [3.5. Error Response Schema](#35-error-response-schema)
- [4. Product API](#4-product-api)
- [5. Price API](#5-price-api)
- [6. Billing Checkout API](#6-billing-checkout-api)
- [7. Subscription API](#7-subscription-api)
- [8. Webhook Events (Billing)](#8-webhook-events-billing)
- [9. Free Trial Configuration](#9-free-trial-configuration)
- [10. Coupons & Discounts](#10-coupons--discounts)
- [11. Tax Configuration](#11-tax-configuration)
- [12. Sandbox Test Cards (Official Airwallex)](#12-sandbox-test-cards-official-airwallex)
- [13. Test Clock (Beta)](#13-test-clock-beta)
- [14. Production Launch Checklist & HPP Migration](#14-production-launch-checklist--hpp-migration)

---


## 1. Authentication

All API calls require an Access Token first:

```shell
curl -X POST https://api-demo.airwallex.com/api/v1/authentication/login \
  -H 'Content-Type: application/json' \
  -H 'x-client-id: <CLIENT_ID>' \
  -H 'x-api-key: <API_KEY>'
```

The `token` field in the response is the `ACCESS_TOKEN`, used in subsequent requests as `Authorization: Bearer <ACCESS_TOKEN>`. Tokens are valid for **30 minutes**; use the `expires_at` field in the response as the source of truth and refresh before it expires.

### 1.1 Environment Variable Management (Recommended)

Create a `.env` file to manage credentials and configuration centrally instead of hardcoding them:

```
# .env.example — copy to .env and fill in (add .env to .gitignore)
AIRWALLEX_CLIENT_ID=<YOUR_CLIENT_ID>
AIRWALLEX_API_KEY=<YOUR_API_KEY>
# Set after checking Airwallex docs/changelog; must be >= the §2 minimum for features you use. Below is a common “full coverage” starting example, not the only valid value.
AIRWALLEX_X_API_VERSION=2025-08-29

# Billing Price IDs (fill in after Step 1)
BILLING_PRICE_BASIC_MONTHLY=
BILLING_PRICE_BASIC_YEARLY=

# Optional
BILLING_TRIAL_DAYS=14
WEBHOOK_SECRET=<YOUR_WEBHOOK_SECRET>
```

> **Security reminder**: `.env` contains sensitive credentials, always add it to `.gitignore` and never commit to version control.

---

## 2. API Versioning

Airwallex uses **date-based version numbers** to manage breaking changes. Each account has a default version, which can be overridden per-request via the **`x-api-version`** header.

**Authoritative references**: [API versioning](https://www.airwallex.com/docs/api/versioning) and [API changelog](https://www.airwallex.com/docs/api/changelog)

### Minimum Version Requirements for Billing Features

| Feature | Minimum Version | Notes |
|---------|-----------------|-------|
| Billing Checkout resource (`SUBSCRIPTION` mode) | **`2025-06-16`** | Introduces `billing_checkouts`, `billing_customers`, `payment_sources`, `billing_transactions`, Credit Notes, new Subscription/Invoice fields |
| `PAYMENT` mode, `SETUP` mode | **`2025-08-29`** | Adds `mode` field (`SUBSCRIPTION` / `PAYMENT` / `SETUP`), `invoice_data`, `invoice_id` |
| Subscription `duration` field | **`2025-08-29`** | Replaces legacy `total_billing_cycles` / `remaining_billing_cycles`; `subscription_data.duration` for Checkout |
| Billing pagination changes (`page` replaces `page_num`) | **`2025-08-29`** | All Billing list endpoints use new pagination structure |
| Payment Source `external_id` points to Payment Method | **`2025-11-11`** | `external_id` changes from Payment Consent ID to Payment Method ID |
| Coupons / Discounts | **`2025-06-16`+** | Coupon resource available after `2025-06-16`; exact fields per latest docs |

**Current latest version**: `2026-02-27`

> **Aligned with SKILL.md examples**: Code and `curl` snippets should read `AIRWALLEX_X_API_VERSION` from `.env` instead of hardcoding a date in the repo; regression-test against the official changelog before bumping.

### How to Set the Version

**Option 1: Per-request (recommended during testing/migration)**

Add the HTTP header (value matches `AIRWALLEX_X_API_VERSION` in `.env`):

```
x-api-version: <AIRWALLEX_X_API_VERSION>
```

Full example:

```shell
curl -X POST https://api-demo.airwallex.com/api/v1/billing_checkouts/create \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <ACCESS_TOKEN>' \
  -H "x-api-version: ${AIRWALLEX_X_API_VERSION}" \
  -d '{ ... }'
```

**Option 2: Upgrade account default version (recommended after production stabilization)**

After migrating production traffic to the new version, contact Airwallex Support to update your account default. Takes effect within ~1 hour, after which you can remove the `x-api-version` header.

### Common Version Mismatch Symptoms

| Symptom | Cause |
|---------|-------|
| `billing_checkouts/create` returns 404 or unknown endpoint | Account version below `2025-06-16`; resource doesn't exist |
| `mode` field is ignored or errors | Version below `2025-08-29`, `PAYMENT` / `SETUP` modes unavailable |
| `duration` field invalid | Version below `2025-08-29`; still using legacy `total_billing_cycles` |
| Pagination response missing `page_before` / `page_after` | Version below `2025-08-29`, old pagination structure |
| `billing_customer_id` not recognized | Version below `2025-06-16`; still using old `customer_id` |

---

## 3. Core Billing Resources

| Resource | API Path | Description |
|----------|----------|-------------|
| Billing Customer | `/api/v1/billing_customers` | Customer who purchases or subscribes, with contact and billing preferences |
| Product | `/api/v1/products` | Goods or services (e.g. Basic Plan, Pro Plan) |
| Price | `/api/v1/prices` | Pricing for a Product, amount, currency, billing frequency |
| Payment Source | `/api/v1/payment_sources` | Customer's payment instrument (e.g. credit card) used for Invoice payment |
| Subscription | `/api/v1/subscriptions` | Recurring purchase relationship between customer and product |
| Billing Checkout | `/api/v1/billing_checkouts` | Manages the lifecycle of Airwallex's hosted Checkout page |
| Coupon | `/api/v1/coupons` | Reusable discount template; once redeemed, becomes a Discount |

---

## 3.5. Error Response Schema

All Airwallex API errors follow a uniform structure:

```json
{
  "code": "validation_error",
  "message": "subscription_data must be provided for SUBSCRIPTION mode in checkout.",
  "source": "subscription_data"
}
```

**Common error codes:**

| `code` | Meaning | Common scenarios |
|--------|---------|------------------|
| `validation_error` | Request parameter validation failed | Missing field, wrong format, invalid value (e.g. `subscription_data` not provided, `trial_ends_at` format error) |
| `resource_not_found` | Resource does not exist | Invalid Price ID, Coupon ID, or Subscription ID |
| `authentication_error` | Authentication failed | Access Token expired or invalid |
| `idempotency_error` | Idempotency conflict | `request_id` reused with different parameters |
| `not_supported` | Feature not supported | API version too low for the field (e.g. `duration` needs `>= 2025-08-29`) |
| `rate_limit_exceeded` | Rate limit exceeded | Too many requests in a short period |

**`source` field**: Points to the erroring field path (e.g. `subscription_data`, `line_items[0].price_id`, `discounts[0].coupon.id`), helpful for pinpointing issues. Some errors (e.g. authentication) may not include `source`.

### HTTP Status Code to `code` Mapping

Developers can triage by HTTP status first, then parse `code` for detailed handling:

| HTTP Status | Common `code` | Recommended action |
|-------------|--------------|---------------------|
| **400** | `validation_error`, `idempotency_error` | Fix request parameters and retry |
| **401** | `authentication_error` | Re-call `authentication/login` for a new Token |
| **403** | `forbidden` | Verify account permissions: Billing enabled, API Key scope |
| **404** | `resource_not_found` | Verify resource IDs (Price, Coupon, Subscription, Checkout) |
| **409** | `idempotency_error` | `request_id` already used with different params; use a new UUID |
| **422** | `not_supported` | API version too low; upgrade `x-api-version` or contact Support |
| **429** | `rate_limit_exceeded` | Exponential backoff (e.g. 1s → 2s → 4s), or reduce concurrency |
| **500 / 502 / 503** | Server error | Temporary Airwallex issue; retry after delay, contact Support if persistent |

**Backend error handling example (Node.js):**

```javascript
const res = await fetch(url, { method: 'POST', headers, body });
if (!res.ok) {
  const err = await res.json().catch(() => ({}));
  if (res.status === 401) {
    accessToken = await refreshToken();
    return retry();
  }
  if (res.status === 429) {
    await sleep(retryDelay);
    return retry();
  }
  throw new Error(`[${res.status}] ${err.code}: ${err.message} (source: ${err.source})`);
}
```

---

## 4. Product API

### Create Product

```
POST /api/v1/products/create
```

**Request body:**

```json
{
  "request_id": "string (UUID, required)",
  "name": "string (product name, required)",
  "description": "string (optional)"
}
```

**Response key fields:**

```json
{
  "id": "prod_xxxx",
  "name": "Basic Plan",
  "description": "Core features for individuals",
  "created_at": "2025-01-01T00:00:00+0000"
}
```

### List Products

```
GET /api/v1/products/list
```

---

## 5. Price API

### Create Price

```
POST /api/v1/prices/create
```

**Request body:**

```json
{
  "request_id": "string (UUID, required)",
  "product_id": "string (associated Product ID, required)",
  "currency": "string (e.g. USD, required)",
  "pricing_model": "FLAT | PER_UNIT | VOLUME | GRADUATED",
  "flat_amount": 10,
  "recurring": {
    "period": 1,
    "period_unit": "DAY | WEEK | MONTH | YEAR"
  }
}
```

**Pricing models:**

| Model | Description | Use case |
|-------|-------------|----------|
| `FLAT` | Fixed price regardless of quantity. Requires `flat_amount` | SaaS monthly fee, membership |
| `PER_UNIT` | Fixed price per unit of quantity. Requires `unit_amount`. Default when `pricing_model` is omitted | Per-seat licensing |
| `VOLUME` | Unit price determined by which tier the **total** quantity falls into. Requires `tiers` | Bulk discounts |
| `GRADUATED` | Unit price changes as quantity increases, charged per tier. Requires `tiers` | API usage billing |

The last entry in `tiers` must omit `upper_bound`.

> **Usage-based billing is not a `pricing_model`.** Set `metered: true` and a `meter_id` on the Price, then report consumption via the Ingest Usage Events API. Sending `TIERED` or `USAGE` as a `pricing_model` fails with `400 validation_error`.

**Response key fields:**

```json
{
  "id": "pri_xxxx",
  "product_id": "prod_xxxx",
  "currency": "USD",
  "pricing_model": "FLAT",
  "flat_amount": 10,
  "recurring": { "period": 1, "period_unit": "MONTH" }
}
```

### List Prices

```
GET /api/v1/prices/list
```

---

## 6. Billing Checkout API

### Create Billing Checkout

```
POST /api/v1/billing_checkouts/create
```

#### SUBSCRIPTION Mode Request Body

```json
{
  "request_id": "string (UUID, required)",
  "mode": "SUBSCRIPTION",
  "legal_entity_id": "string (required for multi-entity accounts, omit for single)",
  "linked_payment_account_id": "string (required for multi-account, omit for single)",
  "customer_data": {
    "email": "string (customer email)"
  },
  "line_items": [
    {
      "price_id": "string (required)",
      "quantity": 1
    }
  ],
  "subscription_data": {
    "Required for SUBSCRIPTION mode, pass {} even if empty": "",
    "trial_ends_at": "string (ISO8601 with timezone, optional)",
    "days_until_due": 5,
    "default_tax_percent": 10,
    "default_invoice_template": {
      "invoice_memo": "string (optional)"
    },
    "duration": {
      "period": 12,
      "period_unit": "MONTH"
    }
  },
  "discounts": [
    {
      "type": "COUPON",
      "coupon": { "id": "string (Coupon ID, e.g. coup_xxx)" }
    }
  ],
  "success_url": "string (required)",
  "back_url": "string (optional, redirect when user clicks back)"
}
```

> **`back_url` vs `cancel_url`**: Billing Hosted Checkout uses **`back_url`**. `cancel_url` is not an Airwallex field in any product, so it is silently ignored here. Developers arriving from Stripe reach for it out of habit; HPP has no `cancel_url` either, and routes shoppers via `successUrl` and `failUrl` in `redirectToCheckout()`.

> **`subscription_data` required**: Omitting `subscription_data` in SUBSCRIPTION mode returns `400 validation_error`. Even without trial/duration, you must pass an empty object `"subscription_data": {}`.

**Line-item-level Coupon (stackable with Checkout-level):** Add a `discounts` array inside `line_items[]`, using the same structure as above.

```json
{
  "line_items": [
    {
      "price_id": "pri_xxxx",
      "quantity": 1,
      "discounts": [
        {
          "type": "COUPON",
          "coupon": { "id": "coup_xxxx" }
        }
      ]
    }
  ]
}
```

> **Application order**: Line-item discounts are applied before Checkout/subscription-level discounts; Checkout-level discounts are **pro-rata** distributed across line items based on their relative subtotals (see "Coupons via API", Pro-rata distribution).

#### SETUP Mode Request Body

`SETUP` mode only verifies and saves a customer's payment method (Payment Source) without charging. Use for save-card-first workflows, with subsequent charges via Subscription API or Invoice API. Requires `x-api-version >= 2025-08-29`.

```json
{
  "request_id": "string (UUID, required)",
  "mode": "SETUP",
  "legal_entity_id": "string (required for multi-entity)",
  "linked_payment_account_id": "string (required for multi-account)",
  "customer_data": {
    "email": "string (customer email)"
  },
  "success_url": "string (required)",
  "back_url": "string (optional)"
}
```

> **Note**: `SETUP` mode does not require `line_items`, `subscription_data`, or `invoice_data`. On completion, the system automatically creates a Billing Customer and Payment Source.

#### PAYMENT Mode Request Body

```json
{
  "request_id": "string (UUID, required)",
  "mode": "PAYMENT",
  "legal_entity_id": "string (required for multi-entity, omit for single)",
  "linked_payment_account_id": "string (required for multi-account, omit for single)",
  "line_items": [
    {
      "price_id": "string (required)",
      "quantity": 10
    }
  ],
  "invoice_data": {
    "default_tax_percent": 10,
    "memo": "string (optional)"
  },
  "discounts": [
    {
      "type": "COUPON",
      "coupon": { "id": "string (optional, Checkout-level Coupon)" }
    }
  ],
  "success_url": "string (required)",
  "back_url": "string (optional)"
}
```

`line_items[].discounts` is also available in `PAYMENT` mode, with the same structure as `SUBSCRIPTION`.

#### Response (on creation: ACTIVE status)

```json
{
  "id": "bco_xxxx",
  "url": "https://checkout.airwallex.com/billing/...",
  "status": "ACTIVE",
  "mode": "SUBSCRIPTION",
  "created_at": "2025-01-01T00:00:00+0000",
  "expires_at": "2025-01-01T01:00:00+0000"
}
```

Return the `url` to the frontend and redirect the customer to complete payment.

#### Response (after completion: COMPLETED status, via GET)

```json
{
  "id": "bco_xxxx",
  "status": "COMPLETED",
  "mode": "SUBSCRIPTION",
  "subscription_id": "sub_xxxx",
  "billing_customer_id": "cus_xxxx",
  "payment_source_id": "ps_xxxx"
}
```

> **Retrieving `subscription_id`**: After Checkout completion, query via `GET /api/v1/billing_checkouts/{id}`, the response includes `subscription_id`. You can also obtain it asynchronously via the `subscription.created` / `subscription.active` Webhook events (recommended).

### Billing Checkout Status Lifecycle

```
ACTIVE ──→ COMPLETED   (customer completes payment)
  │
  ├──→ CANCELLED       (merchant cancels via API)
  │
  └──→ EXPIRED         (not completed within 1 hour)
```

| Status | Description |
|--------|-------------|
| `ACTIVE` | Checkout is available for the customer |
| `COMPLETED` | Payment completed or payment method verified |
| `CANCELLED` | Cancelled by merchant via Cancel API |
| `EXPIRED` | Auto-expired 1 hour after creation |

### 6.1 Python (Flask) Minimal Example

```python
import requests, uuid, os

@app.route('/create-checkout', methods=['POST'])
def create_checkout():
    data = request.get_json()
    token = get_access_token()
    api_version = os.environ['AIRWALLEX_X_API_VERSION']  # §1.1 .env; must meet §2 minimums

    body = {
        'mode': 'SUBSCRIPTION',
        'customer_data': {'email': data['email']},
        'line_items': [{'price_id': data['priceId'], 'quantity': 1}],
        'subscription_data': {},
        'request_id': data.get('request_id', str(uuid.uuid4())),
        'success_url': f"{os.environ['BASE_URL']}/success",
        'back_url': f"{os.environ['BASE_URL']}/pricing",
    }

    resp = requests.post(
        'https://api-demo.airwallex.com/api/v1/billing_checkouts/create',
        json=body,
        headers={
            'Authorization': f'Bearer {token}',
            'Content-Type': 'application/json',
            'x-api-version': api_version,
        },
    )
    resp.raise_for_status()
    result = resp.json()
    return {'url': result['url'], 'checkout_id': result['id']}
```

### 6.2 Success Page Handling (Retrieving subscription_id)

After Checkout completes, the customer is redirected to `success_url`. **Do not rely solely on the redirect to confirm the subscription**, verify via the backend.

**Approaches for passing `checkout_id` to the Success page:**

> **Key constraint**: `success_url` is sent to Airwallex when creating the Checkout, at which point the API **has not yet returned** the `checkout_id` (`bco_xxx`). Therefore, you cannot append the actual `checkout_id` to the URL query string.

| Approach | Implementation | Suitable for |
|----------|---------------|--------------|
| **`sessionStorage` (recommended)** | Frontend stores `checkout_id` from API response before redirect | Simple and reliable, same-tab |
| **Backend mapping** | Append `request_id` to `success_url`, backend maintains `request_id → checkout_id` map | Cross-tab, requires Redis etc. |
| **Webhook only** | Don't pass `checkout_id`; backend gets `subscription_id` asynchronously | Most reliable, no frontend feedback |

> **Recommended combo**: `sessionStorage` for immediate frontend feedback + Webhook as authoritative confirmation.

**Frontend Success page logic:**

```javascript
async function handleSuccessPage() {
  const checkoutId = sessionStorage.getItem('checkout_id');
  if (!checkoutId) { showFallbackMessage(); return; }

  const res = await fetch(`/api/billing-checkout/${checkoutId}`);
  const data = await res.json();

  if (data.status === 'COMPLETED' && data.subscription_id) {
    showSuccessMessage(data.subscription_id);
  } else {
    showPendingMessage();
  }
  sessionStorage.removeItem('checkout_id');
}
```

**Backend Checkout status query:**

```javascript
app.get('/api/billing-checkout/:id', async (req, res) => {
  const accessToken = await getAccessToken();
  const apiVersion = process.env.AIRWALLEX_X_API_VERSION;
  if (!apiVersion) return res.status(500).json({ error: 'Missing AIRWALLEX_X_API_VERSION' });
  const checkout = await fetch(
    `https://api-demo.airwallex.com/api/v1/billing_checkouts/${req.params.id}`,
    { headers: { Authorization: `Bearer ${accessToken}`, 'x-api-version': apiVersion } }
  );
  const data = await checkout.json();
  res.json({ id: data.id, status: data.status, subscription_id: data.subscription_id });
});
```

**PAYMENT mode Success page**: Returns `invoice_id` (not `subscription_id`) on completion; otherwise the logic is the same.

### 6.3 SETUP Mode Full Example

```javascript
const apiVersion = process.env.AIRWALLEX_X_API_VERSION;
if (!apiVersion) throw new Error('Missing AIRWALLEX_X_API_VERSION');
const checkout = await fetch(
  'https://api-demo.airwallex.com/api/v1/billing_checkouts/create',
  {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${accessToken}`,
      'x-api-version': apiVersion
    },
    body: JSON.stringify({
      mode: 'SETUP',
      customer_data: { email },
      request_id: crypto.randomUUID(),
      success_url: 'https://yoursite.com/setup-success',
      back_url: 'https://yoursite.com/pricing'
    })
  }
);
```

On completion, the response includes `billing_customer_id` + `payment_source_id`, which can be used with the Subscription API or Invoice API for subsequent charges.

### Update Billing Checkout

```
POST /api/v1/billing_checkouts/{id}/update
```

Only supports updating the `metadata` field.

### Cancel Billing Checkout

```
POST /api/v1/billing_checkouts/{id}/cancel
```

Only `ACTIVE` Checkouts can be cancelled.

---

## 7. Subscription API

### Create Subscription (direct creation, not via Checkout)

```
POST /api/v1/subscriptions/create
```

**Key fields:**

| Field | Description |
|-------|-------------|
| `billing_customer_id` | Required, Billing Customer ID |
| `collection_method` | `AUTO_CHARGE` / `CHARGE_ON_CHECKOUT` / `OUT_OF_BAND` (offline) |
| `items[].price_id` | Required, Price ID |
| `duration.period_unit` | `DAY` / `WEEK` / `MONTH` / `YEAR` |
| `legal_entity_id` | Legal entity ID |
| `trial_ends_at` | Free trial end time |

> When using Billing Hosted Checkout, the Subscription is automatically created by the Checkout; you typically don't need to call this API manually.

### Update Subscription

```
POST /api/v1/subscriptions/{id}/update
```

Updatable fields depend on current status:
- `PENDING`: `starts_at`, `trial_ends_at`, `days_until_due`, `cancel_at_period_end`, `metadata`, `payment_source_id`
- `IN_TRIAL`: `trial_ends_at`
- `ACTIVE`: `days_until_due`, `cancel_at_period_end`, `metadata`, `payment_source_id`, `invoice_memo`

### Cancel Subscription

```
POST /api/v1/subscriptions/{id}/cancel
```

**Proration behavior (`proration_behavior`):**

| Value | Description |
|-------|-------------|
| `ALL` | Full refund of current cycle |
| `PRORATED` | Pro-rated refund for unused days |
| `NONE` | No refund |

### Subscription Status Lifecycle

```
PENDING ──→ IN_TRIAL ──→ ACTIVE ──→ CANCELLED
  │                        │
  │                        ├──→ UNPAID
  │                        │
  │                        └──→ EXPIRED
  │
  └──→ ACTIVE (direct activation when no trial)
```

| Status | Description |
|--------|-------------|
| `PENDING` | Subscription created, awaiting start |
| `IN_TRIAL` | In free trial |
| `ACTIVE` | Subscription active, billing normally |
| `UNPAID` | Payment failed, awaiting updated payment method |
| `CANCELLED` | Subscription cancelled (irreversible) |
| `EXPIRED` | Subscription expired |

### Plan Changes (Upgrade/Downgrade)

Use the Update Subscription API to change subscription items (`items`) for plan changes. Billing Hosted Checkout **does not directly support** switching an existing subscription's plan in a single Checkout; this must be done via the Subscription API on the backend.

**Typical flow:**

1. **Query current subscription**: `GET /api/v1/subscriptions/{id}` to get current `items` and `billing_customer_id`
2. **Update subscription items**: `POST /api/v1/subscriptions/{id}/update`, replace `price_id` in `items` with the new plan
3. **Handle proration**: Specify `proration_behavior` in the update request

```json
{
  "items": [
    { "price_id": "<NEW_PRICE_ID>", "quantity": 1 }
  ],
  "proration_behavior": "PRORATED"
}
```

| `proration_behavior` | Upgrade scenario | Downgrade scenario |
|---------------------|-----------------|-------------------|
| `PRORATED` | Pro-rate charge for price difference | Pro-rate refund for price difference |
| `ALL` | Full charge at new price | Full refund at old price |
| `NONE` | No adjustment, takes effect next cycle | No adjustment, takes effect next cycle |

> **Note**: Plan changes are done via the Subscription API, no new Billing Checkout is needed. The customer does not need to re-enter payment information.

---

## 8. Webhook Events (Billing)

Configure your Webhook URL in Airwallex Web App → Webhooks to receive the following Billing events:

### Subscription Events

| Event Type | Triggered When | Recommended Action |
|------------|---------------|-------------------|
| `subscription.created` | Subscription is created | Record subscription info |
| `subscription.active` | Subscription enters ACTIVE state | Grant service access |
| `subscription.in_trial` | Enters free trial | Provide trial features |
| `subscription.unpaid` | Payment failed | Notify customer to update payment method |
| `subscription.cancelled` | Subscription cancelled | Revoke service access |
| `subscription.expired` | Subscription expired | Clean up resources, prompt renewal |
| `subscription.updated` | Subscription info changed | Sync local records |

### Invoice Events

| Event Type | Triggered When | Recommended Action |
|------------|---------------|-------------------|
| `invoice.created` | Invoice created | Record billing info |
| `invoice.paid` | Invoice paid successfully | Confirm payment received |
| `invoice.payment_failed` | Invoice payment failed | Notify customer |
| `invoice.voided` | Invoice voided | Update local status |

### Webhook Signature Verification

Production environments **must** verify signatures. Airwallex includes these headers with every Webhook request:

| Header | Description |
|--------|-------------|
| `x-timestamp` | Unix timestamp in **milliseconds** when the request was sent (e.g. `1357872222592`) |
| `x-signature` | Hex digest of `HMAC-SHA256(secret, timestamp + rawBody)` |

**Verification steps:**

```javascript
import crypto from 'crypto';

function verifyWebhookSignature(rawBody, headers, secret) {
  const timestamp = headers['x-timestamp'];
  const signature = headers['x-signature'];

  const payload  = `${timestamp}${rawBody}`;
  const expected = crypto.createHmac('sha256', secret).update(payload).digest('hex');

  // Constant-time compare; guard length first (timingSafeEqual throws on a length mismatch).
  const sigBuf = Buffer.from(signature || '', 'hex');
  const expectedBuf = Buffer.from(expected, 'hex');
  if (sigBuf.length !== expectedBuf.length || !crypto.timingSafeEqual(sigBuf, expectedBuf)) {
    throw new Error('Webhook signature verification failed');
  }

  // Optional: check timestamp is within 5 minutes to prevent replay attacks.
  // `x-timestamp` is in milliseconds, so compare against Date.now() directly —
  // dividing by 1000 here would compare seconds to milliseconds and reject every delivery.
  const ageMs = Math.abs(Date.now() - Number(timestamp));
  if (ageMs > 5 * 60 * 1000) {
    throw new Error('Webhook timestamp too old');
  }
}
```

> **Secret retrieval**: Find the Signing Secret in Airwallex Web App → Webhooks → corresponding Endpoint settings.

### Webhook Best Practices

1. **Always return 200 OK**: Even if processing logic errors, return 200 first, otherwise Airwallex will retry
2. **Idempotent processing**: The same event may be delivered multiple times, deduplicate by `event.id`
3. **Async processing**: Use message queues for time-consuming operations; return 200 within 5 seconds
4. **Signature verification**: See verification section above
5. **Raw body**: Signature is computed on the raw request body; in Express, use `express.raw({ type: 'application/json' })` middleware for unparsed body
6. **Express middleware conflict**: Global `app.use(express.json())` will parse the body first, causing the Webhook route to receive a parsed object instead of a raw Buffer, signature verification will **always fail**. Solutions:

```javascript
// Option 1: Register Webhook route before express.json()
app.post('/webhook', express.raw({ type: 'application/json' }), webhookHandler);
app.use(express.json());

// Option 2: Isolate with Router
const webhookRouter = express.Router();
webhookRouter.use(express.raw({ type: 'application/json' }));
webhookRouter.post('/', webhookHandler);
app.use('/webhook', webhookRouter);
```

---

## 9. Free Trial Configuration

Free trial is **independent of** billing cycles: total subscription duration = **Trial duration + billing duration** (trial does not count toward the paid cycles represented by `duration`).

**Setting trial via Billing Checkout (`SUBSCRIPTION` mode):**

Specify `trial_ends_at` in `subscription_data` (ISO8601 with timezone):

```json
{
  "subscription_data": {
    "trial_ends_at": "2025-02-01T00:00:00+0000",
    "duration": { "period": 12, "period_unit": "MONTH" }
  }
}
```

* **Trial start**: Determined by the subscription's `starts_at`; defaults to subscription creation time if not specified.
* **No trial**: Simply omit `trial_ends_at`.
* **PAYMENT mode**: One-off `PAYMENT` Checkout has no `subscription_data` and does not support Free trial, trial is subscription-only.

**Setting trial via Subscription API directly:**

When creating a Subscription, pass `trial_ends_at` at the request body root level (different path from Checkout auto-creation, same logic). See "Subscriptions via API" docs.

---

## 10. Coupons & Discounts

### Create Coupon

```
POST /api/v1/coupons/create
```

**Common fields:**

| Field | Description |
|-------|-------------|
| `name` | Customer-visible name, displayed on Invoice / Checkout |
| `discount_model` | `FLAT` (fixed amount off) or `PERCENTAGE` (percentage, 0-100) |
| `amount_off` + `currency` | Required for `FLAT` |
| `percentage_off` | Required for `PERCENTAGE` |
| `duration_type` | `ONCE` (first cycle only) / `CUSTOM` (multiple cycles) / `INDEFINITELY` |
| `duration` | Required when `duration_type` is `CUSTOM`: `period` + `period_unit` |
| `expires_at` | Optional, latest time the Coupon can be redeemed |
| `active` | `true` allows redemption; set to `false` to deactivate (see Update Coupon) |
| `metadata` | Optional, for campaign/channel tagging |

> **Field exclusivity (per `discount_model`), mutually exclusive; sending the wrong pair returns `400 validation_error`:**
> - **`FLAT`** → send `amount_off` **and** `currency`; do **not** send `percentage_off`.
> - **`PERCENTAGE`** → send `percentage_off` **only**; do **not** send `amount_off` or `currency`. A common mistake is copying `currency` from the price/product onto the coupon → `amount_off and currency should not be provided when discount_model is not FLAT`.

**Example, percentage discount for custom duration:**

```shell
curl -X POST https://api-demo.airwallex.com/api/v1/coupons/create \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  -H 'Authorization: Bearer <ACCESS_TOKEN>' \
  -d '{
    "request_id": "unique-uuid-here",
    "name": "WELCOME20_3MO",
    "discount_model": "PERCENTAGE",
    "percentage_off": 20,
    "duration_type": "CUSTOM",
    "duration": { "period": 3, "period_unit": "MONTH" },
    "active": true,
    "description": "20% off for first 3 billing cycles"
  }'
```

**Example, fixed amount, one-time only:**

```shell
curl -X POST https://api-demo.airwallex.com/api/v1/coupons/create \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  -H 'Authorization: Bearer <ACCESS_TOKEN>' \
  -d '{
    "request_id": "unique-uuid-here",
    "name": "TAKE10USD",
    "discount_model": "FLAT",
    "amount_off": 10,
    "currency": "USD",
    "duration_type": "ONCE",
    "active": true
  }'
```

### Update / Archive Coupon

```
POST /api/v1/coupons/{id}/update
```

Set `active` to `false` to archive the Coupon, preventing **new** redemptions (handling of existing Discounts follows official documentation).

### Apply to Billing Checkout

In `POST /api/v1/billing_checkouts/create`:

* **Checkout-level**: Root field `discounts: [{ "type": "COUPON", "coupon": { "id": "coup_xxx" } }]`
* **Line-level**: `line_items[].discounts`, same structure

**Application order & distribution**: Line-item discounts are applied first; Checkout/subscription-level discounts are **pro-rata** distributed across line items based on their relative subtotals, for alignment with line-level tax calculations; lines reflect `discount_amount` etc. (see "Coupons via API").

**End-to-end (coupon → checkout):**

```shell
# 1) Create the coupon → response returns { "id": "coup_xxx", ... }
curl -X POST https://api-demo.airwallex.com/api/v1/coupons/create \
  -H 'Authorization: Bearer <ACCESS_TOKEN>' -H 'Content-Type: application/json' \
  -H "x-api-version: ${AIRWALLEX_X_API_VERSION}" \
  -d '{ "request_id": "<UUID>", "name": "WELCOME20", "discount_model": "PERCENTAGE", "percentage_off": 20, "duration_type": "ONCE", "active": true }'

# 2) Reference the returned coup_xxx in the checkout's discounts array
curl -X POST https://api-demo.airwallex.com/api/v1/billing_checkouts/create \
  -H 'Authorization: Bearer <ACCESS_TOKEN>' -H 'Content-Type: application/json' \
  -H "x-api-version: ${AIRWALLEX_X_API_VERSION}" \
  -d '{
    "request_id": "<UUID>",
    "mode": "SUBSCRIPTION",
    "line_items": [{ "price_id": "<PRICE_ID>", "quantity": 1 }],
    "subscription_data": {},
    "discounts": [{ "type": "COUPON", "coupon": { "id": "coup_xxx" } }],
    "success_url": "https://yoursite.com/success",
    "back_url": "https://yoursite.com/pricing"
  }'
```

### `applied_discounts` Audit Field

In Subscription / Invoice API responses, `applied_discounts` describes each discount's source and calculation:

* `discount_model`, `amount_off` / `percentage_off`
* `duration_type` and `ends_at` when `CUSTOM`
* `source` = `COUPON`, `source_id` = Coupon ID
* `applied_to`: `SUBSCRIPTION` / `SUBSCRIPTION_ITEM` / `INVOICE` / `INVOICE_LINE_ITEM` etc.

### Other Management Endpoints

* **Retrieve**: `GET /api/v1/coupons/{id}`
* **List**: `GET /api/v1/coupons/list`

---

## 11. Tax Configuration

Set `default_tax_percent` (0-100) in `subscription_data` or `invoice_data`. Tax is exclusive (added on top).

```json
{
  "subscription_data": {
    "default_tax_percent": 10
  }
}
```

Generated Invoices will automatically include tax calculations.

---

## 12. Sandbox Test Cards (Official Airwallex)

**Authoritative source:** [Test card numbers](https://www.airwallex.com/docs/payments/test-and-go-live/test-card-numbers). Use the **Payment scenarios** and **3DS authentication scenarios** tables there for current PANs, amount-triggered decline rules, 3DS and OTP behaviour, and Registered user checkout, Apple Pay, Google Pay, and AVS conditions.

Billing-specific notes (not obvious from the doc):

* **Do not use** test cards common to other gateways such as `4242...` or `4000...`; they are not in the Airwallex list.
* For a successful payment, avoid the `80.xx` amount pattern, which the doc uses to trigger issuer or risk declines.
* Card numbers may be entered with or without spaces.

---

## 13. Test Clock (Beta)

### Overview

Test Clock enables **simulated time progression** in the Demo environment, accelerating testing of Billing subscription lifecycles without waiting for real time to pass. When time is advanced, the system immediately executes all time-dependent behaviors for the associated Customer's Billing resources.

### Activation

| Step | Description |
|------|-------------|
| 1 | Confirm you're using an **Organisation-enabled** Demo account |
| 2 | Contact the **Managed Service (MS) team** |
| 3 | Provide your **Organisation ID** |
| 4 | MS team enables Test Clock in the Demo environment |

> Beta phase: only available to specific integration customers; not available in production.

### Usage Flow (via Airwallex Demo Web App UI)

**Step 1, Create Test Clock**

Select a Billing Customer and click "Create Test Clock" to create one.

**Step 2, Move Test Clock**

Select a target date/time and click "Move Test Clock". The system will immediately simulate all Billing behaviors between the current Test Clock time and the target time:

- Subscription state transitions (e.g. `IN_TRIAL` → `ACTIVE`, `ACTIVE` → `UNPAID`)
- Invoice generation and payment (renewal Invoices, payment retries)
- Webhook event delivery (`subscription.active`, `invoice.paid`, `subscription.unpaid`, etc.)

**Step 3, Verify**

Check:
- Subscription status changed as expected
- Invoices generated and paid correctly
- Webhooks fired with correct payload
- Your system responded correctly to each event

### Important Limitations

| Limitation | Description |
|-----------|-------------|
| **Irreversible** | Once a Customer is associated with a Test Clock, all their Billing resources are permanently detached from real time and only respond to Test Clock time |
| **Advance limit** | Each advance is capped at **2 billing cycles** (based on the minimum billing frequency across all of the Customer's Subscriptions) |
| **Demo only** | Only available in Demo environment; not supported in production |
| **Organisation only** | Requires an Organisation-enabled Demo account |

### Typical Test Scenarios

| Scenario | Steps |
|----------|-------|
| Annual renewal | Create yearly Subscription → advance Test Clock by 12 months → verify renewal Invoice + Webhook |
| Trial expiration | Create Subscription with `trial_ends_at` → advance to trial end → verify `subscription.active` + first charge |
| Failed payment retry | Associate a failing Payment Source → advance to billing date → verify `subscription.unpaid` + retry logic |
| Multi-cycle renewal | Monthly Subscription → advance 2 months → verify 2 Invoices generated + 2 Webhooks |

---

## 14. Production Launch Checklist & HPP Migration

### Production Launch Checklist

Before switching from Sandbox to production, verify each item:

| # | Check Item | Action |
|---|-----------|--------|
| 1 | **API domain** | `api-demo.airwallex.com` → `api.airwallex.com` |
| 2 | **API credentials** | Use production `CLIENT_ID` and `API_KEY` |
| 3 | **Price IDs** | Recreate Product/Price in production, update `.env` |
| 4 | **Coupon IDs** | If applicable, recreate in production |
| 5 | **Webhook URL** | Configure production endpoint (publicly accessible, HTTPS) |
| 6 | **Webhook Secret** | Use production Signing Secret |
| 7 | **Signature verification** | Confirm using `crypto.timingSafeEqual()` (timing-attack safe) |
| 8 | **`success_url` / `back_url`** | Update to production domain |
| 9 | **`legal_entity_id`** | Confirm production account entity ID |
| 10 | **`AIRWALLEX_X_API_VERSION`** | `.env` / production config matches what you tested and meets §2 minimums for features in use |
| 11 | **Idempotency & retries** | `request_id` uses unique UUIDs; Webhook handling is idempotent |
| 12 | **Error monitoring** | Set up alerts for Checkout creation failures and Webhook processing errors |

> Recommended: run a small-amount real transaction for end-to-end verification before full launch.

### Migrating from HPP (Hosted Payment Page)

| HPP Concept | Billing Checkout Equivalent |
|-------------|----------------------------|
| `PaymentIntent` + `redirectToCheckout()` | `billing_checkouts/create` → returns `url` |
| `successUrl` / `failUrl` in `redirectToCheckout()` | `success_url` / **`back_url`** in the create call (different names, and set server-side) |
| Manual Customer management | Checkout auto-creates Billing Customer + Payment Source |
| `PaymentIntent` concept | Replaced by `line_items` + `Price` |
| Card-saving only | `mode: SETUP` |

---

*This reference document is built from the Airwallex Billing API documentation, serving as a progressive supplement to SKILL.md.*
