---
name: beneficiary-creation
description: Extract bank details from supplier invoices or documents, validate per-country requirements, and create beneficiaries in Airwallex. Use when the user says "set up this supplier", "onboard these vendors", "create beneficiary from invoice", "add a payee", or uploads supplier documents with bank details. Do NOT use for creating invoices, checking balances or FX rates, or provisioning cards.
metadata:
  author: Airwallex
  version: 0.1.0
compatibility: Requires airwallex CLI installed and on PATH. Authenticated via airwallex auth login (OAuth browser flow). Best results with Claude Opus.
---

# Beneficiary Creation

Reads supplier invoices or documents, extracts bank details, validates against country-specific schemas, and creates beneficiaries in Airwallex. **This skill creates beneficiaries only — money movement (transfers) is not in scope.**

## When to use

- User uploads supplier invoices, contracts, vendor lists, or documents with bank details
- User asks to "set up a supplier" or "onboard a vendor"
- User wants to create beneficiaries from extracted bank information
- User has multiple suppliers to onboard in batch

## When NOT to use

**FIRST ACTION — before any CLI command:** Run `airwallex --tree --compact` to discover available command groups and subcommands. If a command or subcommand does not appear in the tree output, it **does not exist** — do NOT invent it. (Global flags: `--compact` for single-line JSON, `--dry-run` to preview writes, `--confirm` to execute writes — valid only when placed immediately after `airwallex`.)

**Global flag placement:** Global flags must go immediately after `airwallex`, before the resource/action. This applies to `--compact`, `--dry-run`, and `--confirm` (plus `--tree` for discovery). Correct: `airwallex --compact beneficiaries list`, `airwallex --dry-run beneficiaries create --data-file payload.json`, `airwallex --confirm beneficiaries create --data-file payload.json`. Wrong: `airwallex beneficiaries list --compact`, `airwallex beneficiaries create --dry-run --data-file payload.json`, `airwallex beneficiaries create --confirm --data-file payload.json` — these fail with `No such option`.

This skill only covers Payouts-domain CLI commands (`beneficiaries`, `beneficiary-schemas`). If the task requires a command group outside this domain (e.g., `invoices`, `cards`, `conversions`, `balances`), **stop — this is the wrong skill.** Redirect the user:

- Wire transfers / payouts → not yet available (use Airwallex Dashboard)
- Creating invoices (money in) → **contract-to-billing** skill
- FX conversions, balances, treasury → **manage-cashflow** skill
- Provisioning corporate cards → **card-provisioning** skill
- Ad-hoc tasks outside beneficiary workflow → **awx-best-practices** skill (fallback)

## Prerequisites

- `airwallex` CLI installed and on PATH
- Authenticated via `airwallex auth login` (check with `airwallex auth whoami`)
- Environment: sandbox by default. For production, log in with `airwallex auth login --prod`. The environment is locked to the authenticated session — there is no per-command override.
- **Verify commands before use:** Run `airwallex --tree --compact <group>` (e.g., `airwallex --tree --compact beneficiaries`) to confirm the subcommand exists. For write commands, run `airwallex <resource> <action> --api-schema-only` (e.g., `airwallex beneficiaries create --api-schema-only`) to get the request body schema — read every `required: true` field and include them all in the payload. For read commands, use `airwallex <resource> <action> --help` (e.g., `airwallex beneficiaries list --help`) to check flags. For `beneficiary-schemas generate`, also read every `required: true` field from the returned schema response and include them all in the payload.
- **`request_id` is MANDATORY for write commands.** Always include `"request_id"` in the JSON body for every `create`, `update`, and `validate` command — the API rejects writes without it. This prevents duplicate beneficiaries only when the same logical retry reuses the same `request_id`; generate a fresh UUID for each distinct new operation via `uuidgen | tr '[:upper:]' '[:lower:]'` — NEVER hand-write a UUID or use sequential/patterned values like `a1b2c3d4-...`. One known exception: `beneficiaries verify` does NOT take `request_id`. Action commands without a body do not take `request_id`. When in doubt, run `--api-schema-only` to confirm.

---

## Non-negotiables

### HARD GATE — money-movement requests

If the user's message mentions **transferring, sending, wiring, or paying money** (e.g., "set up and transfer", "send $15K to them", "pay this supplier now"):

1. **Very first sentence of your reply** must state the capability boundary: *"I can set up [name] as a beneficiary, but I can't execute transfers — that must be done in the Airwallex Dashboard after the beneficiary is created."*
2. Do NOT ask for payment amount, payment reason, source wallet, or any transfer-related details — these are not inputs to this skill.
3. Do NOT imply that the transfer will happen as part of this workflow or that you will "set up the payment right away."
4. After stating the boundary, proceed normally with the beneficiary-creation workflow.

This gate fires even if the transfer request is mixed with a valid beneficiary-creation request. Acknowledge what you CAN do, clearly state what you CANNOT, then continue with the part you can handle.

### Terminology

- **Beneficiary = first step of the payout workflow.** The full payout flow is: **create beneficiary → (optional) verify bank ownership → initiate transfer (Airwallex Dashboard only)**. This skill covers the first two steps — setting up and validating the payee record. Transfers are out of scope and must be done in the Airwallex Dashboard.
- **Beneficiary ≠ transfer.** Creating a beneficiary saves payee details — does NOT send money. Duplicate beneficiaries waste time during the transfer step, which is why the skill searches for existing records before creating.
- **LOCAL vs SWIFT.** LOCAL = domestic clearing (cheaper, faster). SWIFT = international wire (costlier). Do NOT silently pick a transfer method — determine the available rails from the document, country docs, and schema, then present the options to the user with a brief note on cost/speed differences. If the document specifies a preferred rail, surface that preference but still confirm with the user. Only when the document is silent AND the schema shows exactly one supported rail for the country/currency combo may you proceed without asking.
- **COMPANY vs PERSONAL.** COMPANY = business entity. PERSONAL = individual (freelancer). Determines required identity fields.
- **Schema ≠ country docs — you need both.** The schema's `required` flag is not always accurate (some fields marked optional are actually required by the API). Country docs tell you which fields are truly required AND which values are valid. When in doubt, include all fields the country docs list as required, even if the schema says optional.

### Operational rules

- **For ambiguous-intent requests, do not start the workflow until the action is confirmed.** If the user has not clearly confirmed the exact write action, stop before schema reads, auth checks, or other workflow setup that materially advances execution.
- **NEVER fabricate or assume missing information.** If any required field is uncertain, absent, or ambiguous — STOP and ask the user. Keep asking until you have every parameter needed for the API call. This applies to bank details, routing numbers, addresses, entity types, and all required inputs. Do NOT fill in defaults, placeholders, or "reasonable guesses." The only data you may fill in yourself is `nickname` and `request_id`.
- **NEVER echo back unverified field names from the user.** If the user mentions routing types, code names, or bank-detail parameters that you have not confirmed via schema fetch, do NOT include them in your response as if they were real API fields. Instead: (1) acknowledge what the user asked for, (2) fetch the beneficiary schema for that country/currency/transfer-method combo, (3) reply with only the fields the schema actually requires — and flag any user-mentioned terms that do not map to a schema field. Parroting an unverified parameter name back to the user — even just to ask for its value — treats it as a real field and is a form of hallucination.
- Even when the user says "use example data" — STOP and list the concrete fields needed. Offer to create a JSON template for them to fill in.
- Flag any extraction uncertainty — never guess at bank details.
- **Always fetch fresh data** — re-fetch before every step.
- **Prefer business labels over raw IDs in user-facing output.** In summaries, tables, and explanations, show beneficiary names or other human-readable business labels instead of raw system IDs whenever possible. Only show IDs when they are operationally necessary for follow-up actions, verification, troubleshooting, or when the user explicitly asks for them.
- **Default to sandbox.** Confirm with user before any production write — production beneficiaries are real payee records.
- **Dry-run before every write.** Before executing any write command (POST), first run it with the `--dry-run` global flag to preview the request envelope without sending it. Show the envelope to the user (method, URL, body, environment) and get explicit approval. Only then re-run with `--confirm` to execute. The two-step sequence is: (1) `airwallex --dry-run <command>` → show preview → user approves → (2) `airwallex --confirm <command>` → execute. **Never skip the dry-run step for write operations.**
- **Positional IDs, not `--id`.** E.g., `beneficiaries get <ID>`. Never pass `--id` as a flag.
- Commands with an object body (`create`, `update`, `validate`, `verify`) use `--data-file`, `--data`, or `--data-stdin`. Action commands without a body take only positional IDs. Always check the schema.
- **JSON output is the default.** Use `--compact` only if you need single-line JSON output, and place it immediately after `airwallex`: `airwallex --compact beneficiaries list`. Do NOT put it after the action (`airwallex beneficiaries list --compact`).
- Prefer `--data-file` for beneficiary payloads (nested JSON, fewer shell escaping errors).
- **Always validate before creating** — validation is the primary safety mechanism. Only create after validation passes AND user confirms.
- Search for existing beneficiaries by name before creating — duplicate beneficiaries clutter the payout workflow.

### Beneficiary body template & constraints

Beneficiary create body template (three-level nesting — `bank_country_code` at BOTH top-level and inside `bank_details`):

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
  "bank_country_code": "GB",
  "currency": "GBP",
  "transfer_methods": ["LOCAL"],
  "nickname": "Acme supplier"
}
```

- **Fetch schema AND country docs for EVERY country before building ANY JSON:**
  1. Run `airwallex --confirm beneficiary-schemas generate --bank-country-code XX --account-currency YYY --transfer-method LOCAL --entity-type COMPANY` for each unique combo. **Always pass `--confirm`** — the CLI safety layer treats schema fetches as writes even though they are read-only. Without `--confirm` the call is blocked with a `SAFETY_BLOCK` error. Use `airwallex beneficiary-schemas generate --help` to verify exact flag names — the flag is `--account-currency` (NOT `--currency`).
  2. **Fetch** `https://www.airwallex.com/docs/payouts/payout-network/bank-accounts/{country-slug}` for each country — this is NOT optional (see Terminology: "Schema ≠ country docs"). Skipping this step causes validation failures.
- **`transfer_method` vs `transfer_methods`:** Schema fetch uses singular `transfer_method`. Validate/create body uses plural array (`"transfer_methods": ["LOCAL"]`). Mixing causes API rejection.
- **Top-level vs nested fields are different.** `bank_country_code` / `currency` / `transfer_methods` are top-level routing fields. `beneficiary.bank_details.bank_country_code` / `account_currency` are beneficiary bank fields. Do not substitute one for the other.
- **`account_name` required** — `beneficiary.bank_details.account_name` (account holder name) is required for most countries.
- **SWIFT uses `swift_code`, not routing** — do NOT put BIC in `account_routing_type1` (LOCAL routing only).
- **IBAN countries may still require both `iban` and `swift_code` on SWIFT** — follow schema exactly for each combo.
- **LOCAL routing keys vary by country** (`sort_code`, `aba`, `bsb`, etc.) — use schema + country docs, never hardcode.
- **`bank_account_category`** — required for **US/USD/LOCAL** (both COMPANY and PERSONAL) and some personal accounts (e.g., BR). Valid values: `"Checking"` / `"Saving"`. The schema may omit this field — always include it for US beneficiaries.
- **SE/SEK/LOCAL** — the schema marks `account_routing_type1`, `account_routing_value1`, and `account_number` as optional — **they are actually required**. IBAN alone is NOT enough. Include `account_routing_type1` (`bank_code`), `account_routing_value1` (clearing number, typically 4 digits), and `account_number` (**max 7 digits**). Ask the user for these values (see IBAN rule below).
- **PERSONAL uses `additional_info` for tax IDs** — `personal_id_type` and `personal_id_number` go under `beneficiary.additional_info`, NOT directly under `beneficiary`. COMPANY uses `company_name`; PERSONAL uses `first_name` + `last_name`.
- **`beneficiaries list` uses `--name`, not `--first-name`** — use `--name` for PERSONAL, `--company-name` for COMPANY. Do NOT invent flags from JSON body field names.
- **Pagination:** `beneficiaries list` uses `--page-num` (0-based) + `--page-size` (minimum 10, recommend 20). Increment until `has_more` is false.
- **No `auth refresh` command** — token refresh is automatic on 401. Just retry the command; only `auth login` if retry also fails.
- **Do NOT decompose IBANs into bank_code + account_number yourself** — if the schema requires separate routing and account fields but the document only provides an IBAN, tell the user exactly which fields are needed (e.g., "The SE/SEK/LOCAL schema requires a 4-digit clearing number and up to 7-digit account number separately — I cannot reliably extract these from the IBAN. Could you provide them?"). IBAN BBAN structures vary by country; guessing the split causes validation failures.
- **Preserve original values during extraction; normalize only when building the JSON payload.** In the extraction table (Steps 2/6), show bank details **AS WRITTEN** in the document (e.g., "Agência: 1234-5", "Conta Corrente: 1234567-8") and explicitly label the API field mapping (e.g., "Agência → `bank_branch`", "Conta Corrente → `account_number`"). Do NOT strip formatting during extraction.
- **Strip formatting to match schema `pattern` only in Step 10 (payload construction).** Check the field's `pattern` regex first, then strip only characters that prevent a match. E.g., GB sort code pattern `^[0-9]{6}$` → strip hyphens from `20-32-06` to get `203206`. But if the pattern already allows the characters (e.g., BR `bank_branch` pattern `^[0-9A-Za-z.-]{4,7}$` allows dashes, CA account number `^[0-9A-Za-z]{7,21}$`), preserve the original value. **Always show the before→after transformation** so the user can verify each mapping.
- **`address.state` uses ISO 3166-2 codes** with country prefix (e.g., `CA-ON`, `AU-NSW`, `IN-KA`). Do NOT use bare abbreviation.
- **Account number errors (`066`, `086`)** mean wrong length or invalid format — ask the user, never pad or truncate.
- **`validate` ≠ `create`** — validation only checks payloads, it does NOT create. Always validate first, confirm with user, then create.
- **Multiple banking options in one document** — if an invoice/document lists more than one bank account or transfer method (e.g., LOCAL SEK + SWIFT EUR), surface ALL options to the user and ask which to use. Follow the document's stated preference if one exists. Do NOT silently pick one.

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

**Step 4 — Pre-flight.** Run `auth whoami`. If it fails (no active session), ask the user which environment (sandbox/production). Once the user answers, **immediately execute** `airwallex auth login` (or `airwallex auth login --prod` for production) yourself — do NOT tell the user to run it manually. The command triggers a browser-based OAuth flow; the user completes sign-in in their browser. After the command returns, confirm the session with `auth whoami`. If it succeeds on the first check, tell the user the current environment. No `auth refresh` command — on 401, retry the real command first (auto-refresh); if retry also fails, ask environment and **execute `auth login` yourself**.

**Step 5 — Fetch country-specific schema AND country docs.** For EVERY unique country/currency/transfer-method/entity-type combo, run BOTH:
  1. Schema fetch (required fields + patterns).
  2. Country docs page via WebFetch (`https://www.airwallex.com/docs/payouts/payout-network/bank-accounts/{country-slug}`) — valid enum values, routing formats, extra field requirements. **Do NOT skip the country docs** — the schema alone does not provide valid values for routing types, state formats, or fields like `bank_account_category`.

**Step 6 — Build beneficiary table:**

| # | Company/Name | Entity Type | Bank Country | Currency | Transfer Method | Key Bank Fields | State | Status |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |

Fill `State` with ISO 3166-2 code or `n/a`. Mark incomplete rows with `[?]`.

**Step 7 — Validate and confirm.** Cross-check against schema. Do NOT proceed until every row passes and user confirms.

### Phase 2: Validate & Create

**Step 8 — Completeness check.** Validate all planned beneficiaries before first write.

**Step 9 — Match existing beneficiaries.** Search by name/company_name. Paginate fully. If match exists, user decides: skip, update, or create new.

**Step 10 — Validate.** **Process one at a time, sequentially** — do NOT parallelize (a parallel failure cancels sibling calls). Prepare each payload, validate it, show the result, and fix failures using exact API error messages before moving to the next.

**Step 11 — Confirm environment before writes.** Run `airwallex auth whoami` and tell the user which environment (sandbox/production) the creates will target. Do NOT skip this even if you checked earlier — the session may have changed. Wait for explicit user approval before proceeding.

**Step 12 — Create.** **HARD GATE: NEVER attempt `create` for a beneficiary whose `validate` returned an error.** Fix the validation error first (ask the user for corrected data) or skip that row entirely. Retrying create with the same failing payload wastes turns and will fail identically. **Process one at a time, sequentially** — do NOT parallelize. Create only validated, unmatched rows. Wait for each creation to succeed before starting the next. Report each result immediately.

**Step 13 — Summary & next steps.** Show final summary: created/skipped/failed. Then advise on what the user can do next:
- **Verify** — offer bank account ownership verification if the country supports it (Phase 3).
- **Transfer** — remind the user that transfers must be initiated in the Airwallex Dashboard (this skill cannot move money).
- **Cashflow impact** — if the user plans to pay these suppliers soon, note that each payout will reduce their currency balance. Suggest using the **manage-cashflow** skill to check whether their current balances can cover the planned payments and whether any FX conversion is needed before initiating transfers.

### Phase 3: Verify bank account (optional)

Bank account ownership verification confirms the account belongs to the named beneficiary. Not all countries or transfer methods support this.

**Step 14 — Check verify eligibility.** Run `airwallex --tree --compact beneficiaries` to confirm the `verify` subcommand exists, then run `airwallex beneficiaries verify --help` to check its required flags. If the subcommand is not in the tree, inform the user that verification is not available via CLI for this beneficiary and suggest the Airwallex Dashboard.

**Step 15 — Submit verification.** `beneficiaries verify` takes NO positional ID and NO `beneficiary_id` in the body — the payload identifies the account by its bank details, not by a beneficiary record. Run `airwallex beneficiaries verify --api-schema-only` first to confirm the exact body schema (currently `entity_type`, `transfer_method`, `bank_details`). Pass via `--data-file`/`--data`/`--data-stdin`. This command does not require `request_id`. Show the verification status to the user. If the API rejects with an unsupported-country or unsupported-method error, explain clearly and suggest the Airwallex Dashboard as fallback.

---

## Error handling

| Situation | Action |
| --- | --- |
| Document unreadable | Ask for content another way |
| Extraction ambiguous | Mark `[?]`, ask user, do not guess |
| Bank country unclear | Ask user — wrong country cascades to wrong fields |
| Required bank field missing | Show which field for which country schema |
| Schema fetch fails | Try alternate transfer method (LOCAL → SWIFT) |
| Validation fails | Show exact API error, ask user to correct |
| `066` / `086` account errors | Ask user to verify account format/length; never pad or truncate |
| Partial batch completion | Report succeeded (with IDs) and failed |
| Duplicate detected | Show existing, let user choose |
| 401 / auth expired | Retry command (auto-refresh). If retry also fails: ask user which environment (sandbox/production), **immediately execute** `auth login` or `auth login --prod` yourself (do NOT tell user to run it), confirm with `auth whoami`, then resume |
| API error | Run `airwallex <resource> <action> --api-schema-only` (e.g., `airwallex beneficiaries create --api-schema-only`) to verify body structure |

---

## Country reference

For each country, fetch: `https://www.airwallex.com/docs/payouts/payout-network/bank-accounts/{country-slug}`

Common slugs: `united-kingdom`, `united-states`, `australia`, `india`, `canada`, `new-zealand`, `united-arab-emirates`, `singapore`, `hong-kong-sar`, `brazil`, `japan`. EU countries use the name (e.g., `germany`, `france`, `sweden`).

---

## Workflow summary

```
Phase 1: Extract
  get document(s) → extract bank details → clarify intent
    → environment + auth → fetch country schema + docs
      → build table → validate → user confirms

Phase 2: Validate & Create
  completeness check → match existing → validate each
    → confirm environment → create → summary & next steps

Phase 3: Verify bank account (optional)
  check eligibility → submit verification → show status
```
