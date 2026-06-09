# Response templates

Use these as **output templates**, not literal canned text. The illustrative examples below use fictitious data — **always replace** all names, amounts, currencies, and dates with values fetched from the API.

## Contents

- [Skeleton A — broad cash-health ask](#skeleton-a--broad-cash-health-ask)
- [Skeleton B — shortfall or crunch ask](#skeleton-b--shortfall-or-crunch-ask)
- [Skeleton C — money-movement refusal (conversions/transfers/payments)](#skeleton-c--money-movement-refusal-conversionstransferspayments)
- [Skeleton C′ — money-movement refusal (rate locking)](#skeleton-c--money-movement-refusal-rate-locking)
- [Skeleton D — full money-in / money-out ask](#skeleton-d--full-money-in--money-out-ask)
- [Skeleton E — runway or exposure ask](#skeleton-e--runway-or-exposure-ask)
- [Skeleton F — weekly timeline ask](#skeleton-f--weekly-timeline-ask)

The skeletons map onto the routing table in `SKILL.md → Ongoing monitoring`:

| Ask | Skeleton |
| --- | --- |
| Broad cash-health | A |
| Shortfall / crunch | B |
| Money movement — conversions, wires, payments | C |
| Money movement — rate locking | C′ |
| Full money-in / money-out | D |
| Runway or exposure | E |
| Weekly timeline | F |

For all other response formats, follow the output contracts defined in `SKILL.md` — Step 6 (Cash Health Briefing), Deep-dive #1–#4, Table rules, and Weekly cashflow roll-up.

---

## Skeleton A — broad cash-health ask
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

## Skeleton B — shortfall or crunch ask
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

## Skeleton C — money-movement refusal (conversions/transfers/payments)
```
I can't execute [FX conversions / wire transfers / payments] through this tool — that needs to be done in the Airwallex Dashboard.

Here's what I can tell you: [indicative rate context, position impact, or recommended amount].
[NEVER offer to "help in sandbox" or imply transfer capability.]
```

## Skeleton C′ — money-movement refusal (rate locking)
```
Rate locking isn't available — not through this tool and not on the Airwallex Dashboard. The Airwallex Dashboard supports executing conversions at the prevailing market rate, but there's no way to reserve or guarantee a rate.

I can show you the current indicative rate for reference if that helps.
[Do NOT redirect to Airwallex Dashboard for locking. Do NOT imply locking exists elsewhere.]
```

## Skeleton D — full money-in / money-out ask
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

## Skeleton E — runway or exposure ask
```
Here's the runway by currency for the next [horizon] days, based on dated inflows and obligations — not an average burn-rate estimate.

Using [horizon]-day horizon and [home currency] as home currency.

| Currency | Available | Incoming | Outgoing | Net | Exposure % | Runway | Status |
| [Currency] | [available] | [incoming] | [outgoing] | [net] | [X%] | [No crunch within [horizon] / [N] days / 0 days] | [Action needed / Covered / Watch / Healthy / Idle] |
...
[One sentence per Action needed / Covered / Watch currency.]
Total across all currencies: ~[home-currency total]
```

## Skeleton F — weekly timeline ask
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
