---
name: e9-export
description: >-
  Create, run, and debug engine9 export files with the e9 CLI
  (`e9 exportworker export`). Covers canonical export bundles with top-level
  `universe` entries, person-search plugin exports (`plugin:exports:<name>`),
  definition_path, inventory.json5 on bundle export, legacy bundle compatibility,
  and empty/missing export debug. Pre-run warehouse checks: [e9-inventory](../e9-inventory/SKILL.md).
  Use when working with ExportWorker, export, definition_path, hockeystick, idv1
  copies, export parquet, or a missing/wrong export file.
---

# engine9 exports

Exports write account data out of the warehouse: **table parquet**, **idv1 file copies**, and/or **person-search CSVs**. Run them with the **e9 CLI**. Do not show JavaScript `exportWorker.export({...})` with legacy method names (`exportAll`, `exportTables`).

```
e9 exportworker <method> -a <account_id> --<option>=<value>
```

`-a` is the account id from `accounts.d` (same as `e9 sqlworker ok`). Options are `--snake_case` flags. Complex values can be piped as JSON5 on stdin.

Related: warehouse inventory [e9-inventory](../e9-inventory/SKILL.md); person-search remotes [e9-person-remote](../e9-person-remote/SKILL.md); identity [e9-person-id](../e9-person-id/SKILL.md); timeline files [e9-timeline](../e9-timeline/SKILL.md).

Naming: export roots, worker options, and universe entries are canonical
`snake_case`; JavaScript implementation variables remain camelCase. Nested
search and transform options remain owned by their individual handlers.
Known camelCase universe aliases are accepted only for compatibility, normalize
to snake_case, and conflicting dual spellings are rejected.

## Methods

| Method | What it does |
|--------|----------------|
| `inventory` | Same as [e9-inventory](../e9-inventory/SKILL.md) via `exportworker`. Prefer `inventoryworker` for inventory-only runs. |
| `export` | Unified export: infers mode from `definition_path` and options (bundle dump, one person-search CSV, or tables-only). Writes `inventory.json5` (plan only) when a bundle runs. |

`exportAll` and `exportTables` were removed; both error with a message to use `export`.

## `definition_path`

| Shape | Meaning |
|-------|---------|
| `engine9-accounts/.../export` | Bundle plugin (canonically a top-level `universe`). No `:exports:` required. Runs bundle tables/files plus named plugin exports. |
| `engine9-accounts/.../file.plugin.js:exports:<name>` | One named person-search export. |
| Path to a `.json5` / `.json` file | Standalone export definition (`search` and/or bundle keys). |

Without `definition_path`, pass `--tables` (and optionally `--extra_tables` / `--exclude_tables`) for a tables-only dump.

Use `--export_name=<name>` with a plugin path to run one named export without `:exports:<name>` in the path.

Pre-run warehouse inventory: [e9-inventory](../e9-inventory/SKILL.md).

## Bundle dump (tables + idv1 files)

A bundle module canonically exports one top-level `universe` array. Each entry describes one source artifact set:

```
universe: [
  {
    type: 'table',
    table: 'person',
    transforms: [{ path: ':transforms:removePII' }]
  },
  {
    type: 'inputs',
    input_type: 'message',
    channel: 'email',
    files: '\\.idv1\\.parquet$',
    transforms: [{ path: ':transforms:removePII' }]
  },
  {
    type: 'inputs',
    entry_types: ['EMAIL_OPEN', 'EMAIL_CLICK'],
    relative_path: 'timeline/{{input_id}}/{{filename}}',
    transforms: [{ path: ':transforms:removePII' }]
  }
]
```

- **Table entries** use `type: 'table'` and `table: '<warehouse_table>'`. Put one table in each universe entry. `relative_path` and file transforms belong to that entry. Default: `tables/{{table}}.parquet`.
- **Input entries** use `type: 'inputs'`. Their selectors are `input_type`, `channel`, `plugin_id` / `plugin_path`, and/or `entry_types`; `files` optionally narrows the selected idv1 basenames. Default: `{{input_type}}/{{input_id}}/{{filename}}` (unknown `input_type` → `timeline`).
- Inputs export `.idv1.parquet` files, not raw CSV/JSON.
- `files` is an optional regular expression applied to idv1 basenames.
- `entry_types` uses exact names; `EMAIL_*` is not a wildcard.
- `relative_path` is always below `export_dir`. Absolute paths, `..`, backslashes, missing template values, and duplicate targets are rejected before writing.
- Record counts: `metadata.json` when `> 0`; otherwise parquet `COUNT(*)`.

Relative-path variables are `table`, `input_id`, `input_type`, `filename`, `export_name`, `export_id`, `date`, and `datetime_prefix`, where applicable.

### Legacy bundle compatibility

Existing definitions may still use top-level `tables` and `input_directories`. Treat those as compatibility syntax, not the format to copy into new definitions:

- `tables: ['person']` or `{ name: 'person', ... }` corresponds to a `universe` entry `{ type: 'table', table: 'person', ... }`.
- An `input_directories` selector corresponds to `{ type: 'inputs', ...selector }`.
- Existing `extra_tables` and `exclude_tables` belong to the legacy table-list form.

When updating a bundle, prefer expressing the intended artifacts directly as top-level universe entries.

### Artifact transforms

`transforms` on an individual `universe` entry are file-to-file transforms for that table or selected input file. The referenced transform definition must declare `scope: 'file'`; this distinguishes artifact transforms from searched-export row transforms. Each file transform receives:

```
{ filename, target, artifact, options, account_id, ...resolvedBindings }
```

It must write `target` and may return `{ filename: target }`. Transform chains feed each returned file into the next step. Put the implementation in the definition plugin's `transforms` map and reference it as `:transforms:<name>` or `@self:transforms:<name>`; full plugin paths also work. Artifact transforms fail closed: a transform error aborts the run and the original file is not copied to that target.

`removePII` is intentionally not built yet. Once available, add it without server changes:

```
transforms: {
  removePII: {
    scope: 'file',
    transform: removePII
  }
}

// On a type:'table' or type:'inputs' universe entry:
transforms: [{ path: ':transforms:removePII', options: { fields: ['email', 'phone'] } }]
```

Example (Hockeystick):

```
e9 inventoryworker inventory -a <account_id> --definition_path=engine9-accounts/frakture/liftoff/hockeystick/export
e9 exportworker export -a <account_id> --definition_path=engine9-accounts/frakture/liftoff/hockeystick/export
```

Optional: `--limit=N`, `--export_id=<uuid>`, `--start=-30d`, `--end=`.

## Person-search export

A searched export is a definition with top-level `search` (EQL) and top-level `transforms` for row/column mapping. These are not `scope: 'file'` universe transforms:

```
exports: {
  transactions: {
    search: { /* EQL */ },
    transforms: [{ /* row mapping */ }]
  }
}
```

Invoke the named path:

```
e9 exportworker export -a <account_id> --definition_path=engine9-accounts/frakture/advantage_ai/index.plugin.js:exports:transactions --start=-30d
```

A bundle `export` also runs named `:exports:` entries that have `search`.

`--person_ids=1,2,3` scopes the search. `--dedupe=false` ignores prior export files. `--add_to_input_store` writes into an input store instead of only `exports/`.

Empty “remote people for plugin X” audiences: [e9-person-remote](../e9-person-remote/SKILL.md) — filter is `person_remote` ⋈ `input.plugin_id`, not `{prefix}person`.

## Tables-only

```
e9 exportworker export -a <account_id> --tables=person,transaction --limit=100
e9 exportworker export -a <account_id> --extra_tables=custom --exclude_tables=setting
```

Without `--tables`, tables-only mode uses the standard warehouse list (`plugin`, `input`, `person*`, `source_code_dictionary`, `global_message`, `transaction`, …).

## Output

Default directory: `{store_path}/{account_id}/exports/{export_id}/{date}/`

Bundle `export` and `inventory.json5` include `source_directory`: the export root path. Strip it from any absolute output `filename` to recover the relative path under that root (same value as `export_dir` on the export result). Input file and directory entries also carry `source_directory` for the input-store root the file was copied from.

| Path | Contents |
|------|----------|
| `tables/{table}.parquet` | Default warehouse table dumps |
| `{input_type}/{input_id}/*.idv1.parquet` | Default copied idv1 files (`message`, `person`, `timeline`, …) |
| Definition-owned `relative_path` | Custom universe artifact destination below `export_dir` |
| `inventory.json5` | Bundle export: export **plan** only (paths, counts, skipped). Monthly statistics: [e9-inventory](../e9-inventory/SKILL.md). |
| `search/{export_name}.export.csv` + metadata | Default named person-search output during a bundle export |

`directories[].files` is the **file list** (name, filename, records, source_directory), not a count.

## Authoring a bundle

Put `index.js` at `engine9-accounts/<org>/<account>/export/` (or use a `*.plugin.js`) and export a top-level `universe`. Use one `{ type: 'table', table }` entry per warehouse table and `{ type: 'inputs', ...selectors }` entries for input-store files. A plugin may also contain searched exports under `exports: { name: { search, transforms } }` and file-transform implementations under the plugin's top-level `transforms` map.

`channel: 'email'` needs `global_message` or `message` with that channel. An `entry_types` selector finds stores whose `metadata.json` contains at least one exact listed type.

Only idv1 files are copied. Raw files in the store are ignored even if `metadata.json` lists them.

## Debug (A–G)

Walk **A → G**. Stop at the first gap. Run [inventory](../e9-inventory/SKILL.md) before `export`.

```
Export pipeline:
- [ ] A: definition_path resolves (bundle and/or :exports:name)
- [ ] B: `type: 'table'` universe entries inventory the expected tables (or skipped does_not_exist)
- [ ] C: `type: 'inputs'` selectors inventory the expected directories / idv1 files
- [ ] D: file records are useful (not metadata zeros)
- [ ] E: planned `relative_path` and transforms are correct and collision-free
- [ ] F: export wrote every planned artifact under exports/{id}/{date}/
- [ ] G: person-search file has rows (search, remotes, dates)
```

### A) Path

`inventory` / bundle `export` errors “No exports found” or “requires definition_path” → wrong path or no recognized bundle entries. See [e9-inventory](../e9-inventory/SKILL.md). Bundle directories do **not** use `:exports:`.

### B) Tables

`inventory.skipped_tables` with `does_not_exist` → that table is not deployed (summaries often need a stats job). Table export skips the same way. Confirm with `e9 sqlworker query -a <account_id>` / `DESCRIBE`.

### C) Files missing from inventory

`directories` with no nested `files`, or `files: []`:

- A `type: 'inputs'` entry's `channel=email` selector needs matching `global_message` / `message` rows; without one it matches no stores. A separate `entry_types` selector can find stores by metadata.
- `metadata.json` `entry_types` missing the exact names in the selector → store skipped (debug log).
- Only raw files on disk → correctly omitted. Need `.idv1.parquet`.

### D) `records: 0`

Stale `metadata.json` / `input.records` of 0 is ignored; inventory should count parquet. If still 0 after that, the idv1 is empty. Check `inventory.json5` `directories[].files[].records` and `file_records`.

### E) Plan

Check `inventory.json5` before inspecting output. Inventory format and statistics: [e9-inventory](../e9-inventory/SKILL.md).

### F) Files on disk

Bundle `export` returns `export_dir` and `source_directory` (same root). Every file listed by `inventory.json5` should exist at its `relative_path`. Missing copies: listing found no idv1s (C), a transform failed, or a file copy was skipped (logged).

### G) Person-search empty

`search` too tight (`start`/`end`, plugin filter). “Remotes for plugin X” with no rows → [e9-person-remote](../e9-person-remote/SKILL.md). `limit` on a bundle export applies per named export **and** per bundle table.

Record the first failing step.
