---
name: e9-cli
description: Connect Cursor to engine9 MCP — log in first via mcp_auth, set account scope with /e9a, and handle /e9 command-style interactions including person search and task scheduling.
---

# Engine9 CLI

Use this skill when setting up or troubleshooting an MCP connection to an engine9 server from Cursor (or another MCP client), and when handling `/e9` and `/e9a` command-style requests.

For MCP tool selection and invocation strategy, see [e9-mcp](../e9-mcp/SKILL.md).

**On any MCP tool error, follow [e9-mcp — MCP tool errors — stop immediately](../e9-mcp/SKILL.md#mcp-tool-errors--stop-immediately).** Do not call downstream account-scoped tools after a failed `account`, `task`, `search`, or similar call.

## Step 0 — Log in (always first)

**Every `/e9` request starts here.** Do not grep, curl, read config files, start servers, or run CLI commands to "figure out" auth — just log in.

1. Call **`mcp_auth`** on the engine9 MCP server with **`{}`**.
2. Cursor opens a **sign-in prompt for the user** — wait for them to complete it.
3. Call **`ok`**, then **`user`**, to confirm signed in.
4. Proceed with the user's request.

If `user` already succeeds this session, skip to step 4.

**Login is only `mcp_auth` → user completes prompt → verify with `user`.** Nothing else.

## Troubleshooting after login

Only if login succeeded (`user` works) but other MCP calls still fail:

- Ask the user to reload MCP servers in Cursor (Settings → MCP) if config recently changed.
- For local dev server issues, see [Server endpoint and startup](#server-endpoint-and-startup) below.

Do **not** use shell commands or config greps as a substitute for step 0 login.

### Local dev without OAuth

Project `.cursor/mcp.json` may define `engine9.local_noauth` with `Authorization: Bearer localdev`. That entry skips the OAuth prompt; it is not a login workaround when the OAuth server is configured — still call `mcp_auth` on the correct server entry first.

## MCP-only discovery

When handling `/e9` or `/e9a` requests, **do not read local workspace code** to discover plugins, worker methods, paths, or options. Local source does not reliably match the connected MCP server or the account's installed plugins. Use MCP tool schemas, MCP `account` (cached as `engine9.plugins`), and MCP `task` / `worker_invoke` only. See [e9-mcp — MCP-only discovery](../e9-mcp/SKILL.md#mcp-only-discovery--do-not-use-local-code).

## What this skill covers

1. Cursor MCP config (`mcp.json`, `permissions.json`)
2. Auth expectations (OAuth vs `localdev` bearer)
3. `/e9a` account scope and plugin cache
4. `/e9` session behavior, search parsing, and task scheduling

## Server endpoint and startup

Engine9 MCP is exposed at `POST /mcp` (with `GET /mcp/health` for health checks).

- Preferred server startup: run the main API (`node api/index.js`) so MCP and other APIs share one port.
- Standalone MCP startup: from `engine9/server`, run `npm run mcp`.
- Default standalone MCP URL:
  - `http://127.0.0.1:3334/mcp` (when TLS is off)
  - `https://127.0.0.1:3334/mcp` (when `ENGINE9_SSL_CERT_PATH` is set)

Quick health checks:

- HTTP: `curl -sS http://127.0.0.1:3334/mcp/health`
- HTTPS (self-signed local cert): `curl -k -sS https://127.0.0.1:3334/mcp/health`

## Cursor MCP configuration

Create or update `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "engine9.ai": {
      "url": "https://<your-api-host>/mcp"
    }
  }
}
```

Notes:

- The server key (`engine9.ai` above) should match engine9's MCP server name (`ENGINE9_MCP_SERVER_NAME`, default `engine9.ai`).
- If local standalone MCP runs without TLS, use `http://127.0.0.1:3334/mcp`.
- After config changes, reload MCP servers in Cursor.

## Cursor permissions configuration

Create or update `~/.cursor/permissions.json`:

Allow all engine9 tools:

```json
{
  "mcpAllowlist": ["engine9.ai:*"]
}
```

Or allow a subset:

```json
{
  "mcpAllowlist": ["engine9.ai:ok", "engine9.ai:user", "engine9.ai:account", "engine9.ai:search", "engine9.ai:task"]
}
```

## Auth expectations

- MCP requests to `/mcp` require Bearer auth (OAuth or `localdev`).
- In Cursor, normal flow is OAuth connect on first request.
- For local/offline testing, `Authorization: Bearer localdev` can be used with server-side localdev account configuration (`ENGINE9_MCP_LOCALDEV_ACCOUNTS`).

## `/e9a` — account scope

Primary command forms:

- `/e9a <lookup>` — single exact account id (default)
- `/e9a parent <parent_id>` — active child accounts of `<parent_id>` (comma-separated parent ids allowed)
- `/e9a all` — all active accounts

Examples: `/e9a test`, `/e9a engine9`, `/e9a parent frakture_master`, `/e9a all`

The CLI bin script (`server/bin/e9a`) writes `.e9_parameters` for subsequent `e9` worker runs. For `parent` and `all`, it resolves account ids from server account config and writes `-a id1,id2,...`. For a single lookup it writes `-a <lookup>` as before.

### Required behavior

**Default `/e9a <lookup>`:**

1. Do not call `user` (or any account lookup tool) to resolve `<lookup>`.
2. Do not perform fuzzy matching, alias matching, or ambiguity checks.
3. Set the provided value as active account scope immediately.
4. Call MCP `account` with `{ "account_id": "<lookup>" }`.
5. On success (`{ ok: true, plugins: [...] }`), persist the returned `plugins` array for subsequent `/e9` requests.
6. On failure (`isError: true`, or text like `Cannot connect to the … database` / `Not authorized for account`): **stop**. Report the error. Do not cache plugins. Do not proceed to `/e9 task` or other account-scoped calls.
7. Reuse stored account and plugins unless the user runs another `/e9a`.

**`/e9a parent <parent_id>` or `/e9a all`:**

1. Resolve account ids from server account config (active = `disabled` not true; parent mode matches `parent_ids`).
2. Set `engine9.account_ids` to the full list; set `engine9.account_id` to the first id.
3. Do not load plugins for every account; report the resolved ids and count.

### Session persistence

Persist:

- `engine9.account_id` — exact lookup value
- `engine9.account_ids` — single-item array with that same value
- `engine9.plugins` — array from MCP `account` on success (replace entirely on each `/e9a`)

Rules:

- Always replace prior account scope and plugin list on each `/e9a`.
- Do not call `account` again on every `/e9` subcommand if `engine9.plugins` is already loaded for the current `engine9.account_id`; only refresh when the user runs `/e9a` again or explicitly asks to reload plugins.

### Response requirements

On success: confirm the exact `account_id`, state how many plugins were loaded, and note that future `/e9` requests will use this account.

On MCP failure: confirm `account_id` was set, report the MCP error verbatim, note that plugins were not cached, and **stop** — do not run further account-scoped tools in the same request.

## `/e9` command contract

Supported forms:

- `/e9` — bootstrap (connectivity, identity, account scope)
- `/e9 search foo@bar.com`
- `/e9 segment list` — MCP `segment` with `command: list`
- `/e9 segment build <segment_id|definition_path>` — MCP `segment` with `command: build`
- `/e9 task <plugin-path-or-alias> <method> [options...]`

### Session and account requirements

- Every MCP call that is account-scoped MUST include `account_id`.
- If `account_id` is already known in session context, reuse it.
- If `account_id` is not known, request it from the user after `/e9` before running account-scoped tools, or direct them to `/e9a <account_id>`.
- Do not guess `account_id`; require explicit user/session context.

### Recommended `/e9` bootstrap flow

0. **[Step 0 — Log in](#step-0--log-in-always-first)** — `mcp_auth` → user completes prompt → `user`.
1. `/e9` with no subcommand:
   - Confirm signed in via MCP `user`.
   - Confirm active `account_id` (from session or user); if missing, suggest `/e9a <account_id>`.
   - If `engine9.plugins` is not cached for that account, call MCP `account` or suggest `/e9a`.
2. `/e9 search ...`:
   - Ensure active `account_id` exists (or request it / suggest `/e9a`).
   - Parse search terms into MCP `search` arguments.
   - Call `search` with mapped filters and `account_id`.
3. `/e9 task ...`:
   - Require `engine9.account_id` and `engine9.plugins` (run `/e9a` first if missing).
   - If `/e9a` or MCP `account` failed for this account, **stop** — do not call `task`.
   - Resolve the plugin path from cached plugins before calling MCP `task`.
   - Pass the canonical **plugin path** (colon submodule form), not legacy Frakture dotted paths.
   - If MCP `task` returns `isError: true` or a fatal error message, **stop** — do not retry or call other account tools.
4. `/e9 segment list`:
   - Ensure active `account_id` exists (or request it / suggest `/e9a`).
   - Call MCP `segment` with `{ "command": "list", "account_id": "<account_id>" }`.
5. `/e9 segment build ...`:
   - Ensure active `account_id` exists.
   - Call MCP `segment` with `command: build` and either `segment_id` or `definition_path` from the user args.

## `/e9 search` parsing rules

For tokenized arguments after `search`:

- Tokens containing `@` MUST be treated as email inputs and mapped to `emails`.
- Tokens that are straight integers (digits only, e.g. `12345`) MUST be treated as `person_ids`.
- Other terms may map to other supported filters (`phones`, `given_names`, `last_names`) based on context.

When multiple values exist for a filter, pass them as arrays.

Example mapping for `/e9 search foo@bar.com 12345`:

```json
{
  "account_id": "test",
  "emails": ["foo@bar.com"],
  "person_ids": [12345],
  "limit": 10
}
```

## `/e9 task` parsing and path resolution

Command form: `/e9 task <plugin-path-or-alias> <method> [options...]`

Examples:

- `/e9 task renxt/people listCustomFields`
- `/e9 task @frakture-com/channelbots/RENxtBot:People listCustomFields`
- `/e9 task @engine9/plugins/e9workers:SQLWorker queries`

### Path forms (in priority order)

1. **Canonical plugin path with colon submodule** (preferred for `task.path`):
   - `@frakture-com/channelbots/RENxtBot:People`
   - `@engine9/plugins/e9workers:SQLWorker`
2. **Slash alias shorthand** (resolve via `engine9.plugins` before MCP):
   - `renxt/people` → find plugin where `metadata.metadata.alias` (or `metadata.alias`) is `renxt`, submodule `People` from `metadata.submodules`
3. **Bare plugin path** (no submodule): `@frakture-com/channelbots/RENxtBot` — only when the method lives on the plugin root

Do **not** pass legacy remote job paths such as `channelbots.RENxtBot.People`.

### Resolution algorithm

Before MCP `task`:

1. Require `engine9.plugins` from `/e9a` (or call MCP `account` once if missing).
2. If the user path is already `plugin.path` or `plugin.path:Submodule` and matches an installed plugin, use it.
3. If the path is `alias/submodule` (e.g. `renxt/people`), scan `engine9.plugins`:
   - Match alias (case-insensitive) on `plugin.metadata.metadata.alias` or `plugin.metadata.alias`
   - Match submodule (case-insensitive) on keys of `plugin.metadata.submodules`
   - Build `{plugin.path}:{Submodule}` (preserve submodule casing from metadata keys)
4. Call MCP `task` with `account_id` from `engine9.account_id`, resolved `path`, and `method`.

The MCP server also resolves slash/colon paths against the account plugin table if the agent passes shorthand.

### Example MCP `task` payload

User: `/e9 task renxt/people listCustomFields`  
Account: `bfred_lambda_legal` (from `/e9a`)

```json
{
  "account_id": "bfred_lambda_legal",
  "path": "@frakture-com/channelbots/RENxtBot:People",
  "method": "listCustomFields",
  "label": "renxt/people listCustomFields"
}
```

Use `action: "check"` with `flow_run_id` and optional `task_run_ids` from the schedule response to poll status.
