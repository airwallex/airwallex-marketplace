---
name: airwallex-ai-provider-card-mit
description: Use this skill to guide an end-to-end implementation plan document for card-only Airwallex Payments flows used by AI providers that self-manage subscriptions, token balances, manual top-ups, and auto-recharge with merchant-initiated transactions. Covers Web JS SDK Split Card Elements, card binding, billing information collection, fraud-data checks, scheduled MIT consent for subscriptions, unscheduled MIT consent for recharge, webhooks, and merchant-owned ledger, invoice, receipt, retry, notification, and cancellation logic. Do not use for non-card methods, customer-initiated one-click checkout, hosted checkout, Drop-in, HPP, or provider-managed subscription products.
allowed-tools: Read Grep Bash Write Edit WebFetch
argument-hint: "[scenario: plan | recharge | auto-recharge | subscription | saved-card | renewal | full]"
---

# Airwallex Card MIT For AI Providers

## Source Policy

For current Airwallex API field names and SDK shapes, prefer the Airwallex docs MCP over any embedded or cached examples. Verify the relevant Airwallex behavior with MCP before producing API fields, SDK code, or integration claims.

Keep the solution card-only, Web JS SDK-only, and merchant-managed. Do not route the design to hosted checkout, Drop-in, HPP, non-card methods, or provider-managed subscription engines.

## Prerequisite

Install only the Airwallex Docs MCP connector needed for documentation lookup. Use the official setup guide:

https://www.airwallex.com/docs/developer-tools/ai/developer-connector.md

## No Embedded Code Examples In The Skill

Do not store SDK snippets, API request examples, HTML examples, JSON payloads, or curl commands in this skill. Use this skill for interaction design, decisions, field intent, state handling, and implementation checklists only.

## Plan First

Always settle the plan before implementation. Start by identifying the merchant business entries, shopper interaction flows, consent type for each flow, billing-information collection, fraud-data responsibilities, backend/webhook responsibilities, and open questions.

Do not move into code generation until the plan is accepted, unless the user explicitly asks to skip planning. If the user asks for code without an existing plan, first provide a compact plan and call out any assumptions that the code will depend on.

## Plan Artifact

The primary deliverable of this skill is an implementation plan document. First discuss and settle the plan in conversation. Then use this skill and the Airwallex docs MCP to create the plan artifact.

Default the plan filename to `airwallex-ai-provider-card-mit-plan.md` in the current project unless the user specifies another location. If a file already exists at that path, confirm before overwriting it.

The completed plan document should be suitable for user review before implementation begins. It should include MCP-verified JavaScript frontend examples for Split Card Elements and MCP-verified backend API call examples for the relevant PaymentIntent and PaymentConsent flows.

## Core Model

This skill designs merchant-initiated card-on-file flows for AI products:

- Token wallet top-ups, manual recharge, low-balance auto-recharge, and usage-driven charges use an `unscheduled` MIT consent.
- Fixed-amount, fixed-cycle subscriptions use a separate `scheduled` MIT consent.
- Every later charge must be created by the merchant backend and tied to the correct `payment_consent_id`.
- The shopper participates only when entering a card or authorizing a new agreement. Later recharge and renewal charges are merchant-initiated.

## Arguments

`$ARGUMENTS` selects which flow(s) to plan (default `plan`; scope the full plan collaboratively): `plan`, `recharge`, `auto-recharge`, `subscription`, `saved-card`, `renewal`, `full`. These map to the flows in Business Coverage below.

## Business Coverage

This skill guides the full merchant-managed card MIT plan for these AI provider business flows:

- Add card for recharge authorization.
- Manual recharge.
- Auto-recharge setup and runtime charging.
- Subscription signup with a new card.
- Subscription signup with a saved card.
- Subscription renewal.

The plan must define the shopper interaction, consent type, billing-information handling, fraud-data handling, backend workflow, webhook confirmation, ledger impact, and failure recovery for each applicable flow.

## Reference Loading

Load only the reference needed for the task:

- Read `references/interaction-flows.md` first when planning user journeys, product flows, or end-to-end implementation. Use its step-by-step ASCII flows before API mapping.
- Read `references/airwallex-objects-and-consents.md` for object mapping, consent selection, local data models, and charge routing.
- Read `references/frontend-split-card-elements.md` for Web JS SDK Split Card Elements UX, validation, 3DS, success, and failure handling.
- Read `references/server-and-webhooks.md` for backend endpoints, webhook processing, idempotency, ledger updates, invoices, receipts, and retries.
- Read `references/fraud-data-standards.md` whenever designing or implementing PaymentIntent creation or confirmation. The coding agent must use the Airwallex docs MCP to verify the current fraud-prevention data standards before writing API calls.
- Read `references/ux-copy-and-checklists.md` for authorization copy, settings screens, cancellation UX, and implementation checklists.

## Output Standard

For planning answers, include:

1. Step-by-step ASCII interaction map from the shopper's point of view.
2. Business flow coverage.
3. Consent strategy for each applicable flow.
4. Airwallex object mapping.
5. Frontend Split Card Elements flow if card collection is involved.
6. Backend and webhook responsibilities.
7. Fraud-prevention data fields to collect and pass, with MCP verification noted.
8. Merchant-owned ledger and reconciliation notes.
9. MCP-verified JS frontend examples and API call examples to include in the plan document.
10. Proposed `airwallex-ai-provider-card-mit-plan.md` filename and path for user confirmation.
11. Open assumptions or questions.

The plan should explicitly cover the relevant business flows from this set: add card for recharge authorization, manual recharge, auto-recharge setup, subscription signup with a new card, subscription signup with a saved card, and subscription renewal. Omit flows only when they are clearly out of scope for the merchant.

For implementation answers or final plan documents, produce code only after checking the relevant MCP docs. Treat frontend completion as provisional until the backend confirms the PaymentIntent and consent status.
