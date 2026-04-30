# Airwallex AI

Agent skills for integrating Airwallex with LLMs and agent frameworks.

## Plugins

| Plugin | Transport | Description |
| --- | --- | --- |
| [airwallex](/plugins/airwallex) | CLI | Agent skills for the `airwallex` CLI (Claude Code / Cursor) |

| Skill | Category | Description |
| --- | --- | --- |
| contract-to-billing | Billing | Extract from POs/contracts/quotes → create invoices and/or subscriptions |
| beneficiary-creation | Payouts | Extract bank details from supplier docs → validate per-country → create beneficiaries |
| card-provisioning | Issuing | Create cardholders, issue virtual/physical cards with spend limits, manage spending |
| manage-cashflow | Treasury | Aggregate multi-currency balances, receivables, obligations, FX exposure, indicative rates |
| awx-best-practices | Fallback | Ad-hoc operations, troubleshooting, and domains not covered by a workflow skill above |

**Prerequisites:**
- **airwallex:** `airwallex` CLI installed.

## Installation

### Claude Code (CLI skills)

```sh
claude plugin marketplace add https://github.com/airwallex/airwallex-marketplace
claude plugin install airwallex@airwallex-marketplace
```

Requires the `airwallex` CLI installed.

## Managing plugins

### List installed marketplaces and plugins

```sh
claude plugin marketplace list          # show all configured marketplaces
claude plugin list                      # show all installed plugins
```

### Update plugins

Re-sync from the marketplace and reinstall to pick up changes (works even if version number is unchanged):

```sh
claude plugin marketplace update airwallex-marketplace
claude plugin install airwallex@airwallex-marketplace    # reinstall to refresh cache
```

### Remove a plugin

```sh
claude plugin remove airwallex@airwallex-marketplace
```

### Remove the marketplace

```sh
claude plugin marketplace remove airwallex-marketplace
```

## Important Notice

> **Beta Feature.** These plugins are provided on an "as-is" basis for use with
> Airwallex APIs and CLI. By using these plugins, you acknowledge that:
>
> - Actions taken via your Airwallex API keys — whether initiated by you
>   directly or by an AI agent acting on your behalf — are governed by the
>   [Airwallex Terms of Service](https://www.airwallex.com/terms) accepted
>   during account creation and CLI authentication (`airwallex auth login`).
> - You are responsible for safeguarding your API keys and ensuring they are
>   used only in accordance with your intended purpose.
> - These plugins do **not** provide financial, legal, or tax advice. AI agent
>   outputs are probabilistic and should be reviewed by a qualified person
>   before acting on them.
> - These plugins do **not** initiate money-out actions (transfers, FX
>   conversions, or payouts) on your behalf. Where such capabilities exist in
>   the underlying API, the plugins are explicitly instructed not to execute
>   them.

## License

Apache-2.0 — see [LICENSE](LICENSE).
