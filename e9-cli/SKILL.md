---
name: e9-cli
description: Connect Cursor to engine9 MCP — log in first via mcp_auth, set account scope with /e9a, and handle /e9 command-style interactions including plugin install, person search, and task scheduling.
---

# engine9 CLI

Use this skill to configure or troubleshoot an engine9 MCP connection in Cursor or another MCP client. It also defines `/e9` and `/e9a` command behavior, including account scope, plugin install, person search, reports, segments, API keys, and task scheduling.

## Quick reference

| Need | Action |
|------|--------|
| Sign in | Call `mcp_auth` with `{}`, then verify with `ok` and `user` |
| Select one account | `/e9a <account_id>` |
| Select children of a parent | `/e9a parent <parent_account_id>` |
| Install a plugin | `/e9 plugin install <path>` — [e9-plugin](../e9-plugin/SKILL.md) |
| Search people | `/e9 search <terms>` |
| Run an on-demand method | `/e9 task <plugin-path-or-alias> <method> [options...]` |
| Configure Cursor | Add the server to `~/.cursor/mcp.json` and permissions to `~/.cursor/permissions.json` |

**Rule:** On any MCP tool error, follow [e9-mcp — MCP tool errors — stop immediately](../e9-mcp/SKILL.md#mcp-tool-errors--stop-immediately); do not call downstream account-scoped tools after a failed `account`, `task`, `search`, or similar call.

## Step 0 — Log in (always first)

**Rule:** Every `/e9` request starts by logging in. Do not grep, curl, read config files, start servers, or run CLI commands to "figure out" auth.

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

When handling `/e9` or `/e9a` requests, **do not read local workspace code** to discover plugins, worker methods, paths, or options. Local source does not reliably match the connected MCP server or the account's installed plugins. Use MCP tool schemas, MCP `account` (cached as `engine9.plugins`), and MCP `task` / `sql` / `analyze` / `timelinePerson` / `timelinePersonLegacy` / `file` / `apiKey` only. See [e9-mcp — MCP-only discovery](../e9-mcp/SKILL.md#mcp-only-discovery--do-not-use-local-code).

When an account or parent is not found, do NOT dig deeper into compiled account catalogs, etc. Account discovery when using MCP should only be through that MCP, not through any other mechanisms. Do not read `accounts.d/`, `accounts.compiled.json5`, or other on-disk catalogs.

**Rule:** Discover accounts, plugins, methods, paths, and options from the connected MCP server only.

## Concepts

1. Cursor MCP config (`mcp.json`, `permissions.json`)
2. Auth expectations (OAuth vs `localdev` bearer)
3. `/e9a` account scope and plugin cache
4. `/e9` session behavior, search parsing, and task scheduling

## Server endpoint and startup

engine9 MCP is exposed at `POST /mcp` (with `GET /mcp/health` for health checks).

**Production default:** `https://data.engine9.ai/mcp` (same host as the Task API and `/data` routes).

- Preferred server startup: run the main API (`node api/index.js`) so MCP and other APIs share one port.
- Standalone MCP startup: from `engine9/server`, run `npm run mcp`.
- Local standalone MCP URL:
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
      "url": "https://data.engine9.ai/mcp"
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

Examples: `/e9a <account_id>`, `/e9a parent <parent_account_id>`, `/e9a all`

The CLI bin script (`server/bin/e9a`) writes `.e9_parameters` for subsequent **local `e9` worker CLI runs only**. Agents handling `/e9a` in Cursor must **not** run that bin script or read the files it writes.

**Do not use leftover CLI files or compiled catalogs for MCP / Cursor session scope.** Never read `.e9_parameters`, `.e9_config.json5`, `accounts.d/`, `accounts.compiled.json5`, or similar local state to infer `account_id`. If chat session `engine9.account_id` is unset, **ask the user** (or suggest `/e9a <account_id>`) — do not invent scope from disk.

**Discovering accounts (MCP):** when the user asks which accounts match a prefix, parent, type, tags, or installed plugin, call MCP `account` with `command: "search"` in **one** request — do not use `/e9a all` plus per-account plugin loads. Example: `{ "command": "search", "prefixes": ["<prefix>"], "plugins": ["acoustic"] }`. If that search returns `count: 0`, **stop** and report that MCP found no matching accounts. Use `/e9a <id>` afterward to pin a single primary account and cache its plugins.

### Required behavior

**Default `/e9a <lookup>`:**

1. Do not call `user` (or any account lookup tool) to resolve `<lookup>`.
2. Do not perform fuzzy matching, alias matching, or ambiguity checks.
3. Set the provided value as active account scope immediately.
4. Call MCP `account` with `{ "account_id": "<lookup>" }` (plugins command; `command: "plugins"` optional).
5. On success (`{ ok: true, plugins: [...] }` or `{ ok: true, command: "plugins", plugins: [...] }`), persist the returned `plugins` array for subsequent `/e9` requests.
6. On failure (`isError: true`, or text like `Cannot connect to the … database` / `Not authorized for account`): **stop**. Report the error. Do not cache plugins. Do not proceed to `/e9 task` or other account-scoped calls.
7. Reuse stored account and plugins unless the user runs another `/e9a`.

**`/e9a parent <parent_id>` or `/e9a all`:**

1. Resolve ids **only via MCP**. Parent: `account` `{ "command": "search", "parents": ["<parent_id>"] }` (comma-separated parents → `parents` array; results are flat rows with `parent_ids`). All: MCP `user` and take active keys of `accounts` (`disabled` not true); each entry includes flat `parent_ids` (no nested tree).
2. If search `count` is `0` or `user.accounts` is empty: **stop**. Report that MCP did not find the parent or any accounts. Do **not** read `accounts.d` / compiled catalogs.
3. Set `engine9.account_ids` to the MCP list; set `engine9.account_id` to the first id (label only — not a DB probe target). Clear `engine9.plugins`.
4. Do **not** load plugins for every account (or for the first id). Do **not** call MCP `account` plugins / open account databases to “check access” across children. Report the resolved ids and count.
5. For **remote task listing** under parent/all scope, see [Multi-account remote flow runs](#multi-account-remote-flow-runs-parent--all) — those calls must not fan out per-child DB access.

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
- `/e9 plugin listAvailable` — MCP `plugin` `command: listAvailable`
- `/e9 plugin install <path>` — MCP `plugin` `command: install` on session account(s); see [e9-plugin](../e9-plugin/SKILL.md)
- `/e9 plugin settings [path]` — MCP `plugin` `command: settings`
- `/e9 search foo@bar.com`
- `/e9 segment list` — MCP `segment` with `command: list`
- `/e9 segment build <segment_id|definition_path>` — MCP `segment` with `command: build`
- `/e9 report list` — MCP `report` with `command: list`
- `/e9 report run <path>` — MCP `report` with `command: run` (see [e9-reports](../e9-reports/SKILL.md))
- `/e9 apiKey list` / `/e9 apiKey create …` — MCP `apiKey` (see [e9-api-key](../e9-api-key/SKILL.md))
- `/e9 task <plugin-path-or-alias> <method> [options...]`

Account/domain create and secrets: use **e9-account** (`cloud-services/e9-account`), not MCP `/e9 domain`.

### Session and account requirements

- Every MCP call that is account-scoped MUST include `account_id` **when the op is single-account** (search, sql, eql, plugin schedule, etc.).
- If `account_id` is already known in **this chat session** (`engine9.account_id` from `/e9a` or an explicit user statement), reuse it.
- If scope is missing, **ask the user** (single id, `/e9a parent …`, or `/e9a all`) before running scoped tools. Stop and wait — do not proceed with a guessed id.
- Do not guess `account_id`. Do **not** treat `.e9_parameters`, `.e9_cli_history`, `accounts.d/`, `accounts.compiled.json5`, or other leftover local files as session scope.

### Multi-account remote flow runs (parent / all)

Use this for requests like “list current errored tasks”, cross-account job status, or anything backed by MCP `task` `action: "list"` → `TaskWorker.listRemoteFlowRuns` / remote-legacy `POST /flow_runs/filter` when the user wants **parent** or **all** scope. For FAILED / RUNNING / COMPLETED totals without paging the list, use `action: "metrics"` (`POST /flow_runs/metrics`) and omit `status`; use `action: "count"` with the current `status` for “N of total”. To list tasks inside one flow run, use MCP `action: "listTasks"` → `TaskWorker.listRemoteTaskRuns` / remote-legacy `POST /task_runs/filter` (or REST `POST /task_runs/filter`).

- Prefer remote multi-account filters: `parent_account_id` for parent scope, or the remote API’s multi-account / auth-scoped listing for all — **not** a loop of per-account MCP calls.
- Use Prefect `state_type` filters only (`FAILED`, `RUNNING`, `COMPLETED`, `PAUSED`, …). Legacy Mongo tokens (`complete`, `error`, `in_progress`) are rejected with 422.
- Do **not** call MCP `account` (plugins) for each child, and do **not** pick `engine9.account_id` (the first resolved id) and probe that account’s database as a gate.
- Do **not** treat account-DB failures (`Cannot connect to the … database`) as blocking for these ops — remote flow-run listing does not need account DBs.
- Do **not** require `engine9.plugins` for remote flow-run listing; plugin cache is for single-account plugin method scheduling.
- Single-account rules (load plugins via `/e9a <id>`, hard-stop on DB connect for `account`/`task` schedule path resolution) still apply when scheduling a plugin method on one account.
- **Archive / bulk retry** of those listed runs is also one request: reuse `parent_account_id` / `account_ids` and send every `flow_run_id` together. Do **not** fan out one MCP call per child. See [e9-mcp — Bulk archive](../e9-mcp/SKILL.md#bulk-archive--retry-of-flow-runs).

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
   - **Multi-step flow** (user asks to deploy/schedule a flow such as identity rebuild): use MCP `task` with `flow_id` (e.g. `identity-rebuild`) — see [e9-tasks-api/deploy-flow.md](../e9-tasks-api/deploy-flow.md). Do not expand steps into on-demand path/method calls. If MCP returns flow-not-found and you have a local `engine9/server` checkout with the flow file, fall back to `e9 task runFlow -a <account> --flow=rebuild/identity-rebuild.flow.json5`.
   - **On-demand method**: require `engine9.account_id` and (for non-`e9workers` paths) `engine9.plugins` (run `/e9a` first if missing).
   - If `/e9a` or MCP `account` failed for this account, **stop** — do not call `task`.
   - Resolve the plugin path from cached plugins before calling MCP `task`.
   - Pass the canonical **plugin path** (colon submodule form), not remote-legacy dotted paths.
   - If MCP `task` returns `isError: true` or a fatal error message, **stop** — do not retry or call other account tools.
4. `/e9 segment list`:
   - Ensure active `account_id` exists (or request it / suggest `/e9a`).
   - Call MCP `segment` with `{ "command": "list", "account_id": "<account_id>" }`.
5. `/e9 segment build ...`:
   - Ensure active `account_id` exists.
   - Call MCP `segment` with `command: build` and either `segment_id` or `definition_path` from the user args.
6. `/e9 report list`:
   - Ensure active `account_id` exists (or request it / suggest `/e9a`).
   - Call MCP `report` with `{ "command": "list", "account_id": "<account_id>" }`.
7. `/e9 report run <path>`:
   - Ensure active `account_id` exists.
   - Call MCP `report` with `command: run`, `path`, and any filter options (`start`, `end`, `limit`).
8. `/e9 apiKey …`:
   - Catalog needs no account. Other commands need `account_id`.
   - Call MCP `apiKey`. Never schedule `createApiKey` via `task`. Show plaintext `key` to the user immediately on create/rotate.
9. `/e9 plugin …`:
   - Follow [e9-plugin](../e9-plugin/SKILL.md). Native `plugin` only — do not use `task`.
   - `listAvailable` / `settings`: session `account_id`.
   - `install <path>`: resolve path from `listAvailable`; if `engine9.account_ids` is a parent/all list, install on **each** id (parallel batches). Do not call `account` plugins first.

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

- `/e9 task @engine9/plugins/e9workers:EchoWorker echo`
- `/e9 task renxt/people listCustomFields`
- `/e9 task @frakture-com/channelbots/RENxtBot:People listCustomFields`
- `/e9 task @engine9/plugins/e9workers:SQLWorker query`

### Path forms (in priority order)

1. **Reserved engine9 Workers path** (no plugin lookup; every bootstrapped account):
   - `@engine9/plugins/e9workers:EchoWorker` + method `echo` (smoke test)
   - `@engine9/plugins/e9workers:SQLWorker` + method `query`
2. **Canonical plugin path with colon submodule** (account plugins):
   - `@frakture-com/channelbots/RENxtBot:People`
3. **Slash alias shorthand** (resolve via `engine9.plugins` before MCP):
   - `renxt/people` → find plugin where `metadata.metadata.alias` (or `metadata.alias`) is `renxt`, submodule `People` from `metadata.submodules`
4. **Bare plugin path** (no submodule): `@frakture-com/channelbots/RENxtBot` — only when the method lives on the plugin root

Do **not** pass legacy remote job paths such as `channelbots.RENxtBot.People`. Do **not** treat Echo as a separate installed plugin.

### Resolution algorithm

Before MCP `task`:

1. If the path is `@engine9/plugins/e9workers:<Worker>`, pass it through — **do not** require `engine9.plugins` / MCP `account`.
2. Otherwise require `engine9.plugins` from `/e9a` (or call MCP `account` once if missing).
3. If the user path is already `plugin.path` or `plugin.path:Submodule` and matches an installed plugin, use it.
4. If the path is `alias/submodule` (e.g. `renxt/people`), scan `engine9.plugins`:
   - Match alias (case-insensitive) on `plugin.metadata.metadata.alias` or `plugin.metadata.alias`
   - Match submodule (case-insensitive) on keys of `plugin.metadata.submodules`
   - Build `{plugin.path}:{Submodule}` (preserve submodule casing from metadata keys)
5. Call MCP `task` with `account_id` from `engine9.account_id`, resolved `path`, and `method`.

The MCP server also resolves slash/colon paths against the account plugin table if the agent passes shorthand.

### Example MCP `task` payload

User: `/e9 task renxt/people listCustomFields`  
Account: `<account_id>` (from `/e9a`)

```json
{
  "account_id": "<account_id>",
  "path": "@frakture-com/channelbots/RENxtBot:People",
  "method": "listCustomFields",
  "label": "renxt/people listCustomFields"
}
```

Use `action: "listTasks"` with `flow_run_id` and optional `task_run_ids` from the schedule response to poll status. That MCP action calls remote-legacy `POST /task_runs/filter`.

## Related documentation

| Topic | Documentation |
|-------|---------------|
| MCP tools and invocation rules | [e9-mcp](../e9-mcp/SKILL.md) |
| Install plugins on accounts | [e9-plugin](../e9-plugin/SKILL.md) |
| API-key management | [e9-api-key](../e9-api-key/SKILL.md) |
| Direct HTTP Task API | [e9-tasks-api](../e9-tasks-api/SKILL.md) |
| Plugin-installed reports | [e9-reports](../e9-reports/SKILL.md) |
| Flow deployment | [deploy-flow.md](../e9-tasks-api/deploy-flow.md) |
