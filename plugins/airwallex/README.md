# airwallex plugin (Claude Code / Cursor)

Teaches Claude Code and Cursor how to use the **`airwallex`** CLI — an agent-oriented command-line tool for [Airwallex](https://www.airwallex.com/) APIs.

## Skills

| Skill | Category | Description |
| --- | --- | --- |
| [contract-to-billing](skills/contract-to-billing/) | Billing | Extract billing details from POs/contracts/quotes, match existing resources, and create invoices |
| [beneficiary-creation](skills/beneficiary-creation/) | Payouts | Extract bank details from supplier documents, validate per-country schemas, and create beneficiaries |
| [card-provisioning](skills/card-provisioning/) | Issuing | Provision virtual or physical corporate cards with spend limits |
| [manage-cashflow](skills/manage-cashflow/) | Treasury | Aggregate balances, analyze currency exposure, manage FX quotes (conversion execution via dashboard) |
| [awx-best-practices](skills/awx-best-practices/) | Foundation | Always-on guardrail: CLI conventions, auth, environment safety, domain routing, pitfalls |

### Skill file structure

Each workflow skill is self-contained in a single `SKILL.md` (workflow, examples, gotchas). The best-practices skill additionally carries `api_traps.md` with common API pitfalls and workarounds:

```
skills/awx-best-practices/
├── SKILL.md                # Domain routing table, auth rules
└── references/
    └── api_traps.md        # Common API pitfalls and workarounds
```

## Plugin structure

```
airwallex/
├── .claude-plugin/plugin.json
├── .cursor-plugin/plugin.json
├── README.md
└── skills/
    ├── contract-to-billing/
    │   └── SKILL.md
    ├── beneficiary-creation/
    │   └── SKILL.md
    ├── card-provisioning/
    │   └── SKILL.md
    ├── manage-cashflow/
    │   └── SKILL.md
    └── awx-best-practices/
        ├── SKILL.md
        └── references/
            └── api_traps.md
```

## Recommended models

These skills involve multi-step financial workflows with document extraction and API orchestration. **Use Claude Opus 4.6 for best results.**

## Prerequisites

- `airwallex` CLI installed and authenticated
