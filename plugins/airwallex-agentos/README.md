# airwallex-agentos plugin (Claude Code / Cursor / Codex)

Teaches Claude, Cursor, and Codex how to use Airwallex — Billing, Payouts, Issuing, and Treasury — via either the `airwallex` CLI or an Airwallex MCP server.

The same skills work across all supported surfaces. Each `SKILL.md` is surface-neutral — workflow logic is shared, and the agent picks up the surface-specific details (command names, tool names, auth, pagination) from `awx-best-practices/references/surface-quickstart.md` and from the connected toolchain itself.

## Skills

| Skill | Category | Description |
| --- | --- | --- |
| [contract-to-billing](skills/contract-to-billing/) | Billing | Extract billing details from POs/contracts/quotes, match existing resources, and create invoices and/or subscriptions |
| [beneficiary-creation](skills/beneficiary-creation/) | Payouts | Extract bank details from supplier documents, validate per-country schemas, and create beneficiaries |
| [card-provisioning](skills/card-provisioning/) | Issuing | Create cardholders and issue virtual or physical corporate cards with spend limits, and manage card spending |
| [manage-cashflow](skills/manage-cashflow/) | Treasury | Aggregate multi-currency balances, receivables, obligations, FX exposure, and indicative rates (conversion execution via Airwallex Dashboard) |
| [awx-best-practices](skills/awx-best-practices/) | Fallback | Ad-hoc operations, troubleshooting, and domains not covered by a workflow skill above |

### Skill file structure

Each workflow skill is self-contained in a single `SKILL.md` (workflow, examples, gotchas). Larger skills carry a `references/` folder with templates and pitfall notes that the agent loads progressively, only when needed:

```
skills/awx-best-practices/
├── SKILL.md                       # Domain routing, auth rules, refusal templates
└── references/
    ├── api_traps.md               # Common API pitfalls and workarounds
    └── surface-quickstart.md      # CLI- and MCP-specific operating rules
```

## Plugin structure

```
airwallex-agentos/
├── .claude-plugin/plugin.json
├── .codex-plugin/plugin.json
├── .cursor-mcp.json
├── .cursor-plugin/plugin.json
├── .mcp.json
├── assets/
│   ├── icon.svg
│   └── logo.svg
├── README.md
└── skills/
    ├── contract-to-billing/
    │   └── SKILL.md
    ├── beneficiary-creation/
    │   └── SKILL.md
    ├── card-provisioning/
    │   ├── SKILL.md
    │   └── references/
    │       └── card-templates.md
    ├── manage-cashflow/
    │   ├── SKILL.md
    │   └── references/
    │       └── response-templates.md
    └── awx-best-practices/
        ├── SKILL.md
        └── references/
            ├── api_traps.md
            └── surface-quickstart.md
```

## Prerequisites

You need **at least one** of the following toolchains available; the skills detect which one the agent is using and act accordingly.

- **CLI:** `airwallex` CLI installed and authenticated (`airwallex auth login`).
- **MCP:** An Airwallex MCP server connected to your Claude Code, Cursor, or Codex client. Configure the MCP server entry per your host's MCP setup docs.
