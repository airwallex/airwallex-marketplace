---
name: airwallex-billing-checkout
description: Guide integration of Airwallex Billing Hosted Checkout for subscription, one-off payment (PAYMENT), and card-saving (SETUP) scenarios. Covers authentication, product/price creation, free trial, coupons, success page, webhooks, troubleshooting, and API version via .env. Use when user mentions billing checkout, subscription integration, recurring billing, free trial, coupon, promo code, billing_checkouts API, hosted billing page, 订阅, 周期扣款, 优惠券. Do NOT use for HPP / Hosted Payment Page / PaymentIntent; those belong to Online Payments, not Billing.
allowed-tools: Read Grep Bash Write Edit WebFetch
---
# Airwallex Billing Hosted Checkout

## 1. Description

This skill guides end-to-end integration of Airwallex **Billing Hosted Checkout**.

> **Canonical source:** the API steps below distill [Hosted Billing Checkout](https://www.airwallex.com/docs/billing/billing-components/checkout/hosted-billing-checkout) and the related Billing docs. Full mapping in [references/api-reference.md](references/api-reference.md). Those pages are authoritative. Prefer them if anything here drifts.

**How it differs from HPP (Hosted Payment Page):**

| Dimension | HPP (Online Payments) | Billing Hosted Checkout |
|-----------|----------------------|------------------------|
| Core API | `PaymentIntent` + `redirectToCheckout()` | `billing_checkouts/create` → returns `url` |
| Typical use case | One-off payments | Subscriptions / recurring billing / one-off invoices / card-only setup |
| Auto-created resources | None | Customer, Payment Source, Subscription |

**Checkout Modes:** `SUBSCRIPTION` · `PAYMENT` (one-off) · `SETUP` (card-saving only, requires `x-api-version >= 2025-08-29`)

## 2. Trigger

* Mentions "subscription", "recurring billing", "subscription integration"
* Mentions `billing_checkouts`, "hosted billing checkout", "billing checkout"
* Asks about Product/Price creation, pricing models, free trial, Coupon/discount
* Asks about `subscription.active`, `subscription.cancelled`, or other Billing Webhooks

## 3. Core Principles

* **Language**: Detect the user's language. Code, API fields, and enum values stay in English; explanations and comments localized.

* **Interaction first**: When intent is unclear, ask questions per §4 before outputting content; never dump all sections at once
* **Output strategy**: Only output sections relevant to the user's needs; use "conclusion → steps" structure
* **Security**: Use `<PLACEHOLDER>` for sensitive values in code examples
* **Environments**: Demo = `api-demo.airwallex.com`, Production = `api.airwallex.com`
* **API versioning**: Minimum `x-api-version` per capability and the current latest are in [api-reference.md §2](references/api-reference.md). **Set `AIRWALLEX_X_API_VERSION` in `.env`** (template [api-reference.md §1.1](references/api-reference.md)); examples below read that value and **do not hardcode a date**, so nobody assumes one fixed version is mandatory. If you rely on the account default and omit the header, confirm the default meets the §2 minimums for what you use.

## 4. Guided Intake Workflow

If the user's first message is already specific (e.g. "SUBSCRIPTION + 14-day trial + 20% off coupon"), skip known questions and jump straight to the relevant path.

### Round 1: Scenario & Mode

> **Q1: What's your business scenario?**
> - (A) Subscription / recurring billing → `SUBSCRIPTION`, continue to Round 2
> - (B) One-off invoice payment → `PAYMENT`, skip to Round 3
> - (C) Save card only, no immediate charge → `SETUP`, skip to Round 3
> - (D) Not sure → Follow-up: "Will customers repeatedly purchase the same product/service on a schedule? Yes → A; No → B"
> - (E) Already integrated, need to troubleshoot → Jump to §8 Troubleshooting

### Round 2: Subscription Details (SUBSCRIPTION only)

> **Q2: Free trial?** Yes → Ask for number of days; No → Skip
>
> **Q3: Coupon / discount?** Yes → Ask for type (percentage `PERCENTAGE` / fixed amount `FLAT`) and duration (`ONCE`/`CUSTOM`/`INDEFINITELY`); No → Skip
>
> **Q4: Pricing model?** Fixed `FLAT` / Per-unit `PER_UNIT` / Tiered `VOLUME` (unit price by total-quantity tier) or `GRADUATED` (unit price changes as quantity grows) / Not sure → Suggest `FLAT`
>
> Usage-based billing is **not** a `pricing_model`. Set `metered: true` plus a `meter_id` on the Price, then report usage via the Ingest Usage Events API.

### Round 3: General Info

> **Q5: Tech stack?** Frontend + backend language
>
> **Q6: Currency and region?**
>
> **Q7: Do you have an Airwallex account and Product/Price?**
> - Have both → Skip Step 1
> - Have account but no Product/Price → Start from Step 1
> - Have neither → Prompt to sign up and get API key
> - Have an existing HPP integration, want to migrate → See [api-reference.md §14](references/api-reference.md)

### Output Routing Table

| Signal | Output section |
|--------|----------------|
| `SUBSCRIPTION` | §5 (follow Step order) |
| `PAYMENT` | §6 |
| `SETUP` | §6.5 |
| Free trial = Yes | §5 Free trial notes |
| Coupon = Yes | §5 Coupon notes |
| Already has Product/Price | Skip Step 1 |
| Troubleshooting | §8 |
| All paths | End with Webhook + Testing + Production checklist |

### Fallback (when Billing Checkout isn't the right fit)

| User signal | Recommended alternative |
|-------------|------------------------|
| One-off online payment, no subscription concept | **HPP** (Hosted Payment Page) |
| Embed payment components in own page | **Embedded Elements** |
| Fully custom UI, PCI ROC certified | **Native API** |

## 5. SUBSCRIPTION Mode

### Step 0: Authentication

Call `POST /api/v1/authentication/login` (Headers: `x-client-id` + `x-api-key`) to obtain an `ACCESS_TOKEN` (valid 30 minutes; use the `expires_at` field as the source of truth). See [api-reference.md §1](references/api-reference.md). Recommended: manage credentials via `.env` (template in [api-reference.md §1.1](references/api-reference.md)).

### Step 1: Create Product and Price

> These `curl` snippets use `AIRWALLEX_X_API_VERSION` (same as `.env`). Export it before running (`export AIRWALLEX_X_API_VERSION=...`) or load `.env` with your tooling; the value must satisfy [api-reference.md §2](references/api-reference.md) for the features you use.

```shell
curl -X POST https://api-demo.airwallex.com/api/v1/products/create \
  -H 'Authorization: Bearer <ACCESS_TOKEN>' \
  -H 'Content-Type: application/json' \
  -H "x-api-version: ${AIRWALLEX_X_API_VERSION}" \
  -d '{ "request_id": "<UUID>", "name": "Basic Plan" }'
```

```shell
curl -X POST https://api-demo.airwallex.com/api/v1/prices/create \
  -H 'Authorization: Bearer <ACCESS_TOKEN>' \
  -H 'Content-Type: application/json' \
  -H "x-api-version: ${AIRWALLEX_X_API_VERSION}" \
  -d '{
    "request_id": "<UUID>",
    "product_id": "<PRODUCT_ID>",
    "currency": "USD",
    "pricing_model": "FLAT",
    "flat_amount": 10,
    "recurring": { "period": 1, "period_unit": "MONTH" }
  }'
```

> You can also create these via the Web App → Products & Prices UI.

### Step 1.5: Create Coupon (if discount needed)

Call `POST /api/v1/coupons/create`, setting `discount_model` (`PERCENTAGE`/`FLAT`) and `duration_type` (`ONCE`/`CUSTOM`/`INDEFINITELY`).

> **Field exclusivity**: `FLAT` uses `amount_off` **+** `currency`; `PERCENTAGE` uses `percentage_off` **only**. Sending `amount_off`/`currency` on a `PERCENTAGE` coupon returns `400 validation_error`; don't copy `currency` from the price onto the coupon.

```shell
curl -X POST https://api-demo.airwallex.com/api/v1/coupons/create \
  -H 'Authorization: Bearer <ACCESS_TOKEN>' \
  -H 'Content-Type: application/json' \
  -H "x-api-version: ${AIRWALLEX_X_API_VERSION}" \
  -d '{
    "request_id": "<UUID>",
    "name": "WELCOME20",
    "discount_model": "PERCENTAGE",
    "percentage_off": 20,
    "duration_type": "ONCE",
    "active": true
  }'
```

Note the returned `coup_xxx` ID for use in Step 3. Full examples (incl. `FLAT` + custom duration) and the end-to-end coupon → checkout block in [api-reference.md §10](references/api-reference.md).

### Step 2: Frontend

Frontend responsibilities: display plans → collect email → bind unique `request_id` → POST to backend → redirect to Checkout URL.

```javascript
async function handleSubscribe(email, priceId) {
  const res = await fetch('/create-checkout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, priceId, request_id: crypto.randomUUID() })
  });
  const { url, checkout_id } = await res.json();
  sessionStorage.setItem('checkout_id', checkout_id);
  window.location.href = url;
}
```

### Step 3: Backend: Create Billing Checkout (Core)

> **Pre-flight check (ask the user before generating code)**:
> - How many **Legal entities** are under your Airwallex account? → If more than one, `legal_entity_id` is **required**
> - How many **Payment accounts** are linked to your account? → If more than one, `linked_payment_account_id` is **required**
>
> If the user is unsure, ask them to check in Web App → **Account settings → Payment accounts** and **Legal entities**. A single-entity, single-account setup can omit both fields; otherwise missing them will return `Need to specify the linked_payment_account_id ...` or similar 400 errors.

```javascript
app.post('/create-checkout', async (req, res) => {
  const { email, priceId, request_id } = req.body;
  const accessToken = await getAccessToken();
  const apiVersion = process.env.AIRWALLEX_X_API_VERSION;
  if (!apiVersion) {
    return res.status(500).json({ error: 'Missing AIRWALLEX_X_API_VERSION in .env — see api-reference.md §1.1' });
  }

  const resp = await fetch(
    'https://api-demo.airwallex.com/api/v1/billing_checkouts/create',
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${accessToken}`,
        'x-api-version': apiVersion
      },
      body: JSON.stringify({
        mode: 'SUBSCRIPTION',
        // legal_entity_id: '<LEGAL_ENTITY_ID>',              // Required for multi-entity accounts
        // linked_payment_account_id: '<PAYMENT_ACCOUNT_ID>', // Required for multi-account accounts
        customer_data: { email },
        line_items: [{ price_id: priceId, quantity: 1 }],
        subscription_data: {
          // trial_ends_at: new Date(Date.now() + 14*86400e3).toISOString().replace(/\.\d{3}Z$/, '+0000'),
          // duration: { period: 12, period_unit: 'MONTH' },
          // days_until_due: 5
        },
        // discounts: [{ type: 'COUPON', coupon: { id: '<COUPON_ID>' } }],
        // locale: 'ZH',
        request_id,
        success_url: 'https://yoursite.com/success',
        back_url: 'https://yoursite.com/pricing'
      })
    }
  );
  const data = await resp.json();
  if (!resp.ok) return res.status(resp.status).json({ error: data.message });
  res.json({ url: data.url, checkout_id: data.id });
});
```

**Key fields:**

| Field | Description |
|-------|-------------|
| `mode` | `SUBSCRIPTION` / `PAYMENT` / `SETUP` |
| `subscription_data` | **Required for SUBSCRIPTION** (pass `{}` even if empty, otherwise 400) |
| `line_items` | `[{ price_id, quantity }]` |
| `request_id` | Idempotency key (UUID) |
| `success_url` | Redirect after successful payment |
| `back_url` | URL for the back button on the checkout page. Not to be confused with `cancel_url`, which no Airwallex API has |
| `discounts` | Checkout-level Coupon |
| `line_items[].discounts` | Line-item-level Coupon, stackable with Checkout-level |
| `locale` | Hosted page language (`EN`/`ZH`/`JA` etc.) |
| `legal_entity_id` | Required when multiple legal entities exist (can omit for single entity) |
| `linked_payment_account_id` | Required when multiple payment accounts exist (can omit for single account) |
| `trial_ends_at` | ISO8601 with timezone offset (e.g. `+0000`) |

> For Python / Java / Go backends, the logic is the same, POST JSON to the same endpoint. Python example in [api-reference.md §6.1](references/api-reference.md).

### Step 4: Success Page

After Checkout completes, the customer is redirected to `success_url`. Use the `checkout_id` stored in `sessionStorage` to query `GET /api/v1/billing_checkouts/{id}` and retrieve the `subscription_id`. **Always treat Webhooks as the authoritative source**; polling Checkout status is only for immediate frontend feedback. Detailed implementation in [api-reference.md §6.2](references/api-reference.md).

### Step 5: Webhooks

**Always rely on Webhooks, not client-side redirects.**

```javascript
app.post('/webhook', express.raw({ type: 'application/json' }), (req, res) => {
  const sig = req.headers['x-signature'];
  const ts = req.headers['x-timestamp'];
  const expected = crypto.createHmac('sha256', process.env.WEBHOOK_SECRET)
    .update(`${ts}${req.body.toString()}`).digest('hex');
  // Constant-time compare; guard length first (timingSafeEqual throws on a length mismatch).
  const sigBuf = Buffer.from(sig || '', 'hex');
  const expectedBuf = Buffer.from(expected, 'hex');
  if (sigBuf.length !== expectedBuf.length || !crypto.timingSafeEqual(sigBuf, expectedBuf)) {
    return res.sendStatus(401);
  }

  // Replay protection: reject deliveries whose timestamp is more than 5 minutes old.
  // `x-timestamp` is in milliseconds, so compare against Date.now() directly.
  if (Math.abs(Date.now() - Number(ts)) > 5 * 60 * 1000) return res.sendStatus(401);

  const event = JSON.parse(req.body.toString());
  switch (event.type) {
    case 'subscription.active':    /* Grant access */           break;
    case 'subscription.in_trial':  /* Provide trial features */ break;
    case 'subscription.cancelled': /* Revoke access */          break;
    case 'subscription.unpaid':    /* Notify to update payment */ break;
  }
  res.sendStatus(200);
});
```

> **Note**: `express.raw()` must be registered **before** `express.json()`, otherwise signature verification will fail. The sample compares signatures with `crypto.timingSafeEqual()` (guarding buffer length first) so signature validity is not leaked through response timing; a plain `sig !== expected` is timing-attack vulnerable. It also rejects deliveries older than 5 minutes (replay protection) and deduplicating by `event.id` is recommended, since the same event can be delivered more than once. Full details in [api-reference.md §8](references/api-reference.md).

### Step 6: Testing

1. **Test card**: Visa `4035 5010 0000 0008` (any expiry/CVC); **do not use** `4242…`. Full card list in the canonical [Test card numbers](https://www.airwallex.com/docs/payments/test-and-go-live/test-card-numbers) doc (see [api-reference.md §12](references/api-reference.md) for the billing-specific notes)
2. **Dashboard**: Web App → Subscriptions to verify the new subscription
3. **Webhooks**: Confirm events fire correctly
4. **Test Clock (Beta, optional)**: Simulate time progression; requires prior approval from MS team. See [api-reference.md §13](references/api-reference.md)

### Free Trial Notes

Set via `subscription_data.trial_ends_at` (ISO8601 with timezone). Trial period is independent of billing cycles. Omit the field to skip trial. Webhook flow: `subscription.in_trial` → `subscription.active`.

### Coupon Notes

Create Coupon in Step 1.5 → reference in Step 3 via `discounts` (Checkout-level) or `line_items[].discounts` (line-level). Line-level discounts are applied before Checkout-level; Checkout-level discounts are pro-rata distributed across line items. Hosted Checkout discounts are pre-filled via the API; users cannot enter promo codes on the page. If user input is needed, collect the code on your site and pass it in the `create` call. See [api-reference.md §10](references/api-reference.md).

## 6. PAYMENT Mode

One-off invoice payment. No `subscription_data`; use `invoice_data` instead. Creates an Invoice (not a Subscription) on completion. No Free trial. Coupons are supported.

```shell
curl -X POST https://api-demo.airwallex.com/api/v1/billing_checkouts/create \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <ACCESS_TOKEN>' \
  -H "x-api-version: ${AIRWALLEX_X_API_VERSION}" \
  -d '{
    "request_id": "<UUID>",
    "mode": "PAYMENT",
    "line_items": [{ "price_id": "<PRICE_ID>", "quantity": 10 }],
    "invoice_data": { "default_tax_percent": 10 },
    "success_url": "https://yoursite.com/success",
    "back_url": "https://yoursite.com/back"
  }'
```

## 6.5. SETUP Mode (Card-saving only)

Verifies and saves a Payment Source without charging. Requires `x-api-version >= 2025-08-29`. No `line_items`/`subscription_data`/`invoice_data` needed. Returns `billing_customer_id` + `payment_source_id` on completion, which can be used with the Subscription API or Invoice API later. See [api-reference.md §6.3](references/api-reference.md).

## 7. Post-Launch

* **Plan changes (upgrade/downgrade)**: `POST /api/v1/subscriptions/{id}/update`, replace `price_id`, set `proration_behavior` (`PRORATED`/`ALL`/`NONE`)
* **Cancellation**: `POST /api/v1/subscriptions/{id}/cancel`; for end-of-period cancellation, set `cancel_at_period_end: true`
* **Production launch checklist**: Domain switch, credentials, Price IDs, Webhook URL, signature verification, etc. (12 items). Full checklist in [api-reference.md §14](references/api-reference.md)

## 8. Troubleshooting

### A. Checkout Page

| Symptom | Investigation |
|---------|---------------|
| URL shows error | Checkout expires after **1 hour**; recreate |
| No payment methods shown | Verify Billing permissions are active and `linked_payment_account_id` is correct |
| 400 error | Check `source` field in response; verify `legal_entity_id` and environment match |
| Wrong page language | Set `locale` parameter |

### B. API Calls

| Symptom | Investigation |
|---------|---------------|
| `subscription_data must be provided` | SUBSCRIPTION mode requires `subscription_data: {}` |
| `resource_not_found` on price_id | Verify via `GET /api/v1/prices/list` |
| `request_id` conflict | Idempotent design; use a new UUID |
| `authentication_error` | Token expires after 30 min (use `expires_at`), re-login |
| `cancel_url` ignored | No Airwallex API accepts `cancel_url`. Billing Checkout uses `back_url`; HPP uses `failUrl` and `successUrl` |
| `Need to specify the linked_payment_account_id ...` | Account has multiple Payment Accounts. The request must explicitly include `linked_payment_account_id`; retrieve it from Web App → Account settings → Payment accounts (similarly `legal_entity_id` corresponds to "Need to specify the legal_entity_id ...") |

### C. Webhooks

| Symptom | Investigation |
|---------|---------------|
| Events not received | Verify URL is configured correctly and publicly accessible |
| Signature failure | Check Secret; confirm `express.raw()` is used for raw body |
| Status stuck at `PENDING` | Customer hasn't completed payment; possible risk block |

### D. Subscription Status

| Symptom | Investigation |
|---------|---------------|
| Not activated | Check `collection_method` and Payment Source validity |
| No charge after trial | Verify `trial_ends_at` format (timezone offset `+0000`); check auto-charge config |
| Still charging after cancellation | Check `cancel_at_period_end`; confirm cancellation request succeeded |

### E. Coupon

| Symptom | Investigation |
|---------|---------------|
| Redemption 400 | Verify `coup_xxx` exists and `active: true`; for `FLAT` type, `currency` must match |
| `validation_error`: `amount_off and currency should not be provided when discount_model is not FLAT` | `PERCENTAGE` coupons take `percentage_off` **only**; remove `amount_off`/`currency` (those are `FLAT`-only). Don't copy `currency` from the price/product onto the coupon. |
| Amount mismatch | Check `applied_discounts` in response; verify `duration_type` and stacking order |

---

*For detailed API schemas, complete code examples, and the production checklist, see [api-reference.md](references/api-reference.md).*
