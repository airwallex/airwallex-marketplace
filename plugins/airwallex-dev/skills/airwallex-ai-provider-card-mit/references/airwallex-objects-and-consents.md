# Airwallex Objects And Consents

> **Canonical source:** the PaymentConsent and MIT consent model (`next_triggered_by: merchant`, `merchant_trigger_reason` `scheduled` or `unscheduled`) maps to [Save and reuse payment details](https://www.airwallex.com/docs/payments/integration-options/web-checkout/save-and-reuse-payment-details). That doc is authoritative for consent semantics. Prefer it if anything here drifts.

## Table of Contents

- [Object Mapping](#object-mapping)
- [Consent Types](#consent-types)
- [Entry-Based Consent Creation](#entry-based-consent-creation)
- [Terms Of Use Guidance](#terms-of-use-guidance)
- [Local Data Model](#local-data-model)
- [Charge Routing](#charge-routing)
- [Status Handling](#status-handling)

---

## Object Mapping

- `Customer`: the Airwallex shopper profile mapped to the AI merchant user.
- `PaymentMethod`: the tokenized saved card. Store only Airwallex identifiers, brand, last4, expiry, and status metadata.
- `PaymentConsent`: the reusable card agreement. Store one consent per business agreement.
- `PaymentIntent`: each actual setup, top-up, recharge, or subscription charge.

The merchant owns users, plans, subscriptions, token balances, low-balance thresholds, invoices, receipts, retries, cancellation, and access control.

## Consent Types

Use `payment_consent.next_triggered_by = merchant` for all flows in this skill.

Use `merchant_trigger_reason = unscheduled` for:

- Manual balance recharge when the merchant initiates the actual card charge.
- Low-balance auto-recharge.
- Usage-driven token wallet top-ups.
- Variable timing or variable amount charges authorized by the shopper.

Use `merchant_trigger_reason = scheduled` for:

- Fixed-amount, fixed-period subscriptions.
- Renewals charged according to the merchant's subscription schedule.

Do not reuse a recharge consent for a subscription renewal, and do not reuse a subscription consent for wallet recharge.

## Entry-Based Consent Creation

Card binding entry:

1. Create or reuse the Airwallex `Customer`.
2. Create a zero-amount setup `PaymentIntent` for card collection.
3. Confirm through Split Card Elements with an `unscheduled` MIT consent.
4. Store `customer_id`, `payment_method_id`, and `recharge_consent_id`.

Subscription entry with new card:

1. Create or reuse the Airwallex `Customer`.
2. Create a setup or first-charge `PaymentIntent` according to the merchant's plan rules.
3. Confirm through Split Card Elements with a `scheduled` MIT consent.
4. Store `subscription_id`, `payment_method_id`, and `subscription_consent_id`.

Subscription entry with saved card:

1. List or retrieve merchant-known saved cards for the customer.
2. The shopper selects a card and accepts the subscription authorization.
3. Create and confirm a PaymentIntent with the selected `payment_method.id` plus a new `scheduled` MIT consent.
4. Store the returned `payment_consent_id` on the merchant subscription.

## Terms Of Use Guidance

For `scheduled` subscription consent, include terms that mirror the merchant plan:

- `payment_amount_type = FIXED`
- `fixed_payment_amount`
- `payment_currency`
- `payment_schedule.period`
- `payment_schedule.period_unit`
- `billing_cycle_charge_day`, **required when `payment_schedule.period_unit` is `WEEK`, `MONTH`, or `YEAR`** (the day within each cycle to charge; e.g. `5` = the 5th of each month for a monthly plan). Omitting it on such a scheduled consent fails with `400` on `payment_consent.terms_of_use.billing_cycle_charge_day`.
- `start_date`, `end_date`, and `total_billing_cycles` when the merchant plan is not open-ended

For `unscheduled` recharge consent, use variable terms when the amount can differ:

- `payment_amount_type = VARIABLE`
- `payment_currency`
- `min_payment_amount`, `max_payment_amount`, or merchant-side limits when applicable
- `total_billing_cycles` only when the mandate should be capped

If exact Airwallex field requirements are needed, verify with MCP before writing code.

## Local Data Model

Minimum user payment profile:

- Merchant user identifier.
- Airwallex customer identifier.
- Default payment method identifier.
- Default recharge consent identifier.
- Card display metadata: brand, last four digits, expiry, and status.
- Created and updated timestamps.

Minimum subscription record:

- Subscription identifier.
- Merchant user identifier.
- Plan identifier.
- Airwallex customer identifier.
- Payment method identifier.
- Subscription consent identifier.
- Amount, currency, period, and period unit.
- Next charge timestamp.
- Subscription status and cancellation behavior.
- Created and updated timestamps.

Minimum wallet recharge settings:

- Merchant user identifier.
- Recharge consent identifier.
- Minimum balance.
- Recharge amount and currency.
- Per-charge and period limits when supported by merchant policy.
- Enabled state.
- Updated timestamp.

## Charge Routing

Always pass the explicit `payment_consent_id` for later merchant-initiated charges. This is required for correctness when the same card has more than one MIT agreement.

Recharge charge:

- Create a PaymentIntent for the merchant recharge amount and wallet currency.
- Confirm it as merchant-triggered.
- Use the stored recharge consent identifier.

Subscription renewal:

- Create a PaymentIntent for the subscription renewal amount and currency.
- Confirm it as merchant-triggered.
- Use the stored subscription consent identifier.

Never infer the consent from only `payment_method_id` when a user may have more than one active agreement.

## Status Handling

Only treat a consent as usable for later charges when backend verification confirms it is ready for subsequent payment use. Handle intermediate states as pending authorization, customer action required, or failed verification according to the latest MCP docs.
