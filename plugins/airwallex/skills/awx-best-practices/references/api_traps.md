# API traps — non-obvious constraints

Body-level constraints that schema discovery may not surface. Read this when you hit a `validation_failed` / `validation_error` you cannot resolve, or before building a complex payload for the first time. **Field types, enums, and required flags live in the schema — this file only covers what the schema does not say.** Surface-specific quirks (CLI flag names, MCP tool names) live in [surface-quickstart.md](surface-quickstart.md).

## Contents

- [General](#general)
- [Billing — Invoices](#billing--invoices)
- [Billing — Products & Prices](#billing--products--prices)
- [Billing — Subscriptions](#billing--subscriptions)
- [Billing — Coupons](#billing--coupons)
- [Billing — Credit Notes](#billing--credit-notes)
- [Issuing — Cards](#issuing--cards)
- [Issuing — Cardholders](#issuing--cardholders)
- [Payouts — Beneficiaries](#payouts--beneficiaries)
- [Treasury — Balances](#treasury--balances)
- [Treasury — FX](#treasury--fx)

## General

- **`request_id` is required on most `create`, `update`, and `validate` writes.** Generate a fresh UUIDv4 per distinct operation. NEVER hand-write or use sequential/patterned values. Reuse the same `request_id` only when retrying the same logical operation after a transient failure. Known exception: the beneficiary-verify operation does NOT take `request_id`. The schema is authoritative — trust the field's required flag for your surface.
- **Timestamp format:** Always include an explicit timezone — bare dates are rejected. Most body fields (`due_at`, `starts_at`, `expires_at`, …) and most list endpoints accept both `+0000` and `Z`. **Tested exceptions:** billing-invoice listing only accepts `+0000` (rejects `Z`); payment-link listing only accepts `Z` (rejects `+0000`). When a listing endpoint's description states a required format, follow it. Do NOT pre-encode the offset to `%2B0000`.

## Billing — Invoices

- **`legal_entity_id`** — not discoverable via API. If the account has multiple legal entities and this field is omitted, the API rejects with `"Need to specify the legal_entity_id in the request"`. Ask the user.
- **`collection_method` must be set before finalize and is not inferable from context.** Always ask the user — do not guess. `CHARGE_ON_CHECKOUT` additionally needs `linked_payment_account_id` (not discoverable via API; ask the user).
- **Pick `due_at` OR `days_until_due`** — passing both is rejected.
- **Line items are the only source of amount.** No top-level amount field exists on invoice create; `invoice_items` in the create body is **silently ignored**. Always add line items via the dedicated line-items operation, then verify `total_amount > 0` before finalize. Only ONE_OFF + PER_UNIT/FLAT prices are accepted; inline price objects on invoice line items do NOT include `currency` (inherited from the invoice).
- **Lifecycle actions need explicit confirmation.** `finalize`, `void`, `delete`, and `mark-as-paid` are state changes — confirm with the user before invoking, even though they take no body.
- **`metadata`** replaces entirely on update — omit to keep existing.
- **Discounts:** Use coupons via `"discounts": [{"type": "COUPON", "coupon": {"id": "..."}}]`, not negative amounts.
- **Tax handling:** Airwallex Billing does NOT have a built-in tax-rate engine. Do NOT invent tax fields or silently compute tax. If tax must be represented, add it explicitly as a line item or bake it into the unit price after user confirmation.

## Billing — Products & Prices

- **Tiered pricing:** Last tier omits `upper_bound` entirely (open-ended top tier).
- **Price immutability:** Cannot change `currency`, `pricing_model`, `amount`, or `tiers` via update. Deactivate the old price and create a new one.

## Billing — Subscriptions

- **`items[*].price_id`** must reference RECURRING prices — ONE_OFF rejected with "Please add at least one recurring item."
- **`starts_at`** must be strictly future. Compute dynamically — never hardcode.
- **`AUTO_CHARGE`** requires `payment_source_id`. Ask the user — not always discoverable from context.
- **`legal_entity_id`** may also be required on subscription create in multi-entity accounts (not discoverable via API; ask the user).

## Billing — Coupons

- **PERCENTAGE vs FLAT are mutually exclusive.** PERCENTAGE: set `percentage_off` (0–100), no `currency`/`amount_off`. FLAT: set `amount_off` + `currency`, no `percentage_off`.
- **`duration_type: CUSTOM`** needs a `duration` object with both `period` and `period_unit`.

## Billing — Credit Notes

- **Lifecycle: `create` → `line-items add` → `finalize`** — analogous to invoice lifecycle.
- **Create body** needs `billing_invoice_id`, `type` (`BEFORE_PAYMENT`/`AFTER_PAYMENT`), `reason`, and `line_items`.
- **Credit-note operations may not be exposed on every surface.** If the operation is not available, direct the user to the Airwallex Dashboard.

## Issuing — Cards

- **`is_personalized` rule:** VIRTUAL → `false`; PHYSICAL → `true`. Ask the user if the form factor is unspecified — do not default silently.
- **`created_by` is the legal name of the requester**, not the cardholder. Ask the user if not provided.
- **Do NOT invent merchant-category / MCC fields** (e.g. `allowed_categories`). MCC restriction support is unconfirmed on either surface.
- **`program` is an object** `{"purpose": "COMMERCIAL"}` — not a string. `authorization_controls.transaction_limits` is an object `{"currency": "...", "limits": [...]}` — not a bare array. Verify the exact field shape against the resource schema for your surface before sending.
- **Status changes** go through the card-update operation. The settable values are **`INACTIVE` / `ACTIVE` / `CLOSED`** only. `BLOCKED`, `LOST`, and `STOLEN` are NOT settable via the update operation — `BLOCKED` is set by Airwallex; `LOST`/`STOLEN` are reported via the Airwallex Console.
- **Physical-card delivery is create-time only.** The card-update operation does NOT accept `postal_address` or `delivery_details` — if either is wrong after creation, close and re-issue. Two valid paths at create time: (a) cardholder has a registered `postal_address` and card create uses it by default, or (b) pass `postal_address` directly on card create to override. For EXPRESS shipment or any China destination, `delivery_details.mobile_number` (E.164) is required. Always confirm the address with the user; never fabricate. Physical cards are created `INACTIVE` — activate after delivery.

## Issuing — Cardholders

- **INDIVIDUAL cardholders** have two schema-invisible quirks: `individual.address` uses `country` (NOT `country_code`), and `individual.express_consent_obtained` is the string `"yes"` (NOT a boolean `true`). Ask the user for `date_of_birth`, address, and `email` — never fabricate these.

## Payouts — Beneficiaries

- **`transfer_methods` is top-level; bank fields are nested.** `transfer_methods` (plural array) sits at the top of the create/validate body. `bank_country_code` and `account_currency` live inside `bank_details`. The schema-fetch endpoint takes singular `transfer_method` plus `bank_country_code` and `account_currency` as top-level query parameters — easy to confuse with the create body shape.
- **`account_name`** is required in `bank_details` for most countries even when the schema does not mark it required.
- **SWIFT uses `swift_code` in `bank_details`** — do NOT put BIC in `account_routing_type1` (LOCAL routing only). IBAN countries may still require both `iban` and `swift_code` on SWIFT transfers.
- **`entity_type` drives required name fields:** COMPANY → `company_name`; PERSONAL → `first_name`/`last_name` + `additional_info` for `personal_id_type`/`personal_id_number` (the schema does not always surface the conditional `additional_info` requirement).
- **`address.state`** uses ISO 3166-2 codes with country prefix (e.g., `CA-ON`, `AU-NSW`).
- **Strip formatting to match schema `pattern`** — e.g., GB sort code `20-32-06` → `203206` for pattern `^[0-9]{6}$`.
- **`bank_account_category`** — required for US/USD/LOCAL (some other countries, e.g., BR, also need it for personal accounts). Values: **`"Checking"` / `"Savings"`** (note the `s`). Ask the user — the schema may omit this field.
- **SE/SEK/LOCAL trap** — the schema marks `account_routing_type1`, `account_routing_value1`, and `account_number` as optional — **they are actually required**. IBAN alone is NOT enough. Ask the user for clearing number (4–5 digits) and bank-specific account number separately — do not decompose IBANs.
- **Validation ≠ creation** — validation only checks payloads, does NOT create the beneficiary. Always validate first, confirm with user, then create.
- **List filters:** PERSONAL beneficiaries match on `name`, COMPANY on `company_name`. There is no `first_name` filter.
- **Verify-account body shape:** `entity_type`, `transfer_method` (singular), and `bank_details` only — no beneficiary ID and no `request_id`. The verify endpoint works on a candidate payload before any beneficiary record exists, which surfaces country-specific failures cheaper than a failed create. Possible response statuses: `VERIFIED`, `INVALID`, `CANNOT_VERIFY`, `EXTERNAL_SERVICE_UNAVAILABLE`.

## Treasury — Balances

- **Balance-history listing — numeric pagination is date-window-capped.** When paginating by numeric page, the date range is capped at **7 days**; the server rejects wider ranges. Use consecutive ≤7-day windows, or switch to cursor pagination (`page=0` to start) to walk all records without date filters.
- **Cursor and numeric pagination are mutually exclusive on balance-history.** Do not pass both `page` and `page_num` in the same call.

## Treasury — FX

- **FX rates are read-only and indicative.** No conversion-create or FX-quote-create operation is exposed on either surface; conversions must be executed in the **Airwallex Dashboard**.
- **No FX rate-lock action exists anywhere** — not on either surface, not in the Airwallex Dashboard. Conversions execute at prevailing market rates; avoid wording like "lock in" or "secure the rate".
- **Specify `sell_amount` OR `buy_amount`** — not both.
- **`conversion_date`** — only T+0 to T+2 business days. No forward FX pricing via this endpoint.
- **Sandbox `amount_above_limit`** — use `sell_amount: 1000` for rate checks; apply the rate mathematically.
- **Conversion amendments** — only for unsettled conversions. Once status is `SETTLED`, conversions are immutable.
