---
name: e9a
description: Set engine9 account scope directly from an exact account id for subsequent /e9 requests.
---

# Engine9 Account Selector

Use this skill to set account scope for later engine9 requests.

Primary command form:

- `/e9a <lookup>`

Examples:

- `/e9a test`
- `/e9a engine9`
- `/e9a authentic_amy_mcgrath`

## Goal

Treat `<lookup>` as the exact `accountId`, persist it immediately for all subsequent `/e9` requests, and load that account’s plugins (with metadata) into session via the MCP `account` tool.

## Required behavior

1. Do not call `user` (or any account lookup tool) to resolve `<lookup>`.
2. Do not perform fuzzy matching, alias matching, or ambiguity checks.
3. Set the provided value as active account scope immediately.
4. Call the MCP tool **`account`** with `{ "account_id": "<lookup>" }` (same string as `accountId`).
5. On a successful `account` response, persist the returned `plugins` array for subsequent `/e9` requests.
6. Reuse stored account and plugins for subsequent `/e9` calls unless the user overrides account scope with another `/e9a`.

## MCP `account` tool

- **Tool name:** `account` (not a REST path; invoked like other engine9 MCP tools).
- **Input:** `{ "account_id": "<exact lookup>" }`
- **Success body:** `{ "ok": true, "plugins": [ ... ] }` — each plugin includes DB fields plus a `metadata` object from `getPluginMetadata`.
- **Failure body:** `{ "ok": false, "error": "..." }` — e.g. not signed in, or `Not authorized for account <id>`.

Parse the tool’s text JSON content the same way as `search` or `worker_invoke` results.

## Input parsing for `/e9a <lookup>`

- Treat `<lookup>` as required text.
- Trim leading/trailing whitespace.
- Use the resulting string directly as `accountId` and as MCP `account_id`.

## Session persistence contract

Persist:

- `engine9.accountId`: exact lookup value
- `engine9.accountIds`: single-item array with that same value
- `engine9.plugins`: array from MCP `account` on success (replace entirely on each `/e9a`)

Rules:

- For `/e9a <lookup>`, always replace prior account scope and plugin list with the new values.
- Subsequent `/e9 ...` requests should use `engine9.accountId` by default and prefer `engine9.plugins` when reasoning about installed plugins (paths, names, metadata, plugin ids, aliases, submodules).
- Do not call `account` again on every `/e9` subcommand if `engine9.plugins` is already loaded for the current `engine9.accountId`; only refresh when the user runs `/e9a` again or explicitly asks to reload plugins.

### Using `engine9.plugins` for `/e9 schedule`

Each cached plugin entry includes:

- `path` — canonical plugin path (e.g. `@frakture-com/channelbots/RENxtBot`)
- `metadata` — marketplace manifest (alias, submodules, methods, auth_fields, …)

For `/e9 schedule renxt/people listCustomFields`:

1. Find plugin where `metadata.metadata.alias` (or `metadata.alias`) equals `renxt`
2. Find submodule `People` under `metadata.submodules` (match `people` case-insensitively)
3. Pass `path: "@frakture-com/channelbots/RENxtBot:People"` to MCP `schedule`

Always resolve against `engine9.plugins` before scheduling; do not invent dotted `channelbots.*` paths.

## Execution order

1. Parse and trim `<lookup>` → `accountId`.
2. Set `engine9.accountId` and `engine9.accountIds` (replace prior scope).
3. Invoke MCP `account` with `account_id: accountId`.
4. If `ok: true`, set `engine9.plugins` to the returned array.
5. If `ok: false`, clear or omit `engine9.plugins` and report the error; account scope remains set from step 2.

## Response requirements

On success:

- Confirm the exact `accountId` that was set.
- State how many plugins were loaded (length of `engine9.plugins`).
- State that future `/e9` requests will use this account and cached plugin list.

On MCP failure:

- Confirm `accountId` was set (if step 2 completed).
- Report the MCP error and that plugins were not cached.

## Suggested user-facing messages

Success:

`Active accountId set to test. Loaded 12 plugins for subsequent /e9 requests.`

Failure (unauthorized):

`Active accountId set to test, but could not load plugins: Not authorized for account test.`
