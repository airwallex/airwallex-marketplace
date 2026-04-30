---
name: manage-cashflow
description: >
  Multi-currency cash management — balances, receivables, obligations, FX exposure, runway, rebalancing plans, and indicative rates. Use this skill whenever the user asks about cash position, balances, treasury health, shortfalls, crunch points, runway, rebalancing, FX rates or positions, who owes them money, what bills are due, money coming in or going out, urgency triage, or gives a cash health check prompt. ALWAYS use this skill when the user mentions Airwallex balances, currencies, or requests money movement (conversions, wires, transfers, rate locking) — even if the request seems simple to refuse, the skill contains mandatory safety protocols for refusal and redirect. Also load this skill when the user asks for a "transaction report", "accounting report", "reconciliation", "P&L", or "ledger" — the skill contains the required scope-boundary rules to refuse these properly and offer supported alternatives. Do NOT use for creating invoices, setting up suppliers/beneficiaries, or provisioning cards.
metadata:
  author: Airwallex
  version: 0.1.0
compatibility: Requires airwallex CLI installed and on PATH. Authenticated via airwallex auth login (OAuth browser flow). Best results with Claude Opus.
---

# Manage Cashflow

## HARD GATE — money movement requests (overrides everything below)

**This gate fires BEFORE anything else — before reading attachments, before choosing scenario/live mode, before checking auth, before loading schemas, before any API call.**

If the user's message contains money-movement intent — convert funds, wire money, transfer, pay a supplier, send money, move funds, lock a rate, execute an FX conversion — apply this gate immediately:

1. Your **very first token** must begin the refusal. No preamble, no softener.
   **Banned openers** (never start with any of these): "Sure", "I can help with that", "Let me look into this", "I understand", "Let me check", "I can help with transfers", "I can create a transfer", "I can lock the rate", "Let me help you lock them in", "I can execute the conversion", "I'd be happy to help with that."
   **Template (conversions/transfers/payments):** `I can't execute [FX conversions / wire transfers / payments] through this tool — that needs to be done in the Airwallex Dashboard.`
   **Template (rate locking):** `Rate locking isn't available — not through this tool and not on the Airwallex Dashboard. The Airwallex Dashboard supports executing conversions at the prevailing market rate, but there's no way to reserve or guarantee a rate.`

2. **Before that refusal sentence, do NOT** (zero tolerance — any of these before the refusal = failure):
   - call the Read tool on any attachment or file
   - call ANY tool at all (no Bash, no Read, no Search — nothing)
   - choose scenario mode or live mode
   - ask clarifying questions
   - check auth or call `auth whoami`
   - fetch balances, invoices, FX rates, or any data
   - request beneficiary or bank-account details
   - imply that execution would be possible if more information were provided
   Even if the user's message ALSO contains an attachment or an analytical ask, **refuse first, then decide whether to proceed with the non-execution part**.

3. **After refusing**, you may only:
   - For conversions/transfers/payments: redirect the user to the Airwallex Dashboard for execution.
   - For rate locking: do **not** redirect to the Airwallex Dashboard (locking doesn't exist there either). Instead, explain that the Airwallex Dashboard supports executing conversions at the prevailing market rate — no reservation or guarantee.
   - Optionally provide non-execution help (indicative rate context, position impact) — but **frame it as informational, not as a step toward execution.** Say "I can show you the current indicative rate for reference" — NOT "I can help you understand what the conversion would look like" or "Want me to analyze the impact?" Phrasing that sounds like preparation for execution implies the action is feasible through this channel.

4. If the request is purely about execution, stop after the refusal and redirect (or, for rate locking, stop after explaining the limitation). **Never** offer "I can help you do this in sandbox" or imply transfer/wire capability exists in any environment.

---

Aggregates balances, receivables, obligations, and FX exposure. Proposes rebalancing and retrieves indicative FX rates to help the user plan conversions.

**Tone:** Sound like a trusted advisor talking to an entrepreneur over coffee — not a Bloomberg terminal printing a report. The user is smart and busy but has no finance team. Every response should answer three unspoken questions: _Can I pay everyone? Is anything about to go wrong? Do I need to do something right now?_

- If you know the user's name, use it (e.g., "Hey [First name] —"). Personalise the opening.
- Lead with a plain-English health summary, not a table. Tables and breakdowns come only in deep-dives.
- Stay in the user's world — say "you're short [amount] for [obligation name]" not "[currency] net exposure is under-funded by [amount]." Never use jargon the user didn't use first.
- Prefer entrepreneur-facing headings: `Money coming in`, `Money going out`, `Needs attention`, `All clear`, `Suggested action` — not `receivables table`, `obligations by currency`, or `rebalancing matrix`, unless the user explicitly asks for technical detail.

## When to use

- Balances, cash position, treasury overview
- Currency exposure or indicative FX rates
- "How much do I owe" / "what's coming in"
- Rebalancing recommendations across currencies
- FX conversions → **direct user to the Airwallex Dashboard** (not executable here)

## When NOT to use

**FIRST ACTION — before any CLI command:** Run `airwallex --tree --compact` to discover available command groups and subcommands. If a command or subcommand does not appear in the tree output, it **does not exist** — do NOT invent it. (Global flags: `--compact` for single-line JSON, `--dry-run` to preview writes, `--confirm` to execute writes — valid only when placed immediately after `airwallex`.)

**Global flag placement:** Global flags must go immediately after `airwallex`, before the resource/action. This applies to `--compact`, `--dry-run`, and `--confirm` (plus `--tree` for discovery). Correct: `airwallex --compact balances get-current`, `airwallex --dry-run global-accounts create --data-file payload.json`, `airwallex --confirm global-accounts create --data-file payload.json`. Wrong: `airwallex balances get-current --compact`, `airwallex global-accounts create --dry-run --data-file payload.json`, `airwallex global-accounts create --confirm --data-file payload.json` — these fail with `No such option`.

This skill only covers Treasury/Cashflow-domain CLI commands (`balances`, `conversion-rates`, `conversions`, `conversion-amendments`, `global-accounts`, `invoices`, `issuing-transactions`, `spend-bills`, `spend-vendors`, `transfers`, `payment-intents`, `beneficiaries`, `billing-customers`, `cards`). If the task requires a command group outside this domain (e.g., `products`, `prices`, `credit-notes`, `coupons`), **stop — this is the wrong skill.** Redirect the user:

- Creating invoices from documents → **contract-to-billing** skill
- Setting up suppliers / beneficiaries → **beneficiary-creation** skill
- Provisioning corporate cards → **card-provisioning** skill
- Wire transfers → not yet available (use Airwallex Dashboard)
- Accounting reports, reconciliation, P&L, balance sheet, or "transaction report" requests → out of scope here; explain this skill only supports cash position / receivables / obligations / indicative FX
- Ad-hoc tasks outside cashflow workflow → **awx-best-practices** skill (fallback)

## Prerequisites

- `airwallex` CLI installed and on PATH
- Authenticated via `airwallex auth login` (check with `airwallex auth whoami`)
- Environment: sandbox by default. For production, log in with `airwallex auth login --prod`. The environment is locked to the authenticated session — there is no per-command override.
- **Verify commands before use:** Run `airwallex --tree --compact <group>` (e.g., `airwallex --tree --compact balances`) to confirm the subcommand exists. For write commands, run `airwallex <resource> <action> --api-schema-only` (e.g., `airwallex global-accounts create --api-schema-only`) to get the request body schema — read every `required: true` field and include them all in the payload. For read commands, use `airwallex <resource> <action> --help` (e.g., `airwallex balances get-current --help`) to check flags.
- **`request_id` is MANDATORY for all write commands.** Always include `"request_id"` in the JSON body for every `create` and `update` command — the API rejects writes without it. Generate a fresh UUID for each distinct operation via `uuidgen | tr '[:upper:]' '[:lower:]'` — NEVER hand-write a UUID or use sequential/patterned values like `a1b2c3d4-...`. If retrying the same logical operation after a transient/network failure, reuse the same `request_id`; only generate a new one for a distinct new operation. Action commands without a body do not take `request_id`.

---

## Non-negotiables

### Terminology

- **Invoices = receivables (money in).** Never say "obligation" for invoices.
- **Bills = payables (money out).**
- **Card transactions (issuing-transactions):** `AUTHORIZED` = pending hold (money reserved, not yet moved). `CLEARED` = money has actually left the balance. Be explicit which you're counting.
- **FX conversions happen within the same Airwallex account across currency balances** — not as separate wallets. Say "AUD balance" — not "AUD wallet."
- **Home currency** = reporting currency. For broad or shorthand treasury asks with no explicit preference, default to USD and state that assumption plainly. If the user asks for a custom reporting currency but leaves it unspecified, ask.
- **Crunch point** = first date a currency's projected balance goes negative.
- **Runway** = days until crunch. Status labels:

| Status | When to use |
| --- | --- |
| **Action needed** | Crunch within 7 days — no scheduled inflow resolves it |
| **Covered** | Would crunch, but a scheduled inflow arrives before the outflow deadline |
| **Watch** | Crunch in 7-14 days — monitor closely |
| **Healthy** | No crunch within the horizon (>14 days runway) |
| **Idle** | Positive balance, zero outflows within the horizon — funds are redeployable |

**These five labels are the ONLY allowed status vocabulary.** Never substitute synonyms — if the word is not in the table above, do not use it as a status label.

**Terminology note:** Always say "balance" (e.g., "AUD balance"), never "wallet."

**Section headings** must be used verbatim — no synonyms or rewordings:
- `Needs attention` / `All clear`
- `Money coming in` / `Money going out`
- `Suggested action`

### Operational rules

- **No money-movement capability.** See the **HARD GATE** at the top of this document. Any request to convert, wire, transfer, pay, or lock a rate must be refused in the very first sentence — no preparatory work, no softeners. This rule outranks everything else.
- **For ambiguous-intent requests, do not start the workflow until the action is confirmed.** If the user has not clearly confirmed the exact write action, stop before schema reads, auth checks, or other workflow setup that materially advances execution.
- **NEVER fabricate or assume missing information.** If any required field is uncertain, absent, or ambiguous — STOP and ask the user. Keep asking until you have every parameter needed to make the API call. This applies to currencies, amounts, conversion directions, and any other field. Do NOT fill in defaults, placeholder values, or "reasonable guesses."
- **Always fetch fresh data** — re-fetch before every step.
- **Prefer business labels over raw IDs in user-facing output.** In summaries, tables, and explanations, show customer, supplier, merchant, or other human-readable business labels instead of raw system IDs whenever possible. Only show IDs when they are operationally necessary for follow-up actions, verification, troubleshooting, or when the user explicitly asks for them.
- **Attachment = data source, not subject of review.** When a scenario/demo file is attached and the user asks a treasury question ("How's my cash?", "Am I okay?", "Cash.", "Do I have shortfalls?"), use **scenario mode** (see Workflow). The file provides the numbers — do NOT describe the document structure, comment on its formatting, or summarize it as a document. Extract the financial data and deliver the requested cashflow output. All conclusions must be grounded in exact amounts, named currencies, named counterparties, and dated events from the attachment.
- **Broad treasury asks should default to the standard cashflow view.** When the user's request is clearly about cash position, shortfalls, exposure, coverage, or runway, give the Cash Health Briefing (or the most obvious matching deep-dive) directly instead of asking a broad "what do you want to see?" question. If horizon or home currency is not specified, default to **30 days** and **USD**, and state those assumptions explicitly. Only ask a follow-up if the user clearly wants custom framing but has left the required value ambiguous.
- **Live-mode coverage is limited to the current surface.** Never imply that a source was checked if the current CLI/MCP surface cannot read it. If transfers, bills, card obligations, or B2C settlement data are unavailable, call the view **partial** instead of sounding complete.
- **CLI live-mode sources** — supported sources are balances, invoices, global-account inflows, card authorizations from `issuing-transactions`, supplier bills from `spend-bills`, outbound transfers from `transfers`, scheduled conversions, and optional successful `payment-intents` as a **B2C activity proxy**. Do NOT present payment-intent activity as settled cash unless a settlement-level surface exists.
- **Default to sandbox.** Confirm with user before any production write or conversion.
- **Capability boundaries.** Never claim or imply the ability to: execute FX conversions, create transfers, lock rates, perform internal P&L accounting, sweep to yield/investment accounts, set up automated top-ups, or any write action. When refusing, frame it as an **architectural boundary** of the tool (not a role/permission issue). For conversions and transfers, name the Airwallex Dashboard as the place to take action. For rate locking, state clearly that **this capability does not exist anywhere** — not in this tool and not on the Airwallex Dashboard.
- **"No action needed" must be said explicitly.**
- **Always end with a home-currency bottom line** — one number.
- **Always show position before proposing any conversion.**
- **Always show rate and cost** with context (e.g., "At the current indicative rate of 1 [sell currency] = [rate] [buy currency], converting [sell amount] would give you ~[buy amount]."). Never say "locked" or "guaranteed."
- **Show business impact after every recommendation** — what can the user do next?
- Preserve exact amounts — no rounding.
- Show all currencies including zero-balance with pending obligations.
- **Never produce accounting-style outputs.** Do NOT label output as a balance sheet, P&L, reconciliation report, or transaction report. Offer supported cashflow views only: balances, receivables, obligations, runway, and indicative FX.
- **Do not suggest the beneficiary-creation skill for obligations-view questions** like "how much do I owe?" unless the user explicitly asks to set up or pay a supplier.
- **Do not provide forecasting or hedging advice.** You may explain current exposure and data gaps, but do not recommend a hedging strategy or say there is "nothing to hedge."
- **Never recommend yield, investment, or automated top-up actions.** Do not suggest moving idle funds into interest-bearing products, yield accounts, or automated top-up rules — these features do not exist in this tool. Only recommend rebalancing across existing currency balances to cover obligations.
- **Disclaimer on every response that mentions FX rates or recommends action.** Always label rates as "indicative" and append once per response: "This is informational only, not financial advice. Rates may differ at execution — please review in the Airwallex Dashboard before acting."
- **Dry-run before every write.** Before executing any write command (POST), first run it with the `--dry-run` global flag to preview the request envelope without sending it. Show the envelope to the user (method, URL, body, environment) and get explicit approval. Only then re-run with `--confirm` to execute. The two-step sequence is: (1) `airwallex --dry-run <command>` → show preview → user approves → (2) `airwallex --confirm <command>` → execute. **Never skip the dry-run step for write operations.**
- **Positional IDs, not `--id`.** E.g., `beneficiaries get <ID>`, `spend-bills get <ID>`. Never pass `--id` as a flag.
- Commands with an object body (`create`, `update`) use `--data-file`, `--data`, or `--data-stdin`. Some action commands take only positional IDs with no body flags; others accept both. Always check the schema.
- **JSON output is the default.** Use `--compact` only if you need single-line JSON output, and place it immediately after `airwallex`: `airwallex --compact balances get-current`. Do NOT put it after the action (`airwallex balances get-current --compact`).

### FX rate & conversion constraints

- **FX rates are read-only.** Use `conversion-rates get-current` to retrieve indicative rates. Do NOT use `quotes create` — the Quote endpoint is not available in this flow. **There is no "lock a rate" action anywhere — not in this tool, not on the Airwallex Dashboard.** The Airwallex Dashboard supports executing conversions at the prevailing market rate, but there is no separate lock/quote-then-execute flow. Never suggest "create a quote and then finalize it on the Airwallex Dashboard" — that workflow does not exist. **Never use the word "lock" in relation to FX rates.** Do not say "lock in", "help you lock", "secure the rate", or any phrasing that implies you can guarantee or reserve an exchange rate. The very first mention of rate locking in any response must be a disclaimer that this capability does not exist here. After that refusal, talk only about **indicative rates** and **Airwallex Dashboard execution** — do not keep echoing the user's lock wording.
- **`sell_amount` vs `buy_amount`** — when fetching rates, specify one (not both). The API calculates the other.
- **`conversion_date` only supports near-term dates** (T+0 to T+2 business days). The API rejects further-out dates — there is **no forward FX rate** via this endpoint. For forward rate analysis, tell the user this data is not available via the Airwallex API. Omit `conversion_date` for spot rates.
- **Sandbox `amount_above_limit`** — sandbox rejects large FX rate requests. Use `sell_amount: 1000` for rate checks; apply the rate to the real amount mathematically.
- **Conversion amendments only for unsettled** — once status is `SETTLED`, conversions are immutable.
- **`conversions create` not available via CLI** — conversions must be executed in the **Airwallex Dashboard**.
- **`conversion-amendments create` not available via CLI** — `create-quote` (preview cost) IS available. Execute cancellation in the **Airwallex Dashboard**. Use `conversion-amendments list` to verify status.
- **Pagination varies by command (minimum `--page-size` is 10 for all commands that support it):** `invoices list` uses cursor `--page` + `--page-size` (pass `page_after` to `--page`). `conversions list` uses `--page-num` + `--page-size` (0-based, increment while `has_more`). `global-accounts list` uses cursor `--page` + `--page-size`. `issuing-transactions list` uses cursor `--page` + `--page-size` (max 100). **`spend-bills list` uses cursor `--page` ONLY — it does NOT support `--page-size` (the CLI will error with "No such option"). Always run `--help` before assuming pagination flags.**
- **`balances list-history`:** When using `--page-num` pagination, the `--from`/`--to` date range is **capped at 7 days** — the API rejects wider ranges. This is a balance-history API window limit, not the cashflow horizon; 30-day or weekly cashflow views should combine invoices, bills, transfers, card obligations, scheduled conversions, and other supported sources. To retrieve history beyond 7 days with `--page-num`, window into consecutive ≤7-day chunks. Alternatively, use `--page` cursor pagination: passing `--page 0` overrides the 7-day default for null date ranges, enabling a complete walk through all balance records without date filters. Do NOT combine `--page` and `--page-num` in the same call — the CLI rejects it.

---

## Workflow

### Mode selection — decide BEFORE doing anything else

| Condition | Mode |
| --- | --- |
| User attached a scenario file, demo brief, spreadsheet, or PDF | **Scenario mode** |
| No attachment, or user explicitly asks for live account data | **Live mode** |

If an attachment exists AND the user also asks to compare with live data, run scenario mode first, then live mode, and label each data set clearly.

---

### Scenario mode (attachment present)

**S1 — Extract the ledger.** Read the attachment in full. Ignore any non-financial content (titles, instructions, commentary) — only extract the numbers. Never review or critique the document itself. Pull out every financial data point into a structured ledger:
- Balances per currency (available, reserved)
- Receivables (counterparty name, amount, currency, due date)
- Obligations — **all types**: scheduled payouts/transfers, supplier payments, card authorizations (AUTHORIZED), scheduled conversions
- Any other dated cash event

This ledger is your **sole ground truth** for balances, receivables, and obligations. Do NOT replace these numbers with live CLI/MCP data.

**S2 — Fill gaps only.** The only live-data calls allowed in scenario mode:
- `auth whoami` (pre-flight)
- `conversion-rates get-current` (indicative FX rates not in the attachment)

Do NOT call `balances get-current`, `invoices list`, `issuing-transactions list`, or any other data-fetch command — the scenario provides that data, and live API data will contradict it.

**S3 — Time horizon + home currency.** For broad or shorthand asks, default to **30 days** and **USD**. **Always label defaults explicitly in the output**, e.g.: "Using 30-day horizon and USD as home currency (let me know if you'd like different settings)." Ask only if the user explicitly wants a custom framing but has not provided the needed value. Use dates from the scenario to anchor "today."

**S4 — Build tables.** Construct "Money coming in" and "Money going out" from the ledger (see table rules below). Then continue to Step 6 (Cash Health Briefing).

**S-verify** — Before presenting: scan every counterparty name and amount in your output. If any name or number cannot be traced back to the attachment, you have drifted to live data — stop and redo from the ledger.

---

### Live mode (no attachment)

**L1 — Pre-flight.** Run `auth whoami`. If it fails (no active session), ask the user which environment (sandbox/production). Once the user answers, **immediately execute** `airwallex auth login` (or `airwallex auth login --prod` for production) yourself — do NOT tell the user to run it manually. The command triggers a browser-based OAuth flow; the user completes sign-in in their browser. After the command returns, confirm the session with `auth whoami`. If it succeeds on the first check, tell the user the current environment. No `auth refresh` command — on 401, retry the real command first (auto-refresh); if retry also fails, ask environment and **execute `auth login` yourself**.

**L2 — Time horizon + home currency.** For broad or shorthand asks, default to **30 days** and **USD**. **Always label defaults explicitly in the output**, e.g.: "Using 30-day horizon and USD as home currency (let me know if you'd like different settings)." Ask only if the user explicitly wants a custom framing but has not provided the needed value. Skip for standalone rate checks.

**L3 — Fetch current balances.** Per-currency Available balance.

**L4 — Fetch receivables (money in).** Build money-in from the supported sources on the current surface:
- Unpaid finalized invoices (`invoices list --status FINALIZED --payment-status UNPAID`)
- Incoming global-account transactions / inflows. Fetch procedure: (1) `global-accounts list --status ACTIVE` to get eligible accounts, then filter to those with a real `account_number` (not empty, null, or `"-"`). (2) For each eligible account, `global-accounts list-transactions <ID> --from-created-at <horizon_start>` to fetch transactions — the command has no `--status` flag, so filter client-side for `status: PENDING` (expected inflow) and `status: SETTLED` (landed inflow); exclude `REJECTED` and `CANCELLED`. Skip non-ready accounts (`PROCESSING`, `FAILED`, `CLOSED`, or placeholder-number accounts) and state in the output that they were excluded from global-account inflows. If a single eligible account's transaction fetch fails, skip that account, keep fetching the rest, and mark global-account inflows as **partial** instead of aborting the workflow.
- Optional B2C payment activity via successful `payment-intents`, clearly labelled as **activity** rather than settled cash unless a settlement-level surface exists
Filter by due date within horizon and flag overdue invoices at top. Never count payment activity as settled cash unless the surface explicitly provides settlement-level data.

**L5 — Fetch obligations (money out).** Build money-out from all supported outgoing-cash sources within the horizon:
- Card authorizations from `issuing-transactions list --status AUTHORIZED` (use exactly this status value — pending holds)
- Supplier bills from `spend-bills` in cash-out states: `AWAITING_PAYMENT`, `PAYMENT_IN_PROGRESS`, `SCHEDULED` (complete list of pending-obligation statuses — other statuses like `DRAFT`, `AWAITING_APPROVAL`, `PAID`, `REJECTED` are not cash-out obligations). To resolve vendor names: use `spend-vendors get <vendor_id>` (NOT `spend-bills get-vendor` — that command does not exist). The `vendor_id` field on each bill maps to the `spend-vendors` endpoint, not to `beneficiaries`.
- Outbound transfers / payouts from `transfers`, using non-final statuses: `IN_APPROVAL`, `SCHEDULED`, `OVERDUE`, `PROCESSING`, `SENT` (these represent pending outflows — final statuses `PAID`, `FAILED`, `CANCELLED` are excluded)
- Scheduled conversions (`conversions list --status SCHEDULED`)

Filter by due/settlement date within horizon and flag items due within 48 hours. If any source is unavailable on the current surface, say the obligations view is **partial** instead of implying full coverage.

**L6 — Continue to Step 6** (Cash Health Briefing).

---

### Table rules (both modes)

**"Money coming in"** — business labels, due dates with relative markers, overdue flagged at top. Total in home currency. **Every row must use a human-readable entity name** (e.g., "NovaTech Industries", "Sterling Partners") — never an invoice ID like "INV-xxx". If the response only has an ID, fetch the parent customer/beneficiary to resolve the name before building the table. If you include B2C payment activity without settlement data, label it separately as activity / proxy rather than mixing it into landed cash.

**"Money going out"** — card nicknames/purpose, supplier/vendor names, due dates, payment type. Total in home currency. **Every row must use a human-readable entity name** (e.g., "Greenleaf Environmental", "Figma card") — never a beneficiary ID or card ID. Resolve names from related entities if needed. Include **all obligation types available from the current surface**, plus any scenario-provided obligations even if not fetchable live. If a source is unavailable, explicitly label the view as **partial** rather than implying full coverage.

**Mandatory per-row fields (both tables):**
- Entity name (customer / supplier / card label)
- Amount in native currency
- Type (invoice / bill / card auth / card settled / conversion / etc.)
- **Absolute date + relative marker** (e.g., "May 16 — 25 days", "Apr 22 — tomorrow") — if the source lacks a date, show `Date unavailable in source`
- Items due/clearing within 48 hours: prefix the entire row with **`[URGENT]`** (e.g., `[URGENT] Acme Corp | $5,000 | Invoice | Apr 21 — today`)

**Sort order:** Overdue items first, then by soonest due date. Never sort alphabetically.

**Net summary** — After both tables, one line: net incoming vs outgoing in home currency. If any outflow is due before the inflow that would cover it, flag the timing mismatch explicitly (e.g., "Net: +~[incoming total] coming in, but the [outgoing amount] to [payee] is due before [counterparty]'s [incoming amount] arrives.").

**Step 6 — Cash Health Briefing (default output, always shown):**

Three blocks: a **prose briefing** (6a-6c, no tables/headers/bullet lists), an **urgency-first alert block** (6d), and a **numbered deep-dive menu** (6e).

**6a — Opening line.** Use the user's name if known (see Tone). Otherwise start directly.

**6b — Health verdict + suggested fix** (2-4 sentences): Can they pay everyone? Is anything about to go wrong? Do they need to act right now? Must include: (a) explicit horizon + home-currency statement (e.g., "Using 30-day horizon and USD as home currency"), (b) total home-currency position, (c) number of active currencies, (d) whether all **known** obligations within the horizon are covered. If the current surface is partial, say so plainly.
- **If there's a shortfall or timing mismatch:** name the problem AND suggest the fix in one breath — e.g., "You're short [amount] for [obligation name] on [date]. You have [amount] idle in [source currency] — converting it would get you ~[amount], and topping up the rest from [backup currency] would close the gap."
- **If all clear:** don't just say "you're fine" — build confidence by naming 1-2 key items that anchor the picture (e.g., "[Incoming item] lands [when], your [ongoing obligation] is running normally, and the [next major payable] isn't due for [time window].")

**6c — Bottom line.** One sentence: "~[home-currency total] across [number of currencies] currencies — you're covered for the next [horizon]." or "…but [currency] needs attention before [date]."

For scenario-based and shorthand asks, the briefing must cover 6a through 6d completely — skipping any block makes it incomplete.

**6d — Urgency-first alerts.** For broad treasury asks, always follow the prose with a short alert block. Use **exactly** these two headings — no synonyms, no rewording (not "Needs Attention Now", "Worth Watching", "Fine", etc.):

**Needs attention**
- [Currency]: [what's happening] — [absolute date] — [amount impact] — [Action needed / short description]

**All clear**
- [Currency]: [status summary] — [Healthy / Covered / Idle]

Each alert line must state what is happening, the specific date/deadline, the amount impact, and whether it is covered. Sort by urgency (soonest deadline or crunch first), not alphabetically by currency. If a currency dips but an inflow lands in time, say `Covered` explicitly instead of silently omitting the risk. Zero-obligation currencies belong in `All clear` as `Idle` / redeployable, not hidden. If there are no urgent issues, say `Needs attention: None` and still include 1-2 anchoring `All clear` points.

**6e — Deep-dive menu (MANDATORY — never skip).** After the prose and alert block, always end with exactly these four numbered options:
1. **Crunch-point detail** — when each currency runs out and which obligation causes it
2. **Runway per currency** — full position table with Available / Incoming / Outgoing / Net / Runway / Status
3. **Obligations & receivables** — who owes you, what you owe, net summary with timing-mismatch flags
4. **Rebalancing plan** — what to convert, why, after-state per currency, and FX cost

The user picks a number (or asks a follow-up); only then expand that section. If the user asks for "full detail" or "show me everything", expand all four.

### Deep-dive output contracts

**Deep-dive #1 — Crunch-point detail.** One entry per currency. Assign exactly one status label per Terminology above. For each entry, show what drives the status:
- **Action needed** — crunch date + which obligation triggers it.
- **Covered** — name the inflow that saves it: "[Currency]: [available balance] on hand, but [incoming amount] from [counterparty] arrives [date] — covers the [outgoing amount] [obligation] due [date]. Covered."
- **Healthy / Idle** — state balance and runway or note as redeployable.

Group under two headings: **Needs attention** (Action needed) and **All clear** (Covered / Healthy / Idle). Sort "Needs attention" by soonest crunch date first. Always show both headings (use "None" if a group is empty). If any obligation source is unavailable on the current surface, end with a caveat naming the missing source(s).

Self-check before presenting #1: (a) Does every currency appear with exactly one status label? (b) If an inflow resolves a would-be crunch, did I label it **Covered** (not omitted, not "Action needed")? (c) Are entries sorted by soonest crunch date first? (d) Did I name the specific counterparty and amount for each crunch or coverage?

**Deep-dive #2 — Runway per currency.** Table with columns: Currency | Available | Incoming | Outgoing | Net | Exposure % | Runway | Status. Status labels are defined in Terminology above (every currency gets exactly one). **Exposure %** = that currency's home-currency equivalent as a percentage of total position across all currencies. Flag any currency holding >40% of total value as concentrated exposure.

Sort: Action needed → Covered → Watch → Healthy → Idle. Runway must come from the first projected negative date after walking dated events chronologically; never use average burn-rate shortcuts. If no crunch occurs within the horizon, say `No crunch within [horizon]` and mark `Healthy`. After the table, one sentence per "Action needed", "Covered", or "Watch" currency explaining the driver. End with a home-currency total line.

**Forbidden runway patterns** — NEVER use any of these: months/years estimates from `balance ÷ outflow`, coverage ratios (e.g., "619x coverage"), `Infinite` / `effectively unlimited`, or assumed recurring burn from one-time obligations. Zero balance + pending obligations = "Action needed" or "Covered", never "Idle."

Wrong: `| [Currency] | [Available] | [Outgoing] | [Net] | ~[coverage ratio]x coverage |`
Correct: `| [Currency] | [Available] | [Incoming] | [Outgoing] | [Net] | No crunch within [horizon] | Healthy |`

Self-check before presenting #2: (a) Did I compute runway from the first date-based negative projection, NOT from balance ÷ average outflow? (b) Does every currency have exactly one Status label? (c) Is there a home-currency total at the end? (d) Currencies with zero balance AND pending obligations = "Action needed" or "Covered", never "Idle." (e) Did I avoid months / years / `Infinite` wording entirely? (f) Does the Exposure % column show each currency's share of total position, and did I flag any >40% concentration?

**Deep-dive #3 — Obligations & receivables.** Use the exact section headings `Money coming in` and `Money going out`. Follow Table rules above. End both sections with a home-currency total, then add the net summary with timing-mismatch flags.

**Receivables-only asks.** If the user asks only who owes them money / what's coming in / receivables, show `Money coming in` only. Follow the business-label rules above. Every row must show customer name, amount, absolute date, relative marker, and status; overdue items go to the top. End with one total in home currency. Use `balance`, never `wallet`.

**Obligations-only asks.** If the user asks only what is going out / what bills are due / what they owe, show `Money going out` only. Every row must show supplier/card label, amount, due or clearing date with a relative marker, and payment type; items due within 48 hours must be flagged `[URGENT]`. If the source does not provide a date, say `Date unavailable in source` instead of omitting the field. End with one total in home currency.

Self-check before presenting #3: (a) Is every row labeled by **entity name** (customer/supplier/card purpose), NOT by invoice ID or transaction ID? (b) Does each row show **both** an absolute date AND a relative marker (e.g., "May 16 — 25 days")? (c) Are items due within 48 hours prefixed with `[URGENT]`? (d) Do both tables end with a home-currency total? (e) Is there a net summary with timing-mismatch flags?

**Deep-dive #4 — Suggested action / rebalancing plan.** Open with why action is needed in business terms. For each move, show all of the following in one compact block: `sell X -> ~buy Y`, indicative rate, linked obligation, source-currency balance `before -> after`, and destination-currency balance `before -> after`. After listing the moves, add a short after-state summary for every affected currency. Then show one home-currency bottom line in `before -> after` form, with the delta explained as the estimated FX cost. Prefer `Suggested action` heading over `rebalancing plan`. If no action is needed, say so explicitly. End with a Airwallex Dashboard redirect. Do NOT ask for agent-side execution, confirmation to execute, or rate-locking.

Expected structure for Deep-dive #4:

```
Suggested action

[1–2 sentences: which currency is short, which obligation drives it, and where the idle funds are]

Move [N] — Sell [amount sell-ccy] → ~[amount buy-ccy] (indicative rate: 1 [sell-ccy] = [rate] [buy-ccy])
  Funds: [obligation name] [amount] [type], due [date]
  [sell-ccy] balance: [before] → [after]
  [buy-ccy] balance: [before] → ~[after]

[Repeat for additional moves]

After-state: [sell-ccy] — [after], [status]. [buy-ccy] — ~[after], [status + buffer note].

Bottom line: ~[home-ccy total before] → ~[home-ccy total after] (est. FX cost ~[delta])

Head to the Airwallex Dashboard to execute.

This is informational only, not financial advice. Actual rates may differ at execution time.
```

Self-check before presenting #4: (a) Does the plan open with WHY rebalancing is needed (which shortfalls)? (b) Does each move show sell amount, buy amount, indicative rate, and the specific obligation it funds? (c) Is there an explicit before/after for every affected currency? (d) Is the total FX cost shown as a home-currency delta? (e) Did I direct the user to execute in the Airwallex Dashboard? (f) Did I use the heading `Suggested action` (not "Rebalancing Plan")?

**"Should I convert now?" guidance.** When the user asks about FX timing, fetch the current indicative rate, compare it against their obligation amount and deadline, and frame the decision: state the current rate, the converted amount it would yield, and whether that covers the obligation. Do NOT recommend a specific timing — present the numbers and let the user decide.

### Computing crunch dates

Agent-side math (no API for this):
1. Build dated event timeline per currency within horizon.
2. Walk chronologically, applying to running balance.
3. First date balance < 0 = crunch point.
4. If inflow arrives before the problem outflow → "covered", not crunching.
5. Timing matters — don't collapse to net-by-horizon.
6. If any obligation source is unavailable on the current surface, carry that caveat into the output and name the missing source(s).

### Phase 2: Analyze & Recommend

**Step 7 — Check the balance.** Which currencies have more than you need? Which don't have enough to cover upcoming payments? Is too much of your money sitting in one currency?

**Step 8 — FX rate check.** Fetch indicative rates via `conversion-rates get-current`. Present with context and always label as "indicative."

**Step 9 — Propose action.** Follow the Deep-dive #4 output contract. Three paths: (A) No action needed — say so explicitly. (B) Single-step or (C) Multi-step — for each move show sell/buy/rate/linked obligation/after-state per currency, total FX cost, and Airwallex Dashboard redirect. Warn ≥$5K equivalent. "The rate shown is indicative and may change at execution time."

### Phase 3: Verify after execution

**Step 10 — Verify and confirm (after user returns).** When the user confirms execution, fetch updated balances and the actual execution rate (via `conversions list`). Present all five parts: (1) Restatement — what was converted, amount, actual rate. (2) Obligation covered — which payment is now funded, with updated balance confirmation (e.g., "Your EUR balance is now 4,480 EUR — that covers Klaus's 5,000 EUR invoice due Friday"). (3) Source currency health — sell-side still healthy? (4) New risks — anything to watch. (5) Home-currency bottom line — one number. The confirmation must tie the new balance back to the specific obligation it was meant to fund, so the user sees the cause-and-effect clearly.

### Phase 4: Cancel or Amend (if needed)

Only for unsettled conversions (status `SCHEDULED`).

Preview cost with `conversion-amendments create-quote`. Execute cancellation in the **Airwallex Dashboard**. Verify with `conversion-amendments list`.

---

## Weekly cashflow roll-up

Show when the user explicitly asks, **or** proactively only when both of these are true:
- there is material multi-week timing complexity (for example, outflows due in earlier weeks that rely on later inflows, or multiple crunch / relief points across weeks), and
- the standard Cash Health Briefing would be materially clearer with a timeline view than with alerts alone.

Do **not** auto-show the roll-up just because there are many active currencies. When in doubt, keep the default response as the Cash Health Briefing and leave the weekly roll-up behind the menu unless the user asked for a timeline.

Columns: Week | Money in | Money out | Net | Cumulative.

Rules:
- **Aggregate by calendar week** — one row per week, NOT one row per transaction. Multiple items in the same week are summed on one line (e.g., "£19,700 + $5,000 (~$30,200)").
- Native currency values first, `~` prefix for home-currency converted totals.
- `—` for empty weeks (not blank).
- Caveat known items only.
- End with a home-currency bottom line.

---

## Ongoing monitoring

Default: broad treasury asks → Cash Health Briefing (Step 6). If horizon or home currency is missing, default to 30 days and USD, state those assumptions explicitly, and proceed. Scenario file attached → scenario mode.

Routing overrides (use the closest matching intent, not exact wording):

| Intent | Route |
| --- | --- |
| Shortfall / crunch — whether a currency will run out or obligations are covered | Deep-dive #1 — factor receivable timing before declaring a gap |
| Runway / exposure — how long currencies last, per-currency risk | Deep-dive #2 — dated-event runway only (see Forbidden runway patterns) |
| Receivables-only — who owes money, what's coming in | `Money coming in` only — customer names, absolute + relative dates, home-currency total |
| Obligations-only — what bills are due, what's going out | `Money going out` only — vendor/card labels, due dates, home-currency total |
| Full money-in / money-out — everything coming in and going out | Deep-dive #3 — do not stop at the menu |
| Rebalancing — what to convert, FX position advice | Deep-dive #4 |
| Timeline — weekly roll-up, next-few-weeks view | Weekly cashflow roll-up |
| Standalone FX rate — current indicative rate for a currency pair | Rate check only (`conversion-rates get-current`) — label indicative |
| Money movement — convert, wire, transfer, pay | **HARD GATE** — refuse first, then optionally offer indicative context + Airwallex Dashboard redirect |
| Rate locking — lock a rate, secure the rate, hold the rate | **HARD GATE** — refuse first (locking unavailable anywhere, including Airwallex Dashboard), then optionally offer indicative rate for reference |
| Accounting-report — transaction report, P&L, balance sheet, reconciliation | Out of scope — offer Deep-dive #3 or #2 instead |
| Amendment — cancel or amend an unsettled conversion | Phase 4 |

---

## Response skeletons

Use these as **output templates**, not literal canned text. The illustrative examples below use fictitious data — **always replace** all names, amounts, currencies, and dates with the actual values from the current scenario or live account.

**Skeleton A — broad cash-health ask**
```
[Opening — use name if known]

Using [horizon]-day horizon and [home currency] as home currency.

You have ~[home-currency total] across [number of currencies] currencies. [Verdict: all covered / shortfall in [currency]].
[If shortfall: name it + suggest fix in one sentence.]
[If all clear: name 1-2 anchoring items.]
~[home-currency total] across [number of currencies] currencies — [covered for [horizon] / but [currency] needs attention before [date]].

Needs attention
- [Most urgent issue, with date, amount impact, and whether it is covered]

All clear
- [Currency]: [Healthy / Covered / Idle — plain-English explanation]

Want to dig deeper?
1. Crunch-point detail
2. Runway per currency
3. Obligations & receivables
4. Rebalancing plan
```

**Skeleton B — shortfall or crunch ask**
```
[Opening]
[For each currency with a shortfall or would-be shortfall:]
  - [Currency]: [Status label]. [Available balance] available, [Obligation name] [amount] due [date].
    [If Covered: "but [Inflow name] [amount] arrives [date] — covered."]
    [If Action needed: "Short [gap]. Suggest converting from [source currency]."]
[For currencies with no issues:]
  - [Currency]: [Healthy / Idle]. [Available balance] available, [runway or "no obligations"].
[Home-currency bottom line.]
```

**Skeleton C — money-movement refusal (conversions/transfers/payments)**
```
I can't execute [FX conversions / wire transfers / payments] through this tool — that needs to be done in the Airwallex Dashboard.

Here's what I can tell you: [indicative rate context, position impact, or recommended amount].
[NEVER offer to "help in sandbox" or imply transfer capability.]
```

**Skeleton C′ — money-movement refusal (rate locking)**
```
Rate locking isn't available — not through this tool and not on the Airwallex Dashboard. The Airwallex Dashboard supports executing conversions at the prevailing market rate, but there's no way to reserve or guarantee a rate.

I can show you the current indicative rate for reference if that helps.
[Do NOT redirect to Airwallex Dashboard for locking. Do NOT imply locking exists elsewhere.]
```

**Skeleton D — full money-in / money-out ask**
```
Money coming in
| From | Amount | Type | Due | Status |
| [Customer / counterparty] | [amount] | [invoice / payment / etc.] | [absolute date — relative marker] | [status] |
...
Total: ~[home-currency incoming total]

Money going out
| To / Purpose | Amount | Type | Due |
| [Supplier / card / purpose] | [amount] | [bill / card auth / card settled / transfer] | [absolute date — relative marker] |
...
Total: ~[home-currency outgoing total]

Net: +~[incoming total] coming in vs ~[outgoing total] going out. [Timing mismatch flag if any.]
```
For one-sided asks, render only the requested section (`Money coming in` or `Money going out`) with the same row-level fields and a final home-currency total.
**Skeleton E — runway or exposure ask**
```
Here's the runway by currency for the next [horizon] days, based on dated inflows and obligations — not an average burn-rate estimate.

Using [horizon]-day horizon and [home currency] as home currency.

| Currency | Available | Incoming | Outgoing | Net | Exposure % | Runway | Status |
| [Currency] | [available] | [incoming] | [outgoing] | [net] | [X%] | [No crunch within [horizon] / [N] days / 0 days] | [Action needed / Covered / Watch / Healthy / Idle] |
...
[One sentence per Action needed / Covered / Watch currency.]
Total across all currencies: ~[home-currency total]
```

**Skeleton F — weekly timeline ask**
```
[Opening — health summary sentence]
| Week | Money in | Money out | Net | Cumulative |
| [Week range] | [incoming items or —] | [outgoing items or —] | [net for the week] | [running cumulative total] |
| [Week range] | [incoming items or —] | [outgoing items or —] | [net for the week] | [running cumulative total] |
| [Week range] | [incoming items or —] | [outgoing items or —] | [net for the week] | [running cumulative total] |
Caveat: Based on known scheduled items only — unscheduled card spend or ad-hoc transfers are not included.
[Needs attention: [short summary of the first timing gap, or "None"].]
[Home-currency bottom line.]
```

For all other response formats, follow the output contracts defined in Step 6 (Cash Health Briefing), Deep-dive #1–#4, Table rules, and Weekly cashflow roll-up above.

---

## Before-sending checklist

Run this checklist mentally before every response. If any item fails, fix it before sending.

1. **Data source** — If an attachment is present, does every counterparty name and amount trace back to the attachment? (If not, I've drifted to live data — redo.)
2. **Entity names** — Are all rows labelled by business entity name, not invoice/transaction/card IDs? Resolve IDs to names before presenting.
3. **Inflow timing** — Before declaring a shortfall, did I check whether a scheduled inflow resolves it? If yes → **Covered**, not "Action needed."
4. **Home-currency bottom line** — Does my response end with a single total-across-all-currencies number?
5. **Refusal-first** — If money movement was requested, is my very first sentence the mandated refusal ("I can't execute…" for conversions/transfers/payments, or "Rate locking isn't available…" for lock requests), not "I can help you with that"?
6. **Status labels** — Does every currency in a crunch/runway view have exactly one label (Action needed / Covered / Watch / Healthy / Idle)?
7. **No lock language** — Did I avoid "lock in", "secure the rate", or anything implying I can guarantee an FX rate?
8. **Runway** — Computed from dated events only (no months / years / `Infinite` / coverage ratios). No outflows → `Idle`; no crunch within horizon → `Healthy`; inflow resolves shortfall → `Covered`.
9. **Dates** — Does each row show both an absolute date AND a relative marker (e.g., "[Absolute date] — [relative marker]")? Are overdue incoming items sorted to the top, items within 48h flagged `[URGENT]`, and missing dates called out explicitly when the source lacks them?
10. **Rebalancing** — If recommending a conversion: did I show WHY (which shortfall), sell/buy/rate, before/after for every affected currency, one home-currency `before -> after` bottom line, estimated FX cost, and a Airwallex Dashboard redirect?
11. **Coverage honesty** — If the current surface lacked transfers, bills, card obligations, or settlement-level B2C data, did I label the snapshot as partial instead of implying full coverage?
12. **Disclaimer** — If I recommended a rebalancing move, FX conversion, or timing action, did I include the "informational only, not financial advice" disclaimer?
13. **No unsupported features** — Did I avoid suggesting yield accounts, automated top-ups, rate locking, quote-then-execute flows, or any other capability that does not exist in this tool or on the Airwallex Dashboard?
14. **Section headings** — Did I use the exact prescribed headings: `Needs attention` / `All clear` / `Money coming in` / `Money going out` / `Suggested action`? (No synonyms like "Critical Alerts", "Receivables", etc.)
15. **S3 defaults stated** — Did I explicitly state the horizon and home currency in the output (e.g., "Using 30-day horizon and USD as home currency")?
16. **Deep-dive menu (6e)** — Did I end with the four numbered deep-dive options? (Never skip this.)
17. **No "wallet"** — Did I use "balance" everywhere? (Never "wallet".)
18. **Type column** — Does every table row include a payment type (invoice / bill / card auth / card settled)?
19. **Sort order** — Are tables sorted by urgency (overdue first, then soonest due date), not alphabetically?

## Error handling

| Situation | Action |
| --- | --- |
| Balance API empty | "No balances found" — verify account |
| No invoices | Skip receivables, note "No outstanding invoices" |
| FX pair not supported | Suggest alternative path (e.g., `[sell currency] -> [bridge currency] -> [buy currency]`) |
| `insufficient_funds` | Check sell-currency balance first |
| Amendment fails | Conversion may have settled (immutable) |
| Partial completion | Report what succeeded and failed with IDs, then ask user how to proceed |
| Duplicate detected | Show details, let user choose |
| 401 / auth expired | Retry command (auto-refresh). If retry also fails: ask user which environment (sandbox/production), **immediately execute** `auth login` or `auth login --prod` yourself (do NOT tell user to run it), confirm with `auth whoami`, then resume |
| API error | Run `airwallex <resource> <action> --api-schema-only` (e.g., `airwallex conversion-rates get-current --api-schema-only`) to verify body structure |
