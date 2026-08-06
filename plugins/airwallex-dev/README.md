# airwallex-dev plugin

Teaches AI coding agents how to build Airwallex integrations: checkout pages, card elements, onboarding flows, and subscription billing.

Where the `airwallex-agentos` plugin provides workflow and reference skills for conversations, `airwallex-dev` skills are aimed at the code in your project. Five of them generate it directly: they pick a scenario, read only the reference files that scenario needs, and produce the routes, components, and config wired into the project. `airwallex-ai-provider-card-mit` is the exception and produces an implementation plan document instead, deliberately holding no code examples of its own.

## Skills

| Skill | Category | Description |
| --- | --- | --- |
| airwallex-hpp | Payments | Hosted Payment Page: redirect-based checkout hosted by Airwallex |
| airwallex-dropin | Payments | Drop-in Element: embedded UI supporting multiple payment methods |
| airwallex-split-card | Payments | Split Card Element: separate card number, expiry, and CVC inputs for full UI control |
| airwallex-billing-checkout | Billing | Billing Hosted Checkout for subscriptions, one-off payments, and card-saving (SETUP) |
| airwallex-kyc | Connected Accounts | Connected account KYC onboarding via embedded component or hosted link |
| airwallex-ai-provider-card-mit | Payments | Implementation plan for card-on-file MIT flows used by AI providers (top-ups, auto-recharge, subscriptions) |

Most skills take a `scenario` argument (see each `SKILL.md` `argument-hint`) so the agent loads only what the task needs. `airwallex-billing-checkout` is the exception: it has no `argument-hint` and instead walks through an interactive intake questionnaire to decide which sections to output.

### Skill file structure

Each skill has a `SKILL.md` holding the routing logic and generation rules, plus a `references/` folder with SDK snippets, API schemas, and styling options that the agent loads progressively, only when needed:

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
├── .cursor-plugin/plugin.json
├── .mcp.json
├── .cursor-mcp.json
├── README.md
└── skills/
    └── <skill-name>/
        ├── SKILL.md
        └── references/
            └── ...
```

## Prerequisites

- **Sandbox account:** Airwallex sandbox credentials for testing generated integrations.
- **Airwallex Developer MCP connector:** lets the agent search current API docs and call sandbox APIs. Optional for most skills, which fall back to it only when a question goes beyond the bundled references, but **required** by `airwallex-ai-provider-card-mit`, which verifies every API field and SDK shape against MCP before writing them. See [Airwallex Developer MCP connector](https://www.airwallex.com/docs/developer-tools/ai/developer-connector.md).
