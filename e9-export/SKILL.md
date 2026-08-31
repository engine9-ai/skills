---
name: e9-export
description: >-
  Create, run, and debug engine9 export files with the e9 CLI
  (`e9 exportworker exportAll`, inventory, export, exportTables). Covers
  bundle dumps (warehouse tables + idv1 input files), person-search plugin
  exports (`plugin:exports:<name>`), definition_path, inventory.json5, and
  empty/missing export debug. Use when working with ExportWorker, exportAll,
  inventory, definition_path, hockeystick, idv1 copies, export parquet, or a
  missing/wrong export file.
---

# engine9 exports

Exports write account data out of the warehouse: **table parquet**, **idv1 file copies**, and/or **person-search CSVs**. Run them with the **e9 CLI**. Do not show JavaScript `exportWorker.exportAll({...})` as the way to invoke an export.

```
e9 exportworker <method> -a <account_id> --<option>=<value>
```

`-a` is the account id from `accounts.d` (same as `e9 sqlworker ok`). Options are `--snake_case` flags. Complex values can be piped as JSON5 on stdin.

Related: person-search remotes [e9-person-remote](../e9-person-remote/SKILL.md); identity [e9-person-id](../e9-person-id/SKILL.md); timeline files [e9-timeline](../e9-timeline/SKILL.md).

## Methods

| Method | What it does |
|--------|----------------|
| `inventory` | Pre-run: `COUNT(*)` per table + list `.idv1.parquet` files with record counts. Writes nothing. |
| `exportAll` | Complete definition: bundle first, then named `:exports:` person searches. Writes `inventory.json5` when a bundle exists. |
| `export` | One person-search definition (`search` + `transforms`). |
| `exportTables` | Warehouse tables only (standard list, or `--tables` / `--extra_tables` / `--exclude_tables`). |
| `exportInputFiles` | Copy already-listed idv1 files into an `--export_dir` (used by `exportAll`). |

## `definition_path`

| Shape | Meaning |
|-------|---------|
| `engine9-accounts/.../export` | Bundle plugin (`tables` and/or `input_directories` on the module). No `:exports:` required. |
| `engine9-accounts/.../file.plugin.js:exports:<name>` | One named person-search export. |
| Path to a `.json5` / `.json` file | Standalone export definition (`search` and/or bundle keys). |

`exportAll` accepts a plugin path (bundle directory or `*.plugin.js`) or a standalone JSON/JSON5 bundle. Only a plugin can also contribute named person exports. `export` requires one person-search definition (`search`). `inventory` accepts a bundle path or `--tables` / `--input_directories` on the CLI.

## Bundle dump (tables + idv1 files)

A bundle module exports table and input-file artifacts. String table names preserve the legacy output layout; object entries can own transforms and relative paths:

```
tables: [
  'person',
  {
    name: 'transaction',
    relative_path: 'warehouse/{{table}}.parquet',
    transforms: [{ path: ':transforms:removePII' }]
  }
]
input_directories: [
  {
    input_type: 'message',
    channel: 'email',
    files: '\\.idv1\\.parquet$',
    relative_path: 'timeline/{{input_id}}/{{filename}}',
    transforms: [{ path: ':transforms:removePII' }]
  },
  { entry_types: ['EMAIL_OPEN', 'EMAIL_CLICK'] }
]
```

- **Tables** become `{datetime_prefix}.{table}.export.parquet` unless `relative_path` is set.
- **Input directories** copy only `.idv1.parquet` (never raw CSV/JSON). Specs filter `input` by `input_type`, `channel` (`global_message` / `message`), `plugin_id` / `plugin_path`, and/or `metadata.json` `entry_types`.
- `files` is an optional regular expression applied to idv1 basenames.
- `entry_types` uses exact names; `EMAIL_*` is not a wildcard.
- `relative_path` is always below `export_dir`. Absolute paths, `..`, backslashes, missing template values, and duplicate targets are rejected before writing.
- Record counts: `metadata.json` when `> 0`; otherwise parquet `COUNT(*)`.

Relative-path variables are `table`, `input_id`, `input_type`, `filename`, `export_name`, `export_id`, `date`, and `datetime_prefix`, where applicable.

### Artifact transforms

Table and input-file `transforms` are optional file-to-file plugin transforms. No transforms means the fast native table export or byte-for-byte file copy. A file transform must declare `scope: 'file'` so a person/batch transform cannot be used accidentally. Each transform receives:

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

// On the table or input-file artifact:
transforms: [{ path: ':transforms:removePII', options: { fields: ['email', 'phone'] } }]
```

Example (Hockeystick):

```
e9 exportworker inventory -a <account_id> --definition_path=engine9-accounts/frakture/liftoff/hockeystick/export
e9 exportworker exportAll -a <account_id> --definition_path=engine9-accounts/frakture/liftoff/hockeystick/export
```

Optional: `--limit=N`, `--export_id=<uuid>`, `--start=-30d`, `--end=`.

## Person-search export

A named export is `search` (EQL) + `transforms` (column mapping). Invoke the named path:

```
e9 exportworker export -a <account_id> --definition_path=engine9-accounts/frakture/advantage_ai/index.plugin.js:exports:transactions --start=-30d
```

`exportAll` runs the bundle first (inventory, tables, idv1 files), then every named `:exports:` entry that has `search`.

`--person_ids=1,2,3` scopes the search. `--dedupe=false` ignores prior export files. `--add_to_input_store` writes into an input store instead of only `exports/`.

Empty “remote people for plugin X” audiences: [e9-person-remote](../e9-person-remote/SKILL.md) — filter is `person_remote` ⋈ `input.plugin_id`, not `{prefix}person`.

## Tables-only

```
e9 exportworker exportTables -a <account_id> --tables=person,transaction --limit=100
e9 exportworker exportTables -a <account_id> --extra_tables=custom --exclude_tables=setting
```

Without `--tables`, `exportTables` uses the standard warehouse list (`plugin`, `input`, `person*`, `source_code_dictionary`, `global_message`, `transaction`, …).

## Output

Default directory: `{store_path}/{account_id}/exports/{export_id}/{date}/`

| Path | Contents |
|------|----------|
| `*.export.parquet` | Legacy/default table dumps |
| `inputs/{input_id}/*.idv1.parquet` | Legacy/default copied idv1 files |
| Definition-owned `relative_path` | Custom table/input-file destination below `export_dir` |
| `inventory.json5` | Bundle `exportAll`: planned `relative_path`, transforms, tables, files, directories, totals, and skipped items |
| `{datetime_prefix}.{export_name}.export.csv` + metadata | Default named person-search output during `exportAll` |

`directories[].files` is the **file list** (name, filename, records), not a count.

## Authoring a bundle

Put `index.js` at `engine9-accounts/<org>/<account>/export/` (or use a `*.plugin.js`) and export `tables` and/or `input_directories`. Optional `extra_tables` / `exclude_tables`. A plugin may also contain named person exports under `exports: { name: { search, transforms } }` and artifact transform implementations under `transforms`.

`channel: 'email'` needs `global_message` or `message` with that channel. An `entry_types` selector finds stores whose `metadata.json` contains at least one exact listed type.

Only idv1 files are copied. Raw files in the store are ignored even if `metadata.json` lists them.

## Debug (A–G)

Walk **A → G**. Stop at the first gap. Use `inventory` before `exportAll`.

```
Export pipeline:
- [ ] A: definition_path resolves (bundle and/or :exports:name)
- [ ] B: inventory tables exist (or skipped does_not_exist)
- [ ] C: inventory directories / files list the expected idv1s
- [ ] D: file records are useful (not metadata zeros)
- [ ] E: planned `relative_path` and transforms are correct and collision-free
- [ ] F: exportAll wrote every planned artifact under exports/{id}/{date}/
- [ ] G: person-search file has rows (search, remotes, dates)
```

### A) Path

`inventory` / `exportAll` errors “No exports found” or “requires definition_path” → wrong path. Bundle directories do **not** use `:exports:`. One named person export does. A JSON5 person-search definition is for `export`; a JSON5 bundle can be used by `inventory` or `exportAll`.

### B) Tables

`inventory.skipped_tables` with `does_not_exist` → that table is not deployed (summaries often need a stats job). `exportTables` skips the same way. Confirm with `e9 sqlworker query -a <account_id>` / `DESCRIBE`.

### C) Files missing from inventory

`directories` with no nested `files`, or `files: []`:

- Spec 1 `channel=email` joins `global_message` / `message` — no channel row → 0 matches. Spec 2 `entry_types` still can find stores.
- `metadata.json` `entry_types` missing the names in the spec → store skipped (debug log).
- Only raw files on disk → correctly omitted. Need `.idv1.parquet`.

### D) `records: 0`

Stale `metadata.json` / `input.records` of 0 is ignored; inventory should count parquet. If still 0 after that, the idv1 is empty. Check `inventory.json5` `directories[].files[].records` and `file_records`.

### E) Plan

Check `inventory.json5` before inspecting output. Every table and file includes its planned `relative_path` and artifact transforms. An unresolved transform, unsafe path, or duplicate target fails the run rather than writing a partial ambiguous layout.

### F) Files on disk

`exportAll` returns `export_dir`. Every file listed by `inventory.json5` should exist at its `relative_path`. Missing copies: listing found no idv1s (C), a transform failed, or `exportInputFiles` skipped the file (logged).

### G) Person-search empty

`search` too tight (`start`/`end`, plugin filter). “Remotes for plugin X” with no rows → [e9-person-remote](../e9-person-remote/SKILL.md). `limit` on `exportAll` applies per named export **and** per bundle table.

Record the first failing step.
