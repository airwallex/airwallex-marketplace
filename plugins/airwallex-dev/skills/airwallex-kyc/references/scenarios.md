# Airwallex Connected Account KYC: Overview & Onboarding Scenarios

> **Canonical source:** this file distills the official Airwallex docs: [Embedded KYC component](https://www.airwallex.com/docs/connected-accounts/onboarding/kyc-and-onboarding/embedded-kyc-component) (Scenario A) and [Hosted onboarding](https://www.airwallex.com/docs/connected-accounts/onboarding/kyc-and-onboarding/hosted-onboarding) (Scenario B). Those pages are authoritative. Prefer them if anything here drifts. The `theme` object in [theme.md](theme.md) is `@hidden` and has no public-doc equivalent.

> **Access token** (Authentication section only): see [Payments integration notes for AI agents](https://www.airwallex.com/docs/developer-tools/ai-agent-payments-integration.md)
> **Optional embedded-component theming**: see [theme.md](theme.md)

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Embedded Element vs Hosted Flow](#embedded-element-vs-hosted-flow)
- [Scenarios](#scenarios)
  - [Scenario A: Embedded KYC Component](#scenario-a-embedded-kyc-component-default)
  - [Scenario B: Hosted KYC Flow](#scenario-b-hosted-kyc-flow)
- [Gotchas & Troubleshooting](#gotchas--troubleshooting)

---

## Overview

The KYC process is a requirement to onboard underlying connected accounts to the platform. This can be done with Airwallex pre-built UI, either using the Embedded KYC Component or via a Hosted Flow.

**Key characteristics:**
- Choose `embedded-kyc-component` (default) when the user wants onboarding to stay inside their own page/UX with advanced theming, minimal redirects, and is comfortable initializing the Airwallex SDK.
- Choose `hosted-flow` when the user wants the least build effort, is happy to redirect users to an Airwallex-hosted form, and onboards business accounts only (hosted flow does not support individuals).
- If the user needs full control over every UI element, neither applies, point them to the Native API onboarding option.

---

## Prerequisites

KYC onboarding is a **connected-accounts (platform)** capability, not a plain Payments-merchant feature. Confirm with your Airwallex Account Manager that your **platform account** has:

- **Connected accounts** enabled, accounts are created via the Create Account API (save the response `data.id`, an `acct_xxx` id, as the connected account id).
- The chosen UI enabled, **Embedded Components** (Scenario A) or **Hosted flow onboarding** (Scenario B).

**Business accounts only**, connected-account KYC onboarding does not support individuals. A plain Payments-merchant account (without the connected-accounts/platform capability) cannot onboard business accounts and will fail; this is an account-tier prerequisite, not a code error.

---

## Embedded Element vs Hosted Flow

| Feature | Embedded Component | Hosted Flow |
|---------|--------------------|-------------|
| Integration effort | Highest | Lowest |
| UI control | Medium (colors, layout, logo) | Lowest (logo + CSS rules) |
| User stays on your site | Yes (iframe) | No (redirect) |
| Event handling | `element.on()` events | Redirect URL |

---

## Scenarios

### Scenario A: Embedded KYC Component (Default)

```
User → sees Embedded Component on platform page → completes form → submits → success event fires
```

1. Follow the steps in the [Embedded KYC component guide](https://www.airwallex.com/docs/connected-accounts/onboarding/kyc-and-onboarding/embedded-kyc-component) for integration and error handling. Follow all code samples provided.

#### Integration Summary

1. Backend creates the connected account (Create Account API; save the response `data.id`, the `acct_xxx` connected account id)
2. Client generates a **PKCE** pair, `code_verifier` (43-128 chars) + `code_challenge = BASE64URL(SHA256(code_verifier))`, method `S256` (RFC 7636)
3. Backend calls **Authorize a connected account** with scope **`w:awx_action:onboarding`**, the `x-on-behalf-of: <connected account id>` header, and the `code_challenge`; it returns an `authorization_code`
4. Client initializes the SDK with the returned auth code (`authCode`) + the `code_verifier`
5. Component mounts and handles user onboarding
6. Events fire on ready/success/error lifecycle states

> **Optional theming**: the embedded component supports a `theme` option for advanced color/typography customization. If the user has specific brand color requirements, see [theme.md](theme.md).

### Scenario B: Hosted KYC Flow

```
User → triggers sign up on platform page → redirect to hosted page → completes form → submits
```

Hosted Flow requires a **template open ID** from Airwallex, the identifier for your hosted-flow template, set up by your Account Manager. This is **not** the connected account id (`acct_xxx`); the two are different identifiers. Request the template open ID from the user before using it as `{{TEMPLATE_OPEN_ID}}`.

1. Request the `{{TEMPLATE_OPEN_ID}}` from the user
2. Follow the steps in the [Hosted onboarding guide](https://www.airwallex.com/docs/connected-accounts/onboarding/kyc-and-onboarding/hosted-onboarding) for integration and error handling. Follow all code samples provided.

#### Integration Summary

1. Backend creates the connected account (Create Account API; save the response `data.id`, the `acct_xxx` connected account id, **not** the template id)
2. Backend creates the hosted flow: `POST /api/v1/hosted_flows/create` with the `{{TEMPLATE_OPEN_ID}}` (template open ID), the connected `account_id`, and a `return_url`
3. Backend authorizes the flow: `POST /api/v1/hosted_flows/{id}/authorize`, the returned `url` is short-lived, so redirect the user promptly
4. Client redirects the user to the authorized `url`

> In the testing environment, open the hosted flow link in an incognito/private browser.

---

## Gotchas & Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| `cannot onboard business accounts` / onboarding unavailable | The account isn't a connected-accounts **platform** account, or Embedded Components / Hosted flow isn't enabled, see [Prerequisites](#prerequisites). Have your Account Manager enable it. Individuals aren't supported. |
| `Invalid value for return_url` | `http://localhost` is rejected. Use `http://127.0.0.1:<port>` locally, and an HTTPS URL in production. |
| Template ID not found / wrong id | Use the **template open ID** from your Account Manager, not the connected account id (`acct_xxx`); they are different identifiers. |
| `401`/`403` authorizing the KYC session (embedded) | Authorize with scope `w:awx_action:onboarding` **and** the `x-on-behalf-of: <connected account id>` header, and send the PKCE `code_challenge`. |
| Authorized hosted-flow URL fails after a delay | The authorized `url` is short-lived, redirect the user immediately after `hosted_flows/{id}/authorize`; re-authorize if it expired. |
