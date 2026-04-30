---
name: contract-to-billing
description: Extract billing details from purchase orders, contracts, or quotes, then set up Airwallex Billing by creating invoices and/or subscriptions — matching existing customers, products, and prices to avoid duplicates. Use when the user says "create invoice from this PO", "set up billing from this contract", "create a subscription from this agreement", "invoice this quote", "bill this customer", or attaches a document and asks to set up one-time, recurring, or mixed billing. Do NOT use for paying suppliers, provisioning cards, or checking FX rates.
metadata:
  author: Airwallex
  version: 0.1.0
compatibility: Requires airwallex CLI installed and on PATH. Authenticated via airwallex auth login (OAuth browser flow). Best results with Claude Opus.
---

# Contract to Billing

Reads a customer document (PO, contract, quote), extracts line items with AI, and creates a fully populated invoice in Airwallex Billing.

## When to use

- User uploads or references a purchase order, contract, quote, or billing document
- User asks to "create an invoice" from a document
- User wants to extract billing details and set up products/prices/customers
- User says "bill this customer" with a document attached

## When NOT to use

**FIRST ACTION — before any CLI command:** Run `airwallex --tree --compact` to discover available command groups and subcommands. If a command or subcommand does not appear in the tree output, it **does not exist** — do NOT invent it. (Global flags: `--compact` for single-line JSON, `--dry-run` to preview writes, `--confirm` to execute writes — valid only when placed immediately after `airwallex`.)

**Global flag placement:** Global flags must go immediately after `airwallex`, before the resource/action. This applies to `--compact`, `--dry-run`, and `--confirm` (plus `--tree` for discovery). Correct: `airwallex --compact invoices list`, `airwallex --dry-run invoices create --data-file payload.json`, `airwallex --confirm invoices finalize <id>`. Wrong: `airwallex invoices list --compact`, `airwallex invoices create --dry-run --data-file payload.json`, `airwallex invoices finalize <id> --confirm` — these fail with `No such option`.

This skill only covers Billing-domain CLI commands (`invoices`, `products`, `prices`, `billing-customers`, `subscriptions`, `coupons`, `credit-notes`, `billing-checkouts`, `billing-transactions`, `meters`, `payment-links`, `payment-sources`, `usage-events`). If the task requires a command group outside this domain (e.g., `cards`, `beneficiaries`, `conversions`, `balances`, `quotes`), **stop — this is the wrong skill.** Redirect the user:

- Wire transfers / payouts → not yet available (use Airwallex Dashboard)
- Setting up suppliers / beneficiaries → **beneficiary-creation** skill
- FX conversions, balances, treasury → **manage-cashflow** skill
- Provisioning corporate cards → **card-provisioning** skill
- Ad-hoc CLI tasks without a document → **awx-best-practices** skill (fallback)

## Prerequisites

- `airwallex` CLI installed and on PATH
- Authenticated via `airwallex auth login` (check with `airwallex auth whoami`)
- Environment: sandbox by default. For production, log in with `airwallex auth login --prod`. The environment is locked to the authenticated session — there is no per-command override.
- **Verify commands before use:** Run `airwallex --tree --compact <group>` (e.g., `airwallex --tree --compact invoices`) to confirm the subcommand exists. For write commands, run `airwallex <resource> <action> --api-schema-only` (e.g., `airwallex invoices create --api-schema-only`) to get the request body schema — read every `required: true` field and include them all in the payload. For read commands, use `airwallex <resource> <action> --help` (e.g., `airwallex invoices list --help`) to check flags.
- **`request_id` is MANDATORY for all write commands.** Always include `"request_id"` in the JSON body for every `create`, `update`, and `line-items add/update/delete` command — the API rejects writes without it. Generate a fresh UUID for each distinct operation via `uuidgen | tr '[:upper:]' '[:lower:]'` — NEVER hand-write a UUID or use sequential/patterned values like `a1b2c3d4-...`. If retrying the same logical operation after a transient/network failure, reuse the same `request_id`; only generate a new one for a distinct new operation. Action commands without a body (e.g., `invoices finalize`, `invoices void`) do not take `request_id`.

---

## Non-negotiables

### Terminology

- **Invoices = receivables (money in).** Issued BY the user TO their customers. Never say "obligation" for invoices.
- **Products & prices.** Every invoice line item needs a product. For document-specific ad-hoc fees (shipping, handling, setup fees, tax), always use the **inline price mechanism** (see Path B in Workflow) with a newly created product rather than matching existing fee products — fee amounts and descriptions vary per order. Only reuse existing products for the core goods/services sold (e.g., "Widget Alpha").
- **Invoice vs Subscription.** One-time quote → Invoice. Recurring terms → Subscription. Choose before creating.
- **ONE_OFF vs RECURRING.** Baked in at price creation — cannot flip later.
- **`billing_type`** only accepts `IN_ADVANCE` or `IN_ARREARS` (or omit it entirely).
- **`collection_method` mapping from document language:**

| Document says | API value |
| --- | --- |
| "send invoice", "bank transfer", "wire transfer", "offline payment", "pay by bank" | `OUT_OF_BAND` |
| "online payment", "checkout", "payment link", "pay online" | `CHARGE_ON_CHECKOUT` |
| "auto-debit", "direct debit", "auto-charge" | `AUTO_CHARGE` |

Never use `SEND_INVOICE`, `MANUAL`, `AUTOMATIC`, or any value not in the exact list: `AUTO_CHARGE`, `CHARGE_ON_CHECKOUT`, `OUT_OF_BAND`. Always ask the user if the document language is ambiguous.

### Operational rules

- **For ambiguous-intent requests, do not start the workflow until the action is confirmed.** If the user has not clearly confirmed the exact write action, stop before schema reads, auth checks, or other workflow setup that materially advances execution.
- **NEVER fabricate or assume missing information.** If any required field is uncertain, absent, or ambiguous — STOP and ask the user. Keep asking until you have every parameter needed to make the API call. This applies to amounts, currencies, customer details, payment terms, collection method, and any other field. Do NOT fill in defaults, placeholder values, or "reasonable guesses".
- Flag extraction uncertainty with `[?]` — never guess currencies, quantities, or amounts.
- Never round or modify extracted amounts.
- **Always fetch fresh data** — re-fetch before every step.
- **Prefer business labels over raw IDs in user-facing output.** In summaries, tables, and explanations, show customer names, product names, or other human-readable business labels instead of raw system IDs whenever possible. Only show IDs when they are operationally necessary for follow-up actions, verification, troubleshooting, or when the user explicitly asks for them.
- **One wallet, multiple currencies.** Say "AUD balance" — never "AUD wallet."
- **Default to sandbox.** Confirm with user before any production write.
- Show extracted data in five tables and get user confirmation **before any API call**.
- Search for existing customers and core products/prices before creating — avoid duplicates. (Ad-hoc fee products like shipping/handling are exempt — see Terminology.)
- Always confirm before finalizing — finalization is **irreversible**.
- Prices must match target: **ONE_OFF** for invoice line items, **RECURRING** for subscription items.
- **Infer collection method from the document** using the mapping table in Terminology above. If the document clearly implies a method (e.g., "bank transfer" → `OUT_OF_BAND`, "online payment" → `CHARGE_ON_CHECKOUT`), use it and note the choice in the extraction summary. **Ask the user when the document language is genuinely ambiguous or absent.** If the inferred method is `CHARGE_ON_CHECKOUT`, ask for `linked_payment_account_id` — without it the invoice has no checkout link and is unusable.
- **Never claim external payment-gateway setup.** External checkout routing, gateway configuration, and non-Airwallex collection integrations are outside this skill unless the exact capability is explicitly confirmed by current docs and schemas. This skill creates Airwallex Billing resources only.
- **Split bundled requests cleanly — unsupported extras must not block the supported workflow.** If the user asks for an Airwallex invoice/subscription plus an unsupported extra (e.g., Stripe gateway, external payment processor), **complete the Airwallex Billing setup first** without waiting for the unsupported portion to be resolved. After the supported work is done, separately state what was not configured and why.
- **Never invent billing automation fields.** If dunning, reminder cadence, external collection, or downstream automation support is unconfirmed, say so plainly and omit guessed JSON fields.
- **Dry-run before every write.** Before executing any write command (POST), first run it with the `--dry-run` global flag to preview the request envelope without sending it. Show the envelope to the user (method, URL, body, environment) and get explicit approval. Only then re-run with `--confirm` to execute. The two-step sequence is: (1) `airwallex --dry-run <command>` → show preview → user approves → (2) `airwallex --confirm <command>` → execute. **Never skip the dry-run step for write operations.** This applies to ALL writes — including action commands like `invoices finalize`, `invoices void`, `invoices delete`, and `invoices mark-as-paid`. Without `--confirm`, these commands are blocked with `SAFETY_BLOCK`.
- **Positional IDs, not `--id`.** E.g., `invoices get <ID>`, `products get <ID>`. Never pass `--id` as a flag.
- Commands with an object body (`create`, `update`, `line-items add/update/delete`) use `--data-file`, `--data`, or `--data-stdin`. Action commands (`invoices finalize`, `invoices void`, `invoices delete`, `invoices mark-as-paid`) take only positional IDs — no body flags, but **still require `--confirm`** (or `--dry-run` for preview). Example: `airwallex --confirm invoices finalize <id>`. Exception: `credit-notes finalize` does accept body flags. Always check the schema.
- **JSON output is the default.** Use `--compact` only if you need single-line JSON output, and place it immediately after `airwallex`: `airwallex --compact invoices list`. Do NOT put it after the action (`airwallex invoices list --compact`).

### Invoice & subscription constraints

- **`legal_entity_id`** — **Before creating any invoice or subscription**, ask the user: *"Does your account have multiple legal entities? If so, please provide the `legal_entity_id` (available in the Airwallex Dashboard)."* If the account has multiple legal entities and this field is omitted, the API rejects with `"Need to specify the legal_entity_id in the request"`. This ID is **not discoverable via API**. If the user confirms only one legal entity, omit the field.
- **`collection_method` MUST be set BEFORE finalize** — set at create time or via `invoices update`. See Terminology above for exact valid values.
- **`invoices create` has no `--amount` or `--customer-name`** — amounts go in line items, customer is `billing_customer_id` in JSON body, notes go in `memo`. There is no `description` field on invoices.
- **Pick `due_at` OR `days_until_due`** — passing both is rejected.
- **Timestamp format:** `+0000` offset everywhere (e.g., `2026-05-15T00:00:00+0000`). `Z` suffix and bare dates rejected. Applies to `due_at`, `starts_at`, etc.
- **`line-items add` / `line-items update` body must be `{"line_items": [...]}`** — bare array rejected. `line-items delete` body is `{"line_item_ids": ["id1","id2"]}`.
- **Line items only accept ONE_OFF + PER_UNIT/FLAT prices** — RECURRING, VOLUME, GRADUATED rejected. If the contract has tiered pricing, split into separate PER_UNIT line items per band.
- **Inline price objects do NOT include `currency`** — inherited from invoice.
- **Invoice must have line items before finalize** — use `invoices line-items add` on the draft to add items. Do NOT rely on passing `invoice_items` in the `invoices create` body — it may be silently ignored, resulting in a draft with `total_amount: 0` that fails to finalize. Always verify the draft has items (check `total_amount` > 0) before finalizing.
- **Discounts:** Use coupons (`"discounts": [{"type": "COUPON", "coupon": {"id": "..."}}]`), not negative `flat_amount`. For credits, use `credit-notes create`.
- **Coupons** — command group is `coupons` (NOT `discounts`, `promo-codes`, or `vouchers`). Required fields: `name`, `discount_model` (`FLAT` or `PERCENTAGE`), `duration_type` (`ONCE`/`CUSTOM`/`INDEFINITELY`), `request_id`. PERCENTAGE: set `percentage_off` (0–100) — do NOT include `currency` or `amount_off`. FLAT: set `amount_off` + `currency`. `duration_type: CUSTOM` needs `duration` object: `{"period": N, "period_unit": "MONTH"}`. Apply to invoices/subscriptions via `"discounts": [{"type": "COUPON", "coupon": {"id": "..."}}]`.
- **`metadata`** replaces entirely on update — omit to keep existing.
- **Tiered pricing** uses `upper_bound` (not `up_to`). Last tier omits `upper_bound` entirely.
- **`starts_at`** (subscriptions) must be strictly future. Compute dynamically — never hardcode. Omit to default to "now".
- **Subscription `items[*].price_id`** must reference RECURRING prices — ONE_OFF rejected.
- **`AUTO_CHARGE`** requires `payment_source_id`.
- **Recurring prices** need full `recurring` object: `{"interval": 1, "period_unit": "MONTH"}`. Missing `period_unit` causes validation error. `period_unit`: `DAY`/`WEEK`/`MONTH`/`YEAR`.
- **`products create`** body: `{"name", "description", "unit"}` only — no address.
- **`billing-customers create`** requires `address.country_code` (ISO-3166 alpha-2).
- **Tax handling** — Airwallex Billing does not have a built-in tax-rate engine. If the document includes GST, VAT, or sales tax, extract the tax amount and create it as an explicit line item (with a dedicated "Tax" or "GST" product) or include the tax-inclusive amount in the unit price. Flag `[?]` if it is unclear whether document prices are tax-inclusive or exclusive, and ask the user. Do NOT silently compute tax.
- **External payment gateways and custom dunning** — see operational rules above for scope boundaries. Do NOT claim third-party gateway was configured or invent dunning fields (`dunning.enabled`, `dunning.reminders`, `days_after_due`, etc.).
- **Subcommand naming:** `invoices line-items add` / `credit-notes line-items add` use nested `line-items` group; `subscriptions items list` / `subscriptions items get` also use a nested `items` group. Always check schema.
- **Pagination:** Most billing commands use cursor-based `--page` + `--page-size` (minimum 10; response has `page_after` — pass to `--page`, repeat until absent). Exception: `payment-links list` uses `--page-num` (0-based) + `--page-size` (minimum 10). Always check `--help` for the exact pagination style of each command.

---

## Workflow

### Phase 1: Extract

**Step 1 — Get the document.** Accept local files (`.pdf` `.docx` `.txt` `.md` `.png` `.jpg` `.webp`), folder paths, download URLs (via `curl`), or pasted text.

Reading strategy: PDF → Read tool (fall back to `pdfplumber`). DOCX → `python-docx`. Image → Read tool. TXT/MD → Read tool.

If the host/model cannot read a PDF, image, or other attachment reliably, do **not** pretend the document was read. Ask the user to re-upload, paste the relevant text, or provide a readable format before extracting billing details.

**Step 2 — Extract billing details.** Read the **entire** document. Distinguish between:
- **Product catalog** — all products/pricing defined in the contract
- **Current order** — specific items being billed now

Identify: customer (name, address, email), document reference, currency, payment terms, all products and pricing, fees, subscription terms if applicable.

**Step 3 — Build five tables:**

Always use structured tables, not prose summaries. If a section does not apply, still show the table and mark it `N/A`.

1. **Products** — name, description, unit, active, in current order?
2. **Prices** — product, unit price, currency, frequency, pricing model, tiers
3. **Customers** — name, email (required), location
4. **Subscriptions** — only if document has recurring terms (N/A otherwise)
5. **Invoices & Fees** — shipping, setup fees, tax (GST/VAT/sales tax rate and whether prices are tax-inclusive or exclusive), reference, payment terms, due date

**Step 4 — Validate and confirm.** Cross-check against API requirements. List all gaps explicitly. Accept natural-language corrections and loop until user approves. Do NOT proceed to any create/finalize call until every required field is complete and the user confirms.

### Phase 2: Match & Create

**Step 5 — Pre-flight checks.** Run `auth whoami`. If it fails (no active session), ask the user which environment (sandbox/production). Once the user answers, **immediately execute** `airwallex auth login` (or `airwallex auth login --prod` for production) yourself — do NOT tell the user to run it manually. The command triggers a browser-based OAuth flow; the user completes sign-in in their browser. After the command returns, confirm the session with `auth whoami`. If it succeeds on the first check, tell the user the current environment. Validate required fields for all planned operations. No `auth refresh` command — on 401, retry the real command first (auto-refresh); if retry also fails, ask environment and **execute `auth login` yourself**.

**Step 6 — Match existing resources.** Search for customers by email (`billing-customers list --email`). For products, `products list` has **no name filter** — paginate fully (`page_after` → next page until absent) and match by name client-side. Search prices by product (`prices list --product-id`). Present matches and get confirmation. **Only match core goods/services** (e.g., "Widget Alpha"). Do NOT match existing products for per-order ad-hoc fees (shipping, handling, setup charges) — always create fresh products for these and use inline prices in line items.

**Step 7 — Create missing resources.** Create ALL missing products and needed prices from the contract (full catalog), not just current order.

**Price type depends on Table 4:** N/A → ONE_OFF. Has subscription data → RECURRING.

**Pricing model depends on contract:** FLAT (fixed amount), PER_UNIT (per-seat), VOLUME/GRADUATED (tiered).

**Step 7b — Confirm collection method.** Use the method inferred from the document (see operational rules). If the document was ambiguous or silent, ask the user now. If `CHARGE_ON_CHECKOUT`, require `linked_payment_account_id` from the user or Airwallex Dashboard context — do not guess or invent one.

**Step 8 — Route based on Table 4:**

The path comes from the extracted document terms, not from user preference.
Requests for external payment gateways, or custom dunning do **not** change the billing route. Create the Airwallex invoice/subscription that is supported, then clearly state any unsupported extra was not configured.

| Table 4 | Action |
| --- | --- |
| Has subscription data | → Create subscription |
| N/A (one-time) | → Create invoice |
| Both | → Subscription for recurring + invoice for one-time fees |

#### Path A: Subscription

Create subscription with `billing_customer_id`, `currency`, `collection_method`, `items` (RECURRING prices). Verify after creation.

**Receivables note** — after creation, remind the user that this subscription will generate recurring invoices (expected money in). If they use the **manage-cashflow** skill, upcoming subscription invoices will appear in their receivables picture. Mention the subscription ID, currency, billing interval, and next billing date so they can cross-reference.

#### Path B: Invoice

**Create draft** with `billing_customer_id`, `currency`, `collection_method`, optional `days_until_due`/`due_at`, `memo`.

**Add line items** — body: `{"line_items": [{"price_id": "...", "quantity": N}, ...]}`. For ad-hoc fees, use inline price: `{"price": {"product_id": "...", "pricing_model": "FLAT", "flat_amount": 350.00, "description": "Shipping"}, "quantity": 1}`. Do NOT include `currency` in inline price. If the contract uses tiered pricing, convert the billed tiers into invoice-compatible PER_UNIT/FLAT line items per band — do not attach VOLUME/GRADUATED `price_id`s to invoice line items.

**Finalize** — confirm with user first. Show summary: resources created/reused, total, due date, `pdf_url`, `hosted_url`.

**Receivables note** — after finalize, remind the user that this invoice is now a receivable (expected money in). If they use the **manage-cashflow** skill, this finalized invoice will appear in their receivables/obligations picture. Mention the invoice ID, amount, currency, and due date so they can cross-reference. When multiple invoices are created in a batch, also show a one-line aggregate: total outstanding amount per currency (e.g., "3 invoices totalling 47,200 GBP in outstanding receivables") so the user can immediately see the cash-position impact without switching to manage-cashflow.

### Phase 3: Share with Customer (opt-in only)

Must be explicitly requested. After finalize, present:
- **`pdf_url`** — direct link to the PDF version of the invoice (always available on finalized invoices).
- **`hosted_url`** — online checkout page for the customer to pay. Only available when `collection_method` is `CHARGE_ON_CHECKOUT`. For `OUT_OF_BAND` invoices, `hosted_url` may be absent — share the `pdf_url` instead and instruct the customer on bank transfer details.

Airwallex Billing has **no API to email invoices directly** — the agent cannot send on behalf of the user. Offer to draft a payment email the user can copy and send themselves (e.g., via their email client or Slack).

---

## Error handling

| Situation | Action |
| --- | --- |
| Document unreadable | Ask for content another way and stop |
| Required field missing | List gaps, stop, ask user |
| Ambiguous extraction | Flag with `[?]`, ask user, do not guess |
| API error | Stop, show full error, ask user |
| `legal_entity_id` required (missed pre-check) | The account has multiple legal entities. Ask the user which `legal_entity_id` to use — this is not discoverable via API. Include it in the request body and retry. |
| User requests external gateway setup | Explain that external gateway configuration is outside this skill's scope. Continue with Airwallex Billing setup only if the user still wants that. |
| User requests custom dunning cadence not confirmed by docs | Do not invent fields. Say the reminder customization is unsupported or handled separately, and continue with the supported invoice/subscription setup only. |
| Partial completion | Report what succeeded and failed with IDs, then ask user how to proceed |
| Duplicate detected | Show details, let user choose |
| 401 / auth expired | Retry command (auto-refresh). If retry also fails: ask user which environment (sandbox/production), **immediately execute** `auth login` or `auth login --prod` yourself (do NOT tell user to run it), confirm with `auth whoami`, then resume |
| API error | Run `airwallex <resource> <action> --api-schema-only` (e.g., `airwallex invoices create --api-schema-only`) to verify body structure |

---

## Workflow summary

```
Phase 1: Extract
  get document → read → extract → build 5 tables → validate → user confirms

Phase 2: Match & Create
  environment + auth → match existing → user confirms
    → create missing → confirm collection method
      → if recurring: subscription → if one-time: invoice → finalize
      → receivables note (cross-ref with manage-cashflow)

Phase 3: Share (opt-in)
  present URLs (pdf_url + hosted_url) → draft email if requested
```
