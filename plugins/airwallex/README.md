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

Each workflow skill is self-contained in `SKILL.md` (workflow, examples, gotchas) plus a `references/cli_schema/` directory with auto-extracted CLI parameter schemas. The best-practices skill additionally carries `api_traps.md` and the full set of CLI schemas across all command groups:

```
skills/awx-best-practices/
├── SKILL.md                # Domain routing table, auth rules
└── references/
    ├── api_traps.md        # Common API pitfalls and workarounds
    └── cli_schema/         # All command groups (38 schema files)
```

## Plugin structure

```
airwallex/
├── .claude-plugin/plugin.json
├── .cursor-plugin/plugin.json
├── README.md
└── skills/
    ├── contract-to-billing/
    │   ├── SKILL.md
    │   └── references/cli_schema/
    ├── beneficiary-creation/
    │   ├── SKILL.md
    │   └── references/cli_schema/
    ├── card-provisioning/
    │   ├── SKILL.md
    │   └── references/cli_schema/
    ├── manage-cashflow/
    │   ├── SKILL.md
    │   └── references/cli_schema/
    └── awx-best-practices/
        ├── SKILL.md
        └── references/
            ├── api_traps.md
            └── cli_schema/
```

## Recommended models

These skills involve multi-step financial workflows with document extraction and API orchestration. **Use Claude Opus 4.6 for best results.**

## Prerequisites

- Python >= 3.11
- [uv](https://docs.astral.sh/uv/) package manager
- `airwallex` CLI installed and authenticated
