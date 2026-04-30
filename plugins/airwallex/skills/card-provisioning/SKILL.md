---
name: card-provisioning
description: Provision virtual or physical corporate cards in Airwallex Issuing — create cardholders, issue cards with spend limits, and manage card spending. Use when the user says "create a card for", "spin up a virtual card", "set up a card for Adobe", "provision a card", or needs to manage corporate card spending. Do NOT use for bank transfers, creating invoices, or checking FX rates.
metadata:
  author: Airwallex
  version: 0.1.0
compatibility: Requires airwallex CLI installed and on PATH. Authenticated via airwallex auth login (OAuth browser flow). Best results with Claude Opus.
---

# Card Provisioning

Creates virtual or physical corporate cards in Airwallex Issuing — one workflow to set up a cardholder, issue a card with spend limits, and optionally manage ongoing spend.

**Tone:** The target user is a busy entrepreneur, not a finance analyst. Keep language conversational and action-oriented — say "Your Adobe card is set up with a $50/month limit" rather than "Card ID card_xxx created with authorization_controls.transaction_limits.limits[0].amount = 50.00." Show business labels (card nicknames, cardholder names) first; keep raw IDs and technical details in the background unless the user asks for them.

## When to use

- User asks to create a virtual or physical card
- User wants to set up a card for a specific purpose (e.g., "card for Adobe", "travel card")
- User needs to provision cards for team members (batch)
- User wants to update card limits or review card spend
- User asks "what are we spending on software cards?" or wants spend aggregation by category

## When NOT to use

**FIRST ACTION — before any CLI command:** Run `airwallex --tree --compact` to discover available command groups and subcommands. If a command or subcommand does not appear in the tree output, it **does not exist** — do NOT invent it. (Global flags: `--compact` for single-line JSON, `--dry-run` to preview writes, `--confirm` to execute writes — valid only when placed immediately after `airwallex`.)

**Global flag placement:** Global flags must go immediately after `airwallex`, before the resource/action. This applies to `--compact`, `--dry-run`, and `--confirm` (plus `--tree` for discovery). Correct: `airwallex --compact cardholders list`, `airwallex --dry-run cardholders create --data-file payload.json`, `airwallex --confirm cards create --data-file payload.json`. Wrong: `airwallex cardholders list --compact`, `airwallex cardholders create --dry-run --data-file payload.json`, `airwallex cards create --confirm --data-file payload.json` — these fail with `No such option`.

This skill only covers Issuing-domain CLI commands (`cards`, `cardholders`, `issuing-transactions`). If the task requires a command group outside this domain (e.g., `invoices`, `beneficiaries`, `conversions`, `balances`), **stop — this is the wrong skill.** Redirect the user:

- Viewing sensitive card details (PAN, CVV) → direct to Airwallex Dashboard
- Wire transfers / payouts → not yet available (use Airwallex Dashboard)
- Setting up suppliers / beneficiaries → **beneficiary-creation** skill
- Creating invoices → **contract-to-billing** skill
- FX conversions, balances, treasury → **manage-cashflow** skill
- Ad-hoc tasks outside card workflow → **awx-best-practices** skill (fallback)

## Prerequisites

- `airwallex` CLI installed and on PATH
- Authenticated via `airwallex auth login` (check with `airwallex auth whoami`)
- Environment: sandbox by default. For production, log in with `airwallex auth login --prod`. The environment is locked to the authenticated session — there is no per-command override.
- **Verify commands before use:** Run `airwallex --tree --compact <group>` (e.g., `airwallex --tree --compact cards`) to confirm the subcommand exists. For write commands, run `airwallex <resource> <action> --api-schema-only` (e.g., `airwallex cards create --api-schema-only`) to get the request body schema — read every `required: true` field and include them all in the payload. For read commands, use `airwallex <resource> <action> --help` (e.g., `airwallex cards list --help`) to check flags.
- Issuing enabled on the account (check Airwallex Dashboard if creation fails)
- **`request_id` is MANDATORY for all write commands.** Always include `"request_id"` in the JSON body for every `create` and `update` command — the API rejects writes without it with `"request_id is mandatory"`. Generate a fresh UUID for each distinct operation via `uuidgen | tr '[:upper:]' '[:lower:]'` — NEVER hand-write a UUID or use sequential/patterned values like `a1b2c3d4-...`. If retrying the same logical operation after a transient/network failure, reuse the same `request_id`; only generate a new one for a distinct new operation. Action commands without a body (e.g., `cards activate`) do not take `request_id`.

---

## Non-negotiables

### Terminology

- **Cards draw from the account's currency balance — no "card balance."** Say "$X/month limit" or "$X drawn this month."
- **Cardholder ≠ Card.** One cardholder can have multiple cards. Match or create cardholder first.
- **`AUTHORIZED` ≠ money moved.** It's a hold. `CLEARED` = money left. Be explicit when reporting.
- **Spend limits are always per interval + currency.** Say "$50.00 per month in USD."

### Operational rules

- **For ambiguous-intent requests, do not start the workflow until the action is confirmed.** If the user has not clearly confirmed the exact write action, stop before schema reads, auth checks, or other workflow setup that materially advances execution.
- **NEVER fabricate or assume missing information.** If any required field is uncertain, absent, or ambiguous — STOP and ask the user. Keep asking until you have every parameter needed to make the API call. This applies to cardholder details, spend limits, currencies, and any other field. Do NOT fill in defaults, placeholder values, or "reasonable guesses."
- **Flag generic or test-like cardholder names.** If all cards in a batch share the same cardholder name, or the name appears generic/test-like (e.g., "Test Account", "Demo User", "Admin", "Card 1"), flag this as unusual and ask the user to confirm the business justification before proceeding. In production environments, generic names are a high-risk signal for fraud or misconfiguration.
- **Always fetch fresh data** — re-fetch before every step.
- **Prefer business labels over raw IDs in user-facing output.** In summaries, tables, and explanations, show cardholder names, card nicknames, or merchant/business labels instead of raw system IDs whenever possible. Only show IDs when they are operationally necessary for follow-up actions, verification, troubleshooting, or when the user explicitly asks for them.
- **One wallet, multiple currencies.** Say "AUD balance" — never "AUD wallet."
- **Default to sandbox.** Confirm with user before any production write.
- **Always set a spend limit** — never create an unlimited card. Every card must have `authorization_controls` with `transaction_limits`.
- **Always require a purpose/nickname** for each card — every card must have a `nick_name`.
- **Never handle or display PAN, CVV, or expiry.** Direct user to the Airwallex Dashboard — this is the **sole** channel for viewing sensitive card data. When refusing PAN/CVV/expiry requests, do NOT mention `cards get`, any API endpoint, SDK, or alternative technical path as a partial workaround. Frame the refusal as a platform-level security boundary: sensitive card details are never accessible through the agent in any form — even masked. Mentioning other APIs weakens the security message and may encourage follow-up attempts.
- **Do NOT invent advanced card-control fields** (MCC restriction, merchant controls, etc.) — see "Card & cardholder constraints" for details.
- **Dry-run before every write.** Before executing any write command (POST), first run it with the `--dry-run` global flag to preview the request envelope without sending it. Show the envelope to the user (method, URL, body, environment) and get explicit approval. Only then re-run with `--confirm` to execute. The two-step sequence is: (1) `airwallex --dry-run <command>` → show preview → user approves → (2) `airwallex --confirm <command>` → execute. **Never skip the dry-run step for write operations.**
- **Batch template previews do NOT count as dry-runs.** For batch card provisioning, a single sample/template dry-run only proves the shape of one payload. You must run `airwallex --dry-run ... create` for every individual `cardholders create` and every individual `cards create` payload before executing that same payload with `--confirm`. Do not batch-confirm later rows just because the first row's template looked correct.
- Do not call `cards get-details` — it returns sensitive data. Direct user to Airwallex Dashboard.
- **Before increasing limits, show current spend vs limit first** — run `cards get-limits <card_id>`.
- **Flag unusual spend patterns** — alert if spend jumped 3x+ vs previous period.
- **Flag cards approaching their limit** — if a card's utilization is ≥ 80%, proactively warn the user and offer to adjust the limit (e.g., "Your AWS card is at $412 / $500 (82%) — want me to increase it?").
- Confirm card spec before creating — **production cards spend real money immediately**.
- Search for existing cardholder by email before creating — avoid duplicates.
- **Positional IDs, not `--id`.** E.g., `cards get <ID>`, `cardholders get <ID>`. Never pass `--id` as a flag.
- Commands with an object body (`create`, `update`) use `--data-file`, `--data`, or `--data-stdin`. Action commands like `cards activate` take only positional IDs — no body flags. Always check the schema.
- **JSON output is the default.** Use `--compact` only if you need single-line JSON output, and place it immediately after `airwallex`: `airwallex --compact cardholders list`. Do NOT put it after the action (`airwallex cardholders list --compact`).
- **Card ID is a UUID.** Never use placeholders (`card_abc`). Always fetch real IDs from `cards list`.
- **Never fabricate cardholder or card IDs.** If the user gives a placeholder ID, run `cards list` or `cardholders list` to find the real UUID.
- **Batch requests:** If user gives names/emails but no `form_factor`/`currency`/`purpose` (COMMERCIAL vs PERSONAL)/`authorization_controls.interval`, ASK ONCE for shared defaults before creating. Process rows sequentially.

### Card & cardholder constraints

- **`created_by` is MANDATORY on every `cards create` body** — full legal name of the person requesting the card (not the cardholder name). Omitting returns `"created_by is mandatory"`.
- **`is_personalized` is MANDATORY on every `cards create` body** (omitting returns `"is_personalized is mandatory"`). VIRTUAL → `false`; PHYSICAL → `true`. If user didn't specify, ask — do not default silently for production.
- **`issue_to` required on card create** — must be `INDIVIDUAL` or `DELEGATE`, and should match cardholder type. Without it the API returns generic `BAD_REQUEST`.
- **`program` is an object** `{"purpose": "COMMERCIAL"}`, NOT a string. Copy the complete JSON template from Step 6 — do NOT build incrementally or guess fields.
- **`authorization_controls` structure:** `transaction_limits` is an object `{"currency": "USD", "limits": [{"amount": ..., "interval": "..."}]}` — NOT an array. `allowed_transaction_count` is `MULTIPLE` (not `MULTI`).
- **Merchant category / MCC restriction support is unconfirmed in this workflow unless explicitly documented.** Do NOT invent fields like `allowed_categories`. Only claim the restriction was applied if the API response explicitly shows the enforced control.
- **INDIVIDUAL cardholder** needs `name` (as an object `{"first_name": "...", "last_name": "..."}`), DOB, address, and `express_consent_obtained: "yes"` inside the `individual` object. Address uses `country` (not `country_code`). Missing any of these causes `invalid_argument`.
- **DELEGATE cardholder** has minimal fields — no DOB or address required.
- **Physical cards** need explicit activation (`cards activate`) after delivery. Virtual cards auto-activate.
- **Physical cards require `postal_address`** in the card create body. Look up the cardholder's registered address (`individual.address` from `cardholders list` / `cardholders get`), show it, and ask the user to confirm or provide an alternative. Do NOT silently reuse it and do NOT fabricate an address based on the card's currency. The API restricts delivery to the account's eligible countries (sandbox may differ from production). If no address on file, ask the user.
- **Command names:** `cards`, `cardholders` — NOT `issuing-cards` or `issuing-cardholders`. Only `issuing-transactions` has the `issuing-` prefix.
- **Authorizations vs transactions:** No separate authorizations command — use `issuing-transactions list --status AUTHORIZED` to see pending holds.
- **Spend aggregation is manual.** No `cards spending` command. List transactions per card via `issuing-transactions list` and sum in post-processing.
- **Pagination:** `cards list` and `cardholders list` use `--page-num` (0-based) + `--page-size` (minimum 10, recommend 20). `issuing-transactions list` uses cursor `--page` + `--page-size` (max 100). Repeat until `has_more` is false or `page_after` is absent.

---

## Workflow

### Phase 1: Gather Requirements

**Step 1 — Understand the card request.** Collect: purpose/nickname, cardholder (name + email), card type (Virtual/Physical), currency, spend limit (amount + interval), and any requested merchant restriction (MCC support is unconfirmed — see constraints above).

If the user gives a natural-language request like "Create a virtual card for Adobe, $50.00/month", extract what you can and ask for gaps (e.g., "Who should this card be assigned to?").

If the user provides a **document** (spreadsheet, PDF, email, list) with card specifications, extract each person's currency, limit, form factor, and other details **AS WRITTEN** in the document — do NOT normalize currencies or limits to a single default. Present the extracted table for confirmation before proceeding. This is the same principle as beneficiary-creation (extract bank details as-is) and contract-to-billing (show extracted data in tables).

**Step 2 — Build card spec table.** For a **single card**, present:

| Field | Value |
| --- | --- |
| Purpose / Nickname | _(e.g., Adobe Subscription)_ |
| Cardholder | _(e.g., Jane Doe, jane@company.com)_ |
| Form factor | _VIRTUAL / PHYSICAL_ |
| Currency | _(e.g., USD)_ |
| Spend limit | _(e.g., $50.00/month)_ |
| MCC restriction | _All merchants (unconfirmed — see constraints)_ |
| Personalized | _No (virtual) / Yes (physical)_ |

For a **batch request** (multiple cards from a document or list), present a full extraction table with per-row status. Run `cardholders list` BEFORE building the table so you can show cardholder match results inline.

| # | Name | Email | Currency | Limit/mo | Cardholder | Status | Issue |
| --- | --- | --- | --- | --- | --- | --- | --- |
| … | _(as extracted)_ | _(as extracted)_ | _(as extracted)_ | _(as extracted)_ | ✅ Exists / ❌ New | ✅ Ready / ❌ Blocked / ⚠️ Dup | _(specific reason)_ |

Required columns:
- **Cardholder:** whether the cardholder already exists (with status) or needs to be created.
- **Status:** ✅ Ready (all document-extracted fields present), ❌ Blocked (missing document data), or ⚠️ Duplicate.
- **Issue:** the specific missing field or problem for non-ready rows; "—" for ready rows.
- **All value columns extracted AS WRITTEN** from the document — especially per-person currencies and limits.

**Distinguish document issues from system defaults.** When determining row status:

**Tier 1 — Document issues (per-row blockers):** Missing data the document should have provided — name, email, currency, spend limit, DOB (for INDIVIDUAL cardholders) — plus conflicting values, duplicates, or ambiguous fields. These determine the Status column. A row is ✅ Ready when all document-extracted fields are complete and unambiguous.

**Tier 2 — System defaults (shared across batch):** Fields the API requires but card-request documents typically don't include — card nicknames, `created_by`, `express_consent_obtained`, delivery/postal addresses. Present these **after** the table as "I'll also need from you" items. Do NOT mark rows as ❌ Blocked on Tier 2 fields — this buries the real document issues and makes it look like nothing can proceed.

After the table, summarize:
1. How many rows are **ready to proceed** (Tier 1 complete).
2. Which rows are **blocked** on document issues and what is needed to unblock each.
3. Which rows are **duplicates** and your recommendation (e.g., use the more complete entry).
4. Which **shared defaults** (Tier 2) are still needed from the user — ask for these once, not per row.

Then ask the user to confirm ready rows and resolve blocked rows.

**Handling blanket overrides:** If the user provides a blanket override (e.g., "set all limits to $2,000 USD") that conflicts with document-extracted values (e.g., document lists GBP for some rows, AUD for others), flag the conflict and ask the user to choose before applying. Do NOT silently override document-extracted currencies or limits.

Do NOT proceed until user confirms.

### Phase 2: Create Card

**Step 3 — Pre-flight.** Run `auth whoami`. If it fails (no active session), ask the user which environment (sandbox/production). Once the user answers, **immediately execute** `airwallex auth login` (or `airwallex auth login --prod` for production) yourself — do NOT tell the user to run it manually. The command triggers a browser-based OAuth flow; the user completes sign-in in their browser. After the command returns, confirm the session with `auth whoami`. If it succeeds on the first check, tell the user the current environment. Verify Issuing is enabled. No `auth refresh` command — on 401, retry the real command first (auto-refresh); if retry also fails, ask environment and **execute `auth login` yourself**.

**Step 4 — Match existing cardholder** by email. Reuse only if status is `READY`; otherwise stop and explain the cardholder must reach `READY` before issuing. Paginate fully until `has_more` is false.

**Step 5 — Create cardholder** (if needed). Use the **exact template** — copy it, fill in values. For each new cardholder, run a dry-run for that exact cardholder payload, show/confirm it, then execute that same payload with `--confirm`. In a batch, do this row-by-row; do not dry-run only the first cardholder and then confirm-create the rest.

INDIVIDUAL cardholder template (named person) — replace every `<...>` with real values from the user:

```json
{
  "request_id": "<generate via uuidgen>",
  "email": "<cardholder_email>",
  "type": "INDIVIDUAL",
  "individual": {
    "name": {"first_name": "<first_name>", "last_name": "<last_name>"},
    "date_of_birth": "<YYYY-MM-DD>",
    "address": {
      "line1": "<street_address>",
      "city": "<city>",
      "postcode": "<postcode>",
      "country": "<country_code_2_letter>"
    },
    "express_consent_obtained": "yes"
  }
}
```

DELEGATE cardholder template (purpose card, minimal fields) — replace `<...>` with real values:

```json
{
  "request_id": "<generate via uuidgen>",
  "email": "<team_or_purpose_email>",
  "type": "DELEGATE"
}
```

**Step 6 — Create card.** Use the **exact template** — copy it, fill in values. Do NOT construct the payload from scratch or guess fields. Set `issue_to` to match cardholder type. Do NOT add extra JSON fields for MCC or merchant restrictions unless the exact field is documented and verified first. **Process card creates sequentially** — do NOT parallelize. A parallel failure cancels sibling calls, causing cascading errors. Wait for each creation to succeed before starting the next. For each card, dry-run that exact `cards create` payload first, get approval, then execute the same payload with `--confirm`; a prior sample/template preview is not approval for later cards.

Virtual card template — replace every `<...>` with real values from the user:

```json
{
  "request_id": "<generate via uuidgen>",
  "cardholder_id": "<cdh_id>",
  "form_factor": "VIRTUAL",
  "issue_to": "<INDIVIDUAL_or_DELEGATE>",
  "created_by": "<requesting_persons_full_name>",
  "is_personalized": false,
  "nick_name": "<card_purpose>",
  "authorization_controls": {
    "allowed_transaction_count": "MULTIPLE",
    "transaction_limits": {
      "currency": "<currency>",
      "limits": [{"amount": "<amount>", "interval": "<MONTHLY_or_other>"}]
    }
  },
  "program": {"purpose": "COMMERCIAL"}
}
```

Physical card template — same as virtual, plus `postal_address` (show cardholder's registered address and confirm with user; ask if none on file), `form_factor: "PHYSICAL"`, `is_personalized: true`:

```json
{
  "request_id": "<generate via uuidgen>",
  "cardholder_id": "<cdh_id>",
  "form_factor": "PHYSICAL",
  "issue_to": "<INDIVIDUAL_or_DELEGATE>",
  "created_by": "<requesting_persons_full_name>",
  "is_personalized": true,
  "nick_name": "<card_purpose>",
  "authorization_controls": {
    "allowed_transaction_count": "MULTIPLE",
    "transaction_limits": {
      "currency": "<currency>",
      "limits": [{"amount": "<amount>", "interval": "<MONTHLY_or_other>"}]
    }
  },
  "program": {"purpose": "COMMERCIAL"},
  "postal_address": {
    "line1": "<street_address>",
    "city": "<city>",
    "state": "<state>",
    "postcode": "<postcode>",
    "country": "<country_code_2_letter>"
  }
}
```

Physical cards are created `INACTIVE` — activate with `cards activate <card_id>` after delivery.

**Step 7 — Verify and confirm.** Re-fetch the created card (and limits if needed), then show: card ID, nickname, type, currency, limits, status. **In the final confirmation, do NOT display any part of the card number — including masked/last-4 digits.** The API response includes `card_number` with partial masking, but even last-4 digits must not appear in agent output. Identify cards by nickname and card ID only. Direct user to the Airwallex Dashboard for PAN, CVV, and expiry — do NOT construct Airwallex Dashboard URLs. If the user mentioned a specific vendor or subscription (e.g., "card for Adobe"), remind them of the next step: go to the Airwallex Dashboard to copy the card details (PAN, CVV, expiry), then enter them on the vendor/subscription site to complete setup.

### Phase 3: Manage Cards (ongoing)

**Update limits:** Always show current spend vs limit first (via `get-limits`), then update after user confirms.

**Review spend:** List transactions per card (`issuing-transactions list --card-id`), sum amounts. When presenting spend summaries:

- **Show utilization for every card:** display spent amount, limit, and percentage used (e.g., "$412 / $500 — 82%").
- **Flag cards approaching their limit:** if utilization is ≥ 80%, add a warning and proactively ask whether the user wants to increase the limit.
- **Map merchant descriptors to business labels** in user-facing output. Transaction descriptions like "STRIPE* NOTION" or "AMZN MKTP US" are cryptic — translate them to recognizable names (e.g., "Notion", "Amazon") wherever possible. When uncertain, show both: "AMZN MKTP US (likely Amazon)".

**Category aggregation** (e.g., "what are we spending on software?"): There is no API-level category filter. Build the category view by combining two signals:
1. **Card nickname** — cards named "Adobe Subscription", "AWS Dev", "Notion" are likely software. Group by the business purpose in the nickname.
2. **Transaction merchant descriptors** — for each card, scan transaction `merchant_name` / `description` fields and map to recognizable business names.

Combine both signals to classify cards into user-friendly categories (Software, Travel, Office, etc.). Present a grouped summary with per-card breakdown and category total:
```
Software spend this month: $847
  Figma:  $30  / $50   (60%)
  AWS:    $412 / $500  (82%) ⚠️ approaching limit
  Notion: $24  / $30   (80%) ⚠️ approaching limit
  GitHub: $21  / $50   (42%)
  Other (3 cards): $360
```
If a card's category is ambiguous, ask the user rather than guessing.

> **Spend aggregation is manual.** No `cards spending` command. Use `issuing-transactions list` with `--card-id`, `--status`, `--page` (cursor), and `--page-size` (max 100). Filter dates in post-processing if needed.

**Activate physical card** after delivery.

**Batch provisioning:** Follow the batch extraction table from Step 2. Process ✅ Ready rows first — create cardholders where needed, then create cards row by row sequentially. After all ready rows are done, report results and re-present the ❌ Blocked rows for the user to resolve. ASK ONCE for shared defaults (form_factor, interval) only if the document does not specify them per row. Report each `card_id` with its cardholder nickname.

---

## Error handling

| Situation | Action |
| --- | --- |
| Cardholder details incomplete | Ask for missing required fields (name, email, DOB for INDIVIDUAL) |
| All required fields present | Proceed — do NOT block on optional fields (address, phone, etc.) unless the card type requires them (e.g., physical cards need postal_address) |
| Card creation fails | Show full error. Go back to Step 6 template — include ALL required fields and retry once. For any other rejection, stop and show error |
| Limit format unclear | Ask: amount + currency + interval (per transaction / daily / monthly) |
| Cardholder not READY | Stop — the cardholder must reach `READY` before issuance. May need KYC verification; check Airwallex Dashboard |
| Physical card missing postal address | Ask for delivery address |
| MCC / merchant restriction requested but not documented | Say support is unconfirmed in this workflow; create the card without guessed restriction fields or direct the user to the Airwallex Dashboard/policy controls |
| Partial completion | Report which cardholders/cards succeeded with IDs, which failed, then ask how to proceed |
| `BELOW_MIN_PAGE_SIZE` | Remove `--page-size` or set it to 10+. Do NOT retry with a smaller value |
| API error (other) | Stop, show full error, ask user |
| Duplicate detected | Show details, let user choose |
| 401 / auth expired | Retry command (auto-refresh). If retry also fails: ask user which environment (sandbox/production), **immediately execute** `auth login` or `auth login --prod` yourself (do NOT tell user to run it), confirm with `auth whoami`, then resume |
| API error | Run `airwallex <resource> <action> --api-schema-only` (e.g., `airwallex cards create --api-schema-only`) to verify body structure |

---

## Workflow summary

```
Phase 1: Gather Requirements
  understand request → build card spec → user confirms

Phase 2: Create Card
  environment + auth → match cardholder → create if needed → create card → confirm

Phase 3: Manage (ongoing)
  show spend vs limit → update limits → aggregate by category → activate physical cards
```
