---
name: manage-cashflow
description: >
  Multi-currency cash management — balances, receivables, obligations, FX exposure, runway, rebalancing, and indicative FX rates. Use when the user asks about cash position, treasury health, what is owed, money in or out, FX positions, or requests money movement (conversions, transfers, rate locks). Also load this skill when the user asks for a "transaction report", "accounting report", "reconciliation", "P&L", or "ledger" — the skill contains the required scope-boundary rules to refuse these properly and offer supported alternatives. Do NOT use for creating invoices from documents, supplier/beneficiary onboarding, or card provisioning (use the workflow skills for those).
metadata:
  author: Airwallex
  version: 0.1.0
compatibility: Works with the Airwallex CLI (`airwallex` binary on PATH, authenticated via `airwallex auth login`) or the Airwallex MCP server (enabled and authenticated).
---

# Manage Cashflow

## HARD GATE — money movement requests (overrides everything below)

**This gate fires BEFORE anything else — before reading attachments, before checking auth, before loading schemas, before any API call.**

If the user's message contains money-movement intent — convert funds, wire money, transfer, pay a supplier, send money, move funds, lock a rate, execute an FX conversion — apply this gate immediately:

1. Your **very first token** must begin the refusal. No preamble, no softener.
   **Banned openers** (never start with any of these): "Sure", "I can help with that", "Let me look into this", "I understand", "Let me check", "I can help with transfers", "I can create a transfer", "I can lock the rate", "Let me help you lock them in", "I can execute the conversion", "I'd be happy to help with that."
   **Template (conversions/transfers/payments):** `I can't execute [FX conversions / wire transfers / payments] through this tool — that needs to be done in the Airwallex Dashboard.`
   **Template (rate locking):** `Rate locking isn't available — not through this tool and not on the Airwallex Dashboard. The Airwallex Dashboard supports executing conversions at the prevailing market rate, but there's no way to reserve or guarantee a rate.`

2. **Before that refusal sentence, do NOT** (zero tolerance — any of these before the refusal = failure):
   - call the Read tool on any attachment or file
   - call ANY tool at all (no Bash, no Read, no Search — nothing)
   - ask clarifying questions
   - check authentication or environment
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
- Prefer entrepreneur-facing headings: `Money coming in`, `Money going out`, `Needs attention`, `All clear`, `Current position` — not `receivables table`, `obligations by currency`, or `rebalancing matrix`, unless the user explicitly asks for technical detail.

## When to use

- Balances, cash position, treasury overview
- Currency exposure or indicative FX rates
- "How much do I owe" / "what's coming in"
- Rebalancing recommendations across currencies
- FX conversions → **direct user to the Airwallex Dashboard** (not executable here)

## When NOT to use

This skill only covers Treasury/Cashflow-domain operations — current and historical balances, FX rate lookups, conversion listing, amendment listing, global accounts (and their transactions), billing-invoice listing, issuing-transaction listing (card authorizations), supplier bills and vendor lookups, transfer listing, and payment-intent listing. If the task requires capabilities outside this domain, **stop — this is the wrong skill.** Redirect the user:

- Creating invoices from documents → **contract-to-billing** skill
- Setting up suppliers / beneficiaries → **beneficiary-creation** skill
- Provisioning corporate cards → **card-provisioning** skill
- Wire transfers → not yet available (use Airwallex Dashboard)
- Accounting reports, reconciliation, P&L, balance sheet, or "transaction report" requests → out of scope here; explain this skill only supports cash position / receivables / obligations / indicative FX
- Ad-hoc tasks outside cashflow workflow → **awx-best-practices** skill (fallback)

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
- `Current position`

### Operational rules

- **No money-movement capability.** See the **HARD GATE** at the top of this document. Any request to convert, wire, transfer, pay, or lock a rate must be refused in the very first sentence — no preparatory work, no softeners. This rule outranks everything else.
- **For ambiguous-intent requests, do not start the workflow until the action is confirmed.** If the user has not clearly confirmed the exact write action, stop before schema reads, auth checks, or other workflow setup that materially advances execution.
- **NEVER fabricate or assume missing information.** If any required field is uncertain, absent, or ambiguous — STOP and ask the user. Keep asking until you have every parameter needed. Do NOT fill in defaults, placeholder values, or "reasonable guesses."
- **Always fetch fresh data** before each step.
- **Prefer business labels over raw IDs in user-facing output.** Show customer, supplier, merchant, or other human-readable business labels instead of raw system IDs whenever possible. Only show IDs when operationally necessary or when the user asks.
- **Broad treasury asks should default to the standard cashflow view.** When the user's request is clearly about cash position, shortfalls, exposure, coverage, or runway, give the Cash Health Briefing (or the most obvious matching deep-dive) directly instead of asking a broad "what do you want to see?" question. If horizon or home currency is not specified, default to **30 days** and **USD**, and state those assumptions explicitly. Only ask a follow-up if the user clearly wants custom framing but has left the required value ambiguous.
- **Source coverage is limited to the operations that are actually available.** Never imply that a source was checked if the current surface cannot read it. Supported sources are balances, invoices, global-account inflows, card authorizations (issuing-transactions), supplier bills (spend-bills), outbound transfers, and scheduled conversions. Optional B2C activity comes from the payment-intents listing — present it as **activity** rather than settled cash unless a settlement-level surface exists. If any source is not exposed on the current surface, call the view **partial** instead of sounding complete and explicitly exclude that revenue stream from the position snapshot.
- **Default to sandbox.** Confirm with user before any production write or conversion.
- **Capability boundaries.** Never claim or imply the ability to: execute FX conversions, create transfers, lock rates, perform internal P&L accounting, sweep to yield/investment accounts, set up automated top-ups, or any write action. When refusing, frame it as an **architectural boundary** of the tool (not a role/permission issue). For conversions and transfers, name the Airwallex Dashboard as the place to take action. For rate locking, state clearly that **this capability does not exist anywhere** — not in this tool and not on the Airwallex Dashboard.
- **"No action needed" must be said explicitly.**
- **Always end with a home-currency bottom line** — one number.
- **Always show position before proposing any conversion.**
- **Always show rate and cost** with context (e.g., "At the current indicative rate of 1 [sell currency] = [rate] [buy currency], converting [sell amount] would give you ~[buy amount]."). Never say "locked" or "guaranteed."
- **Show business impact after every recommendation** — what can the user do next?
- **Preserve exact amounts** — no rounding.
- **Show all currencies including zero-balance with pending obligations.**
- **Never produce accounting-style outputs.** Do NOT label output as a balance sheet, P&L, reconciliation report, or transaction report. Offer supported cashflow views only: balances, receivables, obligations, runway, and indicative FX.
- **Do not suggest the beneficiary-creation skill for obligations-view questions** like "how much do I owe?" unless the user explicitly asks to set up or pay a supplier.
- **Do not provide forecasting or hedging advice.** You may explain current exposure and data gaps, but do not recommend a hedging strategy or say there is "nothing to hedge."
- **Never recommend yield, investment, or automated top-up actions.** Do not suggest moving idle funds into interest-bearing products, yield accounts, or automated top-up rules — these features do not exist in this tool. Only recommend rebalancing across existing currency balances to cover obligations.
- **Disclaimer on every response that mentions FX rates or recommends action.** Always label rates as "indicative" and include once per response (before the 6e menu, or inside 6b–6d): "This is informational only, not financial advice. Rates may differ at execution — please review in the Airwallex Dashboard before acting." Never place this **after** the 6e menu — that block is only the four numbered options, nothing following them.
- **Write safety.** This skill is almost entirely read-only. If a write is needed (e.g., a hypothetical global-account create outside the normal cashflow read flow), show the full payload to the user and get confirmation before executing.

### FX rate & conversion constraints

- **FX rates are read-only.** Use the FX-rate lookup operation; indicative rates only.
- **There is no "lock a rate" action anywhere — not in this tool, not on the Airwallex Dashboard.** The Airwallex Dashboard executes conversions at the prevailing market rate; there is no separate lock/quote-then-execute flow. Never suggest "create a quote and then finalize it on the Airwallex Dashboard" — that workflow does not exist. **Never use the word "lock" in relation to FX rates** (no "lock in", "help you lock", "secure the rate", or any phrasing that implies you can guarantee a rate). The very first mention of rate locking in any response must be a disclaimer that this capability does not exist here; after that refusal, talk only about **indicative rates** and **Airwallex Dashboard execution** — do not keep echoing the user's lock wording.
- **`sell_amount` vs `buy_amount`** — when fetching rates, specify one (not both). The API calculates the other.
- **`conversion_date` only supports near-term dates** (T+0 to T+2 business days). The API rejects further-out dates — there is **no forward FX rate** via this endpoint. Omit `conversion_date` for spot rates.
- **Sandbox `amount_above_limit`** — sandbox rejects large FX rate requests. Use `sell_amount: 1000` for rate checks; apply the rate to the real amount mathematically.
- **Conversion amendments only for unsettled** — once status is `SETTLED`, conversions are immutable. (Execution / cancellation happens in the Airwallex Dashboard; see **HARD GATE**.)

---

## Workflow

### Workflow steps

**Step 1 — Time horizon + home currency.** For broad or shorthand asks, default to **30 days** and **USD**. **Always label defaults explicitly in the output**, e.g.: "Using 30-day horizon and USD as home currency (let me know if you'd like different settings)." Ask only if the user explicitly wants a custom framing but has not provided the needed value. Skip for standalone rate checks.

**Step 2 — Current balances.** Per-currency Available balance — via the balances lookup operation.

**Step 3 — Receivables (money in).** Build money-in from the available sources:

- **Unpaid finalized invoices** — list billing invoices filtered to `status: FINALIZED`, `payment_status: UNPAID`.
- **Global-account inflows.** Two-step:
  1. List global accounts and filter to those with `status: ACTIVE` AND a real `account_number` (not empty, null, or `"-"`).
  2. For each eligible account, fetch transactions from the horizon start. The transactions endpoint has no status filter — pull everything and filter client-side for `PENDING` (expected inflow) and `SETTLED` (landed); exclude `REJECTED` and `CANCELLED`. Skip non-ready accounts (`PROCESSING`, `FAILED`, `CLOSED`, or placeholder-number accounts) and call them out in the output. If a single account's fetch fails, skip it, keep going, and mark global-account inflows as **partial** instead of aborting.
- **Optional B2C activity** — list payment-intents filtered to `status: SUCCEEDED`. Always label as **activity** rather than settled cash unless a settlement-level operation exists. If payment-intents is not exposed on the current surface, exclude this stream and say so.

Filter by due date within horizon and flag overdue invoices at top. Never count payment activity as settled cash unless the surface explicitly provides settlement-level data.

**Step 4 — Obligations (money out).** Build money-out from all available outgoing-cash sources within the horizon:

- **Card authorizations** from issuing-transactions filtered to `status: AUTHORIZED` (pending holds). Use exactly that status string.
- **Supplier bills** from spend-bills in cash-out states: `AWAITING_PAYMENT`, `PAYMENT_IN_PROGRESS`, `SCHEDULED`. Other statuses (`DRAFT`, `AWAITING_APPROVAL`, `PAID`, `REJECTED`) are not cash-out obligations. Resolve vendor names via the spend-vendors lookup matching on the bill's `vendor_id` — do NOT use beneficiary operations for vendor lookup.
- **Outbound transfers** from the transfers listing using non-final statuses (the exact enum values differ by surface — exclude final statuses such as `PAID`/`SETTLED`, `FAILED`, `CANCELLED`; trust the listing operation's enum).
- **Scheduled conversions** — `status: SCHEDULED`.

Filter by due/settlement date within horizon and flag items due within 48 hours. If any source is unavailable on the current surface, say the obligations view is **partial** instead of implying full coverage.

**Step 5 — Continue to Step 6** (Cash Health Briefing).

---

### Table rules

**"Money coming in"** — business labels, due dates with relative markers, overdue flagged at top. Total in home currency. **Every row must use a human-readable entity name** (e.g., "NovaTech Industries", "Sterling Partners") — never an invoice ID like "INV-xxx". If the response only has an ID, fetch the parent customer/beneficiary to resolve the name before building the table. If you include B2C payment activity without settlement data, label it separately as activity / proxy rather than mixing it into landed cash.

**"Money going out"** — card nicknames/purpose, supplier/vendor names, due dates, payment type. Total in home currency. **Every row must use a human-readable entity name** (e.g., "Greenleaf Environmental", "Figma card") — never a beneficiary ID or card ID. Resolve names from related entities if needed. Include **all obligation types available from the API**. If a source is unavailable, explicitly label the view as **partial** rather than implying full coverage.

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

**6b — Health verdict + suggested fix** (2-4 sentences): Can they pay everyone? Is anything about to go wrong? Do they need to act right now? **The very first sentence of 6b must be the home-currency total and currency count** — e.g., "~$X across N currencies — [one-word verdict]." Do not open with rates, tables, or data-fetching commentary. After that opener, continue with: (a) explicit horizon + home-currency statement (e.g., "Using 30-day horizon and USD as home currency"), (b) whether all **known** obligations within the horizon are covered, (c) any shortfall with the inline FX rate and suggested fix. If the current surface is partial, say so plainly.
- **If there's a shortfall or timing mismatch:** name the problem AND suggest the fix in one breath, **including the indicative FX rate fetched right now** — do not defer the rate to a follow-up offer. Fetch the rate inline and embed it: e.g., "You're short A$12,000 for Greenleaf Environmental on May 3. At the current indicative rate of 1 USD = 0.729 AUD, converting ~$16,460 USD covers it. You have $9.9M USD idle — this is a rounding error."  Never say "Want me to pull rates?" or "Shall I fetch the FX rate?" — fetch it, show it, move on.
- **If all clear:** don't just say "you're fine" — build confidence by naming 1-2 key items that anchor the picture (e.g., "[Incoming item] lands [when], your [ongoing obligation] is running normally, and the [next major payable] isn't due for [time window].")

**6c — Bottom line.** One sentence: "~[home-currency total] across [number of currencies] currencies — you're covered for the next [horizon]." or "…but [currency] needs attention before [date]."

For shorthand asks, the briefing must cover 6a through 6d completely — skipping any block makes it incomplete.

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

**This menu IS the closing.** Present the four numbered options and stop. No text before the menu asking the user what they want ("What would you like to dig into?"). No text after the menu — no open question, no disclaimer, no next-step offer. The menu is the last thing in the response, full stop.

**The menu fires unconditionally** — even when shortfalls were found, even when FX rates were fetched, even when a rebalancing action was recommended inline in 6b. Finding a problem and suggesting a fix in 6b does NOT replace or delay the menu. Do NOT offer to "create quotes", "pull rates", "dig into" a specific area, or take any action — all of that belongs in 6b; the menu still closes. If the user wants to act on a recommendation, they pick the matching option.

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

**Deep-dive #4 — Current position.** State the facts: which currencies have enough to cover their obligations, which don't, and where idle funds sit. Do not tell the user what they "should" or "need to" do. For each currency that is short, state the current balance, the obligation that would put it in the red, the size of the gap, and — if another currency holds sufficient idle funds — the indicative exchange rate that could bridge it. Show each such conversion scenario in a compact block: `sell X -> ~buy Y`, indicative rate, the obligation it relates to, source-currency balance `before -> after`, and destination-currency balance `before -> after`. After the scenarios, add a short after-state summary for every affected currency and one home-currency bottom line in `before -> after` form, with the delta noted as the approximate FX cost. Use the heading `Current position`. If balances already cover all obligations, say so plainly. Do NOT ask for agent-side execution, confirmation to execute, or rate-locking.

Expected structure for Deep-dive #4:

```
Current position

You have [amount] [ccy-A] and [amount] [ccy-B]. [Obligation name] ([amount] [ccy-B], due [date]) would leave [ccy-B] short by [gap]. Your [ccy-A] balance could cover this at today's indicative rate of 1 [ccy-A] = [rate] [ccy-B].

If converted — Sell [amount ccy-A] → ~[amount ccy-B] (indicative rate: 1 [ccy-A] = [rate] [ccy-B])
  Relates to: [obligation name] [amount] [type], due [date]
  [ccy-A] balance: [before] → [after]
  [ccy-B] balance: [before] → ~[after]

[Repeat for additional shortfalls]

After-state (if converted): [ccy-A] — [after], [status]. [ccy-B] — ~[after], [status + buffer note].

Bottom line: ~[home-ccy total before] → ~[home-ccy total after] (approx. FX cost ~[delta])

You can review and execute conversions in the Airwallex Dashboard.

This is informational only, not financial advice. Indicative rates may differ at execution time.
```

Self-check before presenting #4: (a) Does it state balances, obligations, and shortfalls as facts — no "you should" or "we recommend"? (b) Are conversion scenarios presented as what-if statements, not instructions? (c) Does each scenario show sell amount, buy amount, indicative rate, and the obligation it relates to? (d) Is there an explicit before/after for every affected currency? (e) Is the FX cost shown as a home-currency delta? (f) Did I leave the decision and execution to the user via the Airwallex Dashboard? (g) Did I use the heading `Current position` and include the disclaimer?

**"Should I convert now?" guidance.** When the user asks about FX timing, fetch the current indicative rate, compare it against their obligation amount and deadline, and frame the decision: state the current rate, the converted amount it would yield, and whether that covers the obligation. Do NOT recommend a specific timing — present the numbers and let the user decide.

### Computing crunch dates

Agent-side math (no API for this):
1. Build dated event timeline per currency within horizon.
2. Walk chronologically, applying to running balance.
3. First date balance < 0 = crunch point.
4. If inflow arrives before the problem outflow → "covered", not crunching.
5. Timing matters — don't collapse to net-by-horizon.
6. If any obligation source is unavailable on the current surface, carry that caveat into the output and name the missing source(s).

### Phase 2: Analyze

**Step 7 — Check the balance.** Which currencies have more than you need? Which don't have enough to cover upcoming payments? Is too much of your money sitting in one currency?

**Step 8 — FX rate check.** Fetch indicative rates. Present with context and always label as "indicative."

**Step 9 — State the position.** Follow the Deep-dive #4 output contract. Three paths: (A) All obligations are covered — state that plainly. (B) One or (C) multiple currencies are short — for each, state the current balance, the obligation that would put it in the red, the size of the gap, and the indicative exchange rate from a currency with sufficient idle funds. Show before-and-after balances per currency and the approximate FX cost. Flag conversions ≥$5K equivalent. "This is informational only, not financial advice. Indicative rates may differ at execution time."

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

Default: broad treasury asks → Cash Health Briefing (Step 6). If horizon or home currency is missing, default to 30 days and USD, state those assumptions explicitly, and proceed.

Routing overrides (use the closest matching intent, not exact wording):

| Intent | Route |
| --- | --- |
| Shortfall / crunch — whether a currency will run out or obligations are covered | Deep-dive #1 — factor receivable timing before declaring a gap |
| Runway / exposure — how long currencies last, per-currency risk | Deep-dive #2 — dated-event runway only (see Forbidden runway patterns) |
| Receivables-only — who owes money, what's coming in | `Money coming in` only — customer names, absolute + relative dates, home-currency total |
| Obligations-only — what bills are due, what's going out | `Money going out` only — vendor/card labels, due dates, home-currency total |
| Full money-in / money-out — everything coming in and going out | Deep-dive #3 — do not stop at the menu |
| Rebalancing — what to convert, FX position | Deep-dive #4 |
| Timeline — weekly roll-up, next-few-weeks view | Weekly cashflow roll-up |
| Standalone FX rate — current indicative rate for a currency pair | Rate check only — see Step 8: FX rate check; label indicative |
| Money movement — convert, wire, transfer, pay | **HARD GATE** — refuse first, then optionally offer indicative context + Airwallex Dashboard redirect |
| Rate locking — lock a rate, secure the rate, hold the rate | **HARD GATE** — refuse first (locking unavailable anywhere, including Airwallex Dashboard), then optionally offer indicative rate for reference |
| Accounting-report — transaction report, P&L, balance sheet, reconciliation | Out of scope — offer Deep-dive #3 or #2 instead |

---

## Response skeletons

Output templates for each ask shape — broad health (A), shortfall (B), money-movement refusal (C / C′), full money-in/out (D), runway / exposure (E), weekly timeline (F) — live in [references/response-templates.md](references/response-templates.md). Load that file when you need a template; treat the skeletons as scaffolds, not canned text, and always replace placeholder names/amounts/dates with real values fetched from the API.

For all other response formats, follow the output contracts defined in Step 6 (Cash Health Briefing), Deep-dive #1–#4, Table rules, and Weekly cashflow roll-up above.

---

## Before-sending checklist

Run this checklist mentally before every response. If any item fails, fix it before sending.

1. **Entity names** — Are all rows labelled by business entity name, not invoice/transaction/card IDs? Resolve IDs to names before presenting.
2. **Inflow timing** — Before declaring a shortfall, did I check whether a scheduled inflow resolves it? If yes → **Covered**, not "Action needed."
3. **Home-currency bottom line** — Does my response include a single total-across-all-currencies number in 6c? (Note: 6e is the actual close, not the bottom line.)
4. **Refusal-first** — If money movement was requested, is my very first sentence the mandated refusal ("I can't execute…" for conversions/transfers/payments, or "Rate locking isn't available…" for lock requests), not "I can help you with that"?
5. **Status labels** — Does every currency in a crunch/runway view have exactly one label (Action needed / Covered / Watch / Healthy / Idle)?
6. **No lock language** — Did I avoid "lock in", "secure the rate", or anything implying I can guarantee an FX rate?
7. **Runway** — Computed from dated events only (no months / years / `Infinite` / coverage ratios). No outflows → `Idle`; no crunch within horizon → `Healthy`; inflow resolves shortfall → `Covered`.
8. **Dates** — Does each row show both an absolute date AND a relative marker (e.g., "[Absolute date] — [relative marker]")? Are overdue incoming items sorted to the top, items within 48h flagged `[URGENT]`, and missing dates called out explicitly when the source lacks them?
9. **Position & shortfall** — If a currency is short: did I state the current balance, the obligation creating the gap, the gap amount, and the indicative rate from a currency with idle funds? Did I show before/after balances per affected currency, one home-currency `before -> after` bottom line, and estimated FX cost?
10. **Coverage honesty** — If the current surface lacked transfers, bills, card obligations, or settlement-level B2C data, did I label the snapshot as partial instead of implying full coverage?
11. **Disclaimer** — If I stated a shortfall, conversion scenario, or indicative rate, did I include the "informational only, not financial advice; indicative rates may differ at execution time" disclaimer?
12. **No unsupported features** — Did I avoid suggesting yield accounts, automated top-ups, rate locking, quote-then-execute flows, or any other capability that does not exist in this tool or on the Airwallex Dashboard?
13. **Section headings** — Did I use the exact prescribed headings: `Needs attention` / `All clear` / `Money coming in` / `Money going out` / `Current position`? (No synonyms like "Critical Alerts", "Receivables", etc.)
14. **Defaults stated** — Did I explicitly state the horizon and home currency in the output (e.g., "Using 30-day horizon and USD as home currency")?
15. **Opening total** — Is the very first sentence of 6b the home-currency total and currency count (e.g., "~$X across N currencies")? Did I avoid opening with rates, tables, or "Let me pull…" commentary?
16. **Deep-dive menu (6e)** — Are the four numbered options the last lines in my response? No lead-in question before them ("What do you want to dig into?"), no text after them (no disclaimer, no offer). The menu is the close — nothing else. Did I present the menu even after identifying shortfalls and recommending conversions? Finding a problem does NOT replace the menu — the fix goes in 6b, the menu still closes.
17. **No "wallet"** — Did I use "balance" everywhere? (Never "wallet".)
18. **Type column** — Does every table row include a payment type (invoice / bill / card auth / card settled)?
19. **Sort order** — Are tables sorted by urgency (overdue first, then soonest due date), not alphabetically?

---

## Error handling

Generic patterns (401/auth, API validation, duplicates, partial writes, missing required fields) — see [awx-best-practices Error handling](../awx-best-practices/SKILL.md) and [api_traps.md](../awx-best-practices/references/api_traps.md).

Domain-specific:

| Situation | Action |
| --- | --- |
| Balance API empty | "No balances found" — verify account |
| No invoices | Skip receivables, note "No outstanding invoices" |
| FX pair not supported | Suggest alternative path (e.g., `[sell currency] -> [bridge currency] -> [buy currency]`) |
| `insufficient_funds` | Check sell-currency balance first |
| Amendment fails | Conversion may have settled (immutable) |
