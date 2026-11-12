# engine9 exports — building

Read [SKILL.md](SKILL.md) first for **what an export contains** (tables, idv1 files, `metadata.json`, entry types). This file is for creating, running, and debugging exports with the **e9 CLI**. Do not show JavaScript `exportWorker.export({...})` with legacy method names (`exportAll`, `exportTables`).

```
e9 exportworker <method> -a <account_id> --<option>=<value>
```

`-a` is an account id the caller already has (the same id `e9 sqlworker ok` prints). Options are `--snake_case` flags. Complex values can be piped as JSON5 on stdin.

Rule: If MCP does not find that account, stop. Do not open `accounts.d`, `account-config.json`, `frakture-account-config.json`, or any other catalog to discover an id, and do not run this CLI as that lookup.

Related: warehouse inventory [e9-inventory](../e9-inventory/SKILL.md); export remotes [e9-person-remote](../e9-person-remote/SKILL.md); identity [e9-person-id](../e9-person-id/SKILL.md); timeline files [e9-timeline](../e9-timeline/SKILL.md).

Naming: export roots, worker options, and universe entries are canonical
`snake_case`; JavaScript implementation variables remain camelCase. Nested
search and transform options remain owned by their individual handlers.
Known camelCase universe aliases are accepted only for compatibility, normalize
to snake_case, and conflicting dual spellings are rejected.

## Methods

| Method | What it does |
|--------|----------------|
| `inventory` | Delegates to InventoryWorker: stats `{account root}/cache/inventory.json.gz` (`ready: true` if present). Prefer `inventoryworker`. Refresh the **account** cache with `e9 inventoryworker buildInventorySummaryFile` (no definition). Export-scoped plans use `--definition_path=…` and write `cache/inventory-plans/` instead. |
| `export` | Unified export. Writes the files a definition describes (see [What an export writes](#what-an-export-writes)). A plugin path run writes `inventory.json5` last (catalog of what was written). Does not write the account inventory cache. |

`exportAll` and `exportTables` were removed; both error with a message to use `export`.

## What an export writes

Every export writes files under `export_dir`. A definition says what rows each file holds. It does not say where the account stores them; that differs between accounts with the same export.

| Key | Each output file holds |
|-----|------------------------|
| `definition` | One row per person the definition finds, plus whatever row `transforms` add. See [Person search export](#person-search-export-definition) |
| `timeline` | One row per timeline entry in the window, from every input in scope. See [Timeline export](#timeline-export-timeline) |
| `universe` alone | Each listed warehouse table as parquet, and each selected input's `.idv1.parquet` files plus `metadata.json`, copied as-is. See [Copy exports](#copy-exports-tables--idv1-files) |

`definition` and `timeline` exports are **query exports**: each writes one new file (`{export_id}.{export_name}.export.csv` by default), and nothing they read is copied. An export with neither is a **copy**.

`universe` lists the tables and inputs the export covers. On a copy, those are what gets copied. On a query export, they limit which inputs are read.

A plugin path (`engine9-accounts/.../export`) runs both in one `export_dir`: the plugin's copies (its top-level `universe`, plus any named export that is a copy), then every named query export, then `inventory.json5`.

### Where timeline entries are read from

A `timeline` export reads each input in scope where that input's entries are. The input's row in the warehouse `input` table decides:

| `input` row | Entries are read from |
|-------------|-----------------------|
| `min_timeline_ts`, `max_timeline_ts`, or `records > 0` set (the input has an input store) | Its `.idv1.parquet` files |
| None of those set | The warehouse `timeline` table, rows with that `input_id` |

Rule: When an input has an input store, only its idv1 files are read, even if the warehouse `timeline` table also has rows for it. An entry is never written twice.

The output is the same either way. `input_id` and `plugin_id` come from the `input` row, and the other columns keep their names and types. A smaller account may keep everything in the `timeline` table and a larger one may keep most entries in input stores; the same export definition works on both.

An input store that is missing its `input` metadata is read as if it had no store. Backfill the metadata before exporting: `e9 inputworker upsertMissingInputMetadata -a <account_id>` (optionally `--plugin_id=<uuid>`).

## `definition_path`

| Shape | Meaning |
|-------|---------|
| `engine9-accounts/.../export` | Export plugin. No `:exports:` required. Copies what its top-level `universe` lists, then runs each named query export. |
| `engine9-accounts/.../file.plugin.js:exports:<name>` | One named query export. |
| Path to a `.json5` / `.json` file | Standalone definition: a query export (`definition` or `timeline`) or a copy (`universe`). |

Without `definition_path`, pass `--tables` (and optionally `--extra_tables` / `--exclude_tables`) for a tables-only dump.

Use `--export_name=<name>` with a plugin path to run one named export without `:exports:<name>` in the path.

Pre-run warehouse inventory: [e9-inventory](../e9-inventory/SKILL.md).

## Copy exports (tables + idv1 files)

A copy export plugin exports one top-level `universe` array. Each entry names something to copy:

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
- Inputs export `.idv1.parquet` files plus each store's `metadata.json` (input identity). Raw CSV/JSON is never copied.
- `files` is an optional regular expression applied to idv1 basenames. `metadata.json` is still included for each selected store.
- `entry_types` uses exact names; `EMAIL_*` is not a wildcard.
- `relative_path` is always below `export_dir`. Absolute paths, `..`, backslashes, missing template values, and duplicate targets are rejected before writing.
- Record counts: `metadata.json` when `> 0`; otherwise parquet `COUNT(*)`.

Relative-path variables are `table`, `input_id`, `input_type`, `filename`, `export_name`, `export_id`, `date`, and `datetime_prefix`, where applicable.

### Legacy table and input lists

Existing definitions may still use top-level `tables` and `input_directories`. Treat those as compatibility syntax, not the format to copy into new definitions:

- `tables: ['person']` or `{ name: 'person', ... }` corresponds to a `universe` entry `{ type: 'table', table: 'person', ... }`.
- An `input_directories` selector corresponds to `{ type: 'inputs', ...selector }`.
- Existing `extra_tables` and `exclude_tables` belong to the legacy table-list form.

When updating a definition, prefer expressing what it copies directly as top-level universe entries.

### File transforms

`transforms` on an individual `universe` entry are file-to-file transforms for that copied table or input file. The referenced transform definition must declare `scope: 'file'`; this distinguishes file transforms from query export row transforms. Each file transform receives:

```
{ filename, target, file, options, account_id, ...resolvedBindings }
```

`file` describes the file being written (`kind` is `'table'`, `'input_file'`, or `'query_export'`, plus `relative_path` and the table name, input fields, or export name). It must write `target` and may return `{ filename: target }`. Transform chains feed each returned file into the next step. Put the implementation in the definition plugin's `transforms` map and reference it as `:transforms:<name>` or `@self:transforms:<name>`; full plugin paths also work. File transforms fail closed: a transform error aborts the run and the original file is not copied to that target.

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

Example:

```
e9 inventoryworker buildInventorySummaryFile -a <account_id> --definition_path=engine9-accounts/<org>/<account>/export
e9 exportworker export -a <account_id> --definition_path=engine9-accounts/<org>/<account>/export
e9 exportworker export -a <account_id> \
  --definition_path=engine9-accounts/<org>/<account>/export \
  --export_dir=gs://bucket/exports/{{account_id}}/{{export_id}}/{{date}} \
  --credentials=s3://engine9-secrets/<account_id>/gcs-sa.json
```

Optional: `--limit=N`, `--sample=true` / `--sample=false`, `--export_id=<uuid>`, `--export_dir=<path-or-uri>`, `--credentials=<path-or-uri>`. Date windows are below — `--start` / `--end` do not mean the same thing on every export type.

`--sample=true` is a QA dump: 100 rows per warehouse table and query export CSV, 10 rows per sampled `.idv1.parquet`, one parquet per `(input_type, filename)` plus that store’s `metadata.json`. The destination last segment gets `_sample` so a sample cannot overwrite a full export (`exports/{export_id}_sample/{date}`, or `{export_dir}_sample`). `--sample=false` (default) is a full run. `--limit=N` without `--sample=true` still only caps SQL tables and person-search (full idv1 copies).

`--export_dir` is the only destination when set. Use a local directory, `temp` (a per-run folder under the OS temp directory, `{os.tmpdir()}/{account_id}/exports/{export_id}`, with no copy to the account store), or an object-store URI (`s3://`, `r2://`, `gs://` / `gcs://`, `gdrive://`). Omit it to use `{store_path}/{account_id}/exports/{export_id}/{date}/`. The bucket may be customer-owned.

`export_id` / date are **not** appended automatically under a custom `export_dir`. To mirror the default layout on a customer bucket, put placeholders in the path:

```
--export_dir=gs://bucket/prefix/{{account_id}}/{{export_id}}/{{date}}
```

Supported tokens: `{{account_id}}`, `{{export_id}}`, `{{date}}`, `{{datetime_prefix}}`. Without those placeholders, `export_dir` is used exactly as given.

`--credentials` is a path or URI to a **destination** key file (GCS service-account JSON, AWS keys, R2 keys, or Drive JSON with `subject_to_impersonate`). FileWorker reads that file with bot/default credentials (the bootstrap store), then uses the parsed key only for the destination scheme. When omitted, account `settings.file_credentials` is used; otherwise process defaults (ADC / AWS chain / `CLOUDFLARE_R2_*`). The key file is never written into `inventory.json5` or any other exported file.

## Start and end dates

`--start` and `--end` are `relativeDate` values: a relative expression such as `-3d`, `-30d`, or `-12h`, or an ISO timestamp. `end` is exclusive (`column < end`). Omit `end` for an open upper bound. The same flags do different work depending on the export type. A flag set on one type does not filter the others.

| Export type | Where the window is set | What it filters |
|-------------|-------------------------|-----------------|
| Warehouse tables (`type: 'table'`, or `--tables`) | CLI `--start` / `--end` only | Rows on the first matching date column, when the table has one |
| Person search export (`definition` + row transforms) | Default in the definition; CLI `--start` / `--end` override it | Whatever the definition's EQL and transform options reference |
| Input-store copies (`type: 'inputs'`) | EQL on that universe entry | Which stores are selected. Copied `.idv1.parquet` files are not row-filtered |
| Timeline export (`timeline`) | `timeline.start` / `end`, usually `{{or overrides.start "-30d"}}` | Each entry's `ts`, whether it is read from idv1 files or the `timeline` table |

Rule: On a copy, `--start` / `--end` filter warehouse table rows that have a recognized date column. idv1 copies stay whole.

### Warehouse tables

Table export adds `date_column >= start` and `date_column < end` when a column exists. The first match in this list wins:

`frakture_last_modified`, `ts`, `last_modified`, `frakture_date_created`, `date_created`, `modified_at`, `remote_last_modified`, `remote_date_created`.

Tables with none of those columns are exported in full. `publish_date` is not in the list, so `global_message` / `global_message_summary` are not sliced by `--start` / `--end`. There is no per-table date column in the universe entry.

```
e9 exportworker export -a <account_id> --tables=transaction --start=-3d
```

### Person search exports

CLI `--start` / `--end` become `overrides.start` / `overrides.end`. The definition must mention `overrides` or the flags are ignored. Put the default in the template; a caller replaces it with `--start=` / `--end=`.

Use double quotes inside the helper. In raw EQL, wrap with `date` so `-3d` becomes an ISO timestamp before SQL runs:

```
conditions: [
  { eql: 'timeline.ts >= \'{{date (or overrides.start "-3d")}}\'' },
  { eql: '{{#if overrides.end}}timeline.ts < \'{{date overrides.end}}\'{{else}}1=1{{/if}}' }
]
```

Leave `end` out of the `date` helper when it is blank. `date` on an empty string does not produce a timestamp.

Search and transform options take the relative string (the search runtime parses it):

```
options: {
  start: '{{or overrides.start "-3d"}}',
  end: '{{or overrides.end ""}}'
}
```

```
e9 exportworker export -a <account_id> \
  --definition_path=engine9-accounts/<org>/<account>/index.plugin.js:exports:transactions \
  --start=-3d --end=-1d
```

On a plugin path run, those flags are forwarded to every named query export except one named `people`. A `people` snapshot does not see the run's `--start` / `--end`; its templates keep the default written in that definition.

### Input-store copies

`--start` / `--end` are not applied to `.idv1.parquet` copies. Each selected store is copied whole (every idv1 file plus `metadata.json`).

To keep a rolling set of stores, put the window in that `type: 'inputs'` entry's EQL. EQL evaluates `date_sub` at runtime; it does not read CLI `--start`. Message stores use `publish_date` (one store is one message). Other stores can use `input.max_timeline_ts` to skip stores with no activity in the window — that still copies every row in a store that qualifies.

```
{
  type: 'inputs',
  eql: {
    table: 'global_message',
    columns: ['id'],
    conditions: [
      { eql: "channel='email'" },
      { eql: 'publish_date >= date_sub(now(), interval 3 day)' }
    ]
  }
}
```

`global_message` and `message` both resolve to message input ids. `entry_types` on the same entry still applies after the EQL picks ids.

To export timeline **rows** in a window (not whole stores), use a [timeline export](#timeline-export-timeline), not an input-store copy.

## Query exports

A query export builds one new file of rows. There are two: a person search (`definition`) and a timeline export (`timeline`). Both take top-level `transforms` for row/column mapping. These are not `scope: 'file'` universe transforms.

### Person search export (`definition`)

`definition` is a segment definition: EQL or segment definition clauses that select people. Rows are people (and whatever the row transforms append). `search` was removed; it errors with a message to use `definition`.

```
exports: {
  transactions: {
    definition: {
      and: [{ path: '@engine9/schemas/transaction/core:segment_definition_clause:all', options: { start: '{{or overrides.start "-30d"}}' } }]
    },
    transforms: [{ /* row mapping */ }]
  }
}
```

Invoke the named path:

```
e9 exportworker export -a <account_id> \
  --definition_path=engine9-accounts/<org>/<account>/index.plugin.js:exports:transactions \
  --start=-30d
```

The definition supplies the default window and must reference `overrides.start` / `overrides.end`. See [Start and end dates](#start-and-end-dates).

#### People found from input files

When the activity lives in idv1 files and is not loaded into warehouse `timeline`, put an input `universe` on the person search export. The export loads those stores into a query engine and runs the definition (and `sql.query` bindings whose table is `timeline`) against that, joined to the warehouse `input` rows for the same ids. The stores are read, not copied.

```
universe: [
  {
    type: 'inputs',
    entry_types: ['EMAIL_SEND', 'EMAIL_OPEN', 'EMAIL_CLICK'],
    eql: {
      table: 'input',
      columns: ['id', 'input_type'],
      conditions: [
        { eql: `(input.max_timeline_ts >= '{{date (or overrides.start "-30d")}}' or (input.max_timeline_ts is null and input.records > 0))` }
      ]
    }
  }
],
definition: {
  eql: {
    table: 'timeline',
    columns: ['person_id'],
    joins: [{ table: 'input', join_eql: 'timeline.input_id=input.id' }]
  }
}
```

`entry_types` still filters each store’s `metadata.json`. The definition EQL keeps saying `timeline`; that name is the loaded files, not the warehouse table. `--start` / `--end` apply through the definition templates, including the universe EQL. A `people` export on a plugin path run still does not receive those flags.

This path loads every selected file into a temporary query engine and searches people first. When the output is one row per entry, use a timeline export instead.

### Timeline export (`timeline`)

A `timeline` export writes one row per timeline entry into **one** output file. There is no person search and nothing is loaded into the warehouse. The definition says which entries and which columns; each input in scope is read [where its entries are](#where-timeline-entries-are-read-from).

```
exports: {
  emailExhaust: {
    timeline: {
      entry_types: ['EMAIL_SEND', 'EMAIL_OPEN', 'EMAIL_CLICK'],
      start: '{{or overrides.start "-30d"}}',
      end: '{{or overrides.end ""}}',
      columns: ['id', 'ts', 'entry_type_id', 'person_id', 'source_code_id', 'input_id', 'plugin_id']
    },
    transforms: [ /* optional: row batches, one entry per row */ ]
  }
}
```

| Key | Meaning |
|-----|---------|
| `entry_types` | Names (`EMAIL_SEND`) or ids (`40`). Omit for every entry type |
| `start` / `end` | `relativeDate` window on each entry's `ts`. `end` is exclusive. Blank means open |
| `columns` | Optional output columns: timeline entry columns plus `input_id` and `plugin_id`. Only these (and any used by the filters or `order_by`) are read. A listed column an input does not have is output empty. Default is every column |
| `order_by` | Optional column list. Default is read order (sorting reads every row before writing) |

`timeline_files` is the older name for `timeline` and still works. An export that sets both is an error.

Without a `universe`, every input is in scope; input stores whose `min_timeline_ts` / `max_timeline_ts` fall outside `start` / `end` are skipped without being listed. With a `universe`, only the inputs its `type: 'inputs'` entries select are read, and a universe with no `type: 'inputs'` entry is an error. Use a universe to narrow the inputs (for example by `plugin_id` or `input_type`), not to say where their entries are stored.

Columns: the warehouse `timeline` table holds `id`, `ts`, `entry_type_id`, `person_id`, and `source_code_id`. An idv1 file can hold more (for example `email`, `remote_entry_id`). Entries read from the table have those extra columns empty, so list only the five shared columns (plus `input_id` / `plugin_id`) when every row must have them, and look the rest up in row `transforms`. Files with different columns are merged by name.

Rule: With no `transforms` and no dedupe, rows are written directly to CSV (one `export_id` column plus the listed columns). With `transforms`, batches of **entries** (not people) flow through them. Look up what you need by `person_id` against people tables (for example `person_email:transforms:appendEmail`) or by `source_code_id` against `source_code_dictionary`. A `sql.query` binding whose table is `timeline` reads the warehouse table only, not input stores.

`--person_ids` and `--limit` apply. `timeline.entry_types` filters rows. `entry_types` on a universe entry prunes stores by `metadata.json`, so leave it off when the stores' `metadata.json` has no `entry_types` map (for example Luminate message stores); select those stores by `input_type` and the `input` EQL instead.

The export result reports `timeline: { files, inputs, warehouse_records, warehouse_inputs }`: how many idv1 files and inputs were read, and how many rows and inputs came from the `timeline` table. Use it to debug; the output file does not depend on it.

```
e9 exportworker export -a <account_id> \
  --definition_path=engine9-accounts/<org>/<account>/index.plugin.js:exports:emailExhaust \
  --start=-30d
```

A plugin path `export` also runs every named `:exports:` entry that is a query export.

### Options for both

`--person_ids=1,2,3` scopes the query. `--dedupe=false` ignores prior export files. `--add_to_input_store` writes into an input store instead of only `exports/`.

Empty “remote people for plugin X” audiences: [e9-person-remote](../e9-person-remote/SKILL.md) — filter is `person_remote` ⋈ `input.plugin_id`, not `{prefix}person`.

## Tables-only

```
e9 exportworker export -a <account_id> --tables=person,transaction --limit=100
e9 exportworker export -a <account_id> --extra_tables=custom --exclude_tables=setting
```

Without `--tables`, tables-only mode uses the standard warehouse list (`plugin`, `input`, `person*`, `source_code_dictionary`, `global_message`, `transaction`, …).

## Output

Default directory: `{store_path}/{account_id}/exports/{export_id}/{date}/`

Override with `--export_dir`. That value is the only write root: a local path, `temp` (OS temp directory, not the account store), or an `s3://` / `r2://` / `gs://` / `gdrive://` URI. There is no second copy under the account store.

What those files mean for a receiver: [SKILL.md](SKILL.md).

A plugin path `export` and its `inventory.json5` include `export_dir`: the export root path (local or object-store URI). Strip it from any absolute output `filename` to recover the relative path under that root. Copied input file and directory entries still carry `source_directory` for the **input-store** root the file was copied from (not the export root).

Rule: A plugin path export writes table copies, idv1 copies, and query export files first, then `inventory.json5`. Treat that file as the completion catalog: if it is missing, the run did not finish. A single named export (`:exports:<name>`) writes no `inventory.json5`.

| Path | Contents |
|------|----------|
| `tables/{table}.parquet` | Default warehouse table dumps |
| `{input_type}/{input_id}/*.idv1.parquet` | Default copied idv1 files (`message`, `person`, `timeline`, …) |
| `{input_type}/{input_id}/metadata.json` | Input-store descriptor (`input_id`, `input_type`, `entry_types`, listed files) |
| Definition-owned `relative_path` | Custom destination for a copied table or file, below `export_dir` |
| `inventory.json5` | Plugin path run: catalog of what was written (copies, plus `exports[]` for each query export file), written **last**. Monthly statistics: [e9-inventory](../e9-inventory/SKILL.md). |
| `{export_id}.{export_name}.export.csv` + metadata | Default query export output, in `export_dir` |

`directories[].files` is the **file list** (name, filename, records, source_directory), not a count.

## Authoring an export plugin

Put `index.js` at `engine9-accounts/<org>/<account>/export/` (or use a `*.plugin.js`). To copy, export a top-level `universe`: one `{ type: 'table', table }` entry per warehouse table and `{ type: 'inputs', ...selectors }` entries for input-store files. Add query exports under `exports: { name: { definition | timeline, transforms } }`, and file-transform implementations under the plugin's top-level `transforms` map. A plugin may have either or both.

`channel: 'email'` needs `global_message` or `message` with that channel. An `entry_types` selector finds stores whose `metadata.json` contains at least one exact listed type.

Each selected input directory copies `.idv1.parquet` files and `metadata.json`. Raw files in the store are ignored even if `metadata.json` lists them.

## Debug (A–G)

Walk **A → G**. Stop at the first gap. Run [inventory](../e9-inventory/SKILL.md) before `export`.

```
Export pipeline:
- [ ] A: definition_path resolves (plugin path and/or :exports:name)
- [ ] B: `type: 'table'` universe entries inventory the expected tables (or skipped does_not_exist)
- [ ] C: `type: 'inputs'` selectors inventory the expected directories / idv1 files
- [ ] D: file records are useful (not metadata zeros)
- [ ] E: planned `relative_path` and transforms are correct and collision-free
- [ ] F: export wrote every planned copy under `export_dir`, then `inventory.json5`
- [ ] G: query export file has rows (definition, remotes, dates)
```

### A) Path

`inventory` / `export` errors “No exports found” or “requires definition_path” → wrong path, or a definition with neither a copy `universe` nor a query export. See [e9-inventory](../e9-inventory/SKILL.md). Plugin directories do **not** use `:exports:`.

A query export's universe never shows up in `inventory.json5`'s copy lists; it is read, not copied. Each query export file appears under `exports[]`. To check what a query export reads, run that one export (`:exports:<name>`) and look at its log and output rows.

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

Pre-flight plans live in `cache/inventory-plans/`. After a completed copy export, check `{export_dir}/inventory.json5` — it is written last. Inventory format and statistics: [e9-inventory](../e9-inventory/SKILL.md).

### F) Files at the destination

A copy `export` returns `export_dir` (the resolved export root). Every file listed by `inventory.json5` should exist at its `relative_path` under that root. Missing `inventory.json5` means the run did not finish. Missing copies listed in the catalog: listing found no idv1s (C), or a file copy was skipped (logged). A transform failure aborts before `inventory.json5` is written.

### G) Query export empty

Definition too tight (`start`/`end`, plugin filter). Confirm the definition actually references `overrides.start` / `overrides.end`, and that `end` is exclusive. Input-store copies ignore those flags — see [Start and end dates](#start-and-end-dates). “Remotes for plugin X” with no rows → [e9-person-remote](../e9-person-remote/SKILL.md). For `timeline`, read the result's `timeline` counts. Files read but no rows: check `timeline.entry_types` and `start` / `end`. An input you expected read from the `timeline` table is missing from `warehouse_inputs`: its `input` row has store metadata, so its idv1 files were read instead. An input store that was not read: its `input` row lacks store metadata (run `upsertMissingInputMetadata`), or its `min_timeline_ts` / `max_timeline_ts` fall outside the window. With a universe, check that it selects the inputs (a universe `entry_types` selector skips stores whose `metadata.json` lacks `entry_types`). `--sample=true` reads only a few stores, so test a timeline export with a full run scoped by `--start`. `limit` on a plugin path run applies per query export **and** per copied table.

Record the first failing step.
