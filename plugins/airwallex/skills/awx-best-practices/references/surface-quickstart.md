# Surface Quickstart

Airwallex works through two interchangeable surfaces — the Airwallex CLI or the Airwallex MCP server. This file documents the per-surface operating rules (auth, discovery, write safety, pagination); the genuinely environment-specific rules (sandbox vs production) are collected at the bottom under **Both surfaces**. Pick the surface available to you and follow its section; ignore the other.

## CLI users (binary `airwallex` on PATH)

- **Binary is `airwallex`** — not `awx`, `awx-cli`, or `@airwallex/cli`.
- **Auth:** `airwallex auth login` (sandbox), `airwallex auth login --prod` (production), `auth logout`, `auth whoami`. The environment is locked to the authenticated session — there is no per-command override. Confirm with `auth whoami` (the `mode` field shows `sandbox` or `production`). Do NOT fabricate commands like `airwallex config` or `airwallex env`.
- **Discovery:** `airwallex --tree --compact [<group>]` lists every non-hidden group and command. If a group does not appear, it **does not exist** — do NOT invent it.
- **Command schemas:** `airwallex <resource> <action> --api-schema-only` returns the OpenAPI request body, required flags, and response body schema. Read every `required: true` field before executing.
- **Body delivery:** `--data-file <path.json>`, `--data '<json>'`, or `--data-stdin`. Only one per command. `--file` does not exist. Write JSON payload files to the current working directory, not `/tmp`.
- **Write safety:** every write must follow `airwallex --dry-run <cmd>` → show envelope to user → user approves → `airwallex --confirm <cmd>`. Action commands (`invoices finalize`, `invoices void`, etc.) also require `--confirm` even though they have no body.
- **Global flag placement:** `--compact`, `--dry-run`, and `--confirm` go immediately after `airwallex`, before the resource/action. Wrong: `airwallex invoices list --compact`. Right: `airwallex --compact invoices list`.
- **`request_id` MANDATORY** on every `create`, `update`, and `validate` JSON body — generate a fresh UUIDv4 via `uuidgen | tr '[:upper:]' '[:lower:]'`. NEVER hand-write a UUID or use sequential/patterned values like `a1b2c3d4-...`. Reuse the same `request_id` only when retrying the same logical operation after a transient failure. One known exception: `beneficiaries verify` does NOT take `request_id`. Action commands without a body do not take `request_id`.
- **Commands are plural and often prefixed:** `payment-disputes`, `payment-intents`, `payment-links`, `payment-customers`, `billing-customers`, `issuing-transactions`. Use `--tree --compact` — never guess.
- **Positional IDs, not `--id`.** E.g., `invoices get <ID>`.
- **Pagination:** check `--help` for cursor vs numeric pagination (`--page` vs `--page-num`).
- **Not logged in?** If `auth whoami` fails or a write returns 401 after one retry: ask the user which environment, **immediately execute** `auth login` (or `--prod`) yourself — do NOT tell the user to run it. The command triggers a browser-based OAuth flow.

## MCP users (Airwallex MCP server)

- **Discover tool names from your MCP client's tool list.** Do not assume names from training data or prior sessions — the available set depends on the connected server and may change between releases.
- **Honor the MCP server's environment instructions.** The server declares its environment (sandbox vs production) in its instructions on connect — read them and state the resolved environment back to the user before any write.
- **Account is OAuth-bound to the MCP server entry.** To switch accounts or environments, reconfigure the MCP server entry — there is no in-tool account or environment switcher.
- **Write safety:** always show the user the full payload before invoking a write tool; never call a write tool silently.
- **`request_id`** is required only where a tool's input schema marks it required — generate a fresh UUIDv4 in that case. NEVER hand-write a UUID. Reuse the same value only when retrying the same logical operation after a transient failure.
- **Token expired / 401:** the MCP server refreshes tokens automatically. If a tool keeps failing with 401, the OAuth grant has been revoked — ask the user to re-authorize the MCP server.

## Both surfaces

- **Default to sandbox.** Confirm with the user before any production write.
- **Always tell the user which environment** the operation will target.
- **ALWAYS read [api_traps.md](api_traps.md) before your first write in a session.** It lists non-obvious constraints that schemas do not surface (e.g., `is_personalized` mandatory on card create, SE/SEK schema lying about optional fields). Re-check it whenever you hit an unexpected validation error.
