# Frontend Split Card Elements

> **Canonical source:** the Split Card Element behavior described here maps to [Split Card Element](https://www.airwallex.com/docs/payments/integration-options/web-checkout/embedded-elements/split-card-element). Verify current SDK shapes via the Airwallex docs MCP. That doc is authoritative for field names.

## Table of Contents

- [Scope](#scope)
- [Layout](#layout)
- [Initialization](#initialization)
- [Fraud Data Collection](#fraud-data-collection)
- [Form State](#form-state)
- [User Flow States](#user-flow-states)
- [Submit Flow: Card Binding Or Recharge Authorization](#submit-flow-card-binding-or-recharge-authorization)
- [Submit Flow: Subscription Authorization](#submit-flow-subscription-authorization)
- [3DS Handling](#3ds-handling)
- [Error Handling](#error-handling)
- [Success Handling](#success-handling)

---

## Scope

Use only the Airwallex Web JS SDK Split Card Elements:

- `cardNumber`
- `expiry`
- `cvc`

The frontend collects card details and starts the authorization flow. It must not store raw card data and must not decide final success without backend confirmation.

Split Card Elements collect card number, expiry, and CVC only. The merchant frontend must separately collect or confirm billing information needed for card authorization and fraud screening.

For product planning, read `interaction-flows.md` first and start with the relevant ASCII journey. Use this file after the journey is clear to specify element behavior, validation, submission, and recovery states.

## Layout

Use separate areas for card number, expiry, CVC, billing information, 3DS authentication, and the submit action. Use field-level labels, errors, and touched states outside the iframes.

Billing information should be collected outside the Airwallex card iframes and handled by the merchant app. At minimum, plan the UI for billing name, email, and billing address fields that the merchant can validate and pass through the backend according to the current Airwallex MCP docs.

## Initialization

Initialize Airwallex.js with the merchant's environment and locale. Create the card number, expiry, and CVC elements once per flow and mount each one once. Configure a known authentication container for 3DS challenge rendering.

If exact SDK imports, initialization signatures, element options, or mount syntax are needed, verify them with MCP before writing final code.

## Fraud Data Collection

The frontend should support the backend's fraud-data checklist:

- Collect or confirm billing details whenever a new card is entered or a new MIT agreement is authorized.
- For saved-card flows, show the existing billing profile and allow update if it is missing or stale.
- Preserve checkout session identifiers needed for device fingerprinting or device data, after checking current Airwallex MCP guidance.
- Pass only client-safe context to the backend; the backend decides which fraud data belongs in PaymentIntent create or confirm calls.
- Do not treat Split Card Elements completion as the only risk signal.

## Form State

Track each split element independently as empty, incomplete, complete, or error.

Listen to `change` on all three elements. Airwallex events expose `detail.completed`, `detail.empty`, and `detail.error`.

Enable the submit action only when all fields are complete, no field has an error, the required authorization action has been accepted, and no submission is in progress. Only show empty-field errors after blur, submit, or user interaction. Show format errors as soon as Airwallex returns an element error.

## User Flow States

Use a visible state machine:

- `loading`: SDK and elements are initializing.
- `ready`: fields are mounted and editable.
- `invalid`: at least one field is incomplete or invalid.
- `submitting`: authorization request is in progress; disable inputs and buttons.
- `authenticating`: 3DS challenge may be displayed in `auth-form-container`.
- `verifying`: frontend call completed; backend is confirming PaymentIntent and consent status.
- `success`: backend confirmed the agreement and saved local records.
- `failed`: user can retry, change card, or contact support depending on failure type.

## Submit Flow: Card Binding Or Recharge Authorization

Use this when the entry point is add card, wallet recharge setup, or recharge with no existing consent.

1. Validate the three elements and consent checkbox.
2. Validate the billing information required by the merchant checkout.
3. Ask the merchant backend to create or retrieve the Airwallex `Customer`.
4. Send billing information and other fraud-data context to the backend.
5. Ask the backend to create a zero-amount setup `PaymentIntent`.
6. Confirm the setup through the card number element using the setup intent. The consent intent must express merchant-triggered subsequent payments with an unscheduled trigger reason.
7. If 3DS challenge appears, keep the UI in `authenticating`.
8. On SDK resolution, ask the backend to retrieve/confirm the result.
9. Show success only after the backend stores the `payment_method_id` and recharge `payment_consent_id`.

## Submit Flow: Subscription Authorization

Use this when the entry point is subscription signup with a new card.

1. Show plan price, currency, period, renewal terms, and cancellation path.
2. Require a distinct subscription authorization action.
3. Collect or confirm billing information before submission.
4. Send billing information and other fraud-data context to the backend.
5. Create the setup or first-charge `PaymentIntent` on the backend.
6. Confirm through the card number element. The consent intent must express merchant-triggered subsequent payments with a scheduled trigger reason and fixed subscription terms that mirror the selected plan. For weekly/monthly/yearly schedules the terms **must include `billing_cycle_charge_day`** (the charge day within each cycle, e.g. `5` for the 5th of each month); omitting it fails with `400` on `payment_consent.terms_of_use.billing_cycle_charge_day`.
7. Confirm backend status before provisioning the subscription.

If the shopper selects a saved card, still require the subscription authorization action. The frontend may skip Split Card Elements for saved-card selection, but the backend must create a new `scheduled` consent tied to the selected card.

## 3DS Handling

Pass `authFormContainer` when creating the card number element so issuer challenge UI has a predictable location.

During 3DS:

- Disable repeated submits.
- Keep the current card fields intact.
- Show a neutral "Complete bank verification" state.
- Treat challenge cancellation or timeout as recoverable failure.

Airwallex may handle frictionless authentication without showing the challenge container.

## Error Handling

Field validation:

- Card number invalid: show error under card number and focus that field.
- Expiry invalid: show error under expiry and focus that field.
- CVC invalid: show error under CVC and focus that field.

Authorization errors:

- Card declined: invite shopper to use another card.
- Verification failed: allow retry or change card.
- 3DS cancelled: return to ready state with a retry option.
- `400` on `payment_consent.terms_of_use.billing_cycle_charge_day`: the scheduled consent omitted `billing_cycle_charge_day`, which is required for `WEEK`/`MONTH`/`YEAR` schedules, add it (e.g. `5` for the 5th of each month) on the backend and resubmit.
- Network timeout after submit: do not create a duplicate setup blindly; switch to `verifying` and ask the backend to check the existing request by idempotency key or merchant order id.
- Backend result unknown: show pending state and continue server-side reconciliation.

Do not expose raw gateway error dumps to shoppers. Log structured error codes server-side and show concise, actionable text client-side.

## Success Handling

After backend confirmation, show:

- Card brand and last four digits.
- Agreement type: recharge authorization or subscription authorization.
- For subscriptions: amount, period, next renewal date, and cancellation path.
- For recharge: auto-recharge settings if enabled.

Clear sensitive UI state by unmounting or clearing elements after success.
