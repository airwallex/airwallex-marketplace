# Airwallex AI

Agent skills for integrating Airwallex with LLMs and agent frameworks.

## Plugins

| Plugin | Surfaces | Archetype | Description |
| --- | --- | --- | --- |
| [airwallex](./plugins/airwallex) | CLI + MCP | Workflow / Reference | Agent skills for Billing, Payouts, Issuing, and Treasury. Works with the `airwallex` CLI or an Airwallex MCP server. |
| [airwallex-dev](./plugins/airwallex-dev) | - | Integration | Developer-focused skills that generate production-ready code for Airwallex API integrations. |

### airwallex - Workflow & Reference Skills

| Skill | Category | Description |
| --- | --- | --- |
| contract-to-billing | Billing | Extract from POs/contracts/quotes → create invoices and/or subscriptions |
| beneficiary-creation | Payouts | Extract bank details from supplier docs → validate per-country → create beneficiaries |
| card-provisioning | Issuing | Create cardholders, issue virtual/physical cards with spend limits, manage spending |
| manage-cashflow | Treasury | Aggregate multi-currency balances, receivables, obligations, FX exposure, indicative rates |
| awx-best-practices | Fallback | Ad-hoc operations, troubleshooting, and domains not covered by a workflow skill above |

### airwallex-dev - Integration Skills

| Skill | Category | Description |
| --- | --- | --- |
| *(coming soon)* | Integration | Integration skills that generate code - checkout pages, webhook handlers, payment flows, and more. |

### Connectors

Claude Code and Cursor installations include the MCP connector automatically. If you install via `npx skills`, you will need to set up a connector separately. See the [Connectors](https://www.airwallex.com/docs/developer-tools/ai/agentos/connectors) page for options:

- **CLI:** `airwallex` CLI installed and authenticated.
- **MCP:** An Airwallex MCP server connected to your AI coding assistant.

## Installation

Pick the method that matches your editor or workflow. All options install the same set of agent skills.

### Claude Code

Add the marketplace and install the plugin:

```sh
claude plugin marketplace add https://github.com/airwallex/airwallex-marketplace
claude plugin install airwallex@airwallex-marketplace
```

> **Skill namespacing:** Installed skills are prefixed by the plugin name. To invoke a skill, use the format `/airwallex:contract-to-billing`.

### Cursor

The Airwallex plugin is available in the [Cursor Marketplace](https://cursor.com/marketplace). Search for **Airwallex** in **Cursor Settings → Plugins** to install it directly.

### npx (skills only)

Install the agent skills with a single command:

```sh
npx skills add airwallex/airwallex-marketplace
```

This installs the agent skills only and does **not** configure an Airwallex MCP server connection. If your workflow requires the MCP server, you will need to set it up separately in your client's MCP configuration. See [Connectors](#connectors) above.

---

After installing, make sure you have at least one of the prerequisite toolchains available (the `airwallex` CLI or an Airwallex MCP server). See [Connectors](#connectors) above.

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
