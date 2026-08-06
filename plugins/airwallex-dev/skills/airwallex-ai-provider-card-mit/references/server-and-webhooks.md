# Server And Webhooks

## Table of Contents

- [Backend Responsibilities](#backend-responsibilities)
- [Suggested Merchant Endpoints](#suggested-merchant-endpoints)
- [Setup Intent](#setup-intent)
- [Setup Result](#setup-result)
- [Later Merchant-Initiated Charges](#later-merchant-initiated-charges)
- [Webhook Events](#webhook-events)
- [Ledger Rules](#ledger-rules)
- [Retry And Failure Policy](#retry-and-failure-policy)

---

## Backend Responsibilities

The backend is responsible for all trusted actions:

- Authenticate to Airwallex.
- Create or retrieve `Customer`.
- Create setup and charge `PaymentIntent` records.
- Attach applicable fraud-prevention data to PaymentIntent create or confirm calls after checking the current Airwallex MCP docs.
- Confirm result by retrieving Airwallex objects or processing webhooks.
- Store `payment_method_id` and `payment_consent_id`.
- Run subscription renewal and wallet recharge jobs.
- Maintain wallet ledger, subscription state, invoice records, receipt records, retries, notifications, and reconciliation.

Never trust the frontend return alone for final success.

## Suggested Merchant Endpoints

Use merchant-owned endpoints for card setup, setup result confirmation, saved card listing, manual recharge, recharge settings, subscription authorization, subscription cancellation, internal recharge jobs, internal renewal jobs, and Airwallex webhooks. Keep route names aligned with the merchant application and do not embed route examples in this skill.

## Setup Intent

For add-card or recharge authorization:

1. Validate authenticated merchant user.
2. Create or reuse Airwallex `Customer`.
3. Validate merchant-collected billing information from the frontend or existing billing profile.
4. Build the applicable fraud-data set from customer, billing, product, and device/session data; verify exact field placement with MCP.
5. Create a zero-amount PaymentIntent with `customer_id` and applicable fraud data.
6. Return only client-safe fields needed by Airwallex.js, such as `intent_id` and `client_secret`.
7. Store a merchant setup attempt with an idempotency key and expected consent type `unscheduled`.

For subscription authorization:

1. Validate plan, amount, currency, period, and user eligibility.
2. Create or reuse Airwallex `Customer`.
3. Validate merchant-collected billing information from the new-card form or selected saved-card profile.
4. Build the applicable fraud-data set from user profile, billing profile, selected plan, and checkout session; verify exact field placement with MCP.
5. Create a setup or first-charge PaymentIntent according to merchant rules.
6. Store a merchant setup attempt with expected consent type `scheduled` and plan metadata.

## Setup Result

After the frontend SDK resolves:

1. Retrieve the PaymentIntent or wait for webhook confirmation.
2. Verify status and inspect returned `payment_consent_id` and `payment_method.id`.
3. Retrieve the PaymentConsent if needed to confirm `next_triggered_by`, `merchant_trigger_reason`, status, and card metadata.
4. Store only identifiers and non-sensitive display details.
5. Mark the merchant setup attempt as succeeded, pending, or failed.

If the result is not final, show pending status to the frontend and let webhooks finish reconciliation.

## Later Merchant-Initiated Charges

Manual recharge with existing consent:

1. Validate user, amount, currency, and merchant limits.
2. Build the applicable fraud-data set for the recharge event; verify exact field placement with MCP.
3. Create a PaymentIntent for the recharge amount.
4. Confirm as merchant-triggered with the stored recharge consent.
5. Credit wallet balance only after payment success is confirmed.

Auto-recharge:

1. Detect low balance from merchant ledger.
2. Apply merchant thresholds, per-charge limits, period limits, and concurrency guards.
3. Build the applicable fraud-data set for an off-session recharge; verify current requirements with MCP.
4. Create and confirm PaymentIntent with the stored recharge consent.
5. Write wallet ledger entries idempotently.

Subscription renewal:

1. Select due subscriptions from merchant records.
2. Build the applicable fraud-data set for an off-session renewal; verify current requirements with MCP.
3. Create and confirm PaymentIntent with the subscription consent.
4. Extend service access only after success, or apply merchant-defined retry and grace rules after failure.

Always send the explicit `payment_consent_id` for later charges.

## Webhook Events

Handle PaymentIntent and PaymentConsent lifecycle events needed by the merchant flow. PaymentConsent events include created, updated, pending, verified, disabled, paused, requires payment method, requires customer action, and verification failed.

Webhook handler rules:

- Verify webhook authenticity according to current Airwallex docs.
- Process events idempotently.
- Store raw event id and normalized event data.
- Re-fetch important objects when event payload is insufficient.
- Separate payment success from wallet crediting or subscription activation with ledger entries.
- Alert operators for unknown, duplicate, or out-of-order states instead of silently ignoring them.

## Ledger Rules

Wallet ledger:

- Wallet transaction identifier.
- Merchant user identifier.
- Source, such as manual recharge, auto-recharge, usage debit, or adjustment.
- PaymentIntent identifier.
- PaymentConsent identifier.
- Amount and currency.
- Status, such as pending, succeeded, failed, or reversed.
- Created timestamp.

Subscription charge ledger:

- Subscription charge identifier.
- Subscription identifier.
- PaymentIntent identifier.
- PaymentConsent identifier.
- Period start and end.
- Amount and currency.
- Status.
- Attempt count.
- Created timestamp.

Use ledger state, not frontend state, as the source of truth.

## Retry And Failure Policy

The merchant owns retry rules:

- Distinguish hard declines from retryable failures.
- Avoid multiple simultaneous attempts for the same subscription period or recharge trigger.
- Send pre-charge and post-failure notifications when required by product policy.
- Pause auto-recharge after repeated failures.
- Keep cancellation and card update paths easy to find.

Before implementing specific Airwallex error-code handling, verify current error fields and statuses with MCP.
