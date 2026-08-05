---
name: e9-mcp
description: Use the engine9 MCP server — log in first via mcp_auth, MCP-only discovery (never local code), prefer native tools over task, discover plugin methods via account when no native match exists, and invoke task as the catch-all for worker execution.
---

# Engine9 MCP

Use this skill when calling engine9 MCP tools from Cursor or another MCP client. For `/e9` and `/e9a` slash commands and client setup, see [e9-cli](../e9-cli/SKILL.md). For the Prefect-compatible REST Task API, see [e9-tasks-api](../e9-tasks-api/SKILL.md).

## Step 0 — Log in (always first)

**Every `/e9` or MCP request starts here.** Do not grep, curl, read config files, start servers, or run CLI commands to "figure out" auth — just log in.

1. Call **`mcp_auth`** on the engine9 MCP server with **`{}`**.
2. Cursor opens a **sign-in prompt for the user** — wait for them to complete it.
3. Call **`ok`**, then **`user`**, to confirm signed in.
4. Proceed with the user's request.

If `user` already succeeds this session, skip to step 4.

**Login is only `mcp_auth` → user completes prompt → verify with `user`.** Nothing else.

| Do not use for login | Why |
|---------------------|-----|
| Shell commands (`curl`, `grep`, `npm run mcp`, …) | Not login; wastes time |
| Reading `mcp.json`, `STATUS.md`, `.env`, local config | Not login |
| `e9 oauth token`, local CLI, repo worker code | Wrong path for Cursor MCP |
| Reporting "MCP unavailable" before `mcp_auth` | Always try `mcp_auth` first |

After login, if MCP tools still fail, see [e9-cli — troubleshooting](../e9-cli/SKILL.md#troubleshooting-after-login).

## MCP tool errors — stop immediately

Engine9 MCP tools return failures with **`isError: true`** (MCP standard) plus optional **`structuredContent`**:

| Field | Meaning |
|-------|---------|
| `status` | `timeout`, `unauthorized`, `validation`, or `error` |
| `requiresUserNotification` | User should be told about this failure |
| `safeToRetryAutomatically` | When `false`, do not retry or route around the error |

Success responses use JSON text in `content` with `{ ok: true, ... }`. Failures use **plain text** in `content` (not `{ ok: false }` JSON).

### Hard stop — do not continue

When **any** of these is true, **stop the current workflow immediately** and report the error to the user. Do **not** call further account-scoped tools (`task`, `search`, `eql`, `sql`, `analyze`, `segment`, `chat`, etc.).

1. Tool result has **`isError: true`**
2. Response text matches a fatal pattern (even when only plain text is visible):
   - `Cannot connect to the <account_id> database`
   - `Not authorized for account`
   - `Authentication required`
3. MCP **`account`** did not return parseable `{ ok: true, plugins: [...] }`
4. **`structuredContent.safeToRetryAutomatically`** is `false`

### What to do on stop

1. Report the MCP error message verbatim to the user.
2. Do **not** retry automatically or call downstream tools as a workaround.
3. Do **not** guess plugin paths or methods when `account` failed — plugin discovery did not succeed.
4. For **`Cannot connect to the <account_id> database`**: the account database is unreachable; every account-scoped operation will fail the same way until connectivity is restored.

### Examples

Database unreachable (`status: timeout`):

```
Cannot connect to the liftoff_maggie_hassan database
```

Unauthorized account (`status: unauthorized`):

```
Not authorized for account liftoff_hassan
```

In both cases: stop. Do not call `task` or other account tools afterward.

## MCP-only discovery — do not use local code

When interacting with an engine9 MCP server, **discover capabilities exclusively from the MCP server**. Do not search, read, or infer behavior from local workspace code (`server/workers/`, `plugins/`, `interfaces/`, etc.).

Local code is an **unreliable** source for MCP work because:

- The connected MCP server may be a different deployment, version, or branch than the workspace on disk.
- Plugin installs and metadata are **account-specific** — only MCP `account` reflects what is installed for the target account.
- Worker allowlists, tool schemas, and routing (`remote`, `workers/...` vs plugin paths) are enforced by the **running server**, not by files in the repo.
- Method names, option keys, and paths in local source may be deprecated, renamed, or not exposed via MCP at all.

**Use these sources instead:**

| Need | Source |
|------|--------|
| Available MCP tools and parameters | MCP tool schemas (client tool descriptors for the connected server) |
| Installed plugins, submodules, methods | MCP `account` → `plugins[].metadata` |
| Schema / tables / indexes / raw SQL | MCP `sql` (`command`: query, describe, indexes, tables, info, histo, transform_eql) |
| Schedule or check async work | MCP `task` (after resolving path/method from `account`) |
| Analyze / summarize / profile table contents | MCP `analyze` (uses `tables` then `analyze`) |
| Date histogram on indexed datetime column | MCP `sql` with `command: "histo"` |
| List flow definitions (REST) | Task API `GET /flows` — see [e9-tasks-api](../e9-tasks-api/SKILL.md) |

If a path, method, or option is not present in MCP responses, report that to the user — do not guess from local code.

## Tool selection strategy

**Prefer explicit native MCP tools when there is a quality match.** Only fall back to `task` when no native tool covers the request.

| User intent | Prefer |
|-------------|--------|
| Am I connected / signed in? | `ok`, then `user` |
| Who am I / which accounts do I have? | `user` |
| Find accounts by prefix, parent, type, tags, or installed plugin | `account` with `command: "search"` (one call — do not fan out) |
| List plugins / methods on one account | `account` with `account_id` (or `command: "plugins"`) |
| Search people by email, phone, name, or id | `search` |
| List segments or schedule segment builds | `segment` |
| Create accounts / manage domains or domain secrets | e9-account Worker (`cloud-services/e9-account`) — not MCP |
| Run a SQL/EQL query | `eql` / `sql` (`command: "query"` or omit when `sql` is set) |
| Analyze / summarize / profile a table | `analyze` |
| Describe tables, indexes, list tables, histo | `sql` with `command: "describe"` / `"indexes"` / `"tables"` / `"histo"` |
| Compute plugin or input UUIDs | `plugin_id`, `input_id` |
| Chat / conversation history | `chat` |
| Run a plugin worker method | `task` (after discovery via `account` plugins) |

`task` is the **catch-all** for behaviors that do not have a native MCP call. Do not reach for `task` when a native tool already covers the request with equal or better fidelity.

## Available MCP tools

These are the native tools registered on the engine9 MCP server (excluding `task`):

### `ok`

Returns server status, current time, and whether the request is authenticated. No sign-in required.

### `user`

Returns the current authenticated user: uid, email, admin flag, and account access map. **Prefer this** over `task` for identity and account-list questions.

### `account`

Two commands:

**`command: plugins`** (default when `account_id` is set) — list plugins installed on one account with marketplace metadata merged onto each plugin.

- Required: `account_id`
- Returns: `{ ok: true, command: "plugins", plugins: [...] }` — each plugin includes `path`, DB fields, and `metadata` (alias, submodules, methods, auth_fields, …)
- Backward compatible: `{ "account_id": "<id>" }` still means plugins.

**`command: search`** — find accessible accounts in **one call** using config filters and optional installed-plugin probes. Prefer this over `user` + many per-account plugin loads when the question is “which accounts match …?”.

- Requires at least one filter: `prefix` / `prefixes`, `parents`, `ids`, `name`, `type`, `tags`, or `plugins`
- Optional: `recursive` (with `parents`), `include_disabled`, `include_plugins`, `limit` (default 50), `max_scan` (default 100 for plugin probes), `concurrency`
- Returns: `{ ok: true, command: "search", count, accounts: [...], warnings, filters, truncated* }`
- `plugins` filter matches installed plugin `path` / `name` / `table_prefix` substrings (e.g. `["acoustic"]`). Apply `prefix`/`parents` first so DB probes stay bounded.
- Per-account DB failures go into `warnings` (do not fail the whole search).

Example — Authentic accounts with Acoustic:

```json
{ "command": "search", "prefixes": ["authentic"], "plugins": ["acoustic"] }
```

Example — direct children of a parent:

```json
{ "command": "search", "parents": ["frakture_master"] }
```

Plugins command is also the **discovery step** before calling `task` when no native tool matches (see fallback workflow below).

### `search`

Searches people by metadata filters and returns person summaries with related records.

- Required: `account_id`
- Filters: `emails`, `person_ids`, `phones`, `given_names`, `last_names`
- Optional: `limit` (max 1000, default 10)

### `segment`

List account segments or schedule `SegmentWorker.buildSegmentPersonFile`.

- Required: `account_id`, `command` (`list` or `build`)
- **list** — rows from `segment` (plugin_name, build_status, segment_directory when present). Optional `fields: '*'` for all columns.
- **build** — enqueue a segment person-file build. Requires `segment_id` or `definition_path`. Optional: `plugin_id`, `filename`, `engine`, `duckdb_file`, `input_id`, `label`, `remote`.

Example list:

```json
{ "command": "list", "account_id": "test" }
```

Example build by definition path:

```json
{
  "command": "build",
  "account_id": "test",
  "definition_path": "local$@engine9/interfaces/channels/email:segments:email_openers_30d"
}
```

### Account / domain management (e9-account)

Account creation and domain secrets are **not** MCP tools. Use the **e9-account** Cloudflare Worker (`cloud-services/e9-account`):

1. Auth with Delegate (`GET /auth/login` → `/auth/delegate`)
2. `POST /v1/accounts/create` — optional `{ slug, name, backend }`; returns endpoints + `public_api_key` (not shared secret)
3. `GET /v1/accounts/:slug/shared-secret` — same owner only

Canonical store: ACCOUNT_REGISTRY D1. KV: `DOMAINS_KV` + `DOMAIN_API_KEYS`.

### `eql`

Runs a SELECT built from an EQL object and returns generated SQL plus query rows.

- Required: `account_id`, `eql` (query object with `table`, `columns`, `conditions`, etc.)

For EQL **expression fragments** (not a full query), use `sql` with `command: "transform_eql"` instead.

### `sql`

Realtime SQL and schema introspection via `SQLWorker` (replaces the former `worker_invoke` allowlist).

| command | Purpose |
|---------|---------|
| `query` (default when `sql` is set) | Execute a single SQL statement |
| `describe` | Column schema for `table` |
| `indexes` | Indexes for `table` |
| `tables` (default when `sql` omitted) | List/filter table names (`filter`, `includeTemp`, …) |
| `info` | Driver/dialect info |
| `histo` | Date histogram on an indexed datetime column |
| `transform_eql` | EQL expression → SQL fragment (`eql` + `table`) |

Examples:

```json
{ "account_id": "test", "sql": "select 1" }
{ "account_id": "test", "command": "describe", "table": "person" }
{ "account_id": "test", "command": "tables", "filter": "person" }
{ "account_id": "test", "command": "histo", "table": "transaction", "column": "ts" }
```

There is **no** `worker_invoke` tool. Plugin methods still use `task` (async).

### `analyze`

Analyzes (summarizes/profiles) table contents via `SQLWorker.analyze`. Returns `columns` (not deprecated `fields`) with types, min/max, distinct counts, and samples. When indexed datetime columns exist, histo buckets/min/max are merged onto those column objects (`bucket_column: true` on the column used for bucketing). Sample analysis and histo run in parallel.

- Required: `account_id`, `table` (passed to `SQLWorker.tables({ filter })`)
- Optional: `max_tables` (default 3, max 10)
- Workflow:
  1. `tables({ filter })` — regex first; if no matches, language-token fallback on `filter`
  2. `analyze` on each matched table (sample + optional histo in parallel)

Standalone date histograms: `sql` with `command: "histo"`. Schema-only: `sql` `describe` / `indexes` / `tables`.

Example — user says "Summarize the ROI transaction table" → call `analyze`:

```json
{ "account_id": "test", "table": "ROI transaction" }
```

Prefer `analyze` over hand-written SQL when the user wants a table profile/summary.

### `plugin_id`

Computes `plugin_id` from `account_id` and `remote_plugin_id`.

### `input_id`

Computes `input_id` from `plugin_id` and `remote_input_id`.

### `chat`

Store and replay account-scoped conversations.

- Required: `account_id`
- Actions: `send` (default), `history`, `list`, `sample`, `list_samples`

## Fallback workflow: no native match → `account` → `task`

When the user's request does not map cleanly to a native tool:

1. **Ensure account scope** — `account_id` must be known (from session `engine9.account_id`, `account` search, or ask the user / suggest `/e9a`). If you only know org/prefix/plugin constraints, call `account` with `command: "search"` first, then pick one `account_id`.
2. **Call `account`** immediately with `{ "account_id": "<account_id>" }` (plugins command). If this fails, **stop** — do not call `task`.
3. **Scan the returned plugins** for a matching path/method combination:
   - Each plugin has a `path` (e.g. `@frakture-com/channelbots/RENxtBot`) and `metadata.submodules` with method lists.
   - Match user intent to a plugin path + submodule + method name.
   - Resolve alias/submodule shorthand (e.g. `renxt/people`) against `metadata.alias` and `metadata.submodules` keys.
4. **Call `task`** with the resolved `path`, `method`, `account_id`, and any `options` the user provided.

Do not invent paths or methods — only use combinations present in the `account` response.

### Path resolution for `task`

Prefer canonical colon paths:

- `@frakture-com/channelbots/RENxtBot:People`
- `@engine9/plugins/e9workers:SQLWorker`

Slash alias shorthand (`renxt/people`) is resolved by the server against installed plugins, but agents should resolve against cached `engine9.plugins` (from `/e9a`) first.

Do **not** use legacy Frakture dotted paths (`channelbots.RENxtBot.People`).

### Example: scheduling a plugin method

User: "List custom fields on RENxt people for account bfred_lambda_legal"

1. No native tool for this specific plugin method → fallback path.
2. Call `account` with `{ "account_id": "bfred_lambda_legal" }`.
3. Find plugin with alias `renxt`, submodule `People`, method `listCustomFields`.
4. Call `task`:

```json
{
  "account_id": "bfred_lambda_legal",
  "path": "@frakture-com/channelbots/RENxtBot:People",
  "method": "listCustomFields"
}
```

5. To poll status, call `task` again with `action: "check"` and `flow_run_id` / `task_run_ids` from the schedule response.

## Account-scoped calls

All tools except `ok`, `plugin_id`, and `input_id` require authentication. Account-scoped tools require an `account_id` the signed-in user can access. Do not guess account ids.

## Quick validation flow

After [Step 0 — Log in](#step-0--log-in-always-first):

1. Call `user` to confirm signed-in identity and account access.
2. If account id is unknown, call `account` with `command: "search"` and the known filters (prefix/parent/plugin). Otherwise set scope via `/e9a <account_id>` or call `account` plugins to cache methods.
3. Call `search` with a known email to validate account-scoped data access.
