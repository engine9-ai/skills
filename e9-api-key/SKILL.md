---
name: e9-api-key
description: >-
  Create and manage engine9 API keys (e9key_ / e9publickey_) and their permission
  scopes via MCP apiKey. Covers catalog, list, get, create, update, revoke, rotate,
  SqlApiKeyStore storage (hash only), Task API scopes (tasks:read / tasks:schedule),
  and building a key-management UI. Use when working with API keys, e9key, e9k,
  key permissions, scopes, rotate/revoke, or a keys admin screen.
---

# engine9 API keys

Layer-1 credentials for core site APIs and the server **Task API**. Humans manage keys through **MCP `apiKey`** (Firebase / session / `localdev`). Machines **use** keys as `Authorization: Bearer e9key_…` (or `e9publickey_…`). Do not mix those credentials.

**Storage and verification always go through `@engine9/core`** (`SqlApiKeyStore` in the account `api_key` table). MCP does not invent a second store. Only the SHA-256 hash is retained; plaintext is returned **once** on create and rotate.

Using a key against the Task API: [e9-tasks-api](../e9-tasks-api/SKILL.md) / [authentication.md](../e9-tasks-api/authentication.md). MCP login and tool selection: [e9-mcp](../e9-mcp/SKILL.md). UI payload contract: [ui.md](ui.md).

## Prefer MCP `apiKey`

Do **not** schedule `SQLWorker.createApiKey` (or rotate/revoke) via MCP `task` — the plaintext key would land in task run output.

| Need | Call |
|------|------|
| Known scopes + form fields | `apiKey` `command: catalog` (no `account_id`) |
| Keys on an account | `apiKey` `command: list` |
| One key (no secret) | `apiKey` `command: get` + `id` |
| New key | `apiKey` `command: create` — show `key` to the user immediately |
| Change name / scopes / role / expiry / active | `apiKey` `command: update` (does not rotate the secret) |
| Disable | `apiKey` `command: revoke` |
| New secret | `apiKey` `command: rotate` — show `key` once; old id is revoked |

CLI fallbacks (no MCP): `e9 sqlworker createApiKey` / `listApiKeys` / `updateApiKey` / `revokeApiKey` / `rotateApiKey` on a server account, or `npx e9 create-api-key` on a core-only site.

## Scopes

Keys **must** list scopes at creation. Empty scopes deny every check. `admin` grants all scopes. Prefix follows scopes at create/rotate: `public` → `e9publickey_…`, otherwise `e9key_…`. Changing scopes later does **not** change an existing key's prefix — rotate if you need the other prefix.

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
- **Using keys:** `Authorization: Bearer e9key_…` + `X-ENGINE9-ACCOUNT-ID` on core APIs and the Task API. MCP `POST /mcp` does not accept `e9key_` keys.

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
