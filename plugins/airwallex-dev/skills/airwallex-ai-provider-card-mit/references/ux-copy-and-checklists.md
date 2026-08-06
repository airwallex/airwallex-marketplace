# UX Copy And Checklists

## Table of Contents

- [Authorization Copy Principles](#authorization-copy-principles)
- [Recharge Authorization Copy](#recharge-authorization-copy)
- [Subscription Authorization Copy](#subscription-authorization-copy)
- [Screen Checklist](#screen-checklist)
- [Failure Copy](#failure-copy)
- [Implementation Checklist](#implementation-checklist)

---

## Authorization Copy Principles

Use plain, explicit language. The shopper should understand:

- Which card is being saved.
- Which business action they are authorizing.
- Whether future charges are fixed schedule or variable timing.
- Who controls thresholds, recharge amounts, limits, renewal dates, and cancellation.
- Where they can update card, disable recharge, or cancel a subscription.

Keep recharge authorization separate from subscription authorization.

## Recharge Authorization Copy

Example:

> By saving this card, you authorize [merchant] to charge this card for future balance top-ups, including manual recharge requests and auto-recharge if you enable it. Recharge amounts, limits, and low-balance thresholds are managed in your account settings.

When auto-recharge is enabled:

> When your balance falls below [minimum balance], [merchant] may recharge [amount] to this card, subject to your configured limits. You can disable auto-recharge at any time.

## Subscription Authorization Copy

Example:

> By authorizing this subscription, you allow [merchant] to charge [amount] [currency] every [period] to the selected card until you cancel. You can cancel from subscription settings before the next renewal date.

If the first charge differs:

> Today's charge is [first amount]. Future renewals will be [renewal amount] every [period] unless you change or cancel the subscription.

## Screen Checklist

Add card screen:

- Card number, expiry, CVC using Split Card Elements.
- Billing name, billing email, and billing address fields or a confirmed existing billing profile.
- Recharge authorization copy.
- Link to recharge settings if auto-recharge exists.
- Submit disabled until all fields are complete and authorization is accepted.
- 3DS challenge area.

Recharge settings screen:

- Current default card display.
- Minimum balance.
- Recharge amount.
- Per-charge and period limits if supported by merchant policy.
- Enable/disable toggle.
- Last recharge status.

Subscription signup screen:

- Plan name.
- Amount, currency, period.
- Next renewal date or first renewal logic.
- Selected card or new card form.
- Billing information collection or confirmation before authorization.
- Explicit subscription authorization action.
- Cancellation path.

Payment method management:

- List saved cards by brand, last4, expiry.
- Show which card is used for recharge and each subscription.
- Replacing a card should create new consents for the affected business agreements.
- Disabling a card should block dependent recharge or subscription flows unless another valid consent is selected.

## Failure Copy

Field error:

> Check your card details and try again.

3DS cancelled:

> Bank verification was not completed. Try again or use another card.

Card declined:

> This card was declined. Use another card or contact your bank.

Unknown pending result:

> We are still confirming this authorization. Do not retry yet.

Repeated recharge failure:

> Auto-recharge is paused because recent payment attempts failed. Update your card to continue automatic top-ups.

## Implementation Checklist

- Airwallex docs MCP checked for the exact APIs and SDK methods used.
- Airwallex docs MCP checked for payment data standards before PaymentIntent create or confirm calls are implemented.
- Only Web JS SDK Split Card Elements are used for card collection.
- Billing information is collected outside Split Card Elements and passed through the backend according to MCP-verified Airwallex fields.
- Card binding creates `unscheduled` consent unless the entry point is subscription signup.
- Subscription signup creates a `scheduled` consent.
- Scheduled consents include every required `terms_of_use` field, in particular `billing_cycle_charge_day` for `WEEK`/`MONTH`/`YEAR` schedules (a common `400`).
- Customer, billing, product, and applicable device data are passed for fraud prevention, or omissions are documented.
- Shipping data is omitted for digital-only AI flows unless the merchant has physical delivery.
- O2O customer-present metadata is omitted unless the merchant has an offline-to-online flow.
- Later merchant-initiated charges pass explicit `payment_consent_id`.
- Frontend success is not final until backend confirmation.
- Webhooks are idempotent.
- Wallet and subscription ledgers are merchant-owned sources of truth.
- Invoices and receipts are generated from merchant records.
- Retry and notification rules are merchant-owned.
- Cancellation and auto-recharge disable paths are visible.
