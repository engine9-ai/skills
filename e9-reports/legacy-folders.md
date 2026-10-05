# Legacy report folders

Stored folders from an old report system. A small number of legacy accounts still open them. Read this file only when the user explicitly asked to list those folders, read one, or change who can open a folder.

Rule: Do not open this file, recommend these folders, or call MCP `legacyReportFolder` unless the user explicitly asked to manage a legacy report folder.

Rule: Do not code against these folders. Do not write dashboards, SQL, widgets, plugins, filters, or new report definitions from them. Do not treat `report_ids`, layouts, or folder fields as a schema. The only report implementation to author or consume is the JSON dashboard format in [SKILL.md](SKILL.md), through MCP `report`.

Rule: These folders are not plugins and are not part of the plugin report system. `legacyReportFolder` exists only as a legacy operation for accounts that still use the old folders. Do not extend it. Do not build features on it.

## Quick reference

| Need | MCP `legacyReportFolder` |
| --- | --- |
| Stored folders for one account | `command: list` with `account_id` |
| Stored folders plus compiled standard slug parents | `list` with `include_standard: true` |
| One stored folder | `command: get` with `id` (stored id or short) |
| Create or flip the one child of a parent | `command: ensureChild` (admin). `folder_id` plus `visibility` |
| Change visibility when the stored id is already known | `command: setVisibility` (admin) |

`account_id` is the account that owns the folder. On `ensureChild` it is the account that owns the child. `target_account_id`, when sent, must equal `account_id`.

Every success is `{ ok: true, legacy: true, warning, data, meta? }`. The warning means the payload is not a schema. Stop if a follow-on task is to build a report from it.

## Rules

Rule: `list` and `get` are reads. `ensureChild` and `setVisibility` change who can open a folder and require the admin role (`user.accounts[id].level`).

Rule: `visibility` is `public` or `private`. Comparison is case-insensitive and the stored value is lowercase. A public folder makes every report in it available without a per-report sign-in. A private folder does not. A report can still be public on its own.

Rule: `ensureChild` keeps one child per account and parent. The parent key (`folder_id`) is a stored id, a short, or a standard slug such as `messages` or `transactions`. Calling it again with the other visibility updates that same child. It does not create a second folder. The response `data.id` is the child to use with `get` and `setVisibility`.

Rule: `get` and `setVisibility` accept a stored id or a short. A standard slug is rejected. Use `ensureChild` when the only key is the parent slug.

Rule: `include_standard: true` on `list` merges compiled standard slug parents with that account's stored folders. Omit it, or send `false`, for stored folders only. There is no paging. `meta.count` is the number of rows in `data`.

Rule: `account_id` on `get` and `setVisibility` must be the folder's owning account. An account outside that scope is unauthorized.

Rule: Show the service `error` text when a call fails. Diagnostic details are not the message.

## Folder object

Report layouts and data sources are not included. These fields describe a stored folder so an explicit legacy request can be completed. They are not a contract for new reports.

| Field | Type | Notes |
| --- | --- | --- |
| `id` | string | Stored id |
| `short` | string | Public URL key. `""` when unset |
| `account_id` | string | Owning account |
| `parent_id` | string or null | Parent stored id, or the standard slug this child was ensured from |
| `label` | string | `""` when inherited or unset |
| `slug` | string | `""` when unset. Standard definitions use this as their identity |
| `visibility` | string | `public`, `private`, or `""` when unset |
| `disabled` | boolean | `true` only when the field is strictly true |
| `type` | string | `standard` on compiled definitions. `""` when unset |
| `report_ids` | string array | Ids stored on the folder. May be empty |
| `date_created` | string | ISO-8601. Omitted when unset |
| `last_modified` | string | ISO-8601. Omitted when unset |
| `slugs` | object | Present only when the folder has slug overrides |

## Examples

List stored folders:

```json
{ "command": "list", "account_id": "<account_id>" }
```

List stored folders and standard slug parents:

```json
{ "command": "list", "account_id": "<account_id>", "include_standard": true }
```

Read one stored folder:

```json
{ "command": "get", "account_id": "<account_id>", "id": "<stored id or short>" }
```

Create the account child of a standard slug, or flip that child to public. Admin role:

```json
{
  "command": "ensureChild",
  "account_id": "<account_id>",
  "folder_id": "messages",
  "visibility": "public"
}
```

Set a stored folder private when its id is already known. Admin role:

```json
{
  "command": "setVisibility",
  "account_id": "<account_id>",
  "id": "<stored id or short>",
  "visibility": "private"
}
```

## Errors

| Status | When |
| --- | --- |
| 400 | List is missing the account. `include_standard` is not a boolean the service accepts. `folder_id`, the target account, or `visibility` is missing. `visibility` is not `public` or `private`. `id` is not a stored id or short |
| 401 | Missing or invalid auth. The account is outside the caller's accounts. The account does not match the child owner or the folder's account |
| 403 | Write with a read-only credential |
| 404 | Stored folder does not exist, or `ensureChild` cannot find the parent |
| 500 | The folder operation failed |

Validation before the call: `id` is required for `get` and `setVisibility`. `folder_id` and `visibility` are required for `ensureChild`. `visibility` is required for `setVisibility`. `target_account_id` must equal `account_id`.

## Troubleshooting

A request to add a dashboard, chart, filter, or plugin report is the JSON format in [SKILL.md](SKILL.md). Do not start from a legacy folder, its `report_ids`, or a standard slug.

A request to make an old folder public or private, or to see which stored folders an account still has, is `legacyReportFolder`. Confirm the user asked for that before calling it.

## Related documentation

- [engine9 reports](SKILL.md) — the report implementation to code against
- [engine9 MCP](../e9-mcp/SKILL.md)
