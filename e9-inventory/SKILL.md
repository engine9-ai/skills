---
name: e9-inventory
description: >-
  Run and interpret engine9 warehouse inventory with the e9 CLI
  (`e9 inventoryworker inventory`, `e9 inventoryworker buildInventoryReport`).
  Covers the account cache at cache/inventory.json.gz, inventory.json5 export plans,
  InventoryWorker, input-store idv1 counts, table/message statistics (channel,
  sent, impressions/opens, clicks, transaction revenue, timeline people),
  plan-only runs, Home dashboard metrics from statistics, and using inventory
  outside export. Use when working with inventory, inventory.json.gz / inventory.json5,
  warehouse statistics, monthly counts, InventoryWorker, message_activity,
  or pre-flight checks before export or analytics loads.
---

# engine9 inventory

**Inventory** describes what is in an account warehouse: row counts, input-store idv1 files, planned export paths, and **monthly statistics** (per-month counts and engagement by table, plugin, channel, entry type, and similar). It writes a report file; it does not copy parquet or run export.

Inventories take a while, so each account caches the last full report at **`{account root}/cache/inventory.json.gz`**. `inventory` only stats that path (ready / size). `buildInventoryReport` writes it. Standard builds omit input-store `files[]` (`include_files: false`).

Run inventory before export, before analytics DB loads, or any time you need a warehouse snapshot. What an export contains: [e9-export](../e9-export/SKILL.md). File production: [e9-export/building.md](../e9-export/building.md). MCP: [e9-mcp](../e9-mcp/SKILL.md) `inventory` tool.

```
e9 inventoryworker inventory -a <account_id>
e9 inventoryworker buildInventoryReport -a <account_id>
```

`-a` is the account id from `accounts.d`. Options are `--snake_case` flags. When the cache exists, both `options_filename` and `inventory_path` are `{store_path}/{account_id}/cache/inventory.json.gz`.

Related: export contents [e9-export](../e9-export/SKILL.md); running an export [e9-export/building.md](../e9-export/building.md); timeline entry types [e9-timeline](../e9-timeline/SKILL.md); input metadata [inputs/timeline](../inputs/timeline/SKILL.md).

## Workers

| Worker | Alias | When to use |
|--------|-------|-------------|
| `@engine9/plugins/e9workers:InventoryWorker` | `inventoryworker` | **Preferred.** `inventory` stats the cache; `buildInventoryReport` writes it. |
| `@engine9/plugins/e9workers:ExportWorker` | `exportworker` | `inventory` delegates to InventoryWorker (cache lookup only). Bundle **export** still builds a plan-only `inventory.json5` in the export dir (`statistics: false`) and does not overwrite the account cache. |

## What inventory produces

Two logical parts in one JSON report (`format_version` **2**):

1. **Plan** — what an export *would* write: tables, idv1 files, `relative_path`, transforms, skipped items, totals.
2. **Statistics** — account-wide **monthly** counts and engagement (revenue, people, sends / impressions / clicks by channel) for warehouse timelines, Home KPIs, and analytics iteration (standalone `buildInventoryReport` only by default). Grain is `YYYY-MM`, not daily.

Account cache: `{account root}/cache/inventory.json.gz` (gzipped JSON; no `files[]` unless `--include_files=true`). Bundle export plan: `{export_dir}/inventory.json5` (statistics omitted; includes files). Examples: [examples.md](examples.md).

## CLI

### Read the cached inventory

Does **not** generate a report. Completes immediately.

```
e9 inventoryworker inventory -a <account_id>
```

If `{account root}/cache/inventory.json.gz` exists, the return value includes `ready: true`, the path, and `size` / `modified_at` from stat. It does **not** parse the file. If it does not exist, `ready: false` with a message to run `buildInventoryReport`.

`e9 exportworker inventory` is the same cache lookup.

### Build (or refresh) the cache

This is the long-running job. Writes `{account root}/cache/inventory.json.gz` with `summary` baked in. Standard runs pass `include_files: false` (no input-store file listing). Use `--include_files=true` to include `files[]`.

With a bundle definition:

```
e9 inventoryworker buildInventoryReport -a <account_id> \
  --definition_path=engine9-accounts/<org>/<account>/export
```

Without `definition_path`, uses the **default definition**: person* tables, `segment`, `source_code_dictionary`, `transaction`, `timeline`, and **all input stores** (`input` EQL):

```
e9 inventoryworker buildInventoryReport -a <account_id>
```

### Plan-only (faster, no monthly statistics)

```
e9 inventoryworker buildInventoryReport -a <account_id> \
  --definition_path=engine9-accounts/<org>/<account>/export \
  --statistics=false
```

Legacy alias: `--coverage=false` (deprecated).

### Tables-only override (no bundle definition)

```
e9 inventoryworker buildInventoryReport -a <account_id> \
  --tables=person,transaction \
  --extra_tables=global_message_summary \
  --exclude_tables=setting
```

When `--tables` (or `universe` / `input_directories`) is set without `definition_path`, defaults are not applied — only what you pass is inventoried.

### Override input selectors (with a bundle)

```
e9 inventoryworker buildInventoryReport -a <account_id> \
  --definition_path=engine9-accounts/<org>/<account>/export \
  --input_directories='[{"entry_types":["EMAIL_OPEN"],"files":"^opens\\.idv1\\.parquet$"}]'
```

## HTTP (Conductor Home / Inventory)

Browser UIs use the account-scoped data API (Firebase / session auth + `X-ENGINE9-ACCOUNT-ID`), not MCP:

| Method | Path | Role |
|--------|------|------|
| `GET` | `/data/inventory` | Status only (`InventoryWorker.inventory`). Does not parse the gzip file. |
| `GET` | `/data/inventory/report` | Streams `cache/inventory.json.gz` (`Content-Type: application/json`, `Content-Encoding: gzip`). `204` when the cache is missing. |
| `POST` | `/data/inventory/build` | Schedule `InventoryWorker.buildInventoryReport` (does not wait). |

## MCP

Prefer the native **`inventory`** tool over `task` (agents / CLI-style MCP clients):

1. **`command: get`** (default) — same as `GET /data/inventory` / `InventoryWorker.inventory`. Status only: `ready`, path, `size`, `modified_at`. Does not parse the file.
2. If `ready: false` (or the user asked to refresh), **`command: build`** — same as `POST /data/inventory/build`. Schedules `InventoryWorker.buildInventoryReport` (does not wait). Poll with MCP `task` list/output, then `get` again.

Full report: `GET /data/inventory/report` (gzip bytes; `204` if missing), or MCP `file` with `filename: cache/inventory.json.gz` (gunzipped, then byte-capped; use `start`/`end` to sample).

## Return value vs full report

`inventory` and `buildInventoryReport` return **status** only (they do not parse or reshape the cache):

| Field | Meaning |
|-------|---------|
| `ready` | `true` when the cache file exists (or was just written). |
| `inventory_path` | `{store_path}/{account_id}/cache/inventory.json.gz` |
| `options_filename` | Same as `inventory_path` when ready; `null` when not ready. |
| `size` / `modified_at` | From `stat` on get (compressed size). |
| `cached_at` | ISO timestamp on build (also stored inside the gzip file). |

`people`, `transactions`, `table_count`, and `statistics` live in the cache file under `summary` and `statistics`. Full report: `GET /data/inventory/report` (gzip bytes; browser decodes via `Content-Encoding`), MCP `file` with `filename: cache/inventory.json.gz`, or `e9 fileworker json` (gunzips `.gz`).

## Default definition (no `definition_path`)

When `definition_path` is omitted and no explicit bundle options are passed, inventory uses:

**Tables:** `person`, `person_remote`, `person_email`, `person_phone`, `person_address`, `segment`, `source_code_dictionary`, `transaction`, `timeline`

**Inputs:** all rows in `input` via `{ type: 'inputs', eql: { table: 'input', columns: [...] } }` — every input store with listable `.idv1.parquet` files

Override with `--tables`, `--extra_tables`, `--exclude_tables`, `--input_directories`, or `--universe` — any of those replaces the default bundle instead of merging.

Implementation: `server/utilities/defaultInventoryDefinition.js`.

## Inventory file format (`cache/inventory.json.gz` and export `inventory.json5`)

The account cache is gzipped JSON. Bundle export still writes uncompressed JSON5. Same object shape (`format_version` **2**). Standard account cache omits `files[]` / `directories[]`.

### Top level

| Key | Purpose |
|-----|---------|
| `format_version` | Schema version (`2`). |
| `cached_at` | ISO timestamp when the account cache was written (`buildInventoryReport` only). |
| `definition_path`, `plugin_path`, `source_directory` | Run context. |
| `universe` | Resolved bundle universe. |
| `summary` | Baked-in totals: `table_count`, `table_records`, `records`, `people` / `transactions` / `messages` when those tables were inventoried. File counts only when `files` were included. |
| `tables[]` | `{ table, relative_path, records }` plus `transforms` only when non-empty. |
| `files[]`, `directories[]` | Planned idv1 copies (export / `--include_files=true` only). |
| `skipped_tables[]`, `skipped_files[]` | Omitted items (`does_not_exist`, selector mismatch, …). |
| `table_records`, `file_records`, `records` | Plan totals (`file_records` omitted when files are omitted). |
| `statistics` | Monthly warehouse statistics (see below). Omitted when `--statistics=false` or during bundle export. |

Empty `transforms: []` arrays are omitted from serialized output.

### Statistics block

Monthly buckets use `YYYY-MM`. `month_range` spans the earliest and latest month seen across all statistic sources.

| Key | Purpose |
|-----|---------|
| `version` | Statistics schema version (`1`). New fields are additive. |
| `generated_at` | ISO timestamp when statistics were collected. |
| `month_range` | `{ min, max }` month keys. |
| `inputs.by_plugin_entry_type_month[]` | Non-unique timeline row counts from input-store `.idv1.parquet`, aggregated by plugin and entry type. Plugin id from `metadata.json` / `plugin`; display name prefers nickname/label over path. `{ plugin_id, plugin_name, entry_type, channel, month, records }`. **`channel`** is inferred from the entry-type prefix (`EMAIL_OPEN` → `email`, `SMS_SEND` → `sms`, `PHONE_*` → `phone`; otherwise omitted/`null`). |
| `tables[]` | Per inventoried table: `{ table, date_column, months: [{ month, records, … }], records, sql }`. `sql` is the monthly-count query that produced `months`. Date column uses the export cascade: `frakture_last_modified`, `ts`, `last_modified`, `frakture_date_created`, … then `created_at` / `modified_at`. Extra measures on `months[]` when the column exists: **`transaction`** → `revenue` (`sum(amount)`), also rolled up as `tables[].revenue`; **`timeline`** → `people` (`count(distinct person_id)` — people with a timeline entry that month). For **`person`**, also **`created_months`** / **`created_date_column`** when a created-date column exists (`frakture_date_created`, `date_created`, `created_at`, `remote_date_created`) — used by Home’s people-created chart. For `transaction` and `person`, also **`by_plugin_month`** (always present when attempted), **`by_plugin_month_sql`**, and optional **`by_plugin_month_skipped`**: transaction via `input_id` → `input` ⨝ `plugin` (rows also carry `revenue`); person via `person_remote.source_input_id` → `input` ⨝ `plugin` with `count(distinct person_id)` (platform people). |
| `messages` | Prefer **`global_message_summary`** (fallback: `global_message` / `message`): lifetime stats by plugin / submodule / **channel** / month on **`publish_date`**, only rows with **`publish_date > 2000-01-01`** (skips junk/empty dates). Plugin id comes from `bot_id` / `plugin_id`. Display name prefers instance **label** (`bot_nickname` / `bot_label`) over type (`bot_path` / `plugin_path`). Each `by_plugin_submodule_month[]` bucket: `{ plugin_id, plugin_name, submodule, channel, month, records }` plus sums of **`sent`**, **`impressions`** (opens), **`clicks`**, **`spend`**, **`attributed_revenue`**, **`attributed_transactions`** when those columns exist. **`by_channel_month[]`** is the same metrics rolled up without plugin/submodule. Includes `sql`, `date_column`. |
| `message_summary_by_date` | From **`global_message_summary_by_date`** on **`date`**: **active ads** with **`spend > 0`**. `count(distinct message_id)` when available. Grain is plugin / submodule / channel / month. Coverage only — no engagement sums. Includes `sql`, `filter`, `count`, `by_channel_month`. |
| `message_activity` | Same view on **`date`**, **no spend filter** (email and SMS daily stats included). Same grain and metric sums as `messages` (`sent`, `impressions`, `clicks`, …). Prefer this for calendar activity charts. Includes `sql`, `count`, `by_channel_month`. |

Metric columns are omitted from a bucket when the warehouse column is missing. Skipped statistic sources include `skipped: { reason }` on the section (e.g. `does_not_exist`, `no_date_column`, `query_failed`).

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
- **Home dashboard** — monthly KPIs and charts (see below). Compare the latest month with data to the same month a year earlier for `% vs`. Charts are the last 12 months.
- **Analytics DB iteration** — walk `statistics.tables` / `inputs.by_plugin_entry_type_month` to decide which months to pull.
- **Gap detection** — compare `statistics.month_range` to expected span; empty `months` on a table that should have data → stats job or load missing.

### Home dashboard

Grain is monthly (`YYYY-MM`). Plot `month` as the date axis (`YYYY-MM-01`). Filter `by_channel_month` on `channel` (e.g. `email`, `sms`) for per-channel cards/charts. Impressions on message views are platform opens.

| Home surface | Statistics source |
|--------------|-------------------|
| Total revenue | `tables[]` where `table === 'transaction'` → `months[].revenue` (`sum(amount)`). Previous period = same month last year. |
| Donations (count) | same `months[].records` |
| Average gift | `revenue / records` for the latest month vs the same month last year. |
| Active people | `tables[]` where `table === 'timeline'` → `months[].people` (`count(distinct person_id)`). Fallback: `person` table `records` (warehouse people, not activity). |
| People created chart | Prefer person `created_months` (`frakture_date_created` / `date_created` / `created_at`). Fallback: person `months[]` only when `date_column` is a created-date column. Last 12 months. |
| Emails sent (and other channel send KPIs) | Prefer **`message_activity.by_channel_month`** → `sent` for that channel. Fallback: `messages.by_channel_month`, then `inputs.by_plugin_entry_type_month` where `entry_type` is `EMAIL_SEND` / `SMS_SEND` / … (`channel` is already on those rows). |
| Revenue and donations chart | transaction `months[]` for the last 12 months: `revenue` (bars) + `records` (donations line) |
| Channel activity chart (sends, opens, clicks) | Prefer **`message_activity.by_channel_month`**: `sent`, `impressions` (opens), `clicks`. Fallback: `messages.by_channel_month`, then inputs (`EMAIL_SEND` / `EMAIL_OPEN` / `EMAIL_CLICK`, `SMS_*`). Last 12 months. |

Paid/social **coverage** (which months had spend) still maps to `message_summary_by_date` (`spend > 0`). Do not use that block for email opens/clicks — it excludes rows without spend.

Message views: [e9-global-message](../e9-global-message/SKILL.md). Say **transaction**, not donation, in warehouse terms; Home copy may still say “donations” for `transaction` row counts. Prefer `attributed_*` on message buckets for last-click fundraising; Home **Total revenue** uses transaction `amount`.

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
- `InventoryWorker.inventory` stats `{account root}/cache/inventory.json.gz` (no parse); `InventoryWorker.buildInventoryReport` writes it (`include_files: false` by default).
- `ExportWorker.inventory` delegates to InventoryWorker. Bundle export calls the plan builder with `statistics: false` and writes `{export_dir}/inventory.json5` without touching the account cache.
- MCP `inventory` `get` / `build` wraps those two methods.

More JSON examples: [examples.md](examples.md).
