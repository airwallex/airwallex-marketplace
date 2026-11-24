# Airwallex AI

The Airwallex AgentOS plugin and standalone developer integration skills.

## Plugins

| Plugin | Surfaces | Archetype | Description |
| --- | --- | --- | --- |
| [airwallex-agentos](./plugins/airwallex-agentos) | CLI + MCP | Workflow / Reference | Agent skills for Billing, Payouts, Issuing, and Treasury. Works with the `airwallex` CLI or an Airwallex MCP server. |

### airwallex-agentos - Workflow & Reference Skills

| Skill | Category | Description |
| --- | --- | --- |
| contract-to-billing | Billing | Extract from POs/contracts/quotes → create invoices and/or subscriptions |
| beneficiary-creation | Payouts | Extract bank details from supplier docs → validate per-country → create beneficiaries |
| card-provisioning | Issuing | Create cardholders, issue virtual/physical cards with spend limits, manage spending |
| manage-cashflow | Treasury | Aggregate multi-currency balances, receivables, obligations, FX exposure, indicative rates |
| awx-best-practices | Fallback | Ad-hoc operations, troubleshooting, and domains not covered by a workflow skill above |

### Connectors

AgentOS plugin installations for Claude Code, Cursor, and Codex include the MCP connector automatically. If you install via `npx skills`, you will need to set up a connector separately. See the [Connectors](https://www.airwallex.com/docs/developer-tools/ai/agentos/connectors) page for options:

- **CLI:** `airwallex` CLI installed and authenticated.
- **MCP:** An Airwallex MCP server connected to your AI coding assistant.

## Developer integration skills

Use these skills to guide your agent when building an Airwallex integration. Install only the skills your project needs.

| Skill | Integration |
| --- | --- |
| [airwallex-hpp](plugins/airwallex-dev/skills/airwallex-hpp/SKILL.md) | Hosted Payment Page |
| [airwallex-dropin](plugins/airwallex-dev/skills/airwallex-dropin/SKILL.md) | Embedded Drop-in Element |
| [airwallex-split-card](plugins/airwallex-dev/skills/airwallex-split-card/SKILL.md) | Split Card Element |
| [airwallex-billing-checkout](plugins/airwallex-dev/skills/airwallex-billing-checkout/SKILL.md) | Billing Hosted Checkout |
| [airwallex-kyc](plugins/airwallex-dev/skills/airwallex-kyc/SKILL.md) | Connected account KYC onboarding |
| [airwallex-ai-provider-card-mit](plugins/airwallex-dev/skills/airwallex-ai-provider-card-mit/SKILL.md) | AI-provider card-on-file integration plan |

Developer skills are excluded from plugin catalogs and the AgentOS bundle.

## Installation

Install AgentOS as a plugin for your client, or install skills with [`npx`](#npx-skills-only).

### Claude Code

Add the marketplace, then install AgentOS:

```sh
claude plugin marketplace add https://github.com/airwallex/airwallex-marketplace
claude plugin install airwallex-agentos@airwallex-marketplace
```

> **Skill namespacing:** Installed skills are prefixed by the plugin name. To invoke a skill, use the format `/airwallex-agentos:contract-to-billing`.

### Cursor

`airwallex-agentos` is listed in the [Cursor Marketplace](https://cursor.com/marketplace). Search for **Airwallex** in **Cursor Settings → Plugins** to install it.

### Codex

Add the marketplace and install the plugin:

```sh
codex plugin marketplace add https://github.com/airwallex/airwallex-marketplace
codex plugin add airwallex-agentos@airwallex-marketplace
```

You can also browse and install from the Codex plugin directory (`/plugins` in the CLI). The Codex plugins use the same skills and MCP server configuration as the Claude Code and Cursor plugins.

### Grok

Add the marketplace, then install AgentOS:

```sh
grok plugin marketplace add https://github.com/airwallex/airwallex-marketplace
grok plugin install airwallex --trust
```

The Grok plugins use the same skills and MCP server configuration as the Claude Code, Cursor, and Codex plugins.

### npx (skills only)

Browse and install skills from this repository:

```sh
npx skills add airwallex/airwallex-marketplace
```

To select only developer integration skills:

```sh
npx skills add airwallex/airwallex-marketplace/plugins/airwallex-dev
```

Select your skills and agent in the installer; choose all six to install the full developer skill set. Add `-g` to install across your projects.

The installer copies instructions and references only; it does not configure MCP connections.

- **AgentOS workflows:** Set up the `airwallex` CLI or an Airwallex MCP server. See [Connectors](#connectors).
- **Developer integrations:** Configure [Developer MCP](https://www.airwallex.com/docs/developer-tools/ai/developer-connector) separately when needed; `airwallex-ai-provider-card-mit` requires it. Test generated code with sandbox credentials.

## Important Notice

> **Beta Feature.** This plugin is provided on an "as-is" basis for use with
> Airwallex APIs (via the `airwallex` CLI or an MCP server). By using this
> plugin, you acknowledge that:
>
> - Actions taken via your Airwallex API keys, whether initiated by you
>   directly or by an AI agent acting on your behalf, are governed by the
>   [Airwallex Terms of Service](https://www.airwallex.com/terms) accepted
>   during account creation and surface authentication (`airwallex auth login`
>   for the CLI, OAuth authorization for the MCP server).
> - You are responsible for safeguarding your API keys and ensuring they are
>   used only in accordance with your intended purpose.
> - This plugin does **not** provide financial, legal, or tax advice. AI agent
>   outputs are probabilistic and should be reviewed by a qualified person
>   before acting on them.
> - This plugin does **not** initiate money-out actions (transfers, FX
>   conversions, or payouts) on your behalf. Where such capabilities exist in
>   the underlying API, the skills are explicitly instructed not to execute
>   them.

## License

Apache-2.0. See [LICENSE](LICENSE).
