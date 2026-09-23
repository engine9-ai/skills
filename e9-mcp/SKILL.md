---
name: e9-mcp
description: Use the engine9 MCP server — log in first via mcp_auth, MCP-only discovery (never local code), prefer native tools over task. Plugin install/settings are first-class MCP `plugin` (see e9-plugin), not task. Use @engine9/plugins/e9workers:EchoWorker + echo for on-demand smoke tests (no plugin lookup), discover other plugin methods via account, and invoke task as the catch-all for worker execution.
---

# engine9 MCP

The engine9 MCP server exposes authenticated, account-scoped tools for discovery, queries, reporting, API-key management, and asynchronous work. Use this skill when selecting and calling engine9 MCP tools from Cursor or another MCP client. **Plugins are first-class:** install, catalog, and settings use native MCP `plugin` — never `task`. Fleet install steps: [e9-plugin](../e9-plugin/SKILL.md).

## Quick reference

| Need | Use |
|------|-----|
| Authenticate | `mcp_auth` with `{}`, then verify with `ok` and `user` |
| Install a plugin / list installable paths | `plugin` `install` / `listAvailable` — [e9-plugin](../e9-plugin/SKILL.md) |
| Discover accounts or installed plugins | `account` |
| Use a purpose-built operation | The matching native MCP tool |
| Run an on-demand worker method | `task` with `path` + `method` |
| Run a predefined flow | `task` with `flow_id` |
| Diagnose generated queries | The response's top-level `sql` field |

**Rule:** Discover accounts, plugins, methods, and options from the connected MCP server only; never infer them from local workspace code.

## Step 0 — Log in (always first)

**Rule:** Every `/e9` or MCP request starts by logging in. Do not grep, curl, read config files, start servers, or run CLI commands to "figure out" auth.

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

**Rule:** Stop the current workflow on an MCP error unless the response explicitly marks it safe to retry automatically.

| Field | Meaning |
|-------|---------|
| `status` | `timeout`, `unauthorized`, `validation`, or `error` |
| `requiresUserNotification` | User should be told about this failure |
| `safeToRetryAutomatically` | When `false`, do not retry or route around the error |

Success responses use JSON text in `content` with `{ ok: true, ... }`. Failures use **plain text** in `content` (not `{ ok: false }` JSON).

### Executed SQL (`sql`)

Tools that run warehouse SQL include a top-level **`sql`** field so you can debug without reconstructing statements. Prefer this over guessing SQL.

**Array form** (`timelinePerson` and any multi-query tool):

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

**String form** (`eql`, `sql` `command: query`): `sql` is the single statement string — equivalent to `[{ "id": "query", "sql": "<statement>", "error": null }]`. SELECT/WITH queries are hard-capped at **10000** rows (`LIMIT` rewritten before execution). Responses include `max_rows` and `truncated`.

When diagnosing timeline or model results, read `sql` first. Do not re-invent those SELECTs via the `sql` tool unless you need a variant.

### Hard stop — do not continue

When **any** of these is true, **stop the current workflow immediately** and report the error to the user. Do **not** call further account-scoped tools (`task`, `search`, `eql`, `sql`, `analyze`, `segment`, `inventory`, `timelinePerson`, `timelinePersonLegacy`, `chat`, `file`, `apiKey`, `report`, `plugin`, etc.).

**Exception — fleet plugin install:** a per-account MCP `plugin` `install` failure (`Cannot connect to the … database`, `Not authorized for account`, unmet dependencies) is a **skip** for that `account_id` only. Continue remaining accounts and report successes and failures. Session-level auth failure and `getPluginMetadata` still stop the whole job. See [e9-plugin](../e9-plugin/SKILL.md).

1. Tool result has **`isError: true`**
2. Response text matches a fatal pattern (even when only plain text is visible):
   - `Cannot connect to the <account_id> database`
   - `Not authorized for account`
   - `Authentication required`
   - `getPluginMetadata` (e.g. `worker.getPluginMetadata is not a function`)
3. MCP **`account`** did not return parseable `{ ok: true, plugins: [...] }`
4. **`structuredContent.safeToRetryAutomatically`** is `false`
5. An **account or parent is not found** in MCP: `user.accounts` is empty, `account` search `count` is `0` for that id/parent/prefix, or plugins says the account is unknown/unauthorized. **Stop.** Do **not** dig deeper into compiled account catalogs, etc. Account discovery when using MCP should only be through that MCP, not through any other mechanisms.

### What to do on stop

1. Report the MCP error message verbatim to the user (or that MCP `user` / `account` search did not find the account or parent).
2. Do **not** retry automatically or call downstream tools as a workaround.
3. Do **not** guess plugin paths or methods when `account` failed — plugin discovery did not succeed.
4. Do **not** open `accounts.d/`, `accounts.compiled.json5`, `account-config.json`, `account.<id>.json5`, `.e9_parameters`, or any other local catalog to “find” ids MCP did not return.
5. For **`Cannot connect to the <account_id> database`**: the account database is unreachable; every account-scoped operation **on that id** will fail the same way until connectivity is restored. Fleet `plugin` `install` skips that id and continues ([e9-plugin](../e9-plugin/SKILL.md)).
6. For **`getPluginMetadata`**: plugin metadata loading is broken on this server. **Abort.** That must be fixed before continuing — do not schedule via local `TaskWorker`, SQL `plugin` / `bot_metadata` lookups, guessed `plugin_id`/`submodule` paths, or the REST Task API as a workaround. `account` plugins and `task` schedule both depend on it.

### Examples

Database unreachable (`status: timeout`):

```
Cannot connect to the <account_id> database
```

Unauthorized account (`status: unauthorized`):

```
Not authorized for account <account_id>
```

Plugin metadata missing (`account` plugins or `task` schedule):

```
worker.getPluginMetadata is not a function
```

In all cases: stop. Do not call `task` or other account tools afterward. For `getPluginMetadata`, tell the user it must be fixed before any plugin discovery or scheduling can continue.

## MCP-only discovery — do not use local code

When interacting with an engine9 MCP server, **discover capabilities and accounts exclusively from the MCP server**. Do not search, read, or infer behavior from local workspace code (`server/workers/`, `plugins/`, `interfaces/`, etc.).

**Rule:** If MCP does not return an account, plugin path, method, or option, report that result and do not guess or search local catalogs.

Local code is an **unreliable** source for MCP work because:

- The connected MCP server may be a different deployment, version, or branch than the workspace on disk.
- Plugin installs and metadata are **account-specific** — only MCP `account` reflects what is installed for the target account.
- Worker allowlists, tool schemas, and routing (`remote`, `workers/...` vs plugin paths) are enforced by the **running server**, not by files in the repo.
- Method names, option keys, and paths in local source may be deprecated, renamed, or not exposed via MCP at all.

### Accounts and parents — MCP only

Account discovery when using MCP should only be through that MCP, not through any other mechanisms.

| Need | MCP source |
|------|------------|
| Who am I / which accounts can I use? | `user` (`accounts` flat map with `parent_ids`) |
| Find accounts by prefix, parent, name, type, tags, plugin | `account` `command: "search"` (one call; flat rows with same listing fields) |
| One account’s plugins / methods | `account` `command: "plugins"` |

When an account or parent is not found, do NOT dig deeper into compiled account catalogs, etc. Do **not** read `accounts.d/`, `accounts.compiled.json5`, `account-config.json`, `account.<slug>.json5`, `.e9_parameters`, `.e9_config.json5`, or account trees on disk. Report that MCP did not return the account or parent, and stop.

**Use these sources instead:**

| Need | Source |
|------|--------|
| Available MCP tools and parameters | MCP tool schemas (client tool descriptors for the connected server) |
| Account ids, names, parents | MCP `user` / MCP `account` search only |
| Installed plugins, submodules, methods | MCP `account` → `plugins[].metadata` |
| Installable package paths on this server | MCP `plugin` `command: "listAvailable"` |
| Install a package on an account | MCP `plugin` `command: "install"` — [e9-plugin](../e9-plugin/SKILL.md) |
| Plugin settings (types, descriptions, current values), grouped by plugin | MCP `plugin` `command: "settings"` (`PluginWorker.listSettings`) |
| Schema / tables / indexes / raw SQL | MCP `sql` (`command`: query, describe, indexes, tables, info, histo, compile_eql) |
| Schedule or check async work | MCP `task`: on-demand = `path`+`method` (`@engine9/plugins/e9workers:EchoWorker` needs no `account` lookup); predefined flow = `flow_id` slug |
| Analyze / summarize table contents (min, max, distinct, samples) | MCP `analyze` (uses `tables` then `analyze`; optional `target_buckets`) |
| Account / person current timeline + identity / model health | MCP `timelinePerson` (`command: inspect`) |
| Account / person **legacy** `timeline_v3*` inspect | MCP `timelinePersonLegacy` (separate tool; not `inspect` + `legacy`) |
| Compare current `model_*_stats` by source code | MCP `timelinePerson` (`command: compareSourceCodes`). Legacy pivot: [e9-model/legacy.md](../e9-model/legacy.md), only when the user explicitly asked for legacy |
| Date histogram on a datetime column (index optional) | MCP `sql` with `command: "histo"` |
| List flow definitions (REST) | Task API `GET /flows` — see [e9-tasks-api](../e9-tasks-api/SKILL.md) |

If a path, method, or option is not present in MCP responses, report that to the user — do not guess from local code.

## Tool selection strategy

**Prefer explicit native MCP tools when there is a quality match.** Only fall back to `task` when no native tool covers the request.

**Rule:** Use a native MCP tool whenever it covers the request with equal or better fidelity; treat `task` as the catch-all.

| User intent | Prefer |
|-------------|--------|
| Am I connected / signed in? | `ok`, then `user` |
| Who am I / which accounts do I have? | `user` (flat `accounts` map with `parent_ids`) |
| Find accounts by prefix, parent, type, tags, or installed plugin | `account` with `command: "search"` (one call — do not fan out; flat rows) |
| List plugins / methods on one account | `account` with `account_id` (or `command: "plugins"`) |
| List plugin settings (types, descriptions, current values) grouped by plugin | `plugin` with `command: "settings"` |
| Install a plugin on one account or a parent’s children | `plugin` `install` — [e9-plugin](../e9-plugin/SKILL.md). Do not use `task` |
| Search people by email, phone, name, or id | `search` |
| Search people in or out of a current model by source code | `search` with a `searchOptions` clause — [e9-model — Person search](../e9-model/SKILL.md#person-search) |
| Search people in a legacy model and out of the current model | [e9-model/legacy.md](../e9-model/legacy.md) — only when the user explicitly asked for legacy |
| List available person-search form options for an account | `searchOptions` |
| Account / person current timeline + identity / model health | `timelinePerson` (`command: inspect`) |
| Account / person **legacy** `timeline_v3*` inspect | `timelinePersonLegacy` |
| Compare current `model_*_stats` by source code | `timelinePerson` with `command: "compareSourceCodes"`. Legacy pivot: [e9-model/legacy.md](../e9-model/legacy.md), only when the user explicitly asked for legacy |
| List segments, load segment detail, or schedule segment builds | `segment` |
| List or run reports (plugin `path` or portable JSON `definition`; filters, date range) | `report` (`list` first for plugins; `run` with `path` or `definition`) |
| Read or schedule account warehouse inventory | `inventory` (`get` first; `build` only if not ready) |
| Create accounts / manage domains or domain secrets | e9-account Worker (`cloud-services/e9-account`) — not MCP |
| Run a SQL/EQL query | `eql` / `sql` (`command: "query"` or omit when `sql` is set) |
| Analyze / summarize a table | `analyze` |
| Describe tables, indexes, list tables, histo | `sql` with `command: "describe"` / `"indexes"` / `"tables"` / `"histo"` |
| Compute plugin or input UUIDs | `plugin_id`, `input_id` |
| Chat / conversation history | `chat` |
| Read a small slice of an account file (S3 / local) | `file` |
| Create / list / update / rotate / revoke API keys and scopes | `apiKey` — see [e9-api-key](../e9-api-key/SKILL.md). Never via `task` |
| Run an on-demand plugin method | `task` with `path` + `method`. Built-in: `@engine9/plugins/e9workers:EchoWorker` + `echo` (no `account` lookup). Other plugins: discover via `account` first |
| Run a published flow (predefined) | `task` with `flow_id` (slug from REST `GET /flows`) — no `path`/`method` |
| Archive or retry flow runs / job lists | `task` with `action: "archive"` or bulk `"retry"` (`flow_run_ids`). One call for all ids. After a parent/all list, pass the same `parent_account_id` — [bulk archive](#bulk-archive--retry-of-flow-runs) |
| Pause / resume / retry a task run, edit options, fetch log/output/checkpoints | `task` with `action: "pause"` / `"resume"` / `"retry"` (`task_run_id`) / `"updateOptions"` / `"log"` / `"output"` / `"get"` (`checkpoints` only on `get`). To walk back checkpoints: `action: "resetCheckpoints"` with `start_index` (0 / omitted clears all; N keeps the first N — cannot yank from the middle) |
| Method option schema for a plugin path (Options form) | `task` with `action: "describe"` and `path` (same naming as schedule; `method` optional) |

`task` is the **catch-all** for behaviors that do not have a native MCP call. Do not reach for `task` when a native tool already covers the request with equal or better fidelity.

## Available MCP tools

These are the native tools registered on the engine9 MCP server (excluding `task`):

### `ok`

Returns server status, current time, and whether the request is authenticated. No sign-in required.

### `user`

Returns the current authenticated user: uid, email, admin flag, and a **flat** account access map. Each `accounts[account_id]` entry shares listing fields with `account` search: `name`, `type`, `parent_ids`, `disabled`, `tags`, plus auth `level`. **No nested children** — rebuild hierarchy from `parent_ids` on the client. **Prefer this** over `task` for identity and full account-list questions. See [How MCP lists accounts](../../server/api/mcp/accounts.md).

### `account`

Two commands:

**`command: plugins`** (default when `account_id` is set) — list plugins installed on one account with marketplace metadata merged onto each plugin.

- Required: `account_id`
- Returns: `{ ok: true, command: "plugins", plugins: [...] }` — each plugin includes `path`, DB fields, and `metadata` (alias, submodules, methods, auth_fields, …)
- Backward compatible: `{ "account_id": "<id>" }` still means plugins.

**`command: search`** — find accessible accounts in **one call** using config filters and optional installed-plugin probes. Prefer this over `user` + many per-account plugin loads when the question is “which accounts match …?”.

- Requires at least one filter: `prefix` / `prefixes`, `parents`, `ids`, `name`, `type`, `tags`, or `plugins`
- Optional: `recursive` (with `parents`), `include_disabled`, `include_plugins`, `include_plugin_metadata`, `limit` (default 50), `max_scan` (default 100 for plugin probes), `concurrency`
- Returns: `{ ok: true, command: "search", count, accounts: [...], warnings, filters, truncated* }`
- Each `accounts[]` row is **flat** with the same core fields as `user.accounts` (`name`, `type`, `parent_ids`, `disabled`, `tags`) plus `account_id`. `parents` / `recursive` only filter which rows appear; they do not nest children.
- `include_plugins` attaches lite plugin rows (`id` / `name` / `path` / `table_prefix`) per account. `include_plugin_metadata` adds one marketplace metadata map keyed by plugin path — do not fan out `command: plugins` per account to build a method catalog.
- `plugins` filter matches installed plugin `path` / `name` / `table_prefix` substrings (e.g. `["acoustic"]`). Apply `prefix`/`parents` first so DB probes stay bounded.
- Per-account DB failures go into `warnings` (do not fail the whole search).
- `count: 0` is the answer. Do **not** fall back to compiled account catalogs to find ids MCP omitted.

Example — accounts matching a prefix with a plugin installed:

```json
{ "command": "search", "prefixes": ["<prefix>"], "plugins": ["acoustic"] }
```

Example — direct children of a parent (flat list):

```json
{ "command": "search", "parents": ["<parent_account_id>"] }
```

Plugins command is also the **discovery step** before calling `task` when no native tool matches (see fallback workflow below).

### `plugin`

**Fast path:** [e9-plugin](../e9-plugin/SKILL.md) — install is native MCP `plugin`, not `task`.

Install plugins and catalog **declared settings** for an account via `PluginWorker`. Settings are warehouse configuration on a plugin row (`setting` table). They are **not** marketplace authorization (`auth_fields` on `account` plugin metadata).

- Required: `account_id`, `command`
- **listAvailable** — installable package paths on this server (server-wide catalog; any accessible `account_id` works)
- **install** — install `path` (full package path or shorthand from listAvailable). One call per account. Unique packages reuse the existing row
- **settings** — catalog grouped by plugin path: `path`, `name`, `plugin.instances`, JSON Schema `form`, `settings[]` (`type`, `description`, `values`, `default`, current `value`). Optional `path` / `plugin_id`. Hidden settings omitted unless `include_hidden: true`. Compile failures go in `errors` (call still succeeds)
- **setSetting** — update one warehouse value. Required: `plugin_id`, `name`. Validates `values` / type when the package declares that name

Example — install:

```json
{
  "command": "install",
  "account_id": "<account_id>",
  "path": "@engine9/plugins/models/deployment/v2026_09_16"
}
```

Example — settings UI catalog:

```json
{ "command": "settings", "account_id": "test" }
```

Example — update a declared setting:

```json
{
  "command": "setSetting",
  "account_id": "test",
  "plugin_id": "<plugin uuid>",
  "name": "summary_model_kind",
  "value": "current"
}
```

HTTP: `GET /data/settings`, `POST /data/settings`. Fleet install: [e9-plugin](../e9-plugin/SKILL.md). Authoring: [create-engine9-plugin](../create-engine9-plugin/SKILL.md#settings).

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

### Model source-code membership

People in or out of a **current** model for a source code use the installed model's `searchOptions` handlers, not ad-hoc SQL. Full paths, forms, and the include/exclude tree: [e9-model — Person search](../e9-model/SKILL.md#person-search).

| Intent | Clause |
| --- | --- |
| In the model's person credit for a code | `{ path: "@engine9/plugins/models/<model>:search:personSourceCode", options: { sourceCode } }` |
| Out of that credit | Same clause with `exclude: true` |
| In person credit vs has a credited transaction | `personSourceCode` vs `transactionSourceCode` — different handlers |

`<model>` is `first_touch`, `crm_origin`, or `last_acquisition`. `sourceCode` containing `%` is `LIKE`. `exclude` is set on the clause; it is not a form property.

Rule: People in a **legacy** model for a source code, and out of the matching current model, are [e9-model/legacy.md](../e9-model/legacy.md). Read that file only when the user explicitly asked for legacy models. Do not add `@engine9/plugins/models/legacy:search:personSourceCode` otherwise, even if `searchOptions` lists it.

### `searchOptions`

Per-account catalog of person-search options for building a UI form. Call this before constructing advanced plugin searches; submit resulting `{ path, options }` clauses to `search` (or `POST /data/search`). Set `exclude: true` on a clause for people outside that handler; `exclude` is not a form field.

- Required: `account_id`
- Returns: `{ ok: true, account_id, standard, searches, errors }`
  - `standard` — fixed filters (`emails`, `phones`, `given_names`, `last_names`, `person_ids`, `limit`) as JSON Schema
  - `searches` — handlers from **installed** plugins: `path`, `title`, canonical `form`, `plugin.instances`. Current model source-code forms (`personSourceCode`, `transactionSourceCode`) are in [e9-model — Person search](../e9-model/SKILL.md#person-search). The legacy model form is in [e9-model/legacy.md](../e9-model/legacy.md) and is used only when the user explicitly asked for legacy.
  - `errors` — per-plugin compile failures (does not fail the whole call)

Example:

```json
{ "account_id": "test" }
```

### `report`

List, load, or run reports for an account. Plugin reports come from installed plugins (`<package>:reports:<key>`). `command: run` also accepts a portable JSON `definition` (hosted or inline). SQL is generated by `ReportWorker`. Full authoring + UI widget contract: [e9-reports](../e9-reports/SKILL.md).

- Required: `account_id`
- **command: list** (default) — catalog of **installed plugin** reports: `path`, `name`, `description`, `tags`, `plugin.instances`, **`filters`** (JSON Schema), component summaries
- **command: get** — definition without running. Requires `path`
- **command: run** — execute `path` **or** `definition` with `options` / `start` / `end` / `limit` / `offset`. Returns component `data` plus top-level `sql`. Mutually exclusive locators.

Example list:

```json
{ "account_id": "<account_id>" }
```

Example run (plugin path):

```json
{
  "command": "run",
  "account_id": "<account_id>",
  "path": "@engine9/plugins/reports/people:reports:subscription_status",
  "limit": 1000
}
```

Example run (inline definition):

```json
{
  "command": "run",
  "account_id": "<account_id>",
  "definition": {
    "name": "People count",
    "data_sources": { "default": { "table": "person" } },
    "sections": [
      {
        "components": [
          { "id": "people", "component": "StatCard", "name": "People", "metric": { "eql": "count(*)" } }
        ]
      }
    ]
  }
}
```

HTTP: `GET /data/reports`, `GET /data/reports/get?path=`, `GET|POST /data/reports/run` (`path` or POST `definition`).

### `timelinePerson`

Account- or person-level **current-identity** timeline + model inspect (`ModelWorker.inspectPerson`) and source-code model compare (`compareSourceCodes`). SQL lives on the server; prefer this over ad-hoc SQL. The conductor Timeline artifact is a shell over `command: inspect`; `/models` is a shell over `command: compareSourceCodes`. There is no separate `auditPeople` tool — inspect includes those **current** aggregations.

Rule: `inspect` never loads `timeline_v3*`. Call **`timelinePersonLegacy`** for legacy tables. Current vs legacy contract: [e9-timeline — MCP: current vs legacy](../e9-timeline/SKILL.md#mcp-current-timeline-vs-legacy-timeline).

- Required: `account_id`
- **command: inspect** (default) — omit `emails` / `person_ids` / `source_codes` for an account-wide load (entry-type min/max/count, identity table counts, transaction min/max, `model_*_person` / `model_*_transaction` counts — not `*_stats`). Person-scoped (`person_ids` / `emails`) returns timeline detail + `model_*_person` only and **skips** those account audit COUNTs. `person_ids` is a number, string, or array of either (warehouse `person.id` is an integer — do not stringify). Optional `source_codes` (comma-delimited; `%` is LIKE) filters those queries. Missing tables are skipped. Do not join `person.id` to `person_id_int`.
- **command: compareSourceCodes** — all current `model_*_stats` by source code (`person_count`, `revenue`, `transactions`). Omit `source_codes` to union each model's top 10 by people and by revenue. Optional `models` subset. Returns `sql` for top-N selection and per-model stats. Pivot stems and `legacy: true` are in [e9-model/legacy.md](../e9-model/legacy.md); use them only when the user explicitly asked for legacy.
- **command: compareSourceCodesLegacy** — legacy pivot vs current. [e9-model/legacy.md](../e9-model/legacy.md). Only when the user explicitly asked for legacy.
- **command: summarizeSourceCodesLegacy** — pivot rows only. [e9-model/legacy.md](../e9-model/legacy.md). Only when the user explicitly asked for legacy.

All commands include top-level **`sql`**: `[{ id, sql, error, table? }]` — the statements executed for this request. Use that log when debugging inspect/compare results.

Example — account-wide inspect (current aggregations):

```json
{ "account_id": "test" }
```

Example — person inspect (current only):

```json
{ "account_id": "test", "emails": "user@example.com" }
```

Example — person inspect by numeric `person.id`:

```json
{ "account_id": "test", "person_ids": 1517 }
```

Example — auto top source codes across current models:

```json
{ "account_id": "test", "command": "compareSourceCodes" }
```

Example — specified source codes:

```json
{ "account_id": "test", "command": "compareSourceCodes", "source_codes": "EM_%,MAIL" }
```

### `timelinePersonLegacy`

Legacy-identity timeline inspect (`ModelWorker.inspectPersonLegacy`). **Separate tool** from `timelinePerson` so the UI can load it independently (or not at all) and so it can be unregistered later.

- Required: `account_id`
- Optional: `emails`, `person_ids` (current `person.id` — email lookup only), `source_codes`, `row_limit`, `people_limit`
- Account-wide: `timeline_v3` min/max/count, `transaction_model_source_code`. `person_model_source_code` totals are skipped (no account-wide COUNT)
- With email / person_ids / source_codes: `person_model_source_code` totals by model; with email also person-level `timeline_v3_summary` / `person_model_source_code`. Type column is **`entry_type_label`**
- Do not join `person.id` to `person_id_int`. Future deployments will drop this tool.

Example — person legacy timeline:

```json
{ "account_id": "test", "emails": "user@example.com" }
```

Example — account-wide legacy aggregations:

```json
{ "account_id": "test" }
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

### `inventory`

Account warehouse inventory cache at `{account root}/cache/inventory.json.gz`. Prefer this over `task` for inventory. Reports take a long time to build — **always `get` first**.

- Required: `account_id`
- **command: get** (default) — `InventoryWorker.inventory`. Status only: if the gzip cache exists, `ready: true` plus path / size / modified_at. Does not parse the file. If not: `ready: false` (do not retry get as a workaround — offer `build`). Full report: `GET /data/inventory/report` (gzip bytes, `204` if missing) or MCP `file` with `filename: cache/inventory.json.gz`.
- **command: build** — schedule `InventoryWorker.buildInventorySummaryFile` via TaskWorker. Optional: `definition_path`, `tables`, `extra_tables`, `exclude_tables`, `input_directories`, `include_files`, `statistics`, `label`, `remote`. Standard builds omit input-store files. Does not wait for the report.

Example get:

```json
{ "account_id": "test" }
```

Example build:

```json
{ "command": "build", "account_id": "test" }
```

### Account / domain management (e9-account)

Account creation and domain secrets are **not** MCP tools. Use the **e9-account** Cloudflare Worker (`cloud-services/e9-account`):

1. Auth with Delegate (`GET /auth/login` → `/auth/delegate`)
2. `POST /v1/accounts/create` — optional `{ slug, name, backend }`; returns endpoints + `public_api_key` (not shared secret)
3. `GET /v1/accounts/:slug/shared-secret` — same owner only

Canonical store: ACCOUNT_REGISTRY D1. KV: `DOMAINS_KV` + `DOMAIN_API_KEYS`.

### `eql`

Runs a SELECT built from an EQL object and returns generated SQL plus query rows. Hard-capped at **10000** rows: a missing or larger `eql.limit` is rewritten before execution. Responses include `max_rows` and `truncated`.

- Required: `account_id`, `eql` (query object with `table`, `columns`, `conditions`, etc.)

For EQL **expression fragments** (not a full query), use `sql` with `command: "compile_eql"` instead.

Full EQL syntax, query-object shape, and samples: [e9-eql](../e9-eql/SKILL.md).

### `sql`

Realtime SQL and schema introspection via `SQLWorker` (replaces the former `worker_invoke` allowlist).

| command | Purpose |
|---------|---------|
| `query` (default when `sql` is set) | Execute a single SQL statement. SELECT/WITH are hard-capped at **10000** rows (`LIMIT` rewritten before execution); `truncated` when the cap is hit |
| `describe` | Column schema for `table` |
| `indexes` | Indexes for `table` |
| `tables` (default when `sql` omitted) | List/filter table names (`filter`, `includeTemp`, …) |
| `info` | Driver/dialect info |
| `histo` | Date histogram on a datetime column (index not required; default prefers indexed) |
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

Analyzes table contents via `SQLWorker.analyze`. Prefer this over hand-written `MIN` / `MAX` / `COUNT(DISTINCT)` when the user wants a summary of a SQL table. Column-stat contract, sample limits, and the date-histogram overwrite rules: [e9-eql — Analyzing a table](../e9-eql/SKILL.md#analyzing-a-table).

- Required: `account_id`, `table` (passed to `SQLWorker.tables({ filter })`)
- Optional: `max_tables` (default 3, max 10), `target_buckets` (default 10; date histogram only)
- Workflow:
  1. `tables({ filter })` — regex first; if no matches, language-token fallback on `filter`
  2. `analyze` on each matched table (sample + optional histo in parallel)

Response: `{ ok, filter, matched_tables, analyzed_tables, truncated, analyses }`. Each analysis is `{ table, records, table_records, columns }`. `table_records` is `count(*)`. Column stats live on `columns`; histogram buckets are merged onto those same objects.

| Column field | Meaning |
|--------------|---------|
| `name`, `type` | Column name and inferred type (`string`, `int`, `bigint`, `double`, `decimal`, `date`, `datetime`, `uuid`) |
| `min`, `max` | Sample range. When a histogram runs, every datetime column is replaced with full-table SQL min/max |
| `empty` | Null or missing values in the sample |
| `distinct` | Distinct values seen in the sample |
| `sample` | Up to 32 most frequent sample values (stringified) |
| `min_length`, `max_length` | Present on string columns |
| `buckets` | On datetime columns when a histogram runs: `{ range, start, end, records, min, max }` |
| `bucket_column` | `true` on the histogram column (`ts`, then `created_at`, then `modified_at`, else the first indexed datetime leading column) |
| `bucket_unit` | `day`, `week`, `month`, `quarter`, or `year` |

Rule: `table_records` is `count(*)`. Sample `min` / `max` / `distinct` / `empty` / `sample` / `records` cover up to 10000 rows (5000 from the start plus 5000 highest primary-key rows when a primary key exists). They are not full-table aggregates. Histogram `MIN` / `MAX` and `buckets[].records` on datetime columns are full-table. `analyze` only auto-runs the histogram when an indexed datetime column exists; for unindexed dates use `sql` `command: "histo"` with an explicit `column`.

Standalone date histograms: `sql` with `command: "histo"`. Schema-only: `sql` `describe` / `indexes` / `tables`.

Example — user says "Summarize the ROI transaction table" → call `analyze`:

```json
{ "account_id": "test", "table": "ROI transaction", "target_buckets": 10 }
```

### `plugin_id`

Computes `plugin_id` from `account_id` and `remote_plugin_id`.

### `input_id`

Computes `input_id` from `plugin_id` and `remote_input_id`.

### `chat`

Store and replay account-scoped conversations.

- Required: `account_id`
- Actions: `send` (default), `history`, `list`, `sample`, `list_samples`

### `file`

Read a small slice of an account file via `FileWorker.stream` (local, `s3://`, `r2://`, `gs://`). Hard-locked to the scoped `account_id`.

- Required: `account_id`, `filename` (or `path`)
- `filename` may be relative (`export/inventory.json5`) or rooted (`s3://engine9-accounts/<account_id>/…`). Relative paths resolve under `ENGINE9_STORED_INPUT_PATH/<account_id>`. Paths outside the account, other accounts, or `..` escapes are rejected.
- Optional: `start` / `end` (inclusive byte offsets, same idea as `GET /log`). Default is the first 300KB. Window is capped at 300KB. Negative `start` tails from the end.
- Returns `{ ok, filename, content, start, end, bytes, size, truncated }`

Example:

```json
{ "account_id": "test", "filename": "export/inventory.json5" }
```

### `apiKey`

Create and manage engine9 API keys (`e9key_` / `e9publickey_`) and their scopes. Keys are **generic non-session auth** for HTTP APIs (public signup/payment forms, inbound, Task API, and other scoped routes); the scope list is what a key can do. MCP itself stays on a user session and does not accept `e9key_`. Storage is `@engine9/core` `SqlApiKeyStore` (account `api_key` table, hash only). Prefer this over CLI or MCP `task`. Full contract: [e9-api-key](../e9-api-key/SKILL.md).

- **catalog** (default when `account_id` omitted) — known scopes and form fields. No database.
- **list** (default when `account_id` set) — keys for the account (no plaintext)
- **get** — one key by `id` (no plaintext)
- **create** — plaintext returned once (`shown_once: true`). Required: `scopes`
- **update** — name / scopes / `default_role_id` / `expires_at` / `active` without rotating
- **revoke** — deactivate
- **rotate** — new plaintext once; old id revoked

Do not schedule `createApiKey` via `task`. Do not send `e9key_` credentials to MCP.

Example catalog:

```json
{ "command": "catalog" }
```

Example create:

```json
{
  "command": "create",
  "account_id": "test",
  "name": "partner-tasks",
  "scopes": ["tasks:read", "tasks:schedule"]
}
```

## On-demand tasks

MCP `task` (default `action: "schedule"`) and REST `POST /tasks/schedule` use the same **on-demand** names: **`path` + `method`** (no `flow_id`).

For **multi-step flows** (identity rebuild, etc.), prefer `flow_id` / `flow_path` — see [e9-tasks-api/deploy-flow.md](../e9-tasks-api/deploy-flow.md). Do not hand-expand flow steps into on-demand schedules.

### Built-in engine9 Workers

Every bootstrapped account has `@engine9/plugins/e9workers`. Pass that path plus a worker submodule. **Do not** call MCP `account` to look up a plugin id. **Do not** install a separate Echo plugin.

| On-demand `path` | Typical `method` |
|------------------|------------------|
| `@engine9/plugins/e9workers:EchoWorker` | `echo` (smoke test) |
| `@engine9/plugins/e9workers:SQLWorker` | `query` |
| `@engine9/plugins/e9workers:SegmentWorker` | `list`, `detail`, `build`, `buildSegmentPersonFile` |
| `@engine9/plugins/e9workers:InventoryWorker` | `inventory` (cache lookup), `buildInventorySummaryFile` |

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

For account-specific plugins (RENxt, …), discover `path` + `method` from MCP `account` plugins, then call `task`. Slash shorthand (`renxt/people`) is resolved against that list.

## Fallback workflow: no native match → `account` → `task`

**Rule:** Installing a package is a native match. Use MCP `plugin` `install` ([e9-plugin](../e9-plugin/SKILL.md)). Do not enter this fallback.

When the user's request does not map cleanly to a native tool:

1. **Ensure account scope** — `account_id` must be known from **this chat session** (`engine9.account_id` after `/e9a`), an explicit user statement, or MCP `account` search when the user asked you to find matching accounts. If missing, **ask the user** or suggest `/e9a <account_id>` and stop — do not read leftover CLI files or compiled account catalogs for scope. If you only know org/prefix/plugin constraints and the user wants discovery, call `account` with `command: "search"` first. If search returns no accounts, **stop** — do not look up ids on disk.
2. **Pick the schedule mode:**
   - **Predefined / built-in flow** (`flow_id` such as `identity-rebuild`, or account-published slug): call `task` with `flow_id` only (optional `label`). First task is **paused** by default — resume `paused_task_run_id` to start. See [e9-tasks-api deploy-flow.md](../e9-tasks-api/deploy-flow.md).
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

Do **not** use remote-legacy dotted paths (`channelbots.RENxtBot.People`).

### Example: scheduling a plugin method

User: "List custom fields on RENxt people for account `<account_id>`"

1. No native tool for this specific plugin method → fallback path.
2. Call `account` with `{ "account_id": "<account_id>" }`.
3. Find plugin with alias `renxt`, submodule `People`, method `listCustomFields`.
4. Call `task`:

```json
{
  "account_id": "<account_id>",
  "path": "@frakture-com/channelbots/RENxtBot:People",
  "method": "listCustomFields"
}
```

5. To poll status, call `task` again with `action: "listTasks"` and `flow_run_id` / `task_run_ids` from the schedule response.

## Account-scoped calls

All tools except `ok`, `plugin_id`, and `input_id` require authentication. Account-scoped tools (including `file` and `apiKey` except `command: catalog`) require an `account_id` the signed-in user can access. Do not guess account ids. Do not infer them from leftover local CLI state or compiled account catalogs — those are for the `e9` / `e9a` bin scripts or the MCP host, not the agent. For MCP, ask for scope, require `/e9a`, or use MCP `account` search. If MCP does not return the account or parent, stop.

### Parent / all scope — do not fan out DB access

When the user asks for **all accounts**, **parent** children, or other multi-account remote flow-run views (e.g. list errored remote runs via MCP `task` `action: "list"` → `TaskWorker.listRemoteFlowRuns` / remote-legacy `POST /flow_runs/filter`):

- Do **not** call `account` plugins (or any account-DB worker) once per child to “check access”.
- Do **not** use the first id in a parent/all list as a required DB-connected `account_id` before the remote list.
- Drive the request with remote multi-account filters (`parent_account_id`, `account_ids`, etc.) and status filters. Use Prefect `state_type` values only (`FAILED`, `RUNNING`, `COMPLETED`, `PAUSED`, …). Legacy Mongo tokens (`complete`, `error`, `in_progress`) are **rejected with 422**. Account database connectivity is not a prerequisite for remote-legacy flow-run list reads.
- The hard-stop on `Cannot connect to the … database` still applies to tools that truly need that account DB (`sql`, `eql`, `search`, `timelinePerson`, `timelinePersonLegacy`, single-account plugin schedule path resolution). It must **not** block multi-account remote flow-run listing.
- Listing is one request. **Counting or metrics** for those runs is also one request (`task` `action: "count"` / `"metrics"` — same `parent_account_id` / `account_ids`). **Archiving or bulk-retrying** those runs is also one request: reuse the same `parent_account_id` / `account_ids` and send every `flow_run_id` together ([bulk archive](#bulk-archive--retry-of-flow-runs)). Do **not** fan out one MCP call per child.

### `task` action `list` — remote flow runs

MCP `task` with `action: "list"` calls `TaskWorker.listRemoteFlowRuns` (`POST /flow_runs/filter` on the remote-legacy Task API). Returns **flow runs only** — nested `task_runs` are not included. Each flow run includes `account_id`, `parent_account_id` (first id in that account's `parent_ids`, or `null`), and `parent_ids`.

MCP `task` with `action: "count"` calls `POST /flow_runs/count` (same filters as list). Returns `{ count }`. Use with the current `status` filter for “N of total”.

MCP `task` with `action: "metrics"` calls `POST /flow_runs/metrics`. Returns `{ count, total, FAILED, RUNNING, COMPLETED }` (extra types such as `PAUSED` only when matching). Rule: omit `status` so pills ignore the current state filter.

MCP `task` with `action: "listTasks"` (or `"debug"`) calls `TaskWorker.listRemoteTaskRuns` (`POST /task_runs/filter` on the remote-legacy Task API) for a specific `flow_run_id` / `task_run_ids`. The result is `{ task_runs: [ … ], flow_run? }` — the same shape as REST `POST /task_runs/filter`. Pass `remote: false` to list local runs. Each `task_run` / `flow_run` includes `account_id`, `parent_account_id`, and `parent_ids`. Each `task_run` includes **`log_link`** (`/task_runs/{id}/log` on the Task API). Listings omit **`checkpoints`**. Display `state.name` (aka `state_name`); color/group by `state.type` (`state_type`). Render commands from `allowed_actions` (`pause`, `resume`, `retry`, `stop`, `update_options`, `reset_checkpoints`). Do **not** read deprecated `status` (Mongo vocabulary).

MCP `task` with `action: "get"` calls `GET /task_runs/:id` (`TaskWorker.getRemoteTaskRun`). This is the single-task detail read: `resolved_options`, `output`, and **`checkpoints`** (`[{ modified, options }]` from worker `modify_history`). A task can have **multiple** checkpoints, oldest first. Checkpoints can exceed 1MB — request them only for one task. Workers write them; `PATCH` does not create a checkpoint. **Retry does not clear them.** `allowed_actions` includes `reset_checkpoints` when the job has checkpoints.

MCP `task` with `action: "resetCheckpoints"` (alias `"reset_checkpoints"`) calls `POST /task_runs/:id/reset_checkpoints`. Reset only walks **back from the most recent**: `start_index` N keeps `checkpoints[0 .. N)` and drops later ones. Omitted / `0` clears all (RESET ALL). You cannot yank a checkpoint out of the middle while keeping later ones. Same as GraphQL `job_reset_checkpoint`. Not status-gated.

MCP `task` per-task-run controls (Firebase / MCP session — **do not** send `e9key_` keys):

| Action | Task API | Notes |
|--------|----------|--------|
| `start` | `POST /task_runs/:id/retry` | Run now. Accepts `force`, `start_after: "previous"`, `start_after_task_run_id` |
| `retry` + `task_run_id` | `POST /task_runs/:id/retry` | Same as start; required for "Run after previous job" |
| `pause` / `resume` | `POST /task_runs/:id/pause` / `/resume` | True pause (not job kill). Offer only when `allowed_actions` contains the command |
| `stop` | `POST /task_runs/:id/stop` | Kill. `set_state` `CANCELLED` equivalent |
| `updateOptions` | `PATCH /task_runs/:id` `{ options }` | Pending/paused only; 409 when RUNNING/terminal |
| `describe` | `POST /tasks/describe` | Method option metadata for `path` (marketplace first, then Frakture). `method` optional. Does not enqueue |
| `get` | `GET /task_runs/:id` | Single-task details including **`checkpoints`**. Do not use `listTasks` for this |
| `resetCheckpoints` | `POST /task_runs/:id/reset_checkpoints` | Walk back checkpoints. `{ "start_index": 0 }` (default) is RESET ALL; `N` keeps the first N (oldest) and drops later ones. Cannot yank from the middle. Retry does not clear. Alias: `reset_checkpoints` |
| `log` / `output` | `GET /task_runs/:id/log` / `/output` | `{ log_link, log, truncated }` and optional signed **`log_url`** from remote-legacy; prefer **`log_link`** for integrations |
| `archive` / bulk `retry` | `POST /flow_runs/archive` / `/retry` | All `flow_run_ids` in one call. After parent/all list, pass `parent_account_id`. **`user_id` is not required**. See [bulk archive](#bulk-archive--retry-of-flow-runs) |
| `count` | `POST /flow_runs/count` | Same filters as `list`. `{ count }`. Ignores limit |
| `metrics` | `POST /flow_runs/metrics` | FAILED / RUNNING / COMPLETED pills. Omit `status` so pills ignore the current state filter |

Same account-scope auth as `action: "list"` (account header + bearer). Do **not** ask the user for a remote-legacy `user_id`.

Status filters take Prefect types only, e.g. `{ "status": ["FAILED"] }`. `422` = invalid body, legacy status token, retry of a RUNNING run without `force`, or `resetCheckpoints` with a non-integer `start_index`. `409` = action not in `allowed_actions` (Prefect-styled message).

Example — walk back the most recent checkpoint (keep the first two):

```json
{
  "action": "resetCheckpoints",
  "account_id": "<account_id>",
  "task_run_id": "<task_run_id>",
  "start_index": 2
}
```

Example — errored flow runs under a parent:

```json
{
  "action": "list",
  "account_id": "<parent_account_id>",
  "parent_account_id": "<parent_account_id>",
  "status": ["FAILED"],
  "limit": 300
}
```

Status pills for the same parent (no `status` so FAILED / RUNNING / COMPLETED are independent of the list filter):

```json
{
  "action": "metrics",
  "account_id": "<parent_account_id>",
  "parent_account_id": "<parent_account_id>"
}
```

### Bulk archive / retry of flow runs

`action: "archive"` and bulk `action: "retry"` take **every** `flow_run_id` in one MCP call. Do **not** archive or retry one id at a time. Do **not** send `user_id`. Confirm with the user before a mass archive (count + how many accounts). Prefer terminal runs (`COMPLETED`, `FAILED`, `CANCELLED`, `CRASHED`) unless the user asked to archive in-progress work. The worker chunks at **500** ids (remote find/archive limit) — that is still one MCP call.

Single account:

```json
{
  "action": "archive",
  "account_id": "<account_id>",
  "flow_run_ids": ["<flow_run_id>", "<flow_run_id>"]
}
```

#### Multiple accounts (parent / all — 100+ children)

`action: "list"` already returns errored runs across children in one request (`parent_account_id` / `account_ids`). Frakture `POST /flow_runs/archive` is the same: it looks up job lists by **global** ids. The only thing that used to force 100 remote hops was engine9 sending `X-Account-Id` for a single owner, which made Frakture skip every other account's ids.

**Do not** make one MCP `task` call per child. **Do not** add a JSON-RPC / generic batch endpoint. Pass the same multi-account flag as list so the remote hop **omits** `X-Account-Id` (server-token admin context, same as list). Then one Frakture POST archives every id, regardless of which child owns it.

1. **List once** — `{ "action": "list", "account_id": "<parent_account_id>", "parent_account_id": "<parent_account_id>", "status": ["FAILED"], "limit": 300 }` (page if needed). No per-child DB.
2. Collect every flow run `id`. Confirm with the user.
3. **Archive once** with those ids and the same `parent_account_id` (or `account_ids`):

```json
{
  "action": "archive",
  "account_id": "<parent_account_id>",
  "parent_account_id": "<parent_account_id>",
  "flow_run_ids": ["<flow_run_id_1>", "<flow_run_id_2>"]
}
```

`account_id` is MCP auth (must be an account the user can access — typically the parent). `parent_account_id` / `account_ids` is what drops the remote account header. Omit that flag and mixed-owner ids are silently skipped.

REST: `POST /flow_runs/archive` with `flow_run_ids` plus `parent_account_id` or `account_ids` in the body. See [e9-tasks-api endpoints](../e9-tasks-api/endpoints.md#post-flow_runsarchive).

Bulk `retry` with `flow_run_ids` uses the same flag. Per-task `retry` (`task_run_id`) stays single-run.

## Quick validation flow

After [Step 0 — Log in](#step-0--log-in-always-first):

1. Call `user` to confirm signed-in identity and account access.
2. If account id is unknown, call `account` with `command: "search"` and the known filters (prefix/parent/plugin). If `count` is `0`, **stop** — do not read compiled catalogs. Otherwise set scope via `/e9a <account_id>` or call `account` plugins to cache methods.
3. Call `search` with a known email to validate account-scoped data access.
4. For parent/all **remote flow-run** requests, skip step 2–3 per-child probes — use multi-account remote filters only.

## Related documentation

| Topic | Documentation |
|-------|---------------|
| How MCP lists accounts (flat `parent_ids`) | [server/api/mcp/accounts.md](../../server/api/mcp/accounts.md) |
| Install plugins on accounts | [e9-plugin](../e9-plugin/SKILL.md) |
| Cursor setup and `/e9` commands | [e9-cli](../e9-cli/SKILL.md) |
| API-key creation and scope rules | [e9-api-key](../e9-api-key/SKILL.md) |
| Direct HTTP task and flow execution | [e9-tasks-api](../e9-tasks-api/SKILL.md) |
| EQL query syntax and SQL table analysis (`analyze`) | [e9-eql](../e9-eql/SKILL.md) |
| Plugin-installed reports | [e9-reports](../e9-reports/SKILL.md) |
| Plugin settings (define + catalog) | [create-engine9-plugin](../create-engine9-plugin/SKILL.md#settings) |
