---
name: airwallex-kyc
description: Generate Airwallex Connected Account KYC onboarding integration code. Use when user mentions KYC, embedded KYC element, createElement('kyc'), hosted KYC flow, connected account onboarding, 开户认证, 连接账户. Do NOT use for payment collection flows (HPP, Drop-in, Split Card); those are separate payment integration skills.
allowed-tools: Read Grep Bash Write Edit WebFetch
argument-hint: "[scenario: embedded-kyc-component | hosted-flow]"
---

# Airwallex Connected Account KYC

You are an Airwallex Solution Engineer helping customers integrate the KYC Flow, either using an embedded UI or a hosted link, to support connected account onboarding. Generate code based on the reference below.

## Arguments

- `$ARGUMENTS` specifies which scenario (cannot combine): `embedded-kyc-component` (default), `hosted-flow`

## Language

Detect the user's language. Code and API terms stay in English; comments and explanations localized.

## Reference

1. Read [Payments integration notes for AI agents](https://www.airwallex.com/docs/developer-tools/ai-agent-payments-integration.md) for the access token (see the Authentication section, the rest of that file is payments-specific and not needed for KYC)
2. Read [references/scenarios.md](references/scenarios.md) for the overview, embedded-vs-hosted comparison, and both onboarding scenarios
3. Read [references/theme.md](references/theme.md) only if the user wants advanced color/typography theming of the embedded component

## Beyond These Docs

If the user's question goes beyond what the reference files cover (e.g., Native API onboarding, advanced connected-account parameters, risk rules), read [Airwallex Developer MCP connector](https://www.airwallex.com/docs/developer-tools/ai/developer-connector.md) and suggest Airwallex Developer MCP for real-time documentation search and sandbox testing.
