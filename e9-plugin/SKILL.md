---
name: e9-plugin
description: >-
  Install, list, and configure engine9 account plugins via native MCP `plugin`
  (listAvailable, install, settings, setSetting). Fast path for deploying a
  package onto one account or every child of a parent. Use when the user says
  install a plugin, deploy a package, listAvailable, plugin settings, setSetting,
  or asks to install `@engine9/plugins`, `@engine9/interfaces`, or a models
  deployment path on accounts. Do not use create-engine9-plugin (authoring) or
  MCP `task` for install.
---

# engine9 plugin install

Plugins are first-class on a connected engine9 MCP server. Installing a package onto accounts is a native `plugin` tool call, not a worker task and not local `PluginWorker` / `e9 pluginworker`. Use this skill for deploy, listAvailable, settings catalog, and setSetting. Use [create-engine9-plugin](../create-engine9-plugin/SKILL.md) only when **authoring** a package.

## Quick reference

| Need | Call |
|------|------|
| Sign in | `mcp_auth` `{}`, then `ok` / `user` — [e9-mcp](../e9-mcp/SKILL.md#step-0--log-in-always-first) |
| Catalog of installable paths (this server) | `plugin` `{ "command": "listAvailable", "account_id": "<any accessible id>" }` |
| Install on one account | `plugin` `{ "command": "install", "account_id": "<account_id>", "path": "<package>" }` |
| Find children of a parent | `account` `{ "command": "search", "parents": ["<parent_account_id>"], "limit": 500 }` |
| Verify a path is installed | `account` `{ "command": "search", "parents": ["<parent_account_id>"], "plugins": ["<path substring>"], "limit": 500, "max_scan": 500 }` |
| Settings catalog / update | `plugin` `settings` / `setSetting` |

**Rule:** Prefer MCP `plugin`. Never schedule `PluginWorker.install` via MCP `task`. Never invent paths from local `plugins/` or `interfaces/`.

**Rule:** `listAvailable` is the installable catalog for the **connected server**. If the path is absent, stop and report it — do not read workspace packages as a workaround.

## Concepts

`plugin` `install` writes a `plugin` row (and schema / settings) into **that account's warehouse**. There is no bulk-install RPC: one MCP call per `account_id`. Account **search** stays one call.

| Path form | Example |
|-----------|---------|
| Full package (preferred) | `@engine9/plugins/models/deployment/v2026_09_16` |
| Shorthand from catalog | `models/deployment/v2026_09_16`, `e9email` |
| User omits `@` | `engine9/plugins/...` → match `listAvailable` (`@engine9/plugins/...`) |

Unique packages (`metadata.unique`, typical `@engine9/plugins/*` and most interfaces) reuse the existing row. Reinstall is idempotent for that path.

Declared `settings` are inserted on first install only. Changing a value later is `setSetting`, not reinstall. Marketplace `auth_fields` are not settings.

## Workflow

Copy this checklist:

```
- [ ] mcp_auth → user
- [ ] listAvailable once; resolve path
- [ ] Resolve account ids (one id, or account search)
- [ ] plugin install per account_id (parallel batches)
- [ ] Verify with one account search (plugins filter)
```

### 1. Log in

Follow [e9-mcp Step 0](../e9-mcp/SKILL.md#step-0--log-in-always-first). If `user` already succeeded this session, skip.

### 2. Resolve the package path

Call `plugin` `listAvailable` with any `account_id` the user can access (session id, parent, or a child). Catalog is server-wide.

```json
{ "command": "listAvailable", "account_id": "<account_id>" }
```

Match the user's string to a returned path (exact, or add `@` / treat as suffix). If nothing matches, **stop**.

### 3. Resolve target accounts

| User said | Accounts |
|-----------|----------|
| One id | That `account_id` only |
| "all clients of `<parent>`" / children | `account` search `parents: ["<parent_account_id>"]` (direct children). Do **not** install on the parent unless named |
| Nested descendants | Same plus `"recursive": true` |
| Prefix / tags / type | `account` search with those filters |
| Session parent/all (`engine9.account_ids`) | Every id in that list |

**Rule:** Discover ids only via MCP `user` / `account` search. If `count` is `0`, stop. Do not read `accounts.d` or compiled catalogs.

Do **not** call `account` `plugins` (or any other per-child DB probe) before install. Install does not need the method catalog.

### 4. Install

For each target `account_id`:

```json
{
  "command": "install",
  "account_id": "<account_id>",
  "path": "@engine9/plugins/models/deployment/v2026_09_16"
}
```

Run in parallel batches of about 10. Success looks like `{ ok: true, command: "install", result: { id, path, name, deployedVersion, tablePrefix, included } }`.

**Rule:** Fleet install is the exception to the global MCP "stop on first `isError`" rule. A per-account `Cannot connect` / `Not authorized` / unmet dependencies is a **skip** for that id. Continue the remaining accounts. Session-level auth failure and `getPluginMetadata is not a function` still **stop** the whole job. Report a success/fail table at the end.

### 5. Verify

One `account` search — do not fan out `command: plugins`:

```json
{
  "command": "search",
  "parents": ["<parent_account_id>"],
  "plugins": ["models/deployment/v2026_09_16"],
  "limit": 500,
  "max_scan": 500
}
```

`count` should equal the child list. `matched_plugins` should include the installed path. Retry only the missing ids.

## `/e9 plugin` commands

After [e9-cli](../e9-cli/SKILL.md) login:

| Command | MCP |
|---------|-----|
| `/e9 plugin listAvailable` | `plugin` `listAvailable` (`account_id` from session) |
| `/e9 plugin install <path>` | Resolve path, then `install` on `engine9.account_ids` if set (parent/all), else `engine9.account_id` |
| `/e9 plugin settings [path]` | `plugin` `settings` |
| `/e9 plugin setSetting <plugin_id> <name> <value>` | `plugin` `setSetting` |

Natural language ("install `<path>` on all `<parent>` clients") is the same workflow without requiring `/e9a` first: search, then install.

## Rules

**Rule:** Native MCP `plugin` only. Do not use `task`, local `PluginWorker`, or `e9 pluginworker` when the connected server exposes `plugin`.

**Rule:** Do not load [create-engine9-plugin](../create-engine9-plugin/SKILL.md) for an install request.

**Rule:** Do not call `account` plugins once per child to decide whether to install. Unique-path reinstall is safe; verify afterward with one search.

**Rule:** `listAvailable` missing the path means it is not installable on **this** server. Local checkout of the package is not sufficient.

## Troubleshooting

| Symptom | Action |
|---------|--------|
| Path not in `listAvailable` | Report the catalog; stop. The MCP host must ship the package |
| `Cannot install … unmet plugin dependencies` | Install listed dependency paths first on that account, then retry |
| `Cannot install … excluded by` | An installed stack forbids the path; do not force |
| `Cannot connect to the <account_id> database` | Skip that account; continue the fleet |
| `Not authorized for account` | Skip that id; do not look it up on disk |
| `getPluginMetadata is not a function` | Abort the job (server plugin metadata is broken) |
| Search `count` below child count | Reinstall only the missing ids; check `warnings` on the search |

## Related documentation

- [MCP tools](../e9-mcp/SKILL.md)
- [CLI / account scope](../e9-cli/SKILL.md)
- [Author a plugin or interface](../create-engine9-plugin/SKILL.md)
- [Plugin settings authoring](../create-engine9-plugin/SKILL.md#settings)
