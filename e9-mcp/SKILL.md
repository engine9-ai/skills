---
name: e9-mcp
description: Use the engine9 MCP server — log in first via mcp_auth, MCP-only discovery (never local code), prefer native tools over task, use @engine9/plugins/e9workers:EchoWorker + echo for on-demand smoke tests (no plugin lookup), discover other plugin methods via account, and invoke task as the catch-all for worker execution.
---

# engine9 MCP

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

engine9 MCP tools return failures with **`isError: true`** (MCP standard) plus optional **`structuredContent`**:

| Field | Meaning |
|-------|---------|
| `status` | `timeout`, `unauthorized`, `validation`, or `error` |
| `requiresUserNotification` | User should be told about this failure |
| `safeToRetryAutomatically` | When `false`, do not retry or route around the error |

Success responses use JSON text in `content` with `{ ok: true, ... }`. Failures use **plain text** in `content` (not `{ ok: false }` JSON).

### Executed SQL (`sql`)

Tools that run warehouse SQL include a top-level **`sql`** field so you can debug without reconstructing statements. Prefer this over guessing SQL.

**Array form** (`timelinePerson`, `auditPeople`, and any multi-query tool):

```json
{
  "ok": true,
  "sql": [
    { "id": "person_email", "table": "person_email", "sql": "SELECT …", "error": null },
    { "id": "timeline", "table": "timeline", "sql": "SELECT …", "error": null }
  ]
}
```

| Field | Meaning |
|-------|---------|
| `id` | Label for the statement |
| `sql` | Statement text, or `null` if it was not generated |
| `error` | `null` on success, `"skipped"` if not run, or the failure message |
| `table` | Optional primary table |

Omit `sql` only when the tool did not execute SQL. Empty `[]` means the tool could have queried but did not.

**String form** (`eql`, `sql` `command: query`): `sql` is the single statement string — equivalent to `[{ "id": "query", "sql": "<statement>", "error": null }]`.

When diagnosing timeline or model results, read `sql` first. Do not re-invent those SELECTs via the `sql` tool unless you need a variant.

### Hard stop — do not continue

When **any** of these is true, **stop the current workflow immediately** and report the error to the user. Do **not** call further account-scoped tools (`task`, `search`, `eql`, `sql`, `analyze`, `segment`, `auditPeople`, `timelinePerson`, `chat`, etc.).

1. Tool result has **`isError: true`**
2. Response text matches a fatal pattern (even when only plain text is visible):
   - `Cannot connect to the <account_id> database`
   - `Not authorized for account`
   - `Authentication required`
   - `getPluginMetadata` (e.g. `worker.getPluginMetadata is not a function`)
3. MCP **`account`** did not return parseable `{ ok: true, plugins: [...] }`
4. **`structuredContent.safeToRetryAutomatically`** is `false`

### What to do on stop

1. Report the MCP error message verbatim to the user.
2. Do **not** retry automatically or call downstream tools as a workaround.
3. Do **not** guess plugin paths or methods when `account` failed — plugin discovery did not succeed.
4. For **`Cannot connect to the <account_id> database`**: the account database is unreachable; every account-scoped operation will fail the same way until connectivity is restored.
5. For **`getPluginMetadata`**: plugin metadata loading is broken on this server. **Abort.** That must be fixed before continuing — do not schedule via local `TaskWorker`, SQL `plugin` / `bot_metadata` lookups, guessed `bot_id/submodule` paths, or the REST Task API as a workaround. `account` plugins and `task` schedule both depend on it.

### Examples

Database unreachable (`status: timeout`):

```
Cannot connect to the liftoff_maggie_hassan database
```

Unauthorized account (`status: unauthorized`):

```
Not authorized for account liftoff_hassan
```

Plugin metadata missing (`account` plugins or `task` schedule):

```
worker.getPluginMetadata is not a function
```

In all cases: stop. Do not call `task` or other account tools afterward. For `getPluginMetadata`, tell the user it must be fixed before any plugin discovery or scheduling can continue.

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
| Schema / tables / indexes / raw SQL | MCP `sql` (`command`: query, describe, indexes, tables, info, histo, compile_eql) |
| Schedule or check async work | MCP `task`: on-demand = `path`+`method` (`@engine9/plugins/e9workers:EchoWorker` needs no `account` lookup); predefined flow = `flow_id` slug |
| Analyze / summarize / profile table contents | MCP `analyze` (uses `tables` then `analyze`) |
| Account people / identity / timeline / model health | MCP `auditPeople` |
| Person timeline + models (current and legacy) | MCP `timelinePerson` (`command: inspect`) |
| Compare current (and opt-in legacy) model scores by source code | MCP `timelinePerson` (`command: compareSourceCodes`) |
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
| List available person-search form options for an account | `searchOptions` |
| Account people / timeline / identity / model health check | `auditPeople` |
| Person timeline + stored models (current and legacy) | `timelinePerson` |
| Compare current `model_*_stats` (and opt-in pivot) by source code | `timelinePerson` with `command: "compareSourceCodes"` |
| List segments, load segment detail, or schedule segment builds | `segment` |
| Create accounts / manage domains or domain secrets | e9-account Worker (`cloud-services/e9-account`) — not MCP |
| Run a SQL/EQL query | `eql` / `sql` (`command: "query"` or omit when `sql` is set) |
| Analyze / summarize / profile a table | `analyze` |
| Describe tables, indexes, list tables, histo | `sql` with `command: "describe"` / `"indexes"` / `"tables"` / `"histo"` |
| Compute plugin or input UUIDs | `plugin_id`, `input_id` |
| Chat / conversation history | `chat` |
| Run an on-demand plugin method | `task` with `path` + `method`. Built-in: `@engine9/plugins/e9workers:EchoWorker` + `echo` (no `account` lookup). Other plugins: discover via `account` first |
| Run a published flow (predefined) | `task` with `flow_id` (slug from REST `GET /flows`) — no `path`/`method` |
| Archive or retry flow runs / job lists | `task` with `action: "archive"` or `"retry"` |

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

Person search by metadata filters and/or a plugin search tree. Prefer this over ad-hoc SQL or `task` when looking up people by email, phone, name, or id. Returns person summaries with related records; each related subsection (`emails`, `phones`, `addresses`, `person_remote`, `transactions`) includes a total count and up to 100 sample records.

- Required: `account_id`
- Filters (string or array each): `emails`, `person_ids`, `phones`, `given_names`, `last_names`
- Optional: `search` — plugin search tree from `searchOptions`, e.g. `{ and: [{ path: "@engine9/interfaces/person_email:search:emails", options: { emailMatch: "a@" } }] }` (merged with metadata filters using AND)
- Optional: `limit` (max 1000, default 10)
- Returns: `{ ok: true, result }` where `result` is the `PersonWorker.search` payload

Example — email lookup:

```json
{ "account_id": "test", "emails": ["foo@bar.com"], "limit": 10 }
```

Example — plugin clause from `searchOptions`:

```json
{
  "account_id": "test",
  "search": {
    "and": [
      {
        "path": "@engine9/interfaces/person_email:search:emails",
        "options": { "subscriptionStatus": "Subscribed" }
      }
    ]
  },
  "limit": 10
}
```

For `/e9 search …` token parsing (emails vs person_ids vs names), see [e9-cli — `/e9 search` parsing rules](../e9-cli/SKILL.md#e9-search-parsing-rules).

### `searchOptions`

Per-account catalog of person-search options for building a UI form. Call this before constructing advanced plugin searches; submit resulting `{ path, options }` clauses to `search` (or `POST /data/search`).

- Required: `account_id`
- Returns: `{ ok: true, account_id, standard, searches, errors }`
  - `standard` — fixed filters (`emails`, `phones`, `given_names`, `last_names`, `person_ids`, `limit`) as JSON Schema
  - `searches` — handlers from **installed** plugins: `path`, `title`, canonical `form`, `plugin.instances`
  - `errors` — per-plugin compile failures (does not fail the whole call)

Example:

```json
{ "account_id": "test" }
```

### `auditPeople`

Read-only account people / identity / timeline / model health check (`AccountWorker.auditPeople`). Independent components: missing tables are skipped, query failures are reported, the rest continue. Prefer this over ad-hoc SQL or MCP `task` when the UI or agent needs an account audit payload. Render `current` and `legacy` as separate sections.

- Required: `account_id`
- Optional: `components` (subset of checks), `exclude`, `legacy` (default true; `false` skips timeline_v3 / person_model_source_code / transaction_model_source_code)
- Returns: `{ ok, account_id, available_components, components, current, legacy, sql, errors, started_at, finished_at }`
- `sql` is `[{ id, sql, error }]` for every warehouse statement this audit ran
- `ok` is false only when a component `status` is `error` (skipped is still success)

Example:

```json
{ "account_id": "test" }
```

Example — current identity/timeline only:

```json
{ "account_id": "test", "legacy": false }
```

### `timelinePerson`

Person-level **current-identity** timeline + model inspect (`ModelWorker.inspectPerson`) and source-code model compare (`compareSourceCodes`). SQL lives on the server; prefer this over ad-hoc SQL. The conductor Timeline artifact is a shell over `command: inspect`; `/models` is a shell over `command: compareSourceCodes`.

- Required: `account_id`
- **command: inspect** (default) — `emails` and/or `person_ids`. `person_ids` is a number, string, or array of either (warehouse `person.id` is an integer — do not stringify). Returns `{ queried, tables[], person_ids, emails, sql }` for current `timeline` / `model_*` only. Pass `legacy: true` to also load `timeline_v3_summary` / `person_model_source_code` (opt-in; future deployments will drop this). Missing tables are skipped. Do not join `person.id` to `person_id_int`.
- **command: compareSourceCodes** — all current `model_*_stats` by source code (`person_count`, `revenue`, `transactions`). Omit `source_codes` to union each model's top 10 by people and by revenue. Pass `legacy: true` to also include `transaction_model_pivot` stems. Optional `models` subset. Returns `sql` for top-N selection and per-model stats.
- **command: compareSourceCodesLegacy** — same-stem pivot vs current delta. `source_codes` required (comma-delimited; `%` is LIKE).
- **command: summarizeSourceCodesLegacy** — pivot rows only. `source_codes` required.

All commands include top-level **`sql`**: `[{ id, sql, error, table? }]` — the statements executed for this request. Use that log when debugging inspect/compare results.

Example — person inspect (current only):

```json
{ "account_id": "test", "emails": "user@example.com" }
```

Example — person inspect by numeric `person.id`:

```json
{ "account_id": "test", "person_ids": 1517 }
```

Example — person inspect with legacy tables:

```json
{ "account_id": "test", "emails": "user@example.com", "legacy": true }
```

Example — auto top source codes across current models:

```json
{ "account_id": "test", "command": "compareSourceCodes" }
```

Example — specified source codes, with legacy pivot:

```json
{ "account_id": "test", "command": "compareSourceCodes", "source_codes": "EM_%,MAIL", "legacy": true }
```

### `segment`

List or load rows from the account **`segment` table**, or schedule `SegmentWorker.buildSegmentPersonFile`. Do not use legacy `global_segment`. There is no free-text filter — **list → match in the agent → detail**.

- Required: `account_id`, `command` (`list`, `detail`, or `build`)
- **list** — summary rows from `segment` joined to `plugin` (`plugin_name`, `plugin_path`). Structured filters only: `plugin_id`, `plugin_path`, `segment_id` / `segment_ids`, `remote_segment_ids`, `build_type`, `submodule`, `fields`. Optional `fields: '*'` for all columns. Returns `sql`.
- **detail** — full `segment` row(s) plus plugin_name / plugin_path. Requires `segment_id` or `segment_ids`. Returns `sql`.
- **build** — enqueue a segment person-file build. Requires `segment_id` or `definition_path`. Optional: `plugin_id`, `filename`, `engine`, `duckdb_file`, `input_id`, `label`, `remote`.

Example list:

```json
{ "command": "list", "account_id": "test" }
```

Example list by plugin path:

```json
{ "command": "list", "account_id": "test", "plugin_path": "@engine9/interfaces/channels/email" }
```

Example detail after matching a listed row:

```json
{ "command": "detail", "account_id": "test", "segment_id": "<uuid>" }
```

Example build by definition path:

```json
{
  "command": "build",
  "account_id": "test",
  "definition_path": "@engine9/interfaces/channels/email:segments:email_openers_30d"
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

For EQL **expression fragments** (not a full query), use `sql` with `command: "compile_eql"` instead.

Full EQL syntax, query-object shape, and samples: [e9-eql](../e9-eql/SKILL.md).

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
| `compile_eql` | EQL expression → SQL fragment (`eql` + `table`) |

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

## On-demand tasks

MCP `task` (default `action: "schedule"`) and REST `POST /tasks/schedule` use the same **on-demand** names: **`path` + `method`** (no `flow_id`).

### Built-in engine9 Workers

Every bootstrapped account has `@engine9/plugins/e9workers`. Pass that path plus a worker submodule. **Do not** call MCP `account` to look up a plugin id. **Do not** install a separate Echo plugin.

| On-demand `path` | Typical `method` |
|------------------|------------------|
| `@engine9/plugins/e9workers:EchoWorker` | `echo` (smoke test) |
| `@engine9/plugins/e9workers:SQLWorker` | `query` |
| `@engine9/plugins/e9workers:SegmentWorker` | `list`, `detail`, `build`, `buildSegmentPersonFile` |

Echo smoke test:

```json
{
  "account_id": "test",
  "path": "@engine9/plugins/e9workers:EchoWorker",
  "method": "echo",
  "options": { "message": "hello from echo", "seconds": 1 }
}
```

### Other installed plugins

For account-specific bots (RENxt, …), discover `path` + `method` from MCP `account` plugins, then call `task`. Slash shorthand (`renxt/people`) is resolved against that list.

## Fallback workflow: no native match → `account` → `task`

When the user's request does not map cleanly to a native tool:

1. **Ensure account scope** — `account_id` must be known from **this chat session** (`engine9.account_id` after `/e9a`), an explicit user statement, or MCP `account` search when the user asked you to find matching accounts. If missing, **ask the user** or suggest `/e9a <account_id>` and stop — do not read leftover CLI files (`.e9_parameters`, `.e9_config.json5`, etc.) for scope. If you only know org/prefix/plugin constraints and the user wants discovery, call `account` with `command: "search"` first, then confirm which `account_id` to use.
2. **Pick the schedule mode:**
   - **Predefined flow** (`flow_id` slug from REST `GET /flows` or the user): call `task` with `flow_id` only. Skip plugin discovery.
   - **On-demand built-in** (`@engine9/plugins/e9workers:<Worker>` such as Echo): call `task` with `path` + `method`. Skip plugin discovery.
   - **On-demand account plugin**: continue with steps 3–4.
3. **Call `account`** immediately with `{ "account_id": "<account_id>" }` (plugins command). If this fails (including `getPluginMetadata`), **stop** — do not call `task`. That metadata load must be fixed before scheduling a non-`engine9` on-demand task can continue.
4. **Scan the returned plugins** for a matching path/method combination:
   - Each plugin has a `path` (e.g. `@frakture-com/channelbots/RENxtBot`) and `metadata.submodules` with method lists.
   - Match user intent to a plugin path + submodule + method name.
   - Resolve alias/submodule shorthand (e.g. `renxt/people`) against `metadata.alias` and `metadata.submodules` keys.
5. **Call `task`** with the resolved `path`, `method`, `account_id`, and any `options`. No `flow_id`.

Do not invent paths or methods for account plugins — only use combinations present in the `account` response. Do not mix `flow_id` with `path`/`method`.

### Path resolution for `task`

Prefer these forms:

- **Built-in workers (no lookup):** `@engine9/plugins/e9workers:EchoWorker`
- **Account plugin colon paths:** `@frakture-com/channelbots/RENxtBot:People`
- **Slash alias** for installed plugins (`renxt/people`) — resolved by the server; agents should resolve against cached `engine9.plugins` (from `/e9a`) first

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

5. To poll status, call `task` again with `action: "listTasks"` and `flow_run_id` / `task_run_ids` from the schedule response.

## Account-scoped calls

All tools except `ok`, `plugin_id`, and `input_id` require authentication. Account-scoped tools require an `account_id` the signed-in user can access. Do not guess account ids. Do not infer them from leftover local CLI state (`.e9_parameters` and similar) — that is for the `e9` / `e9a` bin scripts only; for MCP, ask for scope or require `/e9a`.

### Parent / all scope — do not fan out DB access

When the user asks for **all accounts**, **parent** children, or other multi-account remote flow-run views (e.g. list errored remote runs via MCP `task` `action: "list"` → `TaskWorker.listRemoteFlowRuns` / Frakture `POST /flow_runs/filter`):

- Do **not** call `account` plugins (or any account-DB worker) once per child to “check access”.
- Do **not** use the first id in a parent/all list as a required DB-connected `account_id` before the remote list.
- Drive the request with remote multi-account filters (`parent_account_id`, `account_ids`, etc.) and status filters. Prefer Prefect `state_type` values (`FAILED`, `RUNNING`, `COMPLETED`). Legacy Mongo statuses are accepted and mapped (see below). Account database connectivity is not a prerequisite for Frakture flow-run list reads.
- The hard-stop on `Cannot connect to the … database` still applies to tools that truly need that account DB (`sql`, `eql`, `search`, `auditPeople`, `timelinePerson`, single-account plugin schedule path resolution). It must **not** block multi-account remote flow-run listing.

### `task` action `list` — remote flow runs

MCP `task` with `action: "list"` calls `TaskWorker.listRemoteFlowRuns` (`POST /flow_runs/filter` on the Frakture Task API). Returns **flow runs only** — nested `task_runs` are not included. Each flow run includes `account_id`, `parent_account_id` (first id in that account's `parent_ids`, or `null`), and `parent_ids`.

MCP `task` with `action: "listTasks"` (or `"debug"`) calls `TaskWorker.listRemoteTaskRuns` (`POST /task_runs/filter` on the Frakture Task API) for a specific `flow_run_id` / `task_run_ids`. The result is `{ task_runs: [ … ], flow_run? }` — the same shape as REST `POST /task_runs/filter`. Pass `remote: false` to list local runs. Each `task_run` / `flow_run` includes `account_id`, `parent_account_id`, and `parent_ids`.

MCP `task` with `action: "archive"` or `"retry"` bulk-archives or retries remote flow runs / job lists (`TaskWorker.archiveRemoteFlowRuns` / `retryRemoteFlowRuns` → Frakture `POST /flow_runs/archive` and `/flow_runs/retry`). Same Firebase / MCP session and account header as `action: "list"` — **`user_id` is not required** (listing never required it; archive/retry must not either). Do **not** send `e9key_` Task API credentials from Conductor or Cursor MCP. Do **not** ask the user for a Frakture `user_id`.

| Mongo status | Prefect `state_type` |
|--------------|----------------------|
| `complete` | `COMPLETED` |
| `error` | `FAILED` |
| `in_progress` (or missing) | `RUNNING` |

Prefer Prefect values in `status` (e.g. `["FAILED"]`). Mongo values are mapped before the request is sent.

Example — errored flow runs under a parent:

```json
{
  "action": "list",
  "account_id": "frakture_master",
  "parent_account_id": "frakture_master",
  "status": ["FAILED"],
  "limit": 300
}
```

## Quick validation flow

After [Step 0 — Log in](#step-0--log-in-always-first):

1. Call `user` to confirm signed-in identity and account access.
2. If account id is unknown, call `account` with `command: "search"` and the known filters (prefix/parent/plugin). Otherwise set scope via `/e9a <account_id>` or call `account` plugins to cache methods.
3. Call `search` with a known email to validate account-scoped data access.
4. For parent/all **remote flow-run** requests, skip step 2–3 per-child probes — use multi-account remote filters only.
