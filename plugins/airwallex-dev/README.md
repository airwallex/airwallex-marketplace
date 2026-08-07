# airwallex-dev plugin (Claude Code / Cursor / Codex)

Teaches Claude, Cursor, and Codex how to build Airwallex integrations in your codebase: checkout pages, card elements, onboarding flows, and subscription billing.

Where `airwallex-agentos` provides workflow and reference skills for conversations, `airwallex-dev` skills work on the code in your project. Five of them generate that code directly. Each one picks a scenario, reads only the reference files that scenario needs, and writes the routes, components, and config into the project. `airwallex-ai-provider-card-mit` is the exception. It produces an implementation plan document instead and keeps no code examples of its own.

## Skills

| Skill | Category | Description |
| --- | --- | --- |
| [airwallex-hpp](skills/airwallex-hpp/SKILL.md) | Payments | Hosted Payment Page: redirect-based checkout hosted by Airwallex |
| [airwallex-dropin](skills/airwallex-dropin/SKILL.md) | Payments | Drop-in Element: embedded UI supporting multiple payment methods |
| [airwallex-split-card](skills/airwallex-split-card/SKILL.md) | Payments | Split Card Element: separate card number, expiry, and CVC inputs for full UI control |
| [airwallex-billing-checkout](skills/airwallex-billing-checkout/SKILL.md) | Billing | Billing Hosted Checkout for subscriptions, one-off payments, and card-saving (SETUP) |
| [airwallex-kyc](skills/airwallex-kyc/SKILL.md) | Connected Accounts | Connected account KYC onboarding via embedded component or hosted link |
| [airwallex-ai-provider-card-mit](skills/airwallex-ai-provider-card-mit/SKILL.md) | Payments | Implementation plan for card-on-file MIT flows used by AI providers (top-ups, auto-recharge, subscriptions) |

Most skills take a `scenario` argument (see each `SKILL.md` `argument-hint`) so the agent loads only what the task needs. `airwallex-billing-checkout` is the exception: it has no `argument-hint` and instead walks through an interactive intake questionnaire to decide which sections to output.

### Skill file structure

Each skill has a `SKILL.md` holding the routing logic and generation rules, plus a `references/` folder with SDK snippets, API schemas, and styling options that the agent loads only when it needs them:

```
skills/<skill-name>/
├── SKILL.md                       # Scenario routing, generation rules, reference index
└── references/
    └── ...                        # SDK snippets, API schemas, styling, error handling
```

## Plugin structure

```
airwallex-dev/
├── .claude-plugin/plugin.json
├── .codex-plugin/plugin.json
├── .cursor-plugin/plugin.json
├── .cursor-mcp.json
├── .mcp.json
├── assets/
│   ├── icon.svg
│   └── logo.svg
├── README.md
└── skills/
    └── <skill-name>/
        ├── SKILL.md
        └── references/
            └── ...
```

## Prerequisites

- **Sandbox account:** Airwallex sandbox credentials for testing generated integrations.
- **Airwallex Developer MCP connector:** lets the agent search current API docs and call sandbox APIs. Most skills fall back to it only when a question goes beyond the bundled references, but `airwallex-ai-provider-card-mit` **requires** it and verifies every API field and SDK shape against MCP before writing them. See [Airwallex Developer MCP connector](https://www.airwallex.com/docs/developer-tools/ai/developer-connector).
