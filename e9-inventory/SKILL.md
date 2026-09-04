---
name: e9-inventory
description: >-
  Run and interpret engine9 warehouse inventory with the e9 CLI
  (`e9 inventoryworker inventory`, `e9 exportworker inventory`). Covers
  inventory.json5 format (export plan + monthly statistics), InventoryWorker,
  input-store idv1 counts, table/message statistics, plan-only runs, and using
  inventory outside export. Use when working with inventory, inventory.json5,
  warehouse statistics, monthly counts, InventoryWorker, or pre-flight checks
  before export or analytics loads.
---

# engine9 inventory

**Inventory** describes what is in an account warehouse: row counts, input-store idv1 files, planned export paths, and **monthly statistics** (per-month record counts by table, plugin, entry type, message submodule, and similar). It writes a report file; it does not copy parquet or run export.

Run inventory before export, before analytics DB loads, or any time you need a warehouse snapshot. What an export contains: [e9-export](../e9-export/SKILL.md). File production: [e9-export/building.md](../e9-export/building.md).

```
e9 inventoryworker inventory -a <account_id> --definition_path=<bundle>
e9 exportworker inventory -a <account_id> --definition_path=<bundle>
```

`-a` is the account id from `accounts.d`. Options are `--snake_case` flags. Full report path is returned as `options_filename`.

Related: export contents [e9-export](../e9-export/SKILL.md); running an export [e9-export/building.md](../e9-export/building.md); timeline entry types [e9-timeline](../e9-timeline/SKILL.md); input metadata [inputs/timeline](../inputs/timeline/SKILL.md).

## Workers

| Worker | Alias | When to use |
|--------|-------|-------------|
| `@engine9/plugins/e9workers:InventoryWorker` | `inventoryworker` | **Preferred** for inventory-only runs (no export coupling). |
| `@engine9/plugins/e9workers:ExportWorker` | `exportworker` | Same `inventory` method; use when already in an export workflow. |

Both call the same `buildInventoryReport` utility. Bundle **export** still writes `inventory.json5` but passes `statistics: false` so export stays fast (plan only).

## What inventory produces

Two logical parts in one JSON report (`format_version` **2**):

1. **Plan** — what an export *would* write: tables, idv1 files, `relative_path`, transforms, skipped items, totals.
2. **Statistics** — account-wide **monthly** counts for warehouse timeline views and analytics iteration (standalone inventory only by default).

Examples: [examples.md](examples.md).

## CLI

### Full inventory (plan + statistics)

With a bundle definition:

```
e9 inventoryworker inventory -a <account_id> \
  --definition_path=engine9-accounts/<org>/<account>/export
```

Without `definition_path`, inventory uses the **default definition**: person* tables, `segment`, `source_code_dictionary`, `transaction`, `timeline`, and **all input stores** (`input` EQL):

```
e9 inventoryworker inventory -a <account_id>
```

### Plan-only (faster, no monthly statistics)

```
e9 inventoryworker inventory -a <account_id> \
  --definition_path=engine9-accounts/<org>/<account>/export \
  --statistics=false
```

Legacy alias: `--coverage=false` (deprecated).

### Tables-only override (no bundle definition)

```
e9 inventoryworker inventory -a <account_id> \
  --tables=person,transaction \
  --extra_tables=global_message_summary \
  --exclude_tables=setting
```

When `--tables` (or `universe` / `input_directories`) is set without `definition_path`, defaults are not applied — only what you pass is inventoried.

### Override input selectors (with a bundle)

```
e9 inventoryworker inventory -a <account_id> \
  --definition_path=engine9-accounts/<org>/<account>/export \
  --input_directories='[{"entry_types":["EMAIL_OPEN"],"files":"^opens\\.idv1\\.parquet$"}]'
```

## Return value vs full report

The CLI return value is a **summary** (counts, per-table totals, statistics month range). The **full** inventory is at `options_filename` (under `{store_path}/{account_id}/temp/{date}/`).

Bundle export embeds the plan in `{export_dir}/inventory.json5` (statistics omitted).

## Default definition (no `definition_path`)

When `definition_path` is omitted and no explicit bundle options are passed, inventory uses:

**Tables:** `person`, `person_remote`, `person_email`, `person_phone`, `person_address`, `segment`, `source_code_dictionary`, `transaction`, `timeline`

**Inputs:** all rows in `input` via `{ type: 'inputs', eql: { table: 'input', columns: [...] } }` — every input store with listable `.idv1.parquet` files

Override with `--tables`, `--extra_tables`, `--exclude_tables`, `--input_directories`, or `--universe` — any of those replaces the default bundle instead of merging.

Implementation: `server/utilities/defaultInventoryDefinition.js`.

## Inventory file format (`inventory.json5`)

### Top level

| Key | Purpose |
|-----|---------|
| `format_version` | Schema version (`2`). |
| `definition_path`, `plugin_path`, `source_directory` | Run context. |
| `universe` | Resolved bundle universe. |
| `tables[]` | `{ table, relative_path, records }` plus `transforms` only when non-empty. |
| `files[]`, `directories[]` | Planned idv1 copies; `directories[].files` is the file list, not a count. |
| `skipped_tables[]`, `skipped_files[]` | Omitted items (`does_not_exist`, selector mismatch, …). |
| `table_records`, `file_records`, `records` | Plan totals. |
| `statistics` | Monthly warehouse statistics (see below). Omitted when `--statistics=false` or during bundle export. |

Empty `transforms: []` arrays are omitted from serialized output.

### Statistics block

Monthly buckets use `YYYY-MM`. `month_range` spans the earliest and latest month seen across all statistic sources.

| Key | Purpose |
|-----|---------|
| `version` | Statistics schema version (`1`). |
| `generated_at` | ISO timestamp when statistics were collected. |
| `month_range` | `{ min, max }` month keys. |
| `inputs.by_plugin_entry_type_month[]` | Non-unique timeline row counts from input-store `.idv1.parquet`, aggregated by plugin and entry type. Plugin id/name from `metadata.json` and `plugin` table. `{ plugin_id, plugin_name, entry_type, month, records }`. |
| `tables[]` | Per inventoried table: `{ table, date_column, months: [{ month, records }], records }`. Date column uses the export cascade: `frakture_last_modified`, `ts`, `last_modified`, `frakture_date_created`, … then `created_at` / `modified_at`. Optional **`by_plugin_month`**: for `transaction`, via `input_id` → `input.plugin_id`; for `person`, via `person_remote.source_input_id` → `input.plugin_id` with `count(distinct person_id)` (platform people). |
| `messages` | From `global_message` or `message`: `{ plugin_id, plugin_name, submodule, month, records }` bucketed on `publish_date`. |
| `message_summary_by_date` | From `global_message_summary_by_date` on `date`, only rows where **spend > 0** or **impressions > 0**. Same bucket shape as `messages`. |

Skipped statistic sources include `skipped: { reason }` on the section (e.g. `does_not_exist`, `no_date_column`, `query_failed`).

## Record counts (plan)

| Source | Rule |
|--------|------|
| Warehouse tables | `COUNT(*)` |
| Input idv1 files | `metadata.json` when `records > 0`; else parquet row count (may refresh metadata) |
| Input `metadata.json` | Copied with each selected store so the export describes the input |
| Raw files in input store | Never listed — only `.idv1.parquet` and `metadata.json` |

## Using statistics

Typical uses:

- **Warehouse timeline UI** — one row per source; each month tick is populated when `months[].records > 0` or a matching statistics bucket exists.
- **Analytics DB iteration** — walk `statistics.tables` / `inputs.by_plugin_entry_type_month` to decide which months to pull.
- **Gap detection** — compare `statistics.month_range` to expected span; empty `months` on a table that should have data → stats job or load missing.

Email activity (sends, opens, clicks) maps to `statistics.inputs.by_plugin_entry_type_month` (`EMAIL_SEND`, `EMAIL_OPEN`, …). Message sends map to `statistics.messages`. Paid/social impressions map to `statistics.message_summary_by_date`.

## Debug (plan)

Walk in order when export plan looks wrong (also applies when inventory precedes export):

```
- [ ] A: definition_path resolves (bundle plugin or JSON5)
- [ ] B: expected tables present or skipped_tables explains gaps
- [ ] C: input selectors match idv1 stores (channel, entry_types, metadata.json)
- [ ] D: file records > 0 (metadata zeros fall back to parquet count)
- [ ] E: relative_path and transforms valid, no duplicate targets
```

Export-specific steps (F, G): [e9-export debug](../e9-export/building.md#debug-ag).

### Common plan gaps

| Symptom | Likely cause |
|---------|----------------|
| `skipped_tables` `does_not_exist` | Table not deployed (summary tables need stats job) |
| Empty `directories` / `files` | Selector mismatch (`channel=email` needs message inputs; `entry_types` must match `metadata.json` exactly) |
| `records: 0` on idv1 | Empty parquet or stale metadata (inventory should count parquet) |
| Transform error | Transform must declare `scope: 'file'` |

## Implementation notes

- Core logic: `server/utilities/inventoryReport.js`, `inventoryStatistics.js`.
- `InventoryWorker` is registered on `@engine9/plugins/e9workers`.
- Export bundle runs reuse the plan builder; they do not collect statistics by default.

More JSON examples: [examples.md](examples.md).
