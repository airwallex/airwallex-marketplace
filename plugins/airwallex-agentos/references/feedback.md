# Submitting Feedback

Shared reference for the Airwallex AgentOS skills. Each skill's `## Feedback` section points here, so this guidance lives in **one place** — edit it here and every skill picks up the change. Skills cite it as `../../references/feedback.md`.

## Why this matters

Users rarely submit feedback on their own — they usually just drop off or live with the pain unless the experience is especially bad. And you (the agent) are often the first to notice when a skill's instructions, an API, or a workflow doesn't hold up in practice. So it's on you to surface feedback in the moment — both the user's reactions and your own observations.

## When to offer

Offer to submit feedback in either of these situations:

- **The user reacts to how a skill performed** — praise, frustration, a gap, a wrong result, a missing capability, or "this should work differently."
- **You hit something worth flagging while using the skill** — a genuine gap or missing capability, instructions that were unclear / contradictory / wrong, an API or tool that behaved unexpectedly or forced a workaround, or a step that repeatedly failed. You don't need the user to raise it first.

In both cases, briefly surface it and, if the user agrees, submit it through whichever channel is available in the environment.

## How to submit

- **CLI:** the entire message is passed as **one positional argument** — there is **no `submit` subcommand and no flags**.

  ```sh
  # ✅ Correct — message is the single positional argument
  airwallex feedback "Refund flow was confusing — couldn't tell which command to use"

  # ❌ Wrong — there is no `submit` subcommand and no --message/--request-id/--category flags
  airwallex feedback submit --message "..."
  ```

  The message must be 10–2000 characters and requires an authenticated session (`airwallex auth login`). The CLI generates its own request/trace IDs internally — do NOT pass one. It returns `{"submitted": true|false}` — report a failure or unauthenticated session honestly rather than claiming success.
- **MCP:** call the feedback tool exposed by your MCP client. Discover its exact name from the tool list (do NOT guess or invent one), then reference it by its fully-qualified `ServerName:tool_name` form.

## Rules

- **Ask first.** Offer once, keep it short, and never submit without the user's explicit go-ahead. Never nag.
- **No sensitive data.** Never put card numbers, CVV, balances, bank details, or customer PII in the feedback text — describe the problem without them.
- **Don't over-trigger.** Offer feedback only when there's a real signal — a user reaction, or a material issue/improvement you actually hit. Skip it for routine successful requests, one-off transient errors, and your own mistakes that aren't skill or product problems.
- **Scope.** This channel captures product/skill feedback for the team; it does not replace human judgement on whether a skill was effective.
