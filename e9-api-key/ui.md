# API key management UI

Build the keys screen from MCP `apiKey`. Do not query `api_key` with MCP `sql` and do not schedule `createApiKey` via `task`.

## Screen flow

1. **`command: catalog`** — permissions checklist, field list, and plaintext rules. Cache for the session.
2. **`command: list`** + `account_id` — table of keys (name, scopes, active, expires_at, timestamps). No secret column.
3. **Create** — form from `catalog.fields` where `create: true`. `scopes` is required (`required_on_create`). On success, show a **copy-once** dialog for `key` (`shown_once: true`). Never persist that value in app storage.
4. **Edit permissions** — `command: update` with `id` plus changed fields. Scope edits do not issue a new secret.
5. **Rotate** — confirm, then `command: rotate`. Same copy-once dialog. Old id is in `revokedId`.
6. **Revoke** — `command: revoke`. Row stays on list with `active: false` unless you pass `include_inactive: false` or `active: true`.

`get` / `update` / `revoke` return `{ record }`. `list` returns `{ keys, count }`. Filters on list: `name`, `id`, `active`, `include_inactive` (default true).

## Catalog (form source)

`command: catalog` returns:

| Field | Use |
|-------|-----|
| `prefixes.standard` / `prefixes.public` | Labels (`e9key_` vs `e9publickey_`) |
| `scopes[]` | Checkbox list: `id`, `allows`, `surface`, `prefix` |
| `fields[]` | Which inputs appear on create vs update (`create` / `update` / `nullable`) |
| `plaintext.shown_on` | `["create","rotate"]` — when to show the copy-once dialog |
| `plaintext.stored` | Always `false` |
| `commands` | Supported MCP commands |

Do not hardcode the scope list in the client — render `scopes[]`.

Prefix is chosen at **create/rotate** from whether `public` is in `scopes`. Updating scopes later does not change the existing token prefix. If the UI needs the other prefix, rotate.

## Record shape (list / get / update / revoke)

```json
{
  "id": "<uuid>",
  "name": "partner-tasks",
  "scopes": ["tasks:read", "tasks:schedule"],
  "default_role_id": null,
  "active": true,
  "expires_at": null,
  "created_at": "…",
  "modified_at": "…"
}
```

Never displayed or stored: plaintext `key`, `key_hash`.

## Create / rotate success

```json
{
  "ok": true,
  "command": "create",
  "shown_once": true,
  "key": "e9key_…",
  "id": "<uuid>",
  "name": "partner-tasks",
  "scopes": ["tasks:read", "tasks:schedule"],
  "default_role_id": null,
  "account_id": "<account_id>"
}
```

Rotate adds `revokedId`. Show `key` immediately; closing the dialog loses it.

## Auth for the UI

The management UI is an MCP client: Google / engine9 session, then `apiKey` with `account_id` the user can access. Issued keys are for Task API / core HTTP callers — do not send `e9key_` on `POST /mcp`.
