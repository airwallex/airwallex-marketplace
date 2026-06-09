---
name: contract-to-billing
description: Extract billing details from purchase orders, contracts, or quotes, then set up Airwallex Billing by creating invoices and/or subscriptions — matching existing customers, products, and prices to avoid duplicates. Use when the user says "create invoice from this PO", "set up billing from this contract", "create a subscription from this agreement", "invoice this quote", "bill this customer", or attaches a document and asks to set up one-time, recurring, or mixed billing. Do NOT use for paying suppliers, provisioning cards, or checking FX rates.
metadata:
  author: Airwallex
  version: 0.1.0
compatibility: Works with the Airwallex CLI (`airwallex` binary on PATH, authenticated via `airwallex auth login`) or the Airwallex MCP server (enabled and authenticated).
---

# Contract to Billing

Reads a customer document (PO, contract, quote), extracts line items with AI, and creates a fully populated invoice in Airwallex Billing.

## When to use

- User uploads or references a purchase order, contract, quote, or billing document
- User asks to "create an invoice" from a document
- User wants to extract billing details and set up products/prices/customers
- User says "bill this customer" with a document attached

## When NOT to use

This skill only covers Billing-domain operations — invoices (list, create, retrieve, finalize, void, mark-as-paid, plus line-item add / update / delete / list), products (list, create), prices (list, create), customers (list, retrieve, create, update), subscriptions (create, list items), coupons (list, create, update), meters (list, create, update), payment sources (list), and billing transactions (list, retrieve). Credit notes follow `create` → `line-items add` → `finalize` — if the operation is not exposed on the current surface, direct the user to the Airwallex Dashboard.

If the task requires capabilities outside this domain, **stop — this is the wrong skill.** Redirect the user:

- Wire transfers / payouts → not yet available (use Airwallex Dashboard)
- Setting up suppliers / beneficiaries → **beneficiary-creation** skill
- FX conversions, balances, treasury → **manage-cashflow** skill
- Provisioning corporate cards → **card-provisioning** skill
- Ad-hoc tasks outside billing workflow → **awx-best-practices** skill (fallback)

## Non-negotiables

### Terminology

- **Invoices = receivables (money in).** Issued BY the user TO their customers. Never say "obligation" for invoices.
- **Invoice lifecycle.** **DRAFT → add line items → finalize → FINALIZED (immutable)**. To correct after finalize: void → create new.
- **Products & prices.** Every invoice line item needs a product. For document-specific ad-hoc fees (shipping, handling, setup fees, tax), always use the **inline price mechanism** (see Path B in Workflow) with a newly created product rather than matching existing fee products — fee amounts and descriptions vary per order. Only reuse existing products for the core goods/services sold (e.g., "Widget Alpha").
- **Invoice vs Subscription.** One-time quote → Invoice. Recurring terms → Subscription. Choose before creating.
- **ONE_OFF vs RECURRING.** Baked in at price creation — cannot flip later.
- **`collection_method` mapping from document language:**

| Document says | API value |
| --- | --- |
| "send invoice", "bank transfer", "wire transfer", "offline payment", "pay by bank" | `OUT_OF_BAND` |
| "online payment", "checkout", "payment link", "pay online" | `CHARGE_ON_CHECKOUT` |
| "auto-debit", "direct debit", "auto-charge" | `AUTO_CHARGE` |

Never use `SEND_INVOICE`, `MANUAL`, `AUTOMATIC`, or any value not in the exact list: `AUTO_CHARGE`, `CHARGE_ON_CHECKOUT`, `OUT_OF_BAND`. Always ask the user if the document language is ambiguous.

### Operational rules

- **For ambiguous-intent requests, do not start the workflow until the action is confirmed.** If the user has not clearly confirmed the exact write action, stop before schema reads, auth checks, or other workflow setup that materially advances execution.
- **NEVER fabricate or assume missing information.** If any required field is uncertain, absent, or ambiguous — STOP and ask the user. Keep asking until you have every parameter needed. Do NOT fill in defaults, placeholder values, or "reasonable guesses".
- **Flag extraction uncertainty with `[?]`** — never guess currencies, quantities, or amounts.
- **Never round or modify extracted amounts.**
- **Always fetch fresh data** — re-fetch before every step.
- **Prefer business labels over raw IDs in user-facing output.** Show customer names and product names first; surface IDs only when operationally necessary or when the user asks.
- **One wallet, multiple currencies.** Say "AUD balance" — never "AUD wallet."
- **Default to sandbox.** Confirm with user before any production write.
- **Show extracted data in five tables and get user confirmation before any API call.**
- **Search for existing customers and core products/prices before creating** — avoid duplicates. (Ad-hoc fee products like shipping/handling are exempt — see Terminology.)
- **Always confirm before finalizing** — finalization is **irreversible**.
- **Prices must match target:** **ONE_OFF** for invoice line items, **RECURRING** for subscription items.
- **Infer collection method from the document** using the mapping table above. If the document clearly implies a method (e.g., "bank transfer" → `OUT_OF_BAND`, "online payment" → `CHARGE_ON_CHECKOUT`), use it and note the choice in the extraction summary. Ask the user when the document language is genuinely ambiguous or absent. If the inferred method is `CHARGE_ON_CHECKOUT`, ask for `linked_payment_account_id` — without it the invoice has no checkout link and is unusable.
- **Never claim external payment-gateway setup.** External checkout routing, gateway configuration, and non-Airwallex collection integrations are outside this skill unless the exact capability is explicitly confirmed by current docs and schemas. This skill creates Airwallex Billing resources only.
- **Split bundled requests cleanly — unsupported extras must not block the supported workflow.** If the user asks for an Airwallex invoice/subscription plus an unsupported extra (e.g., Stripe gateway, external payment processor), complete the Airwallex Billing setup first. After the supported work is done, separately state what was not configured and why.
- **Never invent billing automation fields.** If dunning, reminder cadence, external collection, or downstream automation support is unconfirmed, say so plainly and omit guessed JSON fields.
- **Write safety.** Show the full payload to the user and get confirmation before every write — invoice create / line-item write / finalize / void / mark-paid / subscription create. **Action commands** (`invoices finalize`, `invoices void`, `invoices delete`, `invoices mark-as-paid`) need the same confirmation as create/update — they are not free passes.

### Invoice & subscription constraints

- **`legal_entity_id`** — **Before creating any invoice or subscription**, ask the user: *"Does your account have multiple legal entities? If so, please provide the `legal_entity_id` (available in the Airwallex Dashboard)."* If the account has multiple legal entities and this field is omitted, the API rejects with `"Need to specify the legal_entity_id in the request"`. This ID is **not discoverable via API**. If the user confirms only one legal entity, omit the field.
- **`collection_method` MUST be set BEFORE finalize** — set at create time or via the invoice update operation. See Terminology above for exact valid values.
- **Invoice body shape.** Amounts live in line items (no top-level amount field); customer is `billing_customer_id`; notes go in `memo`. There is no `description` field on invoices.
- **Amounts come from line items, not the invoice create body** — `invoice_items` in the create body is **silently ignored**. Use the dedicated line-items operation after the draft exists.
- **Pick `due_at` OR `days_until_due`** — passing both is rejected.
- **Timestamp format:** include an explicit timezone — bare dates are rejected. Most body fields accept both `+0000` and `Z`. **Tested exception:** the billing-invoice listing only accepts `+0000` — `Z` returns a 400. Both surfaces URL-encode for you — do NOT pre-encode the offset to `%2B0000`.
- **Line items body shape.** Add / update wraps the array in `line_items`; delete uses `line_item_ids`. Bare arrays are rejected.
- **Line items only accept ONE_OFF + PER_UNIT/FLAT prices** — RECURRING, VOLUME, GRADUATED are rejected. If the contract has tiered pricing, split into separate PER_UNIT line items per band.
- **Inline price objects do NOT include `currency`** — inherited from invoice.
- **Verify the draft has items (`total_amount > 0`) before finalize.** A draft with no line items will fail to finalize — and `invoice_items` in the invoice create body is silently dropped, so this state is easy to land in by accident.
- **Discounts:** Use coupons (`"discounts": [{"type": "COUPON", "coupon": {"id": "..."}}]`), not negative `flat_amount`. **Credit notes:** lifecycle is `create` → `line-items add` → `finalize`; if the operation is not exposed on the current surface, direct the user to the Airwallex Dashboard.
- **Coupons:** PERCENTAGE vs FLAT are mutually exclusive. PERCENTAGE → `percentage_off` (0–100), no `currency`/`amount_off`. FLAT → `amount_off` + `currency`, no `percentage_off`. `duration_type: CUSTOM` needs a `duration` object with both `period` and `period_unit`.
- **`metadata`** replaces entirely on update — omit to keep existing.
- **Tiered pricing** uses `upper_bound` (not `up_to`); the last tier omits `upper_bound` entirely.
- **`starts_at`** (subscriptions) must be strictly future. Compute dynamically — never hardcode. Omit to default to "now".
- **Subscription `items[*].price_id`** must reference RECURRING prices — ONE_OFF rejected.
- **`AUTO_CHARGE`** requires `payment_source_id` — ask the user.
- **Tax handling** — Airwallex Billing does not have a built-in tax-rate engine. If the document includes GST, VAT, or sales tax, extract the tax amount and create it as an explicit line item (with a dedicated "Tax" or "GST" product) or include the tax-inclusive amount in the unit price. Flag `[?]` if it is unclear whether document prices are tax-inclusive or exclusive, and ask the user. Do NOT silently compute tax.
- **External payment gateways and custom dunning** — see operational rules above. Do NOT claim a third-party gateway was configured or invent dunning fields (`dunning.enabled`, `dunning.reminders`, `days_after_due`, etc.).
- **Pagination:** `page_size` minimum 10 across billing list endpoints. Most billing listings use cursor pagination — pass `page_after` back into the next call. The `payment-links list` endpoint is an exception (numeric `page_num`, 0-based). Repeat until cursor absent.

---

## Workflow

### Phase 1: Extract

**Step 1 — Get the document.** Accept local files (`.pdf` `.docx` `.txt` `.md` `.png` `.jpg` `.webp`), folder paths, or pasted text. If shell access is available, you can also fetch download URLs.

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

**Step 5 — Confirm environment.** State sandbox vs production to the user (sandbox default; production only on explicit confirmation). Validate required fields for all planned operations. Include `request_id` (UUIDv4) on every write that accepts it.

**Step 6 — Match existing resources.** Search for customers by email (filter parameter `email` on the billing-customers list). For products, the billing-products list has **no name filter** — paginate fully and match by name client-side. Search prices by product (parameter `product_id` on the billing-prices list). Present matches and get confirmation.

**Only match core goods/services** (e.g., "Widget Alpha"). Do NOT match existing products for per-order ad-hoc fees (shipping, handling, setup charges) — always create fresh products for these and use inline prices in line items.

**Step 7 — Create missing resources.** Create ALL missing products and needed prices from the contract (full catalog), not just current order.

**Price type depends on Table 4:** N/A → ONE_OFF. Has subscription data → RECURRING.

**Pricing model depends on contract:** FLAT (fixed amount), PER_UNIT (per-seat), VOLUME/GRADUATED (tiered).

**Step 7b — Confirm collection method.** Use the method inferred from the document (see operational rules). If the document was ambiguous or silent, ask the user now. If `CHARGE_ON_CHECKOUT`, require `linked_payment_account_id` from the user or Airwallex Dashboard context — do not guess or invent one.

**Step 8 — Route based on Table 4:**

The path comes from the extracted document terms, not from user preference. Requests for external payment gateways or custom dunning do **not** change the billing route. Create the Airwallex invoice/subscription that is supported, then clearly state any unsupported extra was not configured.

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

**Add line items.** Use the dedicated line-items operation after the draft exists. Body shape for an existing price: `{"price_id": "...", "quantity": N}`. For ad-hoc fees, use inline price: `{"price": {"product_id": "...", "pricing_model": "FLAT", "flat_amount": 350.00, "description": "Shipping"}, "quantity": 1}`. Do NOT include `currency` in inline price. If the contract uses tiered pricing, convert the billed tiers into invoice-compatible PER_UNIT/FLAT line items per band — do not attach VOLUME/GRADUATED `price_id`s to invoice line items.

**Finalize** — confirm with user first. Show summary: resources created/reused, total, due date, `pdf_url`, `hosted_url`.

**Receivables note** — after finalize, remind the user that this invoice is now a receivable (expected money in). If they use the **manage-cashflow** skill, this finalized invoice will appear in their receivables/obligations picture. Mention the invoice ID, amount, currency, and due date so they can cross-reference. When multiple invoices are created in a batch, also show a one-line aggregate: total outstanding amount per currency (e.g., "3 invoices totalling 47,200 GBP in outstanding receivables") so the user can immediately see the cash-position impact without switching to manage-cashflow.

### Phase 3: Share with Customer (opt-in only)

Must be explicitly requested. After finalize, present:
- **`pdf_url`** — direct link to the PDF version of the invoice (always available on finalized invoices).
- **`hosted_url`** — online checkout page for the customer to pay. Only available when `collection_method` is `CHARGE_ON_CHECKOUT`. For `OUT_OF_BAND` invoices, `hosted_url` may be absent — share the `pdf_url` instead and instruct the customer on bank transfer details.

Airwallex Billing has **no API to email invoices directly** — the agent cannot send on behalf of the user. Offer to draft a payment email the user can copy and send themselves (e.g., via their email client or Slack).

---

## Error handling

Generic patterns (401/auth, API validation, duplicates, partial writes, missing required fields) — see [awx-best-practices Error handling](../awx-best-practices/SKILL.md) and [api_traps.md](../awx-best-practices/references/api_traps.md).

Domain-specific:

| Situation | Action |
| --- | --- |
| Document unreadable | Ask for content another way and stop |
| Ambiguous extraction | Flag with `[?]`, ask user, do not guess |
| `legal_entity_id` required (missed pre-check) | Account has multiple legal entities. Ask the user which `legal_entity_id` to use — not discoverable via API. Include in the request body and retry. |
| User requests external gateway setup | Explain that external gateway configuration is outside this skill's scope. Continue with Airwallex Billing setup only if the user still wants that. |
| User requests custom dunning cadence not confirmed by docs | Do not invent fields. Say the reminder customization is unsupported or handled separately, and continue with the supported invoice/subscription setup only. |

---

## Workflow summary

```
Phase 1: Extract
  get document → read → extract → build 5 tables → validate → user confirms

Phase 2: Match & Create
  environment / account + auth → match existing → user confirms
    → create missing → confirm collection method
      → if recurring: subscription → if one-time: invoice → finalize
      → receivables note (cross-ref with manage-cashflow)

Phase 3: Share (opt-in)
  present URLs (pdf_url + hosted_url) → draft email if requested
```
