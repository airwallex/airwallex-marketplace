# API traps — non-obvious constraints

These are body-level constraints that `--api-schema-only` may not surface. Read this file when you hit a `validation_failed` / `validation_error` you cannot resolve, or before building a complex payload for the first time.

## Contents

- [General](#general)
- [Billing — Invoices](#billing--invoices)
- [Billing — Products & Prices](#billing--products--prices)
- [Billing — Subscriptions](#billing--subscriptions)
- [Billing — Products](#billing--products)
- [Billing — Customers](#billing--customers)
- [Billing — Coupons](#billing--coupons)
- [Billing — Credit Notes](#billing--credit-notes)
- [Issuing — Cards](#issuing--cards)
- [Issuing — Cardholders](#issuing--cardholders)
- [Payouts — Beneficiaries](#payouts--beneficiaries)
- [Treasury — Balances](#treasury--balances)
- [Treasury — FX](#treasury--fx)

## General

- **`request_id` is MANDATORY for write commands.** Include a fresh `"request_id"` (random UUIDv4, e.g. via `uuidgen`) in every `create`, `update`, and `validate` body. Omitting it returns `"request_id is mandatory"`. NEVER hand-write a UUID or use sequential/patterned values like `a1b2c3d4-...`. One known exception: `beneficiaries verify` does NOT take `request_id`. When in doubt, run `--api-schema-only` to confirm.
- **Timestamp format:** `+0000` offset everywhere (e.g., `2026-05-15T00:00:00+0000`). `Z` suffix and bare dates are rejected. Applies to `due_at`, `starts_at`, `expires_at`, etc.

## Billing — Invoices

- **`legal_entity_id`** — not discoverable via API. If the account has multiple legal entities and this field is omitted, the API rejects with `"Need to specify the legal_entity_id in the request"`. Ask the user for it.
- **`collection_method`** values are case-sensitive: `AUTO_CHARGE`, `CHARGE_ON_CHECKOUT`, `OUT_OF_BAND`. Never `AUTOMATIC`/`AUTO`/`MANUAL`. Must be set before finalize. **Always ask the user** which collection method to use — do not guess from context.
- **`CHARGE_ON_CHECKOUT`** needs `linked_payment_account_id` — without it the invoice has no checkout link. Ask the user for this ID (not discoverable via API).
- **Pick `due_at` OR `days_until_due`** — passing both is rejected.
- **`invoices create` has no `--amount`, `--customer-name`, or `description` field** — amounts go in line items, customer is `billing_customer_id` in the JSON body, notes go in `memo`.
- **`invoices line-items add` / `update`:** Body must be `{"line_items": [...]}` (object wrapping an array, not a bare array). Only ONE_OFF + PER_UNIT/FLAT prices accepted. Inline price objects do NOT include `currency` (inherited from invoice).
- **`invoices line-items delete`:** Body is `{"line_item_ids": ["id1", "id2"]}`.
- **Invoice must have line items before finalize.** Do NOT rely on `invoice_items` in the create body — it may be silently ignored.
- **Action commands (`invoices finalize`, `invoices void`, `invoices delete`, `invoices mark-as-paid`) require `--confirm`** — even though they take no body, the CLI safety layer blocks them without `--confirm`. Use `airwallex --confirm invoices finalize <id>`. Same `--dry-run` → `--confirm` two-step as body-based writes.
- **`metadata`** replaces entirely on update — omit to keep existing.
- **Discounts:** Use coupons via `"discounts": [{"type": "COUPON", "coupon": {"id": "..."}}]`, not negative amounts.
- **Tax handling:** Airwallex Billing does NOT have a built-in tax-rate engine. Do NOT invent tax fields or silently compute tax. If tax must be represented, add it explicitly as a line item or bake it into the unit price after user confirmation.

## Billing — Products & Prices

- **`billing_type`** in `prices create` only accepts `IN_ADVANCE` or `IN_ARREARS` (or omit entirely).
- **Recurring prices** need full `recurring` object: `{"interval": 1, "period_unit": "MONTH"}`. Missing `period_unit` causes validation error. `period_unit`: `DAY`/`WEEK`/`MONTH`/`YEAR`.
- **Tiered pricing:** Uses `upper_bound` (not `up_to`). Last tier omits `upper_bound` entirely.
- **Price immutability:** Cannot change `currency`, `pricing_model`, `amount`, `tiers` via update. Deactivate old, create new.

## Billing — Subscriptions

- **`items[*].price_id`** must reference RECURRING prices — ONE_OFF rejected with "Please add at least one recurring item."
- **`starts_at`** must be strictly future. Compute dynamically — never hardcode.
- **`AUTO_CHARGE`** requires `payment_source_id`. Ask the user for it — not always discoverable from context.
- **`legal_entity_id`** may also be required on subscription create in multi-entity accounts. If omitted, the API can reject with `"Need to specify the legal_entity_id in the request"`. This ID is not discoverable via API.

## Billing — Products

- **`products create`** body is minimal: `name`, `description`, `unit` only — no address or pricing fields (prices are separate resources).

## Billing — Customers

- **`billing-customers create`** requires `address.country_code` (ISO-3166 alpha-2).

## Billing — Coupons

- **Command group is `coupons`** — NOT `discounts`, `promo-codes`, or `vouchers`.
- **PERCENTAGE:** Set `percentage_off` (0–100), no `currency`/`amount_off`. **FLAT:** Set `amount_off` + `currency`, no `percentage_off`.
- **`duration_type: CUSTOM`** needs `duration` object: `{"period": N, "period_unit": "MONTH"}`.

## Billing — Credit Notes

- **Lifecycle: `create` → `line-items add` → `finalize`** — analogous to invoice lifecycle.
- **`credit-notes create`** body needs `billing_invoice_id`, `type` (`BEFORE_PAYMENT`/`AFTER_PAYMENT`), `reason`, and `line_items`.

## Issuing — Cards

- **`is_personalized` is MANDATORY.** VIRTUAL → `false`; PHYSICAL → `true`. Omitting returns `"is_personalized is mandatory"`.
- **`issue_to`** required: `INDIVIDUAL` or `DELEGATE`. Without it the API returns generic `BAD_REQUEST`.
- **`created_by`** is MANDATORY — full legal name of the person requesting the card (not the cardholder name). Ask the user if not provided.
- **`program`** is an object `{"purpose": "COMMERCIAL"}`, NOT a string.
- **`authorization_controls.transaction_limits`** is `{"currency": "...", "limits": [...]}`, NOT an array.
- **`allowed_transaction_count`:** `MULTIPLE` (not `MULTI`).
- **Do NOT invent merchant-category / MCC fields** (e.g. `allowed_categories`). MCC restriction support is unconfirmed in CLI/MCP flows.
- **Status changes** via JSON body (`{"status": "BLOCKED"}`), not flags. No `cards block`/`cards close` subcommands.
- **Command names:** `cards`, `cardholders` — NOT `issuing-cards` or `issuing-cardholders`. Only `issuing-transactions` has the `issuing-` prefix.
- **Physical cards** need `postal_address` (ask the user — do not fabricate) and are created `INACTIVE` — activate after delivery.

## Issuing — Cardholders

| Type | Required fields |
| --- | --- |
| INDIVIDUAL | `individual.name` (object: `{"first_name": "...", "last_name": "..."}`), `individual.date_of_birth` (ask user), `individual.address` (uses `country` not `country_code` — ask user), `individual.express_consent_obtained: "yes"` (string, not boolean), `email` (ask user) |
| DELEGATE | `email` only |

## Payouts — Beneficiaries

- **Top-level vs nested beneficiary fields are different.** Top-level `bank_country_code`, `currency`, and `transfer_methods` are routing fields. Nested `beneficiary.bank_details.bank_country_code` and `account_currency` are bank fields. You need both in the correct places — they do NOT substitute for each other.
- **`account_name`** required in `bank_details` for most countries.
- **SWIFT uses `swift_code` in `bank_details`** — do NOT put BIC in `account_routing_type1` (LOCAL routing only). IBAN countries may still require both `iban` and `swift_code` on SWIFT transfers.
- **`entity_type`** required: `COMPANY` or `PERSONAL`. COMPANY → `company_name`; PERSONAL → `first_name`/`last_name` + `additional_info` (for `personal_id_type`/`personal_id_number`).
- **`transfer_methods`** is a plural array: `["LOCAL"]`. Note: `beneficiary-schemas generate` fetch uses singular `transfer_method`; create/validate body uses plural `transfer_methods`.
- **`beneficiary-schemas generate` requires `--confirm`** — the CLI treats schema fetches as writes; without `--confirm` you get `SAFETY_BLOCK`.
- **`beneficiary-schemas generate` uses `--account-currency`** (NOT `--currency`). Using the wrong flag fails before you even get schema fields.
- **`address.state`** uses ISO 3166-2 codes with country prefix (e.g., `CA-ON`, `AU-NSW`).
- **Strip formatting to match schema `pattern`** — e.g., GB sort code `20-32-06` → `203206` for pattern `^[0-9]{6}$`.
- **`bank_account_category`** — required for US/USD/LOCAL. Values: `"Checking"` / `"Saving"`. Ask the user — the schema may omit this field.
- **SE/SEK/LOCAL trap** — the schema marks `account_routing_type1`, `account_routing_value1`, and `account_number` as optional — **they are actually required**. IBAN alone is NOT enough. Ask the user for clearing number and account number separately — do not decompose IBANs.
- **`validate` ≠ `create`** — validation only checks payloads, does NOT create the beneficiary. Always validate first, confirm with user, then create.
- **`beneficiaries list` filter flags:** Use `--name` for PERSONAL and `--company-name` for COMPANY. Do NOT invent `--first-name` from JSON field names.
- **`beneficiaries verify` input shape:** `verify` takes `--data-file` / `--data` / `--data-stdin` payload, not a positional beneficiary ID. The body contains `entity_type`, `transfer_method`, and `bank_details` only — no `beneficiary_id` and no `request_id`. Always confirm via `--api-schema-only`.

## Treasury — Balances

- **`balances list-history` + `--page-num`:** The `--from`/`--to` date range is **capped at 7 days** — the API rejects wider ranges. Use consecutive ≤7-day windows, or use `--page` cursor pagination with `--page 0` to walk all records without date filters.
- **`--page` and `--page-num` are mutually exclusive** on `balances list-history` — do not pass both in the same call.

## Treasury — FX

- **FX rates are read-only.** Use `conversion-rates get-current` for indicative rates. All conversions must be executed in the **Airwallex Dashboard**.
- **`conversions create` is not available via CLI** — there is no CLI command to execute conversions.
- **`quotes create` is not available in this flow** — do not suggest a quote-then-execute path.
- **`conversion-amendments create` is not available via CLI** — only `create-quote` (preview cost) is available. Execute cancellation in the **Airwallex Dashboard**.
- **No FX rate-lock action exists (CLI or Airwallex Dashboard)** — `create-quote` is preview only and does not reserve/lock a rate. Airwallex Dashboard conversions execute at prevailing market rates; avoid wording like "lock in" or "secure the rate".
- **Specify `sell_amount` OR `buy_amount`** — not both.
- **`conversion_date`** — only T+0 to T+2 business days. No forward FX pricing via this endpoint.
- **Sandbox `amount_above_limit`** — use `sell_amount: 1000` for rate checks; apply the rate mathematically.
- **Conversion amendments** — only for unsettled conversions. Once status is `SETTLED`, conversions are immutable.
