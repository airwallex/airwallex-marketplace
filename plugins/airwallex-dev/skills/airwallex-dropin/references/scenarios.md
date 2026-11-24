# Airwallex Drop-in Element: Overview, Frontend Implementation & Scenarios

> **Canonical source:** this file distills the official Airwallex docs: [Drop-in element](https://www.airwallex.com/docs/payments/integration-options/web-checkout/drop-in-element), [Guest user checkout](https://www.airwallex.com/docs/payments/integration-options/web-checkout/drop-in-element/guest-user-checkout), and [Save and reuse payment details](https://www.airwallex.com/docs/payments/integration-options/web-checkout/save-and-reuse-payment-details). Those pages are authoritative. Prefer them if anything here drifts.

> **Backend APIs** (Create Customer, Create PaymentIntent, Get Saved Methods): see [Payments integration notes for AI agents](https://www.airwallex.com/docs/developer-tools/ai-agent-payments-integration.md)
> **MIT server-side & CIT vs MIT**: see [Payments integration notes for AI agents](https://www.airwallex.com/docs/developer-tools/ai-agent-payments-integration.md)
> **Styling and doc references**: see [styling.md](styling.md)

## Table of Contents

- [Overview](#overview)
- [Drop-in vs Other Integration Methods](#drop-in-vs-other-integration-methods)
- [Frontend Implementation](#frontend-implementation)
- [Scenarios](#scenarios)
  - [Scenario A: Guest Checkout](#scenario-a-guest-checkout)
  - [Scenario B: Payment + Save Card (CIT)](#scenario-b-payment--save-card-cit)
  - [Scenario B2: Save Card Without Payment](#scenario-b2-save-card-without-payment-zero-amount)
  - [Scenario C: Returning User with Saved Cards (CIT Subsequent)](#scenario-c-returning-user-with-saved-cards-cit-subsequent)
  - [Scenario D: Save Card for MIT](#scenario-d-save-card-for-mit-via-drop-in)
- [Troubleshooting](#troubleshooting)

---

## Overview

The Drop-in Element is an embedded UI component that renders inside an iframe on your page. It supports multiple payment methods through a single integration and handles input validation, error display, and 3DS automatically. For the full feature list (payment-method coverage, auto-adjusting fields, device fingerprinting, adding methods without code changes), see the [Drop-in element](https://www.airwallex.com/docs/payments/integration-options/web-checkout/drop-in-element) doc.

---

## Drop-in vs Other Integration Methods

| Feature | HPP | Drop-in | Split Card |
|---------|-----|---------|------------|
| Integration effort | Lowest | Medium | Highest |
| UI control | Limited (colors, layout, logo) | Medium (appearance + CSS rules) | Full (individual elements) |
| PCI scope | SAQ A | SAQ A | SAQ A |
| Payment methods | All supported | All supported | Card only |
| User stays on your site | No (redirect) | Yes (iframe) | Yes (iframe) |
| Event handling | Redirect URL | `element.on()` events | `element.confirm()` promise |

---

## Frontend Implementation

### Step 1: Install and Initialize

```bash
npm install @airwallex/components-sdk
```

```javascript
import { init, createElement } from '@airwallex/components-sdk';

await init({
  env: 'demo',  // 'prod' for production
  enabledElements: ['payments'],
});
```

### Step 2: HTML Container

```html
<div id="dropIn"></div>
```

The Drop-in renders its iframe inside this container. Style the container with your own CSS (width, margin, etc.).

### Step 3: Create and Mount Drop-in Element

> **`await init(...)` from Step 1 must have resolved before this runs**, otherwise the container mounts blank.

```javascript
// Prerequisite: await init({ enabledElements: ['payments'] }) has already completed (Step 1)
const element = createElement('dropIn', {
  intent_id: 'replace-with-your-intent-id',
  client_secret: 'replace-with-your-client-secret',
  currency: 'USD',
});

element.mount('dropIn');
```

### Step 4: Handle Events

Attach event listeners **after** calling `mount()`:

```javascript
// Ready — element is mounted and interactive (no payload)
element.on('ready', () => {
  console.log('Drop-in ready');
});

// Success — the payload is on e.detail: { intent, consent }
element.on('success', (e) => {
  const { intent, consent } = e.detail;   // intent is the PaymentIntent object
  console.log('Payment success:', intent.id, intent.status);
  // Show a "verifying…" state, then confirm server-side (Step 5) before showing success
});

// Error — the payload is on e.detail: { error }
element.on('error', (e) => {
  const { error } = e.detail;
  console.error('Payment error:', error.message);   // display error.message to the shopper
});
```

### Step 5: Verify Payment Result

The `success` event only means the shopper **completed the form**; it is not final confirmation. Always verify server-side via a webhook (`payment_intent.succeeded`) or the Retrieve PaymentIntent API before granting value.

**Event payloads** (all carried on `e.detail`):

| Event | `e.detail` |
|-------|-----------|
| `success` | `{ intent, consent }`, `intent` is the PaymentIntent object (`intent.id`, `intent.status`) |
| `error` | `{ error }`; use `error.message` for display |
| `ready` / `cancel` | no payload |

**Show a verifying state, then poll for the final status.** Right after `success`, the PaymentIntent is often still `PROCESSING` (e.g. just after a 3DS challenge), so a single immediate lookup can fail silently. Show a "Verifying payment…" state and poll your backend (or, preferably, react to the webhook):

```javascript
async function confirmPayment(intentId) {
  for (let i = 0; i < 10; i++) {
    // your backend calls Retrieve PaymentIntent and returns its status
    const { status } = await fetch(`/api/verify-payment?intent_id=${intentId}`).then((r) => r.json());
    if (status === 'SUCCEEDED') return showSuccess();
    if (status === 'FAILED' || status === 'CANCELLED') return showFailure(status);
    await new Promise((r) => setTimeout(r, 1500));   // still PROCESSING — wait and retry
  }
  showPending();   // fall back to "we'll confirm by email" — the webhook is the source of truth
}
```

> **UX**: always move Pay → "Verifying…" → an explicit success/error state; never leave the shopper with no feedback. The webhook (`payment_intent.succeeded`) remains the authoritative signal.

---

## Scenarios

### Scenario A: Guest Checkout

```
User → sees Drop-in on checkout page → selects payment method → pays → success event fires
```

- **Backend**: Create PaymentIntent (no `customer_id` needed)
- **Frontend**:

```javascript
const element = createElement('dropIn', {
  intent_id: intentId,
  client_secret: clientSecret,
  currency: 'USD',
  country_code: 'US',
});
element.mount('dropIn');

element.on('success', (e) => {
  const { intent } = e.detail;   // verify server-side before showing success (see Step 5)
  window.location.href = '/payment-success?id=' + intent.id;
});
element.on('error', (e) => {
  showErrorMessage(e.detail.error.message);
});
```

- **Key point**: Simplest flow. No customer object, no save card.

### Scenario B: Payment + Save Card (CIT)

```
User → sees Drop-in on checkout page → enters card
     → sees "Save card for future use" checkbox (optional)
     → pays → success event fires
```

- **Backend**: Create PaymentIntent with `customer_id` and actual `amount`
- **Frontend**:

```javascript
const element = createElement('dropIn', {
  intent_id: intentId,
  client_secret: clientSecret,
  currency: 'USD',
  country_code: 'US',
});
element.mount('dropIn');
```

- **Key point**: Because the PaymentIntent carries `customer_id`, Drop-in **displays a "Save card" checkbox**, **pre-selected by default**. Saving the card is the shopper's choice; they can either:
  - **Tick the checkbox** → card stored as a payment consent for future CIT.
  - **Leave it unticked** → one-off payment, nothing stored.
- No extra frontend flag is required. If the shopper opted in, query the PaymentIntent (or listen to the `payment_consent.created` webhook) to retrieve `payment_consent_id` for later reuse.

### Scenario B2: Save Card Without Payment (Zero-Amount)

```
User → sees Drop-in on "add payment method" page → enters card
     → sees "Save card" checkbox (optional) → submits → success event fires
```

- **Backend**: Create PaymentIntent with `customer_id` and **`amount: 0`** (see [Payments integration notes for AI agents](https://www.airwallex.com/docs/developer-tools/ai-agent-payments-integration.md))
- **Frontend**: Same as Scenario B, only `amount: 0` differs on backend.

```javascript
const element = createElement('dropIn', {
  intent_id: intentId,
  client_secret: clientSecret,
  currency: 'USD',
  country_code: 'US',
});
element.mount('dropIn');
```

- **Key point**: No funds captured. Shopper chooses whether to save via the checkbox; the $0 auth validates the card.

### Scenario C: Returning User with Saved Cards (CIT Subsequent)

```
Returning user → sees Drop-in with saved cards → selects card + enters CVC → pays → success event fires
```

- **Backend**: Create PaymentIntent with `customer_id` and actual `amount`
- **Frontend**:

```javascript
const element = createElement('dropIn', {
  intent_id: intentId,
  client_secret: clientSecret,
  currency: 'USD',
  country_code: 'US',
});
element.mount('dropIn');
```

- **Key point**: Drop-in **automatically** displays saved payment methods when the PaymentIntent has a `customer_id`. No extra configuration needed. Shoppers can select a saved card and enter CVC, or use a new card.

### Scenario D: Save Card for MIT (via Drop-in)

```javascript
const element = createElement('dropIn', {
  intent_id: intentId,
  client_secret: clientSecret,
  currency: 'USD',
  country_code: 'US',
  payment_consent: {
    next_triggered_by: 'merchant',
    merchant_trigger_reason: 'scheduled',   // or 'unscheduled' or 'installments'
  },
});
element.mount('dropIn');
```

> For `merchant_trigger_reason` values and subsequent MIT server-side payments, see [Payments integration notes for AI agents](https://www.airwallex.com/docs/developer-tools/ai-agent-payments-integration.md).

---

## Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| **Blank Drop-in container** after `mount()` (no iframe or form) | `init()` was not awaited before `createElement('dropIn', …)`. Ensure `await init({ enabledElements: ['payments'] })` completes first (Step 1). |
| **No feedback after clicking Pay** | (1) Read the result from **`e.detail.intent`**, not `event.id`. (2) The PaymentIntent is often still `PROCESSING` right after `success`, poll or await the webhook (Step 5) instead of a single immediate lookup. (3) Render a "Verifying…" state so the shopper isn't left guessing. |
| **Pay button seems stuck / nothing happens** | A 3DS challenge may be rendering **inside the Drop-in iframe**, the shopper must complete it there before `success`/`error` fires. |
