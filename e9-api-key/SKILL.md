---
name: e9-api-key
description: >-
  Create and manage engine9 API keys (e9key_ / e9publickey_) — generic non-session
  auth whose scopes gate HTTP APIs (public signup/payment forms, inbound, Task API,
  and other routes). MCP apiKey covers catalog, list, get, create, update, revoke,
  rotate, SqlApiKeyStore (hash only), and a key-management UI. Use when working with
  API keys, e9key, e9k, scopes, rotate/revoke, public forms, or a keys admin screen.
---

# engine9 API keys

engine9 API keys provide generic authentication for non-session HTTP callers, including public forms, inbound integrations, data routes, and the Task API. Use this skill to create, scope, rotate, revoke, or integrate `e9key_` and `e9publickey_` credentials. MCP and console users authenticate through sessions instead; API keys are for machines and other HTTP clients.

## Quick reference

| Need | Use |
|------|-----|
| Discover scopes and form fields | MCP `apiKey` with `command: catalog` |
| Create or manage a key | MCP `apiKey` with `create`, `update`, `rotate`, or `revoke` |
| Authenticate an HTTP request | `Authorization: Bearer <key>` plus `X-ENGINE9-ACCOUNT-ID` |
| Schedule tasks | Grant `tasks:read` and/or `tasks:schedule` |
| Use a public form | Grant `public` and only the required write scopes |

**Rule:** Never send an `e9key_` or `e9publickey_` credential to MCP.

## Concepts

There is one key model; **scopes** decide what a key can do. The Task API is one consumer, not the definition of a key.

Typical callers:

- Public **signup** and **payment** forms (`public`, `people:write`, …)
- Site / inbound HTTP (`people:write`, `tables:write`, `data:read`)
- Machine integrations, including scheduling work (`tasks:read`, `tasks:schedule`)

MCP (`POST /mcp`) stays on Firebase / session / `localdev`. That path is distinct. Keys remain the common mechanism for HTTP APIs: `Authorization: Bearer e9key_…` (or `e9publickey_…`) plus `X-ENGINE9-ACCOUNT-ID`. Do not send `e9key_` to MCP, and do not send MCP session tokens to key-authenticated routes.

Humans **manage** keys through **MCP `apiKey`**. Machines **use** the issued secret on HTTP.

**Storage and verification always go through `@engine9/core`** (`SqlApiKeyStore` in the account `api_key` table). MCP does not invent a second store. Only the SHA-256 hash is retained; plaintext is returned **once** on create and rotate.

## Workflow

**Rule:** Do not schedule `SQLWorker.createApiKey` (or rotate/revoke) via MCP `task`; the plaintext key would land in task run output.

| Need | Call |
|------|------|
| Known scopes + form fields | `apiKey` `command: catalog` (no `account_id`) |
| Keys on an account | `apiKey` `command: list` |
| One key (no secret) | `apiKey` `command: get` + `id` |
| New key | `apiKey` `command: create` — show `key` to the user immediately |
| Change name / scopes / role / expiry / active | `apiKey` `command: update` (does not rotate the secret) |
| Disable | `apiKey` `command: revoke` |
| New secret | `apiKey` `command: rotate` — show `key` once; old id is revoked |

CLI fallbacks (no MCP): server WorkerRunner `e9 sqlworker createApiKey` / `listApiKeys` / `updateApiKey` / `revokeApiKey` / `rotateApiKey` on a server account, or core `npx e9 create-api-key` on a core-only site (`@engine9/core` `bin/e9.js` — not the same binary as server `bin/e9`; see `@engine9/core` README “The e9 CLI”).

## Scopes

The scope list **is** the permission model. The same key type covers forms, inbound, reads, and tasks; attach only the scopes that surface needs. Keys **must** list scopes at creation. Empty scopes deny every check. `admin` grants all scopes. Prefix follows scopes at create/rotate: `public` → `e9publickey_…`, otherwise `e9key_…`. Changing scopes later does **not** change an existing key's prefix — rotate if you need the other prefix.

**Rule:** Grant only the scopes required by the calling surface; empty scopes deny every permission check, while `admin` grants all scopes.

| Scope | Surface | Allows |
|-------|---------|--------|
| `people:write` | Core `POST /people` | Inbound people pipeline |
| `tables:write` | Core `POST /upsert/:table` | Allowlisted table upserts |
| `data:read` | Core `GET /read/:name` | Configured reads |
| `tasks:read` | Task API | List/read flows and run status |
| `tasks:schedule` | Task API | Schedule and control work |
| `admin` | Any | All scopes |
| `public` | Inbound / forms | Public ingest (`e9publickey_` prefix) |

Constants: `SCOPES` from `@engine9/core`. Canonical list for UIs: MCP `apiKey` `command: catalog` → `scopes[]`.

A public signup or payment form typically needs:

```text
public,people:write
```

A partner that discovers flows and schedules tasks needs:

```text
tasks:read,tasks:schedule
```

## MCP examples

Catalog (build the permissions form from this — do not hardcode scopes):

```json
{ "command": "catalog" }
```

List:

```json
{ "account_id": "<account_id>" }
```

Create (plaintext in `key`, `shown_once: true`):

```json
{
  "command": "create",
  "account_id": "<account_id>",
  "name": "public-signup",
  "scopes": ["public", "people:write"]
}
```

Task-API partner (same key model, different scopes):

```json
{
  "command": "create",
  "account_id": "<account_id>",
  "name": "partner-tasks",
  "scopes": ["tasks:read", "tasks:schedule"]
}
```

Update permissions without rotating:

```json
{
  "command": "update",
  "account_id": "<account_id>",
  "id": "<key_id>",
  "scopes": ["tasks:read"]
}
```

Rotate / revoke: pass `command` and `id`. Rotate copies metadata when omitted.

List returns `keys[]`. Get/update/revoke return `record`. None of those include plaintext or `key_hash`. Create/rotate include plaintext in `key` once (`shown_once: true`) — display it and do not write it to files, chat logs, or task output.

## Auth

- **Managing keys:** signed-in MCP user with access to `account_id` (same gate as `sql` / `task`).
- **Using keys:** `Authorization: Bearer e9key_…` + `X-ENGINE9-ACCOUNT-ID` on any HTTP route that verifies layer-1 keys (core site APIs, public forms, Task API, and other scoped endpoints). MCP `POST /mcp` does not accept `e9key_` keys.

## CLI (when MCP is not available)

```
e9 sqlworker createApiKey -a <account_id> --name partner-tasks --scopes tasks:read,tasks:schedule
e9 sqlworker listApiKeys -a <account_id>
e9 sqlworker updateApiKey -a <account_id> --id <key_id> --scopes tasks:read
e9 sqlworker rotateApiKey -a <account_id> --id <key_id>
e9 sqlworker revokeApiKey -a <account_id> --id <key_id>
```

Core-only (no `accounts.d`):

```
npx e9 create-api-key --db sqlite://./engine9.db --name website --scopes admin
```

## Related documentation

| Topic | Documentation |
|-------|---------------|
| MCP login and tool behavior | [e9-mcp](../e9-mcp/SKILL.md) |
| Task API authentication | [authentication.md](../e9-tasks-api/authentication.md) |
| Task API workflows | [e9-tasks-api](../e9-tasks-api/SKILL.md) |
| Key-management UI contract | [ui.md](ui.md) |
