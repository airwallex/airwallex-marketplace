---
name: awx-best-practices
description: Fallback Airwallex skill — works with the Airwallex CLI or the Airwallex MCP server. Use ONLY when no dedicated workflow skill matches the task. Covers ad-hoc operations (list, get, update, delete, void, cancel), general Airwallex API questions, troubleshooting, and domains not covered by a workflow skill (payment links, refunds, disputes, spend management, financial reports). Do NOT load this skill alongside a workflow skill — each workflow skill is self-contained. For invoices/billing/coupons/meters/credit notes use contract-to-billing, for suppliers/beneficiaries use beneficiary-creation, for cards use card-provisioning, for balances/FX/cashflow use manage-cashflow.
metadata:
  author: Airwallex
  version: 0.1.0
compatibility: Works with the Airwallex CLI (`airwallex` binary on PATH, authenticated via `airwallex auth login`) or the Airwallex MCP server (enabled and authenticated).
---

# Airwallex Best Practices

Fallback skill for Airwallex tasks. **Use only when no dedicated workflow skill fits.** Each workflow skill (beneficiary-creation, card-provisioning, contract-to-billing, manage-cashflow) is self-contained — do NOT load this skill alongside them.

## When to use

- Ad-hoc operations (list, get, update, delete, void, cancel, deactivate)
- General Airwallex API questions or troubleshooting
- Payment links, refunds, disputes, spend management, financial reports, and other domains not covered by a dedicated workflow skill

## Use a workflow skill instead

| Task | Skill |
| --- | --- |
| Create invoice from PO/contract/quote | **contract-to-billing** |
| Coupons, meters, usage events, credit notes | **contract-to-billing** |
| Onboard suppliers / create beneficiaries | **beneficiary-creation** |
| Provision corporate cards | **card-provisioning** |
| Cash position, FX, balances, rebalancing | **manage-cashflow** |

If a workflow skill matches, use that skill instead — do NOT load this one alongside it.

## Out of scope — refuse and redirect

Do NOT attempt to fulfill these by aggregating API calls. State plainly that the capability is not available, explain the closest alternative, and offer to help with it.

| Request pattern | Why out of scope | Redirect |
| --- | --- | --- |
| "Transaction report", "all transactions this month", "ledger export" | No skill produces accounting-grade transaction reports | Offer **manage-cashflow** for cash position, receivables, obligations |
| "Reconciliation", "P&L", "balance sheet", "accounting report" | Accounting functions outside agent capability | Same as above |
| "Forecast", "hedging strategy", "FX prediction" | Agent provides indicative spot rates only | Offer **manage-cashflow** for current exposure and indicative FX rates |
| "Yield", "investment advice", "idle funds", "automated top-up", "should I lock a rate?" | Financial advice or unsupported treasury action | Offer **manage-cashflow** for current balances, obligations, and informational indicative FX only |

---

## Environment quickstart

See [references/surface-quickstart.md](references/surface-quickstart.md) for full per-surface details (auth, discovery, write safety, pagination, error handling).

---

## Operational rules

- **NEVER fabricate or assume missing information.** If any required field is uncertain — STOP and ask the user.
- **Always fetch fresh data** — re-fetch before every step.
- **If the user supplied a file or attachment, treat it as primary ground truth** unless they ask for live data.
- **For ambiguous-intent requests, confirm the action before starting.**
- **Never overclaim unsupported capabilities.** Transfers, payouts, FX execution, PAN/CVV retrieval, accounting reports, reconciliation: refuse immediately, state what is not available, offer the closest alternative.
- **Do not provide financial advice.** Never recommend yield, investment products, automated top-ups, hedging strategy, FX prediction, or rate-locking. For treasury questions, redirect to **manage-cashflow** and keep any FX discussion informational and clearly labelled as indicative.
- **Split supported and unsupported asks.** Complete the supported portion and clearly state what was not configured.
- **Prefer business labels over raw IDs in user-facing output.** Show human-readable business labels (customer names, product names, beneficiary names, card nicknames, etc.) instead of raw system IDs whenever possible. Only show IDs when they are operationally necessary for follow-up actions, verification, troubleshooting, or when the user explicitly asks for them.

## Core Airwallex concepts

- **One wallet, multiple currencies.** Say "AUD balance" — never "AUD wallet."
- **Invoices = receivables (money in).** Issued BY the user TO their customers.
- **Bills = payables (money out).** Money the user owes to suppliers.
- **FX conversions happen within one wallet** — not between wallets.

---

## Consequential operations

These are **irreversible or high-impact**. Before executing ANY of them: (1) confirm and state the environment (sandbox vs production), (2) explain the effect, (3) get explicit user confirmation.

| Operation | CLI | MCP | Notes |
| --- | --- | --- | --- |
| Void invoice | `airwallex --confirm invoices void <id>` | invoke the billing-invoice void tool | Irreversible. FINALIZED+UNPAID only. |
| Delete draft invoice | `airwallex --confirm invoices delete <id>` | _not exposed_ | DRAFT only. |
| Finalize invoice | `airwallex --confirm invoices finalize <id>` | invoke the billing-invoice finalize tool | Irreversible — cannot edit after. |
| Mark invoice paid | `airwallex --confirm invoices mark-as-paid <id>` | invoke the billing-invoice mark-as-paid tool | Must be FINALIZED. Use when paid outside Airwallex. |
| Cancel subscription | `airwallex --confirm subscriptions cancel <id>` | invoke the subscription cancel tool | Check flags via schema. |
| Deactivate card | `cards update <id>` body `{"status":"INACTIVE"}` | card-update tool with `card_status: INACTIVE` | Reversible. See [references/api_traps.md](references/api_traps.md) for which statuses are NOT settable here. |
| Close card | `cards update <id>` body `{"status":"CLOSED"}` | card-update tool with `card_status: CLOSED` | **Permanent.** |
| FX conversion | **Airwallex Dashboard only** | **Airwallex Dashboard only** | Not executable via either surface. |
| Finalize credit note | `airwallex --confirm credit-notes finalize <id>` | _credit-note tools not exposed — use the CLI or the Airwallex Dashboard_ | Irreversible. |
| Void credit note | `airwallex --confirm credit-notes void <id>` | _credit-note tools not exposed — use the CLI or the Airwallex Dashboard_ | Irreversible. |
| Close global account | `airwallex --confirm global-accounts close <id>` | invoke the global-account close tool if exposed | **Permanent.** |
| Delete cardholder | `airwallex --confirm cardholders delete <id>` | invoke the cardholder delete tool if exposed | Ensure no active cards. |

---

## Error handling

| Situation | Action |
| --- | --- |
| Required field missing or ambiguous | STOP, list gaps, ask user |
| API error | Show full error, ask user |
| API validation error | Check [references/api_traps.md](references/api_traps.md). **IF** using the CLI, **THEN** also run `airwallex <resource> <action> --api-schema-only` to verify body structure. **ELSE IF** using the MCP server, **THEN** re-inspect the tool's input schema. |
| 401 / auth expired | **IF** using the CLI, **THEN** retry once (auto-refresh). If retry fails, ask the user which environment, immediately execute `auth login` (or `--prod`) yourself, confirm with `auth whoami`, then resume. **ELSE IF** using the MCP server, **THEN** the server refreshes tokens automatically; if a tool keeps returning 401, the OAuth grant has been revoked — ask the user to re-authorize the MCP server. |
| Duplicate detected | Show details, let user choose |
| Partial completion | Report what succeeded (with IDs) and what failed |

## References

- [references/api_traps.md](references/api_traps.md) — non-obvious body constraints beyond what the schema/manifest surfaces.
- **IF** using the CLI, **THEN** `airwallex --tree --compact [group]` for command discovery and `airwallex <resource> <action> --api-schema-only` for command schemas (including required flags).
- [Airwallex API Introduction](https://www.airwallex.com/docs/api/introduction)
