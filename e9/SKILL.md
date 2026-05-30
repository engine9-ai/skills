---
name: e9
description: Connect Cursor (and similar MCP clients) to an engine9 MCP server and support `/e9` command-style interactions including account-aware person search.
---

# Engine9 MCP

Use this skill when setting up or troubleshooting an MCP connection to an engine9 server from Cursor (or another MCP client), and when handling `/e9` command-style requests (especially `/e9 search ...`).

## What this skill does

1. Documents how to connect a client to engine9 MCP.
2. Shows required Cursor config (`mcp.json`, `permissions.json`).
3. Explains auth expectations (OAuth vs `localdev` bearer).
4. Defines `/e9` session behavior, including required `accountId` and cached plugin list handling.
5. Defines `/e9 search` argument parsing and MCP `search` payload mapping.
6. Defines `/e9 schedule` path resolution against `engine9.plugins` (plugin paths and colon/slash forms).
7. Lists currently registered tools, including `user`, `account`, `search`, and `schedule`.

## `/e9` command contract

This skill supports a command flow along the lines of:

- `/e9`
- `/e9 search foo@bar.com`

### Session and account requirements

- Every MCP call that is account-scoped (including `search`) MUST include `accountId`.
- If `accountId` is already known in session context, reuse it.
- If `accountId` is not known, request it from the user after `/e9` before running account-scoped tools, or direct them to `/e9a <accountId>` to set scope.
- Do not guess `accountId`; require explicit user/session context.

### Session plugin cache (from `/e9a`)

After `/e9a <accountId>`, session should include:

- `engine9.accountId` — active account
- `engine9.plugins` — array returned by MCP `account` for that account (each entry: plugin row + `metadata`)

When answering questions about which plugins are installed, plugin paths, marketplace names, or plugin UUIDs:

- Prefer `engine9.plugins` from session; do not call MCP `account` again unless the user asks to reload or account scope changed.
- If `engine9.accountId` is set but `engine9.plugins` is missing, run MCP `account` once with `{ "account_id": "<accountId>" }` and cache the result, or ask the user to run `/e9a` again.

### Recommended `/e9` bootstrap flow

1. `/e9` with no subcommand:
   - Verify connectivity via `ok`.
   - Verify identity/access via `user`.
   - Confirm active `accountId` (from session or user); if missing, suggest `/e9a <accountId>`.
   - If `engine9.plugins` is not cached for that account, call MCP `account` or suggest `/e9a` to load plugins.
2. `/e9 search ...`:
   - Ensure active `accountId` exists (or request it / suggest `/e9a`).
   - Parse search terms into MCP `search` arguments.
   - Call `search` with mapped filters and `accountId`.
3. `/e9 schedule ...`:
   - Require `engine9.accountId` and `engine9.plugins` (run `/e9a` first if missing).
   - Resolve the plugin path from cached plugins before calling MCP `schedule`.
   - Pass the canonical **plugin path** (colon submodule form), not legacy Frakture dotted paths.

## `/e9 schedule` parsing and path resolution

Command form:

- `/e9 schedule <plugin-path-or-alias> <method> [options...]`

Examples:

- `/e9 schedule renxt/people listCustomFields`
- `/e9 schedule @frakture-com/channelbots/RENxtBot:People listCustomFields`
- `/e9 schedule @engine9/plugins/e9workers:SQLWorker queries`

### Path forms (in priority order)

1. **Canonical plugin path with colon submodule** (preferred for `schedule.path`):
   - `@frakture-com/channelbots/RENxtBot:People`
   - `@engine9/plugins/e9workers:SQLWorker`
2. **Slash alias shorthand** (resolve via `engine9.plugins` before MCP):
   - `renxt/people` → find plugin where `metadata.metadata.alias` (or `metadata.alias`) is `renxt`, submodule `People` from `metadata.submodules`
3. **Bare plugin path** (no submodule): `@frakture-com/channelbots/RENxtBot` — only when the method lives on the plugin root

Do **not** pass legacy remote job paths such as `channelbots.RENxtBot.People`. Those are produced internally when talking to the Frakture job-list API.

### Resolution algorithm (agent)

Before `schedule`:

1. Require `engine9.plugins` from `/e9a` (or call MCP `account` once if missing).
2. If the user path is already `plugin.path` or `plugin.path:Submodule` and matches an installed plugin, use it.
3. If the path is `alias/submodule` (e.g. `renxt/people`), scan `engine9.plugins`:
   - Match alias (case-insensitive) on `plugin.metadata.metadata.alias` or `plugin.metadata.alias`
   - Match submodule (case-insensitive) on keys of `plugin.metadata.submodules`
   - Build `{plugin.path}:{Submodule}` (preserve submodule casing from metadata keys)
4. Call MCP `schedule` with `account_id: engine9.accountId`, resolved `path`, and `method`.

The MCP server also resolves slash/colon paths against the account plugin table if the agent passes shorthand.

### Example MCP `schedule` payload

User: `/e9 schedule renxt/people listCustomFields`  
Account: `bfred_lambda_legal` (from `/e9a`)

```json
{
  "account_id": "bfred_lambda_legal",
  "path": "@frakture-com/channelbots/RENxtBot:People",
  "method": "listCustomFields",
  "label": "renxt/people listCustomFields"
}
```

## `/e9 search` parsing rules

For tokenized arguments after `search`:

- Tokens containing `@` MUST be treated as email inputs and mapped to `emails`.
- Tokens that are straight integers (digits only, e.g. `12345`) MUST be treated as `person_ids`.
- Other terms are at the agent's discretion and may map to other supported filters (`phones`, `given_names`, `last_names`) based on context.

When multiple values exist for a filter, pass them as arrays.

## Server endpoint and startup

Engine9 MCP is exposed at `POST /mcp` (with `GET /mcp/health` for health checks).

- Preferred server startup: run the main API (`node api/index.js`) so MCP and other APIs share one port.
- Standalone MCP startup: from `engine9/server`, run:
  - `npm run mcp`
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
    "engine9.io": {
      "url": "https://<your-api-host>/mcp"
    }
  }
}
```

Notes:

- The server key (`engine9.io` above) should match engine9's MCP server name (`ENGINE9_MCP_SERVER_NAME`, default `engine9.io`).
- If local standalone MCP runs without TLS, use `http://127.0.0.1:3334/mcp`.
- After config changes, reload MCP servers in Cursor.

## Cursor permissions configuration

Create or update `~/.cursor/permissions.json`:

Allow all engine9 tools:

```json
{
  "mcpAllowlist": ["engine9.io:*"]
}
```

Or allow a subset:

```json
{
  "mcpAllowlist": ["engine9.io:ok", "engine9.io:user", "engine9.io:account", "engine9.io:search", "engine9.io:schedule"]
}
```

## Auth expectations

- MCP requests to `/mcp` require Bearer auth (OAuth or `localdev`).
- In Cursor, normal flow is OAuth connect on first request.
- For local/offline testing, `Authorization: Bearer localdev` can be used with server-side localdev account configuration (`ENGINE9_MCP_LOCALDEV_ACCOUNTS`).

## Available tools (current)

The engine9 MCP server currently registers these tools:

1. `ok`  
   Returns server status/time and whether request is authenticated.
2. `plugin_id`  
   Computes `plugin_id` from `account_id` and `remote_plugin_id`.
3. `input_id`  
   Computes `input_id` from `plugin_id` and `remote_input_id`.
4. `user` (often invoked as `/user` in slash-style clients)  
   Returns current authenticated user details and account access.
5. `account`  
   Lists plugins installed on an account with marketplace metadata merged onto each plugin.  
   - Required: `account_id`  
   - Returns: `{ ok: true, plugins: [...] }`  
   - Typically invoked during `/e9a` bootstrap and cached as `engine9.plugins` for the session.
6. `worker_invoke`  
   Invokes allowlisted worker methods (account-scoped).
7. `eql`  
   Runs a query from an EQL object and returns SQL + rows.
8. `search`  
   Searches people by metadata filters and returns person summaries/related records.
   - Supported filters: `emails`, `person_ids`, `phones`, `given_names`, `last_names`
   - Required: `accountId`
   - Optional: `limit` (max 1000, default 10)
9. `schedule`  
   Schedules tasks via flow path or direct worker task spec.  
   - Required: `account_id`  
   - Single task: `path` (plugin path or `alias/submodule`), `method`, optional `options`  
   - Or: `flow_path` to a JSON5 flow file  
   - Resolves `path` against installed account plugins (same data as MCP `account`)  
   - Prefer colon paths: `@frakture-com/channelbots/RENxtBot:People`, `@engine9/plugins/e9workers:SQLWorker`

## Suggested quick validation flow

1. Call `ok` to verify connectivity and auth state.
2. Call `user` (or `/user`) to confirm signed-in identity and account access.
3. Run `/e9a <accountId>` (or call `account`) to set scope and cache `engine9.plugins`.
4. Call `search` with a known email/person id to validate account-scoped data access.

Example `search` payload:

```json
{
  "accountId": "test",
  "emails": "person@example.com",
  "limit": 5
}
```

Example mapping for `/e9 search foo@bar.com 12345`:

```json
{
  "accountId": "test",
  "emails": ["foo@bar.com"],
  "person_ids": [12345],
  "limit": 10
}
```
