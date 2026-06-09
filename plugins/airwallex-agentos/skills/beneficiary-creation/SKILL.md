---
name: beneficiary-creation
description: Extract bank details from supplier invoices or documents, validate per-country requirements, and create beneficiaries in Airwallex. Use when the user says "set up this supplier", "onboard these vendors", "create beneficiary from invoice", "add a payee", or uploads supplier documents with bank details. Do NOT use for creating invoices, checking balances or FX rates, or provisioning cards.
metadata:
  author: Airwallex
  version: 0.1.0
compatibility: Works with the Airwallex CLI (`airwallex` binary on PATH, authenticated via `airwallex auth login`) or the Airwallex MCP server (enabled and authenticated).
---

# Beneficiary Creation

Reads supplier invoices or documents, extracts bank details, validates against country-specific schemas, and creates beneficiaries in Airwallex. **This skill creates beneficiaries only — money movement (transfers) is not in scope.**

## When to use

- User uploads supplier invoices, contracts, vendor lists, or documents with bank details
- User asks to "set up a supplier" or "onboard a vendor"
- User wants to create beneficiaries from extracted bank information
- User has multiple suppliers to onboard in batch

## When NOT to use

This skill only covers Payouts-domain operations — listing, creating, updating, validating, and verifying beneficiaries, plus beneficiary-schema lookup. If the task requires capabilities outside this domain, **stop — this is the wrong skill.** Redirect the user:

- Wire transfers / payouts → not yet available (use Airwallex Dashboard)
- Creating invoices (money in) → **contract-to-billing** skill
- FX conversions, balances, treasury → **manage-cashflow** skill
- Provisioning corporate cards → **card-provisioning** skill
- Ad-hoc tasks outside beneficiary workflow → **awx-best-practices** skill (fallback)

## Non-negotiables

### HARD GATE — money-movement requests

If the user's message mentions **transferring, sending, wiring, or paying money** (e.g., "set up and transfer", "send $15K to them", "pay this supplier now"):

1. **Very first sentence of your reply** must state the capability boundary: *"I can set up [name] as a beneficiary, but I can't execute transfers — that must be done in the Airwallex Dashboard after the beneficiary is created."*
2. Do NOT ask for payment amount, payment reason, source wallet, or any transfer-related details — these are not inputs to this skill.
3. Do NOT imply that the transfer will happen as part of this workflow or that you will "set up the payment right away."
4. After stating the boundary, proceed normally with the beneficiary-creation workflow.

This gate fires even if the transfer request is mixed with a valid beneficiary-creation request. Acknowledge what you CAN do, clearly state what you CANNOT, then continue with the part you can handle.

### Terminology

- **Beneficiary = first step of the payout workflow.** The full payout flow is: **create beneficiary → (optional) verify bank ownership → initiate transfer (Airwallex Dashboard only)**. This skill covers the first two steps. Transfers are out of scope and must be done in the Airwallex Dashboard.
- **Beneficiary ≠ transfer.** Creating a beneficiary saves payee details — does NOT send money. Duplicate beneficiaries waste time during the transfer step, which is why the skill searches for existing records before creating.
- **LOCAL vs SWIFT.** LOCAL = domestic clearing (cheaper, faster). SWIFT = international wire (costlier). Do NOT silently pick a transfer method — determine the available rails from the document, country docs, and schema, then present the options to the user with a brief note on cost/speed differences. If the document specifies a preferred rail, surface that preference but still confirm with the user. Only when the document is silent AND the schema shows exactly one supported rail for the country/currency combo may you proceed without asking.
- **COMPANY vs PERSONAL.** COMPANY = business entity. PERSONAL = individual (freelancer). Determines required identity fields.
- **Schema ≠ country docs — you need both.** The schema's `required` flag is not always accurate (some fields marked optional are actually required by the API). Country docs tell you which fields are truly required AND which values are valid. When in doubt, include all fields the country docs list as required, even if the schema says optional.

### Operational rules

- **For ambiguous-intent requests, do not start the workflow until the action is confirmed.** If the user has not clearly confirmed the exact write action, stop before schema reads, auth checks, or other workflow setup that materially advances execution.
- **NEVER fabricate or assume missing information.** If any required field is uncertain, absent, or ambiguous — STOP and ask the user. Keep asking until you have every parameter needed. Do NOT fill in defaults, placeholders, or "reasonable guesses." The only data you may fill in yourself is `nickname` (and `request_id` for surfaces that require it).
- **NEVER echo back unverified field names from the user.** If the user mentions routing types, code names, or bank-detail parameters that you have not confirmed via schema fetch, do NOT include them in your response as if they were real API fields. Instead: (1) acknowledge what the user asked for, (2) fetch the beneficiary schema for that country/currency/transfer-method combo, (3) reply with only the fields the schema actually requires — and flag any user-mentioned terms that do not map to a schema field. Parroting an unverified parameter name back to the user — even just to ask for its value — treats it as a real field and is a form of hallucination.
- **Even when the user says "use example data"** — STOP and list the concrete fields needed. Offer to create a JSON template for them to fill in.
- **Flag any extraction uncertainty** — never guess at bank details.
- **Always fetch fresh data** — re-fetch before every step.
- **Prefer business labels over raw IDs in user-facing output.** Show beneficiary names first; surface IDs only when operationally necessary or when the user asks.
- **Default to sandbox.** Confirm with user before any production write — production beneficiaries are real payee records.
- **Always validate before creating** — validation is the primary safety mechanism. Only create after validation passes AND user confirms.
- **Search for existing beneficiaries by name before creating** — duplicate beneficiaries clutter the payout workflow.
- **Write safety.** Show the full payload to the user and get confirmation before every beneficiary create / update / validate / verify.

### Beneficiary constraints

The beneficiary create body nests bank fields inside `bank_details`. The example below illustrates the canonical shape — verify exact field names and nesting against the resource schema for your surface before sending, as some surfaces flatten or rename fields:

```json
{
  "request_id": "<generate via uuidgen>",
  "beneficiary": {
    "bank_details": {
      "account_name": "...",
      "account_number": "...",
      "account_routing_type1": "sort_code",
      "account_routing_value1": "123456",
      "bank_country_code": "GB",
      "account_currency": "GBP"
    },
    "entity_type": "COMPANY",
    "company_name": "Acme Ltd",
    "address": {
      "city": "London", "country_code": "GB",
      "postcode": "EC1A 1BB", "street_address": "123 Main St"
    }
  },
  "transfer_methods": ["LOCAL"],
  "nickname": "Acme supplier"
}
```

- **Fetch schema AND country docs for EVERY country before building ANY JSON:**
  1. **Schema fetch** for each unique country/currency/transfer-method/entity-type combo. Parameters: `bank_country_code`, `account_currency` (not `currency`), `transfer_method`, `entity_type`.
  2. **Country docs page** via WebFetch (`https://www.airwallex.com/docs/payouts/payout-network/bank-accounts/{country-slug}`) — valid enum values, routing formats, extra field requirements. **Do NOT skip the country docs** — the schema alone does not provide valid values for routing types, state formats, or fields like `bank_account_category`.
- **`transfer_method` vs `transfer_methods`:** Schema fetch uses singular `transfer_method`. Validate/create body uses plural array (`"transfer_methods": ["LOCAL"]`). Mixing causes API rejection.
- **Top-level vs nested fields.** `transfer_methods` is a top-level array. `bank_country_code` and `account_currency` live inside `bank_details`. The **schema-fetch** endpoint takes singular `transfer_method` plus `bank_country_code` and `account_currency` as top-level query parameters — do not confuse the two shapes.
- **`account_name`** inside `bank_details` is required for most countries even when the schema does not mark it required.
- **SWIFT uses `swift_code`, not routing** — do NOT put BIC in `account_routing_type1` (LOCAL routing only).
- **IBAN countries may still require both `iban` and `swift_code` on SWIFT** — follow schema exactly for each combo.
- **LOCAL routing keys vary by country** (`sort_code`, `aba`, `bsb`, etc.) — use schema + country docs, never hardcode.
- **`bank_account_category`** — required for **US/USD/LOCAL** (both COMPANY and PERSONAL) and some personal accounts (e.g., BR). Valid values: **`"Checking"` / `"Savings"`** (note the `s`). The schema may omit this field — always include it for US beneficiaries.
- **SE/SEK/LOCAL** — the schema marks `account_routing_type1`, `account_routing_value1`, and `account_number` as optional — **they are actually required**. IBAN alone is NOT enough. Include `account_routing_type1` (`bank_code`), `account_routing_value1` (clearing number, 4–5 digits depending on the bank), and `account_number` (length is bank-dependent; the schema allows up to 15 digits). Ask the user for these values (see IBAN rule below).
- **PERSONAL uses `additional_info` for tax IDs** — `personal_id_type` and `personal_id_number` belong in `additional_info` (nested under `beneficiary` per the canonical body; verify exact path against your surface's schema). COMPANY uses `company_name`; PERSONAL uses `first_name` + `last_name`.
- **List search uses `name` (PERSONAL) / `company_name` (COMPANY)** — there is no `first_name` filter. Use the actual filter names exposed by the listing operation; do not invent filters from JSON body field names.
- **Do NOT decompose IBANs into bank_code + account_number yourself** — if the schema requires separate routing and account fields but the document only provides an IBAN, tell the user exactly which fields are needed (e.g., "The SE/SEK/LOCAL schema requires a 4–5 digit clearing number and a bank-specific account number separately — I cannot reliably extract these from the IBAN. Could you provide them?"). IBAN BBAN structures vary by country; guessing the split causes validation failures.
- **Preserve original values during extraction; normalize only when building the JSON payload.** In the extraction table (Steps 2/6), show bank details **AS WRITTEN** in the document (e.g., "Agência: 1234-5", "Conta Corrente: 1234567-8") and explicitly label the API field mapping (e.g., "Agência → `bank_branch`", "Conta Corrente → `account_number`"). Do NOT strip formatting during extraction.
- **Strip formatting to match schema `pattern` only in Step 10 (payload construction).** Check the field's `pattern` regex first, then strip only characters that prevent a match. E.g., GB sort code pattern `^[0-9]{6}$` → strip hyphens from `20-32-06` to get `203206`. But if the pattern already allows the characters (e.g., BR `bank_branch` pattern `^[0-9A-Za-z.-]{4,7}$` allows dashes, CA account number `^[0-9A-Za-z]{7,21}$`), preserve the original value. **Always show the before→after transformation** so the user can verify each mapping.
- **`address.state` uses ISO 3166-2 codes** with country prefix (e.g., `CA-ON`, `AU-NSW`, `IN-KA`). Do NOT use bare abbreviation.
- **Account number errors (`066`, `086`)** mean wrong length or invalid format — ask the user, never pad or truncate.
- **`validate` ≠ `create`** — validation only checks payloads, it does NOT create. Always validate first, confirm with user, then create.
- **Multiple banking options in one document** — if an invoice/document lists more than one bank account or transfer method (e.g., LOCAL SEK + SWIFT EUR), surface ALL options to the user and ask which to use. Follow the document's stated preference if one exists. Do NOT silently pick one.
- **Pagination:** use `page_num` (0-based) + `page_size`; increment until `has_more` is false.

---

## Workflow

### Phase 1: Extract

**Step 1 — Get the document(s).** Accept one or more supplier invoices, contracts, vendor lists, or bank detail documents. Batch supported.

**Step 2 — Extract supplier and bank details.** Identify: supplier name, entity type, bank name, bank country, currency, account number/IBAN, routing code(s), address (all five components: `street_address`, `city`, `state`, `postcode`, `country_code`), contact info. Documents may be in any language — extract bank details regardless of language, keep company/entity names in their original language, and present the extracted summary in English for user confirmation.

**Step 2b — Verify user-supplied field names against schema.** If the user's request mentions routing types, code names, or bank-detail parameters that you cannot confirm exist in the Airwallex API, **do NOT echo them back as required fields.** Instead, proceed to Step 4 (pre-flight) and Step 5 (schema fetch) first, then return to the user with only the fields the schema actually requires — and call out any user-mentioned terms that don't correspond to real API fields.

**Step 3 — Clarify intent before proceeding.** Before entering schema checks or API calls, confirm the user's actual goal. Present the extracted summary and explicitly ask:
- **Create new** beneficiary — proceed to validation and creation.
- **Update existing** — search first, show matches, then update.
- **Check for duplicates only** — search and report without creating.
- **Something else** — clarify before committing to a workflow path.

Do NOT assume "create new" by default. If the user's request is ambiguous (e.g., "set up this supplier" could mean create or update), ask. Only proceed to Step 4 after the user confirms the intended action.

If the user's wording does not unambiguously identify which beneficiary records to act on, and there is more than one possible supplier/payee/vendor/beneficiary in the attachment, prior extraction, or conversation context, do **not** choose for them or process every row by default. Present the candidate list and ask which specific record(s) they mean before schema checks or API calls. This applies to singular references ("this supplier", "that vendor", "them"), vague batch language ("these suppliers", "the new vendors"), and any request where the selected rows are unclear.

**Step 4 — Confirm environment.** State sandbox vs production to the user (sandbox default; production only on explicit confirmation). Validate auth via a low-cost read.

**Step 5 — Fetch country-specific schema AND country docs.** For EVERY unique country/currency/transfer-method/entity-type combo, run BOTH:
  1. Schema fetch (required fields + patterns) — see the constraints section above for the exact form on each surface.
  2. Country docs page via WebFetch (`https://www.airwallex.com/docs/payouts/payout-network/bank-accounts/{country-slug}`) — valid enum values, routing formats, extra field requirements. **Do NOT skip the country docs.**

**Step 6 — Build beneficiary table:**

| # | Company/Name | Entity Type | Bank Country | Currency | Transfer Method | Key Bank Fields | State | Status |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |

Fill `State` with ISO 3166-2 code or `n/a`. Mark incomplete rows with `[?]`.

**Step 7 — Validate and confirm.** Cross-check against schema. Do NOT proceed until every row passes and user confirms.

### Phase 2: Validate & Create

**Step 8 — Completeness check.** Validate all planned beneficiaries before first write.

**Step 9 — Match existing beneficiaries.** Search by name/company_name. Paginate fully. If match exists, user decides: skip, update, or create new.

**Step 10 — Validate.** **Process one at a time, sequentially** — do NOT parallelize (a parallel failure cancels sibling calls). Prepare each payload, validate it, show the result, and fix failures using exact API error messages before moving to the next.

**Step 11 — Confirm environment before writes.** Re-state the active environment / account to the user (the session may have changed since Step 4). Wait for explicit user approval before proceeding.

**Step 12 — Create.** **HARD GATE: NEVER attempt `create` for a beneficiary whose `validate` returned an error.** Fix the validation error first (ask the user for corrected data) or skip that row entirely. Retrying create with the same failing payload wastes turns and will fail identically. **Process one at a time, sequentially** — do NOT parallelize. Create only validated, unmatched rows. Wait for each creation to succeed before starting the next. Report each result immediately.

**Step 13 — Summary & next steps.** Show final summary: created/skipped/failed. Then advise on what the user can do next:
- **Verify** — offer bank account ownership verification if the country supports it (Phase 3).
- **Transfer** — remind the user that transfers must be initiated in the Airwallex Dashboard (this skill cannot move money).
- **Cashflow impact** — if the user plans to pay these suppliers soon, note that each payout will reduce their currency balance. Suggest using the **manage-cashflow** skill to check whether their current balances can cover the planned payments and whether any FX conversion is needed before initiating transfers.

### Phase 3: Verify bank account (optional)

Bank account ownership verification confirms the account belongs to the named beneficiary. Not all countries or transfer methods support this.

**Step 14 — Check verify eligibility.** Confirm the beneficiary-account verification operation is available on your surface. If it is not exposed, suggest the Airwallex Dashboard.

Not all country/transfer-method combinations support verification — if the verify call rejects with an unsupported-country or unsupported-method error, explain and suggest the Airwallex Dashboard as fallback.

**Step 15 — Submit verification.** The verify operation takes a candidate `bank_details` payload — **NOT a beneficiary ID** — so you can verify before the beneficiary record exists. Body shape: `entity_type`, `transfer_method`, `bank_details`. No `request_id`. Verify the exact body schema before sending.

Show the verification status to the user. Possible responses include `VERIFIED`, `INVALID`, `CANNOT_VERIFY`, and `EXTERNAL_SERVICE_UNAVAILABLE`; if the call is rejected outright, suggest the Airwallex Dashboard as fallback.

---

## Error handling

Generic patterns (401/auth, API validation, duplicates, partial writes, missing required fields) — see [awx-best-practices Error handling](../awx-best-practices/SKILL.md) and [api_traps.md](../awx-best-practices/references/api_traps.md).

Domain-specific:

| Situation | Action |
| --- | --- |
| Document unreadable | Ask for content another way |
| Extraction ambiguous | Mark `[?]`, ask user, do not guess |
| Bank country unclear | Ask user — wrong country cascades to wrong fields |
| Required bank field missing | Show which field for which country schema |
| Schema fetch fails | Try alternate transfer method (LOCAL → SWIFT) |
| Validation fails | Show exact API error, ask user to correct |
| `066` / `086` account errors | Ask user to verify account format/length; never pad or truncate |

---

## Country reference

For each country, fetch: `https://www.airwallex.com/docs/payouts/payout-network/bank-accounts/{country-slug}`

Common slugs: `united-kingdom`, `united-states`, `australia`, `india`, `canada`, `new-zealand`, `united-arab-emirates`, `singapore`, `hong-kong-sar`, `brazil`, `japan`. EU countries use the name (e.g., `germany`, `france`, `sweden`).

---

## Workflow summary

```
Phase 1: Extract
  get document(s) → extract bank details → clarify intent
    → environment / account + auth → fetch country schema + docs
      → build table → validate → user confirms

Phase 2: Validate & Create
  completeness check → match existing → validate each
    → confirm environment → create → summary & next steps

Phase 3: Verify bank account (optional)
  check eligibility → submit verification → show status
```
