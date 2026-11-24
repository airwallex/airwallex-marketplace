# Fraud Data Standards

## Table of Contents

- [Purpose](#purpose)
- [Required Coding-Agent Behavior](#required-coding-agent-behavior)
- [Data Categories To Check](#data-categories-to-check)
- [AI Provider Mapping](#ai-provider-mapping)
- [Implementation Guardrail](#implementation-guardrail)

---

## Purpose

Every PaymentIntent create or confirm design must include a fraud-data review. The coding agent must query the Airwallex docs MCP for the latest "Payment data for enhanced fraud protection" guidance before writing API calls.

Do not rely only on this reference for exact request syntax. This reference defines the checklist and AI-provider mapping. Use MCP for the current field paths, supported parameters, and endpoint placement.

## Required Coding-Agent Behavior

Before generating code that creates or confirms a PaymentIntent:

1. Query Airwallex MCP for the current fraud-prevention data standards.
2. Identify which data categories apply to the merchant scenario.
3. Add all available applicable fields to the PaymentIntent create or confirm flow.
4. Explain any category intentionally omitted and why it is not applicable or not available.
5. Keep field collection privacy-aware and avoid inventing unavailable customer data.

## Data Categories To Check

Customer information:

- Merchant customer identifier.
- Customer first and last name, when available.
- Customer email.
- Customer phone number.

Billing information:

- Billing first and last name.
- Billing email.
- Billing address, including city, state or province, country code, and postal code.

For Split Card Elements, billing information is not captured by the card number, expiry, or CVC iframes. The merchant frontend must collect or confirm billing details separately and pass them to the backend for the PaymentIntent create or confirm flow according to the current Airwallex MCP docs.

Product information:

- Product or plan name.
- Product SKU or merchant product identifier.
- Quantity.
- Description.
- Unit price.
- Product URL, if the merchant has one.

Device and browser information:

- Device identifier or order-session identifier.
- Shopper IP address.
- Accept header.
- Browser user agent.
- JavaScript enabled signal when available.
- Language or locale.
- Screen width, screen height, and color depth.
- Timezone.

Shipping information:

- Only applies when the AI provider sells or ships physical goods.
- Omit for pure SaaS, API token, balance recharge, or digital subscription flows unless the merchant has a real physical delivery component.

Offline-to-online signal:

- Only applies to O2O flows such as offline scan-to-pay.
- For standard online AI subscriptions and token recharge, mark this as not applicable.

## AI Provider Mapping

Subscription signup:

- Customer information from the merchant user profile.
- Billing information from merchant-owned fields shown alongside Split Card Elements, or from a confirmed existing billing profile.
- Product information from the selected plan.
- Device and browser information from the current checkout session.
- No shipping information for digital-only plans.

Manual recharge:

- Customer information from the merchant user profile.
- Billing information from the saved card profile if available; if a new card is entered, collect billing fields alongside Split Card Elements.
- Product information from the recharge package or wallet top-up product.
- Device and browser information if the user is in session; for later merchant-side processing, use the best available risk data supported by the current Airwallex docs.

Auto-recharge:

- Customer information from the merchant user profile.
- Billing information from the saved card profile if available. Missing billing details should be collected before enabling auto-recharge where practical.
- Product information from the auto-recharge wallet top-up configuration.
- Device data requirements must be checked with MCP because the user is not in session at charge time.

Subscription renewal:

- Customer information from the merchant user profile.
- Billing information from the saved card profile if available.
- Product information from the subscription plan and billing period.
- Device data requirements must be checked with MCP because the user is not in session at charge time.

## Implementation Guardrail

When a generated implementation creates or confirms a PaymentIntent without fraud data, the coding agent must flag that as incomplete unless the response explicitly documents why the relevant data is unavailable or inapplicable.

For this skill's target use cases, customer, billing, product, and applicable device data should be treated as part of the normal payment integration checklist, not as optional polish.
