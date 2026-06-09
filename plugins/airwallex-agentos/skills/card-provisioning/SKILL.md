---
name: card-provisioning
description: Provision virtual or physical corporate cards in Airwallex Issuing — create cardholders, issue cards with spend limits, and manage card spending. Use when the user says "create a card for", "spin up a virtual card", "set up a card for Adobe", "provision a card", or needs to manage corporate card spending. Do NOT use for bank transfers, creating invoices, or checking FX rates.
metadata:
  author: Airwallex
  version: 0.1.0
compatibility: Works with the Airwallex CLI (`airwallex` binary on PATH, authenticated via `airwallex auth login`) or the Airwallex MCP server (enabled and authenticated). Requires Issuing enabled on the Airwallex account.
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

This skill only covers Issuing-domain operations (cards, cardholders, issuing-transactions). If the task requires anything outside that domain, **stop — this is the wrong skill.** Redirect the user:

- Viewing sensitive card details (PAN, CVV) → direct to Airwallex Dashboard
- Wire transfers / payouts → not yet available (use Airwallex Dashboard)
- Setting up suppliers / beneficiaries → **beneficiary-creation** skill
- Creating invoices → **contract-to-billing** skill
- FX conversions, balances, treasury → **manage-cashflow** skill
- Ad-hoc tasks outside card workflow → **awx-best-practices** skill (fallback)

## Non-negotiables

### Terminology

- **Cards draw from the account's currency balance — no "card balance."** Say "$X/month limit" or "$X drawn this month."
- **Cardholder ≠ Card.** One cardholder can have multiple cards. Match or create cardholder first.
- **`AUTHORIZED` ≠ money moved.** It's a hold. `CLEARED` = money left. Be explicit when reporting.
- **Spend limits are always per interval + currency.** Say "$50.00 per month in USD."

### Operational rules

- **For ambiguous-intent requests, do not start the workflow until the action is confirmed.** If the user has not clearly confirmed the exact write action, stop before schema reads, auth checks, or other workflow setup that materially advances execution.
- **NEVER fabricate or assume missing information.** If any required field is uncertain, absent, or ambiguous — STOP and ask the user. Keep asking until you have every parameter needed. Do NOT fill in defaults, placeholder values, or "reasonable guesses."
- **Flag generic or test-like cardholder names.** If all cards in a batch share the same cardholder name, or the name appears generic/test-like (e.g., "Test Account", "Demo User", "Admin", "Card 1"), flag this as unusual and ask the user to confirm before proceeding. In production, generic names are a high-risk fraud signal.
- **Always fetch fresh data** — re-fetch before every step.
- **Prefer business labels over raw IDs in user-facing output.** Show cardholder names and card nicknames first; surface IDs only when operationally necessary or when the user asks.
- **One wallet, multiple currencies.** Say "AUD balance" — never "AUD wallet."
- **Default to sandbox.** Confirm with user before any production write.
- **Always set a spend limit** — never create an unlimited card. Every card must have an explicit limit amount, currency, and interval.
- **Always require a purpose/nickname** for each card.
- **Never handle or display PAN, CVV, or expiry.** Direct user to the Airwallex Dashboard — this is the **sole** channel for viewing sensitive card data. When refusing PAN/CVV/expiry requests, do NOT mention any get-card endpoint, SDK, or alternative technical path as a partial workaround. Frame the refusal as a platform-level security boundary: sensitive card details are never accessible through the agent in any form — even masked. Mentioning other APIs weakens the security message.
- **Do NOT invent advanced card-control fields** (MCC restriction, merchant controls, etc.) — see "Card & cardholder constraints" below.
- **Write safety.** Show the full payload to the user and get confirmation before every card create / update / cardholder create. **Confirm row-by-row in a batch** — never get a single up-front "yes" and then issue the rest unattended. Batch template previews do NOT count as confirmation — confirm and execute each individual payload (per cardholder, per card).
- **Never retrieve full card details via API.** Direct user to the Airwallex Dashboard.
- **Before increasing limits, show current spend vs limit first** — fetch the card's current limits and recent spend via the card-limits / transactions endpoints.
- **Flag unusual spend patterns** — alert if spend jumped 3x+ vs previous period.
- **Flag cards approaching their limit** — if utilization is ≥ 80%, proactively warn and offer to adjust (e.g., "Your AWS card is at $412 / $500 (82%) — want me to increase it?").
- **Confirm card spec before creating** — **production cards spend real money immediately**.
- **Search for existing cardholder by email before creating** — avoid duplicates.
- **Never fabricate cardholder or card IDs.** If the user gives a placeholder ID, list cardholders or cards to find the real UUID.
- **Batch requests:** if the user gives names/emails but no `form_factor`, currency, interval, or program purpose, ASK ONCE for shared defaults before creating. Process rows sequentially — never in parallel.
- **Card and cardholder IDs are UUIDs** — never use placeholders like `card_abc` or `cardholder_xyz`. If you only have a name or label, look up the real UUID first.

### Card & cardholder constraints

- **`created_by`** — full legal name of the **person requesting** the card, not the cardholder. Ask the user if unspecified.
- **`is_personalized`** — VIRTUAL → `false`, PHYSICAL → `true`. Ask the user if the form factor is unspecified; do not default silently in production.
- **Body shape differs by surface.** Use the templates in [references/card-templates.md](references/card-templates.md) — do NOT build the payload incrementally or guess fields. Verify the exact field shape against the resource schema for your surface before sending. Common pitfalls (`MULTIPLE` vs `MULTI`, `program` wrapper vs flat fields, `authorization_controls` nesting) are catalogued in [api_traps.md](../awx-best-practices/references/api_traps.md).
- **Merchant category / MCC restriction support is unconfirmed in this workflow unless explicitly documented.** Do NOT invent fields like `allowed_categories`. Only claim the restriction was applied if the API response explicitly shows the enforced control.
- **INDIVIDUAL cardholder quirks** (not surfaced by schema or manifest): `individual.address` uses `country` (not `country_code`); `individual.express_consent_obtained` is the string `"yes"` (not boolean `true`). Ask the user for DOB, address, and email — never fabricate.
- **DELEGATE cardholder** has minimal fields — no DOB or address required.
- **Physical-card delivery is create-time only.** The card-update operation does NOT accept `postal_address` or `delivery_details` — if either is wrong after creation, close and re-issue. Two valid paths at create time: (a) cardholder has a registered `postal_address` and card create uses it by default; (b) pass `postal_address` directly on card create to override. For EXPRESS shipment (or any China destination), `delivery_details.mobile_number` (E.164) is required. Always confirm the address with the user.
- **Physical cards are created `INACTIVE`** — activate after delivery via the card-activate operation.
- **Authorizations vs transactions:** there is no separate authorizations resource — list issuing-transactions with `status: AUTHORIZED` to see pending holds.
- **Spend aggregation is manual.** No built-in category filter or `cards spending` endpoint — list transactions per card (filter by `card_id`) and sum in post-processing. Use cursor pagination on issuing-transactions.

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

For a **batch request** (multiple cards from a document or list), list cardholders BEFORE building the table so you can show cardholder match results inline. Then present a full extraction table with per-row status:

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

**Step 3 — Confirm environment.** State sandbox vs production to the user (sandbox default; production only on explicit confirmation). Validate auth via a low-cost read. Verify Issuing is enabled on the active account.

**Step 4 — Match existing cardholder** by email. Reuse only if status is `READY`; otherwise stop and explain the cardholder must reach `READY` before issuing. Paginate fully (`page_size` ≥ 10) until `has_more` is false.

**Step 5 — Create cardholder** (if needed). Copy the appropriate template from [references/card-templates.md](references/card-templates.md) (INDIVIDUAL for a named person, DELEGATE for a purpose card), fill in values, show the full payload to the user and get explicit confirmation, then execute. In a batch, confirm row-by-row.

**Step 6 — Create card.** Do NOT add extra JSON fields for MCC or merchant restrictions unless the exact field is documented and verified first. **Process card creates sequentially** — do NOT parallelize. A parallel failure cancels sibling calls, causing cascading errors. Wait for each creation to succeed before starting the next.

Copy the Virtual or Physical card template from [references/card-templates.md](references/card-templates.md), fill in values, show the full payload to the user and get explicit per-card confirmation, then execute. A prior sample/template preview is not approval for later cards.

**Physical-card delivery address — two valid paths (both surfaces):**

1. **Cardholder default.** The cardholder has a registered `postal_address` (set via the cardholder-create / update operation). Card create uses it by default.
2. **Per-card override.** Pass `postal_address` directly on card create when the delivery destination differs from the cardholder's registered address.

Always show the user the cardholder's registered address alongside the intended delivery address and confirm before issuing. Never fabricate an address based on the card's currency. For EXPRESS shipment (or any China destination), pass `delivery_details` with `preferred_delivery_mode: "EXPRESS"` and an E.164 `mobile_number`. Both `postal_address` and `delivery_details` are create-time only — if either is wrong after creation, close and re-issue.

Physical cards are created `INACTIVE` — activate after delivery via the card-activate operation.

**Step 7 — Verify and confirm.** Re-fetch the created card (and limits if needed), then show: card ID, nickname, type, currency, limits, status. **In the final confirmation, do NOT display any part of the card number — including masked/last-4 digits.** The API response includes `card_number` with partial masking, but even last-4 digits must not appear in agent output. Identify cards by nickname and card ID only. Direct user to the Airwallex Dashboard for PAN, CVV, and expiry — do NOT construct Airwallex Dashboard URLs. If the user mentioned a specific vendor or subscription (e.g., "card for Adobe"), remind them of the next step: go to the Airwallex Dashboard to copy the card details (PAN, CVV, expiry), then enter them on the vendor/subscription site to complete setup.

### Phase 3: Manage Cards (ongoing)

**Update limits:** Always show current spend vs limit first (via the card-limits lookup), then update after user confirms.

**Review spend:** List transactions per card (filter by `card_id`) and sum amounts. When presenting spend summaries:

- **Show utilization for every card:** display spent amount, limit, and percentage used (e.g., "$412 / $500 — 82%").
- **Flag cards approaching their limit:** if utilization is ≥ 80%, add a warning and proactively ask whether the user wants to increase the limit.
- **Map merchant descriptors to business labels** in user-facing output. Transaction descriptions like "STRIPE* NOTION" or "AMZN MKTP US" are cryptic — translate them to recognizable names ("Notion", "Amazon") wherever possible. When uncertain, show both: "AMZN MKTP US (likely Amazon)".

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

> **Spend aggregation is manual.** No `cards spending` endpoint. List issuing-transactions with `card_id`, optional `status`, optional date range, and cursor pagination (`page` / `page_after`, `page_size` 10–100). Filter in post-processing if needed.

**Activate physical card** after delivery.

**Batch provisioning:** Follow the batch extraction table from Step 2. Process ✅ Ready rows first — create cardholders where needed, then create cards row by row sequentially. After all ready rows are done, report results and re-present the ❌ Blocked rows for the user to resolve. ASK ONCE for shared defaults (form_factor, interval) only if the document does not specify them per row. Report each `card_id` with its cardholder nickname.

---

## Error handling

Generic patterns (401/auth, API validation, duplicates, partial writes, missing required fields) — see [awx-best-practices Error handling](../awx-best-practices/SKILL.md) and [api_traps.md](../awx-best-practices/references/api_traps.md).

Domain-specific:

| Situation | Action |
| --- | --- |
| Cardholder details incomplete | Ask for missing required fields (name, email, DOB for INDIVIDUAL) |
| All required fields present | Proceed — do NOT block on optional fields (address, phone, etc.) unless the card type requires them (e.g., physical cards need postal_address) |
| Card creation fails | Show full error. Go back to [references/card-templates.md](references/card-templates.md) — include ALL required fields and retry once. For any other rejection, stop and show error |
| Limit format unclear | Ask: amount + currency + interval (per transaction / daily / monthly) |
| Cardholder not READY | Stop — the cardholder must reach `READY` before issuance. May need KYC verification; check Airwallex Dashboard |
| Physical card missing postal address | Ask for delivery address |
| MCC / merchant restriction requested but not documented | Say support is unconfirmed in this workflow; create the card without guessed restriction fields or direct the user to the Airwallex Dashboard / policy controls |

---

## Workflow summary

```
Phase 1: Gather Requirements
  understand request → build card spec → user confirms

Phase 2: Create Card
  environment / account + auth → match cardholder → create cardholder if needed
  → create card (sequential, per-row confirmed) → verify & confirm

Phase 3: Manage (ongoing)
  show spend vs limit → update limits → aggregate by category → activate physical cards
```
