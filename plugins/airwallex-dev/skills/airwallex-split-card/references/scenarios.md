# Split Card Scenarios

## Table of Contents

- [Overview](#overview)
- [Page Layout](#page-layout)
- [Scenario A: One-time Payment (Guest)](#scenario-a-one-time-payment-guest-checkout)
- [Scenario B: Payment + Save Card (CIT)](#scenario-b-payment--save-card-cit-save)
- [Scenario B2: Save Card Without Payment](#scenario-b2-save-card-without-payment-zero-amount)
- [Scenario C: Pay with Saved Card (CIT Subsequent)](#scenario-c-pay-with-saved-card-cit-subsequent)
- [Scenario D: Save Card for MIT](#scenario-d-save-card-for-mit)
- [CVC Element for Saved Cards](#cvc-element-for-saved-cards)
- [Frontend Decision Flow](#frontend-decision-flow)

---

## Overview

The payment page displays Split Card components (CardNumber / Expiry / CVC) with a **"Save this card" checkbox**:

| User Action | Integration Mode | Description |
|-------------|-----------------|-------------|
| **Unchecked** save card | Guest Checkout | One-time payment, card not saved |
| **Checked** save card (CIT) | CIT save during payment | Payment + save card, customer triggers future payments |
| **Checked** save card (MIT) | MIT save during payment | Payment + save card, merchant triggers future payments (subscriptions, auto top-ups) |
| **Save card only** (no payment) | Zero-amount save | Collect card details for future use without charging (amount=0) |
| **Returning user** selects saved card | CIT subsequent payment | Show saved card list, only CVC input required |
| **Merchant** charges saved card | MIT subsequent payment | Server-side only, no customer session needed |

---

## Page Layout

```
┌──────────────────────────────────────────────────────┐
│                   Payment Page                        │
│                                                       │
│  ┌─ New User / New Card ───────────────────────────┐  │
│  │  [CardNumber Element]                           │  │
│  │  [Expiry Element]    [CVC Element]              │  │
│  │  ☐ Save this card for future payments           │  │
│  └─────────────────────────────────────────────────┘  │
│                                                       │
│  ┌─ Returning User (has saved cards) ──────────────┐  │
│  │  ● VISA **** 4242  Exp 12/26                    │  │
│  │  ○ MasterCard **** 5555  Exp 08/27              │  │
│  │  ○ Use a new card                               │  │
│  │                                                  │  │
│  │  [CVC Element]  ← shown when saved card selected│  │
│  └──────────────────────────────────────────────────┘  │
│                                                       │
│  ┌─ Billing Address ───────────────────────────────┐  │
│  │  ☑ Same as shipping address                     │  │
│  │  (or expand billing form)                       │  │
│  └─────────────────────────────────────────────────┘  │
│                                                       │
│  [Confirm Payment]                                    │
└──────────────────────────────────────────────────────┘
```

---

## Scenario A: One-time Payment (Guest Checkout)

```
User → enters card number/expiry/CVC → does NOT check "save card" → clicks pay
```

- **Backend**: Create PaymentIntent (no `customer_id` needed)
- **Frontend**: `cardNumber.confirm({ intent_id, client_secret, payment_method: { card: { name }, billing } })`
- **Key point**: Simplest flow, no Customer object needed; billing passed via `payment_method.billing`

## Scenario B: Payment + Save Card (CIT Save)

```
User → enters card number/expiry/CVC → checks "save card" → clicks pay
```

- **Backend**: Create PaymentIntent (**must** include `customer_id`)
- **Frontend**: `cardNumber.confirm({ intent_id, client_secret, payment_consent: { next_triggered_by: 'customer' }, payment_method: { card: { name }, billing } })`
- **Key point**: Single operation completes payment + saves card + associates billing; `next_triggered_by: 'customer'` means subsequent payments are customer-initiated

## Scenario B2: Save Card Without Payment (Zero-Amount)

```
User → enters card number/expiry/CVC → on "Add payment method" page (no order) → clicks save
```

This scenario is for collecting card details **without charging**. Common use cases: account settings "add a card" page, pre-registration before first purchase, wallet management.

- **Backend**: Create PaymentIntent with **`amount: 0`** and `customer_id`

- **Frontend**: Same Split Card Elements (cardNumber / expiry / cvc), but the page context is "add a card" rather than checkout.

### B2-CIT: Save for future customer-initiated payments

```javascript
cardNumber.confirm({
  intent_id: intentId,
  client_secret: clientSecret,
  payment_consent: {
    next_triggered_by: 'customer',
  },
  payment_method: {
    card: { name: cardholderName },
    billing,
  },
});
```

### B2-MIT: Save for future merchant-initiated payments (e.g. subscription sign-up, first charge later)

```javascript
// Scheduled recurring (e.g. monthly subscription)
cardNumber.confirm({
  intent_id: intentId,
  client_secret: clientSecret,
  payment_consent: {
    next_triggered_by: 'merchant',
    merchant_trigger_reason: 'scheduled',
  },
  payment_method: {
    card: { name: cardholderName },
    billing,
  },
});

// Unscheduled (e.g. auto top-up when balance is low)
cardNumber.confirm({
  intent_id: intentId,
  client_secret: clientSecret,
  payment_consent: {
    next_triggered_by: 'merchant',
    merchant_trigger_reason: 'unscheduled',
  },
  payment_method: {
    card: { name: cardholderName },
    billing,
  },
});
```

- **Key difference from Scenario B**: Backend creates PaymentIntent with `amount: 0`. No funds are captured. The card is validated (a $0 auth may occur) and saved for future use.
- **Response**: Contains `payment_consent_id` for subsequent payments.
- **CIT vs MIT**: CIT requires CVC on subsequent use; MIT allows server-side charge without customer session.

## Scenario C: Pay with Saved Card (CIT Subsequent)

```
Returning user → sees saved card list → selects a card → enters CVC → clicks pay
```

- **Backend**:
  1. Call `GET /payment_methods?customer_id=xxx` to retrieve saved cards
  2. Create PaymentIntent (with `customer_id`)
- **Frontend**:
  1. Render saved card list
  2. Mount a separate CVC Element
  3. `savedCardCvc.confirm({ intent_id, client_secret, payment_method_id, triggered_by: 'customer' })`
- **Key point**: No need to re-enter card number and expiry, only CVC for identity verification; billing is already saved with the card. To override billing for this payment, pass `payment_method: { card: {}, billing }` in the confirm call.

---

## Scenario D: Save Card for MIT

When saving a card for future merchant-initiated charges, set `next_triggered_by: 'merchant'` with a `merchant_trigger_reason`:

```javascript
// Save card for scheduled recurring payments (e.g. monthly subscription)
cardNumber.confirm({
  intent_id: intentId,
  client_secret: clientSecret,
  payment_consent: {
    next_triggered_by: 'merchant',
    merchant_trigger_reason: 'scheduled',   // or 'unscheduled' or 'installments'
  },
  payment_method: {
    card: { name: cardholderName },
    billing,
  },
});
```

### Installments with terms_of_use

When using `installments`, you must provide additional details:

```javascript
cardNumber.confirm({
  intent_id: intentId,
  client_secret: clientSecret,
  payment_consent: {
    next_triggered_by: 'merchant',
    merchant_trigger_reason: 'installments',
    terms_of_use: {
      payment_amount_type: 'FIXED',       // 'FIXED' or 'VARIABLE'
      fixed_payment_amount: 50,           // required if FIXED
      payment_currency: 'USD',
      start_date: '2026-05-01',
      end_date: '2026-12-01',
      total_billing_cycles: 8,
      payment_schedule: {
        period: 1,                        // every 1 month
        period_unit: 'MONTH',             // 'DAY' | 'WEEK' | 'MONTH' | 'YEAR'
      },
    },
  },
  payment_method: {
    card: { name: cardholderName },
    billing,
  },
});
```

**TermsOfUse type definition:**

```typescript
interface TermsOfUse {
  payment_amount_type: 'FIXED' | 'VARIABLE';
  fixed_payment_amount?: number;    // required if FIXED
  max_payment_amount?: number;      // optional if VARIABLE
  first_payment_amount?: number;    // optional, for first payment with setup fees
  payment_currency: string;
  start_date: string;               // e.g. '2026-05-01'
  end_date?: string;
  total_billing_cycles?: number;    // null = indefinite
  billing_cycle_charge_day?: number; // required when period_unit is WEEK/MONTH/YEAR
  payment_schedule?: {
    period: number;                 // e.g. 1 = every 1 unit
    period_unit: 'DAY' | 'WEEK' | 'MONTH' | 'YEAR';
  };
}
```

---

## CVC Element for Saved Cards

When a user selects a saved card, display a standalone CVC input:

```javascript
// Create a separate CVC Element (different instance from the new card CVC)
const savedCardCvc = createElement('cvc', {
  cvcLength: 3,  // 3 for Visa/MasterCard/JCB, 4 for AMEX
});

// Mount to the saved card CVC container
savedCardCvc.mount('savedCardCvc');

// Use savedCardCvc.confirm() to submit payment
savedCardCvc.confirm({
  intent_id: intentId,
  client_secret: clientSecret,
  payment_method_id: 'mtd_xxx',  // the selected saved card ID
  triggered_by: 'customer',
});
```

> **Key points**:
> - CVC Element can only be mounted **once** per payment flow
> - `cvcLength` depends on card brand: AMEX = 4, others = 3
> - The confirm caller for saved cards is `savedCardCvc` (CVC Element), NOT `cardNumber`
> - Must pass `payment_method_id` and `triggered_by: 'customer'`

---

## Frontend Decision Flow

```
User enters payment page
    │
    ├── User logged in?
    │   ├── Yes → fetch saved card list from backend
    │   │   ├── Has saved cards → show card list + CVC + "use new card" option
    │   │   └── No saved cards → show new card input + "save card" checkbox
    │   └── No → show new card input only (pure Guest, no "save card" option)
    │
    └── User clicks "Confirm Payment"
        │
        ├── Build billing: "same as shipping" checked → reuse shippingAddress
        │                  unchecked                  → get from billing form
        │
        ├── Saved card selected → savedCardCvc.confirm({ payment_method_id, triggered_by: 'customer' })
        │                         billing already saved with card
        ├── New card + save checked → cardNumber.confirm({ payment_consent, payment_method: { card, billing } })
        └── New card + unchecked     → cardNumber.confirm({ payment_method: { card, billing } })  (Guest)
```
