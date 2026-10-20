---
name: e9-custom-fields
description: >-
  Add vendor custom fields to an installed plugin's data pull (people,
  transactions/gifts/donations, and other submodules) by appending to the
  engine9 plugin setting `<submodule>_custom_fields` (people_custom_fields,
  transaction_custom_fields) through MCP `plugin` settings / setSetting.
  Covers locating the account and plugin, the hard stop when no setting
  exists (legacy Frakture-only accounts, method-option-only plugins),
  listCustomFields, dedupe-and-append, optional varchar(255) columns on
  `<table_prefix>person` / `<table_prefix>transaction` tables, and an optional
  backdating load. Use when asked to add, pull, or sync custom fields, extra
  fields, or additional columns from a CRM or vendor plugin.
---

# Add custom fields to a plugin pull

A plugin's people or transaction pull brings in standard fields plus any vendor custom fields named in a plugin setting. Adding fields means appending names to that setting, never replacing it. Use this skill when someone asks to add one or more custom fields to a people, transaction, gift, or donation pull for an account. The work is MCP-only and starts with a check that the setting exists; when it does not, stop.

## Quick reference

| Step | MCP call |
| --- | --- |
| Find the account | `account` `{ "command": "search", "prefix": "<account_id>" }` |
| Find the plugin and submodule | `account` `{ "command": "plugins", "account_id": "<account_id>" }`, plus recent runs with `task` `{ "action": "list", "search": "<plugin>", "include_task_runs": true }` |
| Confirm the setting exists (gate) | `plugin` `{ "command": "settings", "account_id": "<account_id>", "path": "<plugin path>" }` |
| List available custom fields | `task` `{ "path": "<plugin path>:<Submodule>", "method": "listCustomFields" }` |
| Append the fields | `plugin` `{ "command": "setSetting", "plugin_id": "<uuid>", "name": "<setting>", "value": [...] }` |
| Find existing tables (optional) | `sql` `{ "command": "tables", "filter": "^<table_prefix>(person\|transaction)" }` |
| Add columns (optional, after asking) | `sql` `ALTER TABLE <table> ADD COLUMN <field> varchar(255)` |
| Backdate (optional, after asking) | `task` with the submodule's load method and a wide `start` |

| Submodule | Setting |
| --- | --- |
| People | `people_custom_fields` |
| Gifts, Donations, Transactions | `transaction_custom_fields` |
| Other submodules | `<submodule>_custom_fields`. Confirm the name and `section` in the settings catalog |

Rule: The setting name and section come from MCP `plugin` `settings`. The table above is the convention, not a substitute for checking.

## Concepts

Engine9 plugin settings live in the warehouse `setting` table, one set per plugin row. Submodule settings carry `section: "<Submodule>"` in the catalog. A `*_custom_fields` setting is an array of vendor field identifiers that the pull requests in addition to standard fields.

Older plugins accept a `custom_fields` option on the load method itself, set in each dataflow job. That is the legacy mechanism. This skill does not edit method options.

Modern pulls write to input-store files, not database tables. Some accounts also keep plugin-prefixed warehouse tables (`<table_prefix>person`, `<table_prefix>transaction`) that need a matching column before new fields show up there.

## Workflow

```
- [ ] A. Account and plugin found (MCP only)
- [ ] B. <submodule>_custom_fields setting exists — otherwise HARD STOP
- [ ] C. listCustomFields run on the right submodule
- [ ] D. Requested fields matched to exact identifiers
- [ ] E. Setting updated: existing values kept, new values deduped and appended
- [ ] F. Asked whether existing tables need varchar(255) columns
- [ ] G. Offered a backdating load
```

### A. Find the account and plugin

1. Log in and search the account per [e9-mcp](../e9-mcp/SKILL.md#step-0--log-in-always-first). A `count: 0` miss is a stop.
2. Call `account` `plugins`. Note each candidate's `path`, `id` (plugin uuid), `table_prefix`, and `metadata.submodules`.
3. When several instances of the vendor plugin exist, or it is unclear which job is the pull, list recent flow runs with `task` `action: "list"`, a `search` on the vendor name, and `include_task_runs`. The running job's plugin path and submodule (for example `...RENxtBot.People` with method `loadPeople`) identify the pull.

Rule: Discover the plugin from MCP only. Do not read local account configs or plugin source.

### B. Confirm the setting exists (hard stop gate)

Call `plugin` `settings` for the account, filtered by `path` or `plugin_id` when known. Look for a setting named `<submodule>_custom_fields` with `section` equal to the submodule.

Rule: If the setting is not in the catalog, stop. Report what was checked and end the workflow. There is no point listing fields that cannot be saved.

Each of these is a hard stop:

| Signal | Meaning |
| --- | --- |
| `account` `plugins` returns `[]` and `plugin` `settings` returns `plugins: []` | The account has no engine9 plugin rows. Typical of a legacy Frakture-only warehouse (no `plugin` table; plugins appear only in `bot_metadata`) |
| engine9 is deployed (core `@engine9/schemas/*` rows exist) but the vendor plugin is not in `account` `plugins`, and its package is not in `plugin` `listAvailable` | The vendor integration still runs only as a Frakture bot. It cannot be installed on this server, so it has no settings |
| The plugin is in `settings` but has no `<submodule>_custom_fields` | The package does not declare the setting |
| `task` `describe` shows `custom_fields` only as an option on the load method | Legacy method-option mechanism only |

Rule: Do not work around a missing setting. Do not edit `custom_fields` on dataflow jobs, run `addCustomField`, or change the load flow unless the user explicitly asks for that legacy path.

### C. List custom fields

Schedule the submodule's `listCustomFields` with `task` (`path` = `<plugin path>:<Submodule>`, `method: "listCustomFields"`, and `plugin_id` when more than one instance exists). Poll with `action: "listTasks"` and read the result with `action: "output"`.

Rule: Run it on the submodule that matches the setting. People custom fields and gift custom fields are separate catalogs in most CRMs.

### D. Match the requested fields

Match the user's names, which are often vendor UI labels, to the identifiers returned by `listCustomFields`. Use the identifier form that the setting's current values use.

Rule: When a requested name matches zero or several fields, show the candidates and ask. Do not guess.

### E. Append to the setting

1. Read the current `value` from the settings response. It may be an array or a comma-separated string.
2. Build the new list: every existing value in its original order, then each new identifier not already present (case-insensitive compare).
3. Show before and after, then call `setSetting` with the full array.

```json
{
  "command": "setSetting",
  "account_id": "<account_id>",
  "plugin_id": "<plugin uuid>",
  "name": "people_custom_fields",
  "value": ["existing_field_a", "existing_field_b", "new_field_c", "new_field_d"]
}
```

Rule: Append only. Do not remove or reorder existing custom fields without explicit guidance from the user.

4. Call `settings` again and confirm the stored value.

### F. Existing tables (ask first)

Ask whether the new fields must also land in existing warehouse tables. Modern pulls keep data in files, so the answer is often no.

When yes, find candidates with `sql` `tables` using the plugin `table_prefix` (`^<table_prefix>person`, `^<table_prefix>transaction`). Ignore `_archive_*` tables. Check `describe` for columns that already exist, then add only the missing ones:

```sql
ALTER TABLE <table_prefix>person ADD COLUMN <field> varchar(255)
```

Rule: New columns are `varchar(255)` unless the user specifies a type.

Rule: DDL needs the admin role. Show the exact statements and get confirmation before running them.

### G. Backdate (optional)

The setting only affects future pulls. To fill history, offer to schedule the submodule's load method with a wide window. Check its options first with `task` `describe` (`start` / `end`, `interval`, `table`).

```json
{
  "account_id": "<account_id>",
  "path": "<plugin path>:People",
  "method": "loadPeople",
  "plugin_id": "<plugin uuid>",
  "options": { "start": "none" }
}
```

Rule: Offer the backdate; do not schedule it without confirmation. A full-history pull can be large and hit vendor rate limits.

## Troubleshooting

| Symptom | Action |
| --- | --- |
| `account` `plugins` is empty but `sql` shows `<prefix>_person` tables | Legacy Frakture-only account. Hard stop at step B; report it |
| Setting exists but the value is a string | Split on commas, trim, then append and write back an array |
| New fields missing after the next pull | Confirm the stored setting value; check the pull's log; check whether the target table needs the column (step F) |
| `setSetting` validation error | Match the declared `type` (`array`); send JSON, not a quoted list |

## Related documentation

- [MCP tools](../e9-mcp/SKILL.md)
- [Plugin install and settings](../e9-plugin/SKILL.md)
- [Declaring plugin settings](../create-engine9-plugin/SKILL.md#settings)
