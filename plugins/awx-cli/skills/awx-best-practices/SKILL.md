---
name: awx-best-practices
description: Fallback Airwallex CLI skill — use ONLY when no dedicated workflow skill matches the task. Covers ad-hoc CLI operations (list, get, update, delete, void, cancel), general Airwallex API questions, troubleshooting, and domains not covered by a workflow skill (payment links, refunds, disputes, meters, usage events, coupons, credit notes, spend management, financial reports). Do NOT load this skill alongside a workflow skill — each workflow skill is self-contained. For invoices/billing use contract-to-billing, for suppliers/beneficiaries use beneficiary-creation, for cards use card-provisioning, for balances/FX/cashflow use manage-cashflow.
metadata:
  author: Airwallex
  version: 0.1.0
compatibility: Requires airwallex CLI installed and on PATH. Authenticated via airwallex auth login (OAuth browser flow). Best results with Claude Opus.
---

# Airwallex Best Practices

Fallback skill for Airwallex tasks. **Use only when no dedicated workflow skill fits.** Each workflow skill (beneficiary-creation, card-provisioning, contract-to-billing, manage-cashflow) is self-contained — do NOT load this skill alongside them.

## When to use

- Ad-hoc CLI/MCP operations (list, get, update, delete, void, cancel, deactivate)
- General Airwallex API questions or troubleshooting
- Payment links, refunds, disputes, meters, usage events, coupons, credit notes, spend management, financial reports, and other domains not covered by a dedicated workflow skill

## Use a workflow skill instead

| Task | Skill |
| --- | --- |
| Create invoice from PO/contract/quote | **contract-to-billing** |
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

## CLI essentials

- **Binary is `airwallex`** — not `awx`, `awx-cli`, or `@airwallex/cli`.
- **Body via `--data-file <path.json>`, `--data '<json>'`, or `--data-stdin`.** Only one per command. `--file` does not exist.
- **`request_id` is MANDATORY for write commands.** Always include `"request_id"` in the JSON body for every `create`, `update`, and `validate` command — the API rejects writes without it. Generate a fresh UUID for each distinct operation via `uuidgen | tr '[:upper:]' '[:lower:]'` — NEVER hand-write a UUID or use sequential/patterned values like `a1b2c3d4-...`. If retrying the same logical operation after a transient/network failure, reuse the same `request_id`; only generate a new one for a distinct new operation. One known exception: `beneficiaries verify` does NOT take `request_id`. Action commands without a body do not take `request_id`. When in doubt, run `--api-schema-only` to confirm.
- **Commands are plural and often prefixed** (`invoices`, `cards`, `beneficiaries`, `beneficiary-schemas`). Common gotcha: CLI commands use prefixed names — `payment-disputes` (not `disputes`), `payment-intents` (not `intents`), `payment-links` (not `links`), `payment-customers` (not `customers`), `billing-customers` (not `customers`), `issuing-transactions` (not `transactions`). Always run `--tree --compact` first — never guess command names.
- **Positional IDs, not `--id`.** E.g., `invoices get <ID>`, `cards get <ID>`.
- **Global flags go immediately after `airwallex`, before the resource/action.** `--compact` (single-line JSON), `--dry-run` (preview write), and `--confirm` (execute write) are global flags. Correct: `airwallex --compact invoices list`, `airwallex --dry-run cards create --data-file payload.json`, `airwallex --confirm invoices finalize <id>`. Wrong: `airwallex invoices list --compact`, `airwallex cards create --dry-run --data-file payload.json`, `airwallex invoices finalize <id> --confirm` — these fail with `No such option`.
- **Auth:** `auth login` (sandbox), `auth login --prod` (production), `auth logout`, `auth whoami`. No other auth subcommands.
- **File writes:** Write JSON payload files to the **current working directory**, not `/tmp`.
- **Pagination:** All paginated commands require `--page-size` of at least 10 (the API rejects lower values). Use 20 as a sensible default. Check `--help` for pagination style: `--page-num` (0-based numeric) or `--page` (cursor-based, pass `page_after` from response).

## Environment & safety

**Default to sandbox.** Confirm with user before any production write.

**Check environment:** Run `airwallex auth whoami` — the `mode` field shows `"sandbox"` or `"production"`. Do NOT fabricate commands (`airwallex config`, `airwallex env`, etc.) or read config files.

**Not logged in?** If `auth whoami` fails or a command returns 401 after retry: (1) ask the user which environment (sandbox/production), (2) **immediately execute** `auth login` or `auth login --prod` yourself — do NOT tell the user to run it manually (the command triggers a browser-based OAuth flow; the user completes sign-in in their browser), (3) confirm with `auth whoami`, then resume. No `auth refresh` command — on 401, retry the real command first (auto-refresh); if retry also fails, ask environment and **execute `auth login` yourself**.

**HARD RULE: `auth whoami` before any consequential operation.** Always tell the user which environment the command will target.

## Command discovery protocol

Before any CLI call, follow this three-step process. Do NOT rely on memory or training data.

### Step 1 — Identify available commands

Run `airwallex --tree --compact` for the full command tree, or `airwallex --tree --compact <group>` (e.g., `airwallex --tree --compact cards`) for a subtree. The output lists all non-hidden command groups and leaf commands. If a group does not appear in the tree output, it **does not exist** — do NOT invent it.

### Step 2 — Get flags and body schema

- **Read commands** (list, get): run `airwallex <group> <subcommand> --help` (e.g., `airwallex cards list --help`) for exact flags, required vs optional params, and pagination style (`--page` vs `--page-num`).
- **Write commands** (create, update, validate, verify): run `airwallex <resource> <action> --api-schema-only` (e.g., `airwallex cards create --api-schema-only`) to get the OpenAPI request body schema (per-property `required`, `type`, `format`, `enum`, `description`) and response body schema. Construct the JSON body following `openapi_request_body` and pass via `--data`, `--data-file`, or `--data-stdin`.
- Global flags (`--compact`, `--dry-run`, `--confirm`) are valid only in global position: `airwallex --compact <resource> <action>`, `airwallex --dry-run <resource> <action>`, `airwallex --confirm <resource> <action>`. Do not put them after the action, e.g. `airwallex <resource> <action> --compact`.

### Step 3 — Check non-obvious constraints before writes

**ALWAYS read [references/api_traps.md](references/api_traps.md) before your first write in a session.** It lists non-obvious constraints that the OpenAPI schema does not surface (e.g., `beneficiary-schemas generate` requiring `--confirm`, `is_personalized` mandatory on card create, SE/SEK schema lying about optional fields). Re-check it whenever you hit an unexpected validation error.

## Operational rules

- **NEVER fabricate or assume missing information.** If any required field is uncertain — STOP and ask the user.
- **Always fetch fresh data** — re-fetch before every step.
- **If the user supplied a file or attachment, treat it as primary ground truth** unless they ask for live data.
- **For ambiguous-intent requests, confirm the action before starting.**
- **Never overclaim unsupported capabilities.** Transfers, payouts, FX execution, PAN/CVV retrieval, accounting reports, reconciliation: refuse immediately, state what is not available, offer the closest alternative.
- **Do not provide financial advice.** Never recommend yield, investment products, automated top-ups, hedging strategy, FX prediction, or rate-locking. For treasury questions, redirect to **manage-cashflow** and keep any FX discussion informational and clearly labelled as indicative.
- **Split supported and unsupported asks.** Complete the supported portion and clearly state what was not configured.
- **Prefer business labels over raw IDs in user-facing output.** In summaries, tables, and explanations, show human-readable business labels (customer names, product names, beneficiary names, card nicknames, etc.) instead of raw system IDs whenever possible. Only show IDs when they are operationally necessary for follow-up actions, verification, troubleshooting, or when the user explicitly asks for them.
- **Dry-run before every write.** `airwallex --dry-run <command>` → show preview → user approves → `airwallex --confirm <command>`. Never skip.

## Core Airwallex concepts

- **One wallet, multiple currencies.** Say "AUD balance" — never "AUD wallet."
- **Invoices = receivables (money in).** Issued BY the user TO their customers.
- **Bills = payables (money out).** Money the user owes to suppliers.
- **FX conversions happen within one wallet** — not between wallets.

---

## Consequential operations

These are **irreversible or high-impact**. Before executing ANY of them: (1) run `auth whoami` and state the environment, (2) explain the effect, (3) get explicit user confirmation.

| Operation | Command | Notes |
| --- | --- | --- |
| Void invoice | `invoices void <id>` | Irreversible. FINALIZED+UNPAID only. |
| Delete draft invoice | `invoices delete <id>` | DRAFT only. |
| Finalize invoice | `invoices finalize <id>` | Irreversible — cannot edit after. |
| Mark invoice paid | `invoices mark-as-paid <id>` | OUT_OF_BAND collection only. Must be FINALIZED. |
| Cancel subscription | `subscriptions cancel <id>` | Check flags via schema. |
| Block card | `cards update <id>` + `{"status":"BLOCKED"}` | Reversible. |
| Close card | `cards update <id>` + `{"status":"CLOSED"}` | **Permanent.** |
| FX conversion | **Airwallex Dashboard only** | Cannot execute via CLI. |
| Finalize credit note | `credit-notes finalize <id>` | Irreversible. |
| Void credit note | `credit-notes void <id>` | Irreversible. |
| Delete credit note | `credit-notes delete <id>` | DRAFT only. |
| Close global account | `global-accounts close <id>` | **Permanent.** |
| Delete cardholder | `cardholders delete <id>` | Ensure no active cards. |

Card status changes use `cards update` with a JSON body — no `cards block`/`cards close` subcommands exist.

---

## Error handling

| Situation | Action |
| --- | --- |
| Required field missing or ambiguous | STOP, list gaps, ask user |
| API error | Show full error, ask user |
| API validation error | Check [references/api_traps.md](references/api_traps.md) and run `airwallex <resource> <action> --api-schema-only` (e.g., `airwallex cards create --api-schema-only`) to verify body structure |
| 401 / auth expired | Retry command (auto-refresh). If retry also fails: ask user which environment (sandbox/production), **immediately execute** `auth login` or `auth login --prod` yourself (do NOT tell user to run it), confirm with `auth whoami`, then resume |
| Duplicate detected | Show details, let user choose |
| Partial completion | Report what succeeded (with IDs) and what failed |

## References

- `airwallex --tree --compact [group]` — discover available commands and subcommands (e.g., `airwallex --tree --compact invoices`)
- `airwallex <resource> <action> --api-schema-only` — OpenAPI request/response body schema for write commands (e.g., `airwallex invoices create --api-schema-only`)
- [references/api_traps.md](references/api_traps.md) — non-obvious body constraints beyond what the schema surfaces
- [Airwallex API Introduction](https://www.airwallex.com/docs/api/introduction)
