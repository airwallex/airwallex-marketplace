# Interaction Flows

## Table of Contents

- [How To Use This Reference](#how-to-use-this-reference)
- [Flow 1: Add Card For Recharge Authorization](#flow-1-add-card-for-recharge-authorization)
- [Flow 2: Manual Recharge With Existing Consent](#flow-2-manual-recharge-with-existing-consent)
- [Flow 3: Auto-Recharge Setup](#flow-3-auto-recharge-setup)
- [Flow 4: Subscribe With New Card](#flow-4-subscribe-with-new-card)
- [Flow 5: Subscribe With Saved Card](#flow-5-subscribe-with-saved-card)
- [Flow 6: Subscription Renewal](#flow-6-subscription-renewal)
- [Failure Lane Template](#failure-lane-template)
- [Plan Document Section Shape](#plan-document-section-shape)

---

## How To Use This Reference

Start planning with the shopper experience. Draw the interaction first, then map the steps to Airwallex objects and merchant backend actions.

Use these ASCII flows as templates and adapt labels, amounts, settings, and merchant-owned states to the product.

## Flow 1: Add Card For Recharge Authorization

Use when the shopper enters billing settings or recharge settings before subscribing.

```text
User
  |
  v
Billing / Payment settings
  |
  | clicks "Add card"
  v
Add card panel
  |
  | sees recharge authorization copy
  | enters card number / expiry / CVC
  | enters or confirms billing information
  | accepts recharge authorization
  v
Submit authorization
  |
  +--> field invalid
  |      |
  |      v
  |   show field error, keep user on form
  |
  +--> card details valid
         |
         v
      Merchant backend creates setup intent
         |
         v
      Split Card Elements confirm unscheduled MIT consent
         |
         +--> 3DS challenge required
         |      |
         |      v
         |   user completes bank verification
         |
         v
      Backend verifies result
         |
         +--> success
         |      |
         |      v
         |   show saved card + recharge authorization active
         |
         +--> failed
         |      |
         |      v
         |   show retry / use another card
         |
         +--> pending or unknown
                |
                v
             show confirming state; backend reconciles by webhook
```

Result:

```text
payment_method_id stored
unscheduled recharge_consent_id stored
```

## Flow 2: Manual Recharge With Existing Consent

Use when the shopper already has an active recharge consent. The visible action is manual recharge, but the actual card charge is merchant-initiated under the unscheduled agreement.

```text
User
  |
  v
Billing balance page
  |
  | clicks "Add balance"
  v
Recharge amount panel
  |
  | enters amount
  | sees selected saved card
  | confirms recharge request
  v
Merchant backend validates amount and limits
  |
  v
Merchant creates and confirms PaymentIntent
  |
  | uses recharge_consent_id
  v
Payment processing
  |
  +--> success
  |      |
  |      v
  |   credit wallet ledger and show new balance
  |
  +--> failed
  |      |
  |      v
  |   do not credit wallet; show failure and card update path
  |
  +--> pending
         |
         v
      show pending; update balance after webhook confirmation
```

Result:

```text
No card form
No CVC prompt
No new consent unless the user changes card or consent is missing
```

## Flow 3: Auto-Recharge Setup

Use when the shopper enables low-balance automatic recharge.

```text
User
  |
  v
Recharge settings
  |
  | sets minimum balance
  | sets recharge amount
  | sets optional limits
  | toggles auto-recharge on
  v
Consent check
  |
  +--> existing unscheduled consent
  |      |
  |      v
  |   save auto-recharge settings
  |
  +--> no unscheduled consent
         |
         v
      Add card flow
         |
         v
      save auto-recharge settings after consent success
```

Runtime:

```text
Token usage reduces balance
  |
  v
Balance below threshold
  |
  v
Merchant backend checks settings, limits, and concurrency lock
  |
  v
Merchant charges recharge_consent_id
  |
  +--> success: credit wallet and notify user
  +--> failure: pause or retry according to merchant policy
```

## Flow 4: Subscribe With New Card

Use when the shopper starts at a plan page and enters a new card during subscription signup.

```text
User
  |
  v
Plan selection
  |
  | chooses plan
  v
Subscription checkout
  |
  | sees amount, currency, billing period, renewal policy
  | chooses "New card"
  | enters card number / expiry / CVC
  | enters or confirms billing information
  | accepts subscription authorization
  v
Submit subscription authorization
  |
  +--> field invalid
  |      |
  |      v
  |   show field error, keep user on form
  |
  +--> card details valid
         |
         v
      Merchant backend creates setup or first-charge intent
         |
         v
      Split Card Elements confirm scheduled MIT consent
         |
         +--> 3DS challenge required
         |      |
         |      v
         |   user completes bank verification
         |
         v
      Backend verifies result
         |
         +--> success
         |      |
         |      v
         |   activate subscription and show next renewal date
         |
         +--> failed
         |      |
         |      v
         |   show retry / use another card
         |
         +--> pending or unknown
                |
                v
             show confirming state; do not activate until verified
```

Result:

```text
payment_method_id stored
scheduled subscription_consent_id stored on the subscription
```

## Flow 5: Subscribe With Saved Card

Use when the shopper has an existing saved card from recharge or another agreement.

```text
User
  |
  v
Plan selection
  |
  | chooses plan
  v
Subscription checkout
  |
  | sees amount, currency, billing period, renewal policy
  | selects saved card
  | confirms billing information
  | accepts subscription authorization
  v
Submit subscription authorization
  |
  v
Merchant backend creates setup or first-charge intent
  |
  v
Merchant confirms new scheduled MIT consent with selected payment_method_id
  |
  +--> extra authentication required
  |      |
  |      v
  |   complete required action if returned by Airwallex flow
  |
  +--> success
  |      |
  |      v
  |   activate subscription and show next renewal date
  |
  +--> failed
         |
         v
      show retry / choose another card
```

Result:

```text
existing payment_method_id reused
new scheduled subscription_consent_id stored
```

## Flow 6: Subscription Renewal

Use for later renewals. The shopper is not in session.

```text
Merchant renewal job
  |
  v
Find due subscription
  |
  v
Create renewal PaymentIntent
  |
  v
Confirm with subscription_consent_id
  |
  +--> success
  |      |
  |      v
  |   extend entitlement, write ledger, issue receipt
  |
  +--> retryable failure
  |      |
  |      v
  |   schedule retry, notify user, keep grace state if applicable
  |
  +--> hard failure
         |
         v
      mark payment failed, restrict entitlement by merchant policy
```

## Failure Lane Template

Use this as the default handling lane for card setup and subscription authorization:

```text
Submit
  |
  +--> field error
  |      -> show inline error, focus field, keep form editable
  |
  +--> authorization declined
  |      -> show use another card
  |
  +--> bank verification cancelled
  |      -> show retry verification
  |
  +--> network timeout
  |      -> show confirming state, check backend result, avoid duplicate setup
  |
  +--> webhook later confirms success
  |      -> update UI state on refresh or notification
  |
  +--> webhook later confirms failure
         -> show failed state and recovery path
```

## Plan Document Section Shape

When drafting the planning answer or the final plan document, use this order:

```text
1. ASCII user interaction flow
2. Business flow coverage
3. Consent type and reason for each flow
4. Split Card Elements behavior, if card collection is needed
5. Billing information collection and validation
6. Fraud-data fields and MCP verification notes
7. Backend calls and webhook confirmation
8. MCP-verified JS frontend examples for Split Card Elements
9. MCP-verified backend API call examples for PaymentIntent and PaymentConsent flows
10. Merchant ledger and business-state updates
11. Failure and recovery paths
12. Open questions
13. Confirmed file name and path
```
