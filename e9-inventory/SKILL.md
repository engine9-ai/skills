---
name: e9-inventory
description: "Run and interpret engine9 warehouse inventory, account cache reports, export plans, monthly table and message statistics, input-store idv1 counts, Home dashboard metrics, and pre-flight checks with InventoryWorker or the e9 CLI. Use when inspecting inventory.json.gz or inventory.json5, comparing aggregate Messages with per-person Timeline entries, refreshing inventory, or validating data coverage before export or analytics work; analyze the report without automatically querying the warehouse."
---

# engine9 inventory

**Inventory** describes what is in an account warehouse: row counts, input-store idv1 files, planned export paths, and monthly statistics by table, plugin, channel, entry type, and month. It writes a report file; it does not copy Parquet or run an export. Use it for account-wide coverage analysis, cached dashboard metrics, custom export planning, and pre-flight validation.

## Quick reference

| Task | Command or artifact |
|------|---------------------|
| Check cache status | `e9 inventoryworker inventory -a <account_id>` |
| Refresh account inventory | `e9 inventoryworker buildInventorySummaryFile -a <account_id>` |
| Read the full account report | `cache/inventory.json.gz` |
| Inspect a custom plan | `cache/inventory-plans/<slug>.json.gz` |
| Inspect a bundle export plan | `{export_dir}/inventory.json5` |

## Rules

### Analyze from the report only

When the user asks to **check / analyze inventory** (what plugins have, email timeline entry coverage, monthly volume, gaps vs expected sources), **answer from the inventory cache/report alone**.

1. MCP `inventory` `get` (or CLI `inventory`) for cache status.
2. If `ready: true`, read `cache/inventory.json.gz` with MCP `file`, HTTP `/data/inventory/report`, or `e9 fileworker json`.
3. Interpret `summary`, `tables[]`, and `statistics`.

Warehouse SQL is a **follow-up** — only when the user explicitly asks to query the DB, or after you have reported inventory results and they request deeper verification.

Rule: Do not automatically run MCP `sql`, `eql`, `analyze`, or another live warehouse query to confirm inventory findings. Query the warehouse only when the user explicitly requests it.

### Distinguish message and storage layers

Most accounts are audited on **aggregate Messages** (`statistics.messages` from `global_message_summary`: sent, opens, clicks, and spend). **Timeline → Messages** is optional per-person entries (`EMAIL_SEND`, `EMAIL_OPEN`, and similar) from input stores. Audit aggregate data first and more often.

Rule: A large Messages series with a missing or smaller Timeline → Messages row is normal; do not classify missing per-person entries as a failed message load.

Large timeline idv1 files often live only in input stores because loading every entry into `timeline` is expensive and may not be intended. Prefer `statistics.inputs.by_plugin_entry_type_month` for extracted per-person entries and `statistics.tables[]` for rows loaded into warehouse tables.

Rule: Input stores are not warehouse tables. A count mismatch between input statistics and warehouse table statistics is not, by itself, a load failure.

Inventories take a while, so each account caches the last **account-wide** report at **`{account root}/cache/inventory.json.gz`**. `inventory` only stats that path (ready / size). `buildInventorySummaryFile` **without** an export definition writes it. Standard builds omit input-store `files[]` (`include_files: false`).

**Custom / export-scoped builds never use that path.** When `definition_path`, `universe`, `tables`, or other bundle overrides are set, the plan is written to **`cache/inventory-plans/<slug>.json.gz`** (or `{export_dir}/inventory.json.gz` when `export_dir` is set). Bundle **export** still writes `{export_dir}/inventory.json5` and does not touch the account cache.

## Concepts

### Workers

| Worker | Alias | When to use |
|--------|-------|-------------|
| `@engine9/plugins/e9workers:InventoryWorker` | `inventoryworker` | **Preferred.** `inventory` stats the **account** cache; `buildInventorySummaryFile` writes that cache only for default (non-export) builds. With `definition_path` / custom universe → `cache/inventory-plans/`. |
| `@engine9/plugins/e9workers:ExportWorker` | `exportworker` | `inventory` delegates to InventoryWorker (account cache lookup only). Bundle **export** writes `{export_dir}/inventory.json5` last (`statistics: false`) and does not overwrite the account cache. |

### What inventory produces

Two logical parts in one JSON report (`format_version` **2**):

1. **Plan** — what an export *would* write: tables, idv1 files, `relative_path`, transforms, skipped items, totals. `InventoryWorker.buildInventorySummaryFile` writes this to `cache/inventory-plans/` (or `{export_dir}/inventory.json.gz`). Bundle **export** writes the same shape as `{export_dir}/inventory.json5` **after** artifacts, as the completion catalog.
2. **Statistics** — account-wide **monthly** counts and engagement (revenue, people, sends / impressions / clicks by channel) for warehouse timelines, Home KPIs, and analytics iteration (standalone `buildInventorySummaryFile` only by default). Grain is `YYYY-MM`, not daily. Message stats are **aggregate** (`statistics.messages` from `global_message_summary` — Inventory **Messages**). Per-person send/open/click rows are **Timeline → Messages** (`statistics.inputs`, `subcategory: messages`).

Account cache: `{account root}/cache/inventory.json.gz` (gzipped JSON; no `files[]` unless `--include_files=true`). Custom/export plans: `{account root}/cache/inventory-plans/<slug>.json.gz`. Bundle export plan: `{export_dir}/inventory.json5` (statistics omitted; includes files). Examples: [examples.md](examples.md).

## Workflow

### CLI

### Read the cached inventory

Does **not** generate a report. Completes immediately.

```
e9 inventoryworker inventory -a <account_id>
```

If `{account root}/cache/inventory.json.gz` exists, the return value includes `ready: true`, the path, and `size` / `modified_at` from stat. It does **not** parse the file. If it does not exist, `ready: false` with a message to run `buildInventorySummaryFile`.

`e9 exportworker inventory` is the same cache lookup.

### Build (or refresh) the account cache

This is the long-running job. With **no** export definition, writes `{account root}/cache/inventory.json.gz` with `summary` baked in (`account_inventory: true`). Standard runs pass `include_files: false`. Use `--include_files=true` to include `files[]`.

```
e9 inventoryworker buildInventorySummaryFile -a <account_id>
```

### Build an export / custom plan (does not clobber the account cache)

With a bundle definition (or `universe` / `tables` / `input_directories`), writes `cache/inventory-plans/<slug>.json.gz` and returns `account_inventory: false`:

```
e9 inventoryworker buildInventorySummaryFile -a <account_id> \
  --definition_path=engine9-accounts/<org>/<account>/export \
  --statistics=false --include_files=true
```

Without `definition_path`, uses the **default definition**: person* tables, `segment`, `source_code_dictionary`, `transaction`, `timeline`, and **all input stores** (`input` EQL) — that path **is** the account cache:

```
e9 inventoryworker buildInventorySummaryFile -a <account_id>
```

### Plan-only (faster, no monthly statistics)

```
e9 inventoryworker buildInventorySummaryFile -a <account_id> \
  --definition_path=engine9-accounts/<org>/<account>/export \
  --statistics=false
```

Legacy alias: `--coverage=false` (deprecated).

### Sample plan (QA, not the account cache)

`--sample=true` builds a fast QA plan: one `.idv1.parquet` per `(input_type, filename)`, no monthly statistics, never `cache/inventory.json.gz`. Use `--sample=true` or `--sample=false` (not a bare `--sample`).

```
e9 inventoryworker buildInventorySummaryFile -a <account_id> \
  --definition_path=engine9-accounts/<org>/<account>/export \
  --sample=true --include_files=true
```

Writes `cache/inventory-plans/<slug>_sample.json.gz` with `sample: true`, `sample_table_limit` (100 unless `--limit=N`), and `sample_file_limit` (10). Bundle **export** with `--sample=true` materializes that plan under `{export_id}_sample/` (see [export building](../e9-export/building.md)).

### Tables-only override (no bundle definition)

```
e9 inventoryworker buildInventorySummaryFile -a <account_id> \
  --tables=person,transaction \
  --extra_tables=global_message_summary \
  --exclude_tables=setting
```

When `--tables` (or `universe` / `input_directories`) is set without `definition_path`, defaults are not applied — only what you pass is inventoried.

### Override input selectors (with a bundle)

```
e9 inventoryworker buildInventorySummaryFile -a <account_id> \
  --definition_path=engine9-accounts/<org>/<account>/export \
  --input_directories='[{"entry_types":["EMAIL_OPEN"],"files":"^opens\\.idv1\\.parquet$"}]'
```

### HTTP (Conductor Home / Inventory)

Browser UIs use the account-scoped data API (Firebase / session auth + `X-ENGINE9-ACCOUNT-ID`), not MCP:

| Method | Path | Role |
|--------|------|------|
| `GET` | `/data/inventory` | Status only (`InventoryWorker.inventory`). Does not parse the gzip file. |
| `GET` | `/data/inventory/report` | Streams `cache/inventory.json.gz` (`Content-Type: application/json`, `Content-Encoding: gzip`). `204` when the cache is missing. |
| `POST` | `/data/inventory/build` | Schedule `InventoryWorker.buildInventorySummaryFile` (does not wait). |

### MCP

Prefer the native **`inventory`** tool over `task` (agents / CLI-style MCP clients):

1. **`command: get`** (default) — same as `GET /data/inventory` / `InventoryWorker.inventory`. Status only: `ready`, path, `size`, `modified_at`. Does not parse the file.
2. If `ready: false` (or the user asked to refresh), **`command: build`** — same as `POST /data/inventory/build`. Schedules `InventoryWorker.buildInventorySummaryFile` (does not wait). Poll with MCP `task` list/output, then `get` again.

Full report: `GET /data/inventory/report` (gzip bytes; `204` if missing), or MCP `file` with `filename: cache/inventory.json.gz` (gunzipped, then byte-capped; use `start`/`end` to sample).

For “what’s in inventory?” questions, stop after parsing that report. Do not chain into `sql` / `eql` unless the user asks.

## File format

### Return value vs full report

`inventory` and `buildInventorySummaryFile` return **status** only (they do not parse or reshape the cache):

| Field | Meaning |
|-------|---------|
| `ready` | `true` when the cache file exists (or was just written). |
| `inventory_path` | `{store_path}/{account_id}/cache/inventory.json.gz` |
| `options_filename` | Same as `inventory_path` when ready; `null` when not ready. |
| `size` / `modified_at` | From `stat` on get (compressed size). |
| `cached_at` | ISO timestamp on build (also stored inside the gzip file). |

`people`, `transactions`, `table_count`, and `statistics` live in the cache file under `summary` and `statistics`. Full report: `GET /data/inventory/report` (gzip bytes; browser decodes via `Content-Encoding`), MCP `file` with `filename: cache/inventory.json.gz`, or `e9 fileworker json` (gunzips `.gz`).

### Default definition (no `definition_path`)

When `definition_path` is omitted and no explicit bundle options are passed, inventory uses:

**Tables:** `person`, `person_remote`, `person_email`, `person_phone`, `person_address`, `segment`, `source_code_dictionary`, `transaction`, `global_message_summary`, `global_message_summary_by_date`, `timeline`

Aggregate message views come before `timeline` so the usual message audit is in the plan even when per-person entries are not loaded.

**Inputs:** all rows in `input` via `{ type: 'inputs', eql: { table: 'input', columns: [...] } }` — every input store with listable `.idv1.parquet` files

Override with `--tables`, `--extra_tables`, `--exclude_tables`, `--input_directories`, or `--universe` — any of those replaces the default bundle instead of merging.

Implementation: `server/utilities/defaultInventoryDefinition.js`.

### Inventory report (`cache/inventory.json.gz` and export `inventory.json5`)

The account cache is gzipped JSON. Bundle export still writes uncompressed JSON5. Same object shape (`format_version` **2**). Standard account cache omits `files[]` / `directories[]`.

### Top level

| Key | Purpose |
|-----|---------|
| `format_version` | Schema version (`2`). |
| `cached_at` | ISO timestamp when the account cache was written (`buildInventorySummaryFile` only). |
| `definition_path`, `plugin_path`, `source_directory` | Run context. |
| `universe` | Resolved bundle universe. |
| `summary` | Baked-in totals: `table_count`, `table_records`, `records`, `people` / `transactions` / `messages` when those tables were inventoried. File counts only when `files` were included. |
| `tables[]` | `{ table, relative_path, records }` plus `transforms` only when non-empty. |
| `files[]`, `directories[]` | Planned idv1 copies (export / `--include_files=true` only). |
| `skipped_tables[]`, `skipped_files[]` | Omitted items (`does_not_exist`, selector mismatch, …). |
| `table_records`, `file_records`, `records` | Plan totals (`file_records` omitted when files are omitted). |
| `statistics` | Monthly warehouse statistics (see below). Omitted when `--statistics=false`, `--sample=true`, or during bundle export. |
| `sample` | `true` on QA sample plans/exports. |
| `sample_table_limit` / `sample_file_limit` | Row caps used for the sample (tables/person-search vs idv1 copies). |

Empty `transforms: []` arrays are omitted from serialized output.

### Statistics block

Monthly buckets use `YYYY-MM`. `month_range` spans the earliest and latest month seen across all statistic sources.

| Key | Purpose |
|-----|---------|
| `version` | Statistics schema version (`1`). New fields are additive. |
| `generated_at` | ISO timestamp when statistics were collected. |
| `month_range` | `{ min, max }` month keys. |
| `inputs` | Per-person timeline entries (`category: timeline`). `by_plugin_entry_type_month[]` rows also have **`subcategory`** (`messages` for EMAIL_*/SMS_*/PHONE_*/MESSAGE_*, else transactions / signups / forms / segments / other). Plugin id from `metadata.json` / `plugin`; display name prefers nickname/label over path. `{ plugin_id, plugin_name, entry_type, channel, month, records, category, subcategory }`. **`channel`** is inferred from the entry-type prefix (`EMAIL_OPEN` → `email`). These counts describe **files on disk**, not warehouse `timeline`, and are **not** aggregate message stats. Inventory UI: **Timeline → Messages** (per-person). |
| `tables[]` | Per inventoried table: `{ table, date_column, months: [{ month, records, … }], records, sql }`. `sql` is the monthly-count query that produced `months`. Date column uses the export cascade: `frakture_last_modified`, `ts`, `last_modified`, `frakture_date_created`, … then `created_at` / `modified_at`. Extra measures on `months[]` when the column exists: **`transaction`** → `revenue` (`sum(amount)`), also rolled up as `tables[].revenue`; **`timeline`** → `people` (`count(distinct person_id)` — people with a timeline entry that month). For **`person`**, also **`created_months`** / **`created_date_column`** when a created-date column exists (`frakture_date_created`, `date_created`, `created_at`, `remote_date_created`) — used by Home’s people-created chart. For `transaction` and `person`, also **`by_plugin_month`** (always present when attempted), **`by_plugin_month_sql`**, and optional **`by_plugin_month_skipped`**: transaction via `input_id` → `input` ⨝ `plugin` (rows also carry `revenue`); person via `person_remote.source_input_id` → `input` ⨝ `plugin` with `count(distinct person_id)` (platform people). |
| `messages` | **Aggregate** (`kind: aggregate`) — the usual message audit. Prefer **`global_message_summary`** (fallback: `global_message` / `message`): lifetime stats by plugin / submodule / **channel** / month on **`publish_date`**, only rows with **`publish_date > 2000-01-01`** (skips junk/empty dates). Plugin id comes from `bot_id` / `plugin_id`. Display name prefers instance **label** (`bot_nickname` / `bot_label`) over type (`bot_path` / `plugin_path`). Each `by_plugin_submodule_month[]` bucket: `{ plugin_id, plugin_name, submodule, channel, month, records }` plus sums of **`sent`**, **`impressions`** (opens), **`clicks`**, **`spend`**, **`attributed_revenue`**, **`attributed_transactions`** when those columns exist. **`by_channel_month[]`** is the same metrics rolled up without plugin/submodule. Includes `sql`, `date_column`. Inventory UI: top-level **Messages**. |
| `message_summary_by_date` | From **`global_message_summary_by_date`** on **`date`**: **active ads** with **`spend > 0`**. `count(distinct message_id)` when available. Grain is plugin / submodule / channel / month. Coverage only — no engagement sums. Includes `sql`, `filter`, `count`, `by_channel_month`. |
| `message_activity` | Same view on **`date`**, **no spend filter** (email and SMS daily stats included). Same grain and metric sums as `messages` (`sent`, `impressions`, `clicks`, …). Prefer this for calendar activity charts. Includes `sql`, `count`, `by_channel_month`. |

Metric columns are omitted from a bucket when the warehouse column is missing. Skipped statistic sources include `skipped: { reason }` on the section (e.g. `does_not_exist`, `no_date_column`, `query_failed`).

### Record counts

| Source | Rule |
|--------|------|
| Warehouse tables | `COUNT(*)` |
| Input idv1 files | `metadata.json` when `records > 0`; else parquet row count (may refresh metadata) |
| Input `metadata.json` | Copied with each selected store so the export describes the input |
| Raw files in input store | Never listed — only `.idv1.parquet` and `metadata.json` |

## Examples

### Using statistics

Stay on these fields when answering inventory questions. Do not open a live DB session unless the user asks (see [Analyze from the report only](#analyze-from-the-report-only-no-auto-sql)).

Typical uses:

- **Aggregate message coverage (usual audit)** — `statistics.messages` / `message_activity` (`global_message_summary`). Inventory UI: **Messages**.
- **Per-person message entries (optional)** — `inputs.by_plugin_entry_type_month` where `subcategory === 'messages'` (what was extracted into idv1; may never be loaded into warehouse `timeline`). Inventory UI: **Timeline → Messages**.
- **Loaded warehouse volumes** — `tables[]` monthly buckets (what is already in SQL tables such as `timeline` / `transaction`).
- **Warehouse timeline UI** — one row per source; each month tick is populated when `months[].records > 0` or a matching statistics bucket exists.
- **Home dashboard** — monthly KPIs and charts (see below). People and channel StatCards compare the latest month with data to the same month a year earlier for `% vs`, and name both months. Revenue / donations / average gift show the latest month only (a partial month is not comparable to last year). Revenue and people charts are the last 12 months. Email / SMS charts span first to last month with activity (max 36 months).
- **Analytics DB iteration** — walk `statistics.tables` / `messages` first; use `inputs.by_plugin_entry_type_month` only when you need per-person months.
- **Gap detection** — compare `statistics.month_range` to expected span; empty `months` on a table that should have data → stats job or load missing. Empty or smaller warehouse `timeline` vs large email input buckets is often intentional (space), not a gap to “fix” with SQL. Empty Timeline → Messages with a populated Messages series is also normal.

### Home dashboard

Grain is monthly (`YYYY-MM`). Plot `month` as the date axis (`YYYY-MM-01`). Filter `by_channel_month` on `channel` (e.g. `email`, `sms`) for per-channel cards/charts. Impressions on message views are platform opens.

| Home surface | Statistics source |
|--------------|-------------------|
| Total revenue | `tables[]` where `table === 'transaction'` → `months[].revenue` (`sum(amount)`). Latest month only — no year-over-year delta. |
| Donations (count) | same `months[].records` (latest month only). |
| Average gift | `revenue / records` for the latest month (no year-over-year delta). |
| Active people | `tables[]` where `table === 'timeline'` → `months[].people` (`count(distinct person_id)`). Fallback: `person` table `records` (warehouse people, not activity). |
| People created chart | Prefer person `created_months` (`frakture_date_created` / `date_created` / `created_at`). Fallback: person `months[]` only when `date_column` is a created-date column. Last 12 months. |
| Emails sent (and other channel send KPIs) | Prefer **`message_activity.by_channel_month`** → `sent` for that channel when that series has 2+ months and its last **finished** month with sends/opens/clicks reaches as far as `messages`. Fallback: `messages.by_channel_month` (or roll up `messages.by_plugin_submodule_month`) — including when activity is one month, ends earlier, or later activity months are record-only (`sent` / opens / clicks are 0). Do **not** use timeline `inputs` / `EMAIL_SEND` for Home email stats. Score the last finished calendar month with sends; the StatCard caption is that month and the delta names the same month a year earlier. |
| Revenue and donations chart | transaction `months[]` for the last 12 months: `revenue` (bars) + `records` (donations line) |
| Channel activity chart (sends, opens, clicks) | Same source as Emails sent. Span first month with sends/opens/clicks through the last finished month, capped at 36 months. Per-person timeline entries stay on Inventory **Timeline → Messages**. |

Paid/social **coverage** (which months had spend) still maps to `message_summary_by_date` (`spend > 0`). Do not use that block for email opens/clicks — it excludes rows without spend.

Message views: [e9-global-message](../e9-global-message/SKILL.md). Say **transaction**, not donation, in warehouse terms; Home copy may still say “donations” for `transaction` row counts. Prefer `attributed_*` on message buckets for last-click fundraising; Home **Total revenue** uses transaction `amount`.

## Troubleshooting

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

### Implementation notes

- Core logic: `server/utilities/inventoryReport.js`, `inventoryStatistics.js`, `inventorySample.js`.
- `InventoryWorker.inventory` stats `{account root}/cache/inventory.json.gz` (no parse); `InventoryWorker.buildInventorySummaryFile` writes that path **only** for account (default) builds. Export / custom / `--sample=true` builds write `cache/inventory-plans/` and never overwrite the account cache.
- `ExportWorker.inventory` delegates to InventoryWorker. Bundle export calls the plan builder with `statistics: false` and writes `{export_dir}/inventory.json5` **after** artifacts, without touching the account cache. `export_dir` may be a local path or an object-store URI. `--sample=true` writes under `{export_id}_sample/` (or `{export_dir}_sample`).
- MCP `inventory` `get` / `build` wraps those two methods (`sample` is a build-only boolean).

## Related documentation

- [Inventory JSON examples](examples.md)
- [Export package contents](../e9-export/SKILL.md)
- [Build and run exports](../e9-export/building.md)
- [Timeline entries and entry types](../e9-timeline/SKILL.md)
- [Timeline input metadata and file shapes](../inputs/timeline/SKILL.md)
- [Global message views](../e9-global-message/SKILL.md)
- [engine9 MCP](../e9-mcp/SKILL.md)
