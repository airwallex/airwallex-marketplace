# Airwallex Hosted Payment Page (HPP): Overview, Frontend Implementation & Scenarios

> **Backend APIs** (Create Customer, Create PaymentIntent, Get Saved Methods): see [Payments integration notes for AI agents](https://www.airwallex.com/docs/developer-tools/ai-agent-payments-integration.md)
> **MIT server-side & CIT vs MIT**: see [Payments integration notes for AI agents](https://www.airwallex.com/docs/developer-tools/ai-agent-payments-integration.md)
> **Styling and doc references**: see [styling.md](styling.md)

## Table of Contents

- [Overview](#overview)
- [HPP vs Other Integration Methods](#hpp-vs-other-integration-methods)
- [Frontend Implementation](#frontend-implementation)
- [Scenarios](#scenarios)
  - [Scenario A: Guest Checkout](#scenario-a-guest-checkout)
  - [Scenario B: Payment + Save Card (CIT)](#scenario-b-payment--save-card-cit)
  - [Scenario B2: Save Card Without Payment](#scenario-b2-save-card-without-payment-zero-amount)
  - [Scenario C: Returning User with Saved Cards (CIT Subsequent)](#scenario-c-returning-user-with-saved-cards-cit-subsequent)
  - [Scenario D: Save Card for MIT](#scenario-d-save-card-for-mit-via-hpp)

---

## Overview

The Hosted Payment Page (HPP) is a **redirect-based** checkout solution. Instead of embedding payment UI on your site, you redirect shoppers to a secure, pre-built payment page hosted by Airwallex.

**Key characteristics:**
- No iframe, no `createElement`, no `mount`, just `redirectToCheckout()`
- Accepts multiple payment methods through a single integration
- Airwallex handles all payment UI, input validation, 3DS, and device fingerprinting
- Lowest PCI-DSS scope (SAQ A)
- Supports appearance customization (colors, layout, logo)

---

## HPP vs Other Integration Methods

| Feature | HPP | Drop-in | Split Card |
|---------|-----|---------|------------|
| Integration effort | Lowest | Medium | Highest |
| UI control | Limited (colors, layout, logo) | Medium (appearance + CSS rules) | Full (individual elements) |
| PCI scope | SAQ A | SAQ A | SAQ A |
| Payment methods | All supported | All supported | Card only |
| User stays on your site | No (redirect) | Yes (iframe) | Yes (iframe) |

---

## Frontend Implementation

### Step 1: Install and Initialize

```bash
npm install @airwallex/components-sdk
```

```javascript
import { init } from '@airwallex/components-sdk';

// HPP requires enabledElements: ['payments'] to get the payments object
const { payments } = await init({
  env: 'demo',  // 'prod' for production
  enabledElements: ['payments'],
});
```

### Step 2: Redirect to Checkout

```javascript
payments.redirectToCheckout({
  intent_id: 'replace-with-your-intent-id',
  client_secret: 'replace-with-your-client-secret',
  currency: 'USD',
  country_code: 'US',
  successUrl: 'https://yoursite.com/payment-success',
});
```

The shopper is redirected to Airwallex's hosted page. After payment, they are redirected to `successUrl`.

### Step 3: Handle Return

On successful payment, the shopper is redirected to your `successUrl` with the PaymentIntent id appended as the `id` query param:
```
https://yoursite.com/payment-success?id=int_xxxxx
```

**Verify server-side, and show a "verifying" state while you do.** Right after the redirect the PaymentIntent is often still `PROCESSING` (e.g. just after a 3DS challenge), so a single immediate lookup can miss the final status. Poll your backend (or, preferably, react to the `payment_intent.succeeded` webhook):

```javascript
// On your success page
const intentId = new URLSearchParams(window.location.search).get('id');

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
confirmPayment(intentId);
```

> **Critical**: Never trust the client-side redirect alone. The `payment_intent.succeeded` webhook (or the Retrieve PaymentIntent API) is the authoritative signal, always resolve the shopper to an explicit success/error state, never a blank or hanging page.

---

## Scenarios

### Scenario A: Guest Checkout

```
User → clicks "Checkout" → redirected to HPP → enters card → pays → redirected back
```

- **Backend**: Create PaymentIntent (no `customer_id` needed)
- **Frontend**:

```javascript
payments.redirectToCheckout({
  intent_id: intentId,
  client_secret: clientSecret,
  currency: 'USD',
  country_code: 'US',
  successUrl: 'https://yoursite.com/payment-success',
});
```

- **Key point**: Simplest flow. No customer object, no save card.

### Scenario B: Payment + Save Card (CIT)

```
User → clicks "Checkout" → redirected to HPP → enters card
     → sees "Save card for future use" checkbox (optional)
     → pays → redirected back
```

- **Backend**: Create PaymentIntent with `customer_id` and actual `amount`
- **Frontend**:

```javascript
payments.redirectToCheckout({
  intent_id: intentId,
  client_secret: clientSecret,
  currency: 'USD',
  country_code: 'US',
  successUrl: 'https://yoursite.com/payment-success',
});
```

- **Key point**: Because the PaymentIntent carries `customer_id`, HPP **displays a "Save card" checkbox** on the payment form, **pre-selected by default**. Saving the card is the shopper's choice; they can either:
  - **Tick the checkbox** → card stored as a payment consent for future CIT (returning-user checkout).
  - **Leave it unticked** → one-off payment, nothing stored.
- No extra frontend flag is required; simply providing `customer_id` on the PaymentIntent is enough for HPP to surface the option.
- **Response**: If the shopper opted in, query the PaymentIntent (or listen to the `payment_consent.created` webhook) to retrieve the resulting `payment_consent_id` for later reuse.

### Scenario B2: Save Card Without Payment (Zero-Amount)

```
User → clicks "Add payment method" → redirected to HPP → enters card → card saved → redirected back
```

- **Backend**: Create PaymentIntent with `customer_id` and **`amount: 0`** (see [Payments integration notes for AI agents](https://www.airwallex.com/docs/developer-tools/ai-agent-payments-integration.md))
- **Frontend**: Same as Scenario B on the backend except `amount: 0`; on the frontend, add a `payment_consent` block to record the save-only intent (shown below).

```javascript
payments.redirectToCheckout({
  intent_id: intentId,
  client_secret: clientSecret,
  currency: 'USD',
  country_code: 'US',
  payment_consent: {
    next_triggered_by: 'customer',   // or 'merchant' for MIT setup
  },
  successUrl: 'https://yoursite.com/card-saved',
});
```

- **Key point**: No funds captured. Card validated via $0 auth and saved for future use.

### Scenario C: Returning User with Saved Cards (CIT Subsequent)

```
Returning user → clicks "Checkout" → redirected to HPP → sees saved cards → selects card + enters CVC → pays → redirected back
```

- **Backend**: Create PaymentIntent with `customer_id` and actual `amount`
- **Frontend**:

```javascript
payments.redirectToCheckout({
  intent_id: intentId,
  client_secret: clientSecret,
  currency: 'USD',
  country_code: 'US',
  successUrl: 'https://yoursite.com/payment-success',
});
```

- **Key point**: HPP **automatically** displays saved cards when the PaymentIntent has a `customer_id`. No extra frontend code needed. The shopper can select a saved card and enter CVC, or use a new card.

### Scenario D: Save Card for MIT (via HPP)

```javascript
// Scheduled recurring (e.g. monthly subscription)
payments.redirectToCheckout({
  intent_id: intentId,
  client_secret: clientSecret,
  currency: 'USD',
  country_code: 'US',
  payment_consent: {
    next_triggered_by: 'merchant',
    merchant_trigger_reason: 'scheduled',
  },
  successUrl: 'https://yoursite.com/subscription-confirmed',
});
```

> For `merchant_trigger_reason` values and subsequent MIT server-side payments, see [Payments integration notes for AI agents](https://www.airwallex.com/docs/developer-tools/ai-agent-payments-integration.md).
