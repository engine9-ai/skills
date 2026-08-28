# Timeline loading (developer)

User-facing model, entry types, querying, and missing-entry debug: [SKILL.md](SKILL.md). Plugin file shapes and `@engine9/input-tools` helpers: [inputs/timeline](../inputs/timeline/SKILL.md). Timeline rows are **entries**, never events.

Use this file when writing or modifying **load jobs**, or when an engineering investigation of `InputWorker` is requested. Do not use this file to diagnose a missing Timeline tab entry from the product; that is not a load-job change.

engine9 server workers treat timeline data in two stages:

- **Raw input → Timeline ID files** (with `person_id` and `id`) — `InputWorker.id` and `PersonWorker.loadPeople`.
- **Timeline ID files → database tables** — `InputWorker.loadTimeline`, `loadTimelineTables`, `loadTimelineFile`, and `loadTimelineDetails`, often with `SQLiteWorker` / `ClickHouseWorker` / `DuckDBWorker`.

## Timeline ID files

A **Timeline ID file** is typically parquet whose name ends with `.idv1.parquet`. Rows already have:

- **`id`**: UUID (`getTimelineEntryUUID`)
- **`ts`**: entry timestamp
- **`person_id`**: internal numeric person id
- **`entry_type_id`**: integer from `TIMELINE_ENTRY_TYPES`
- **Optional**: `source_code_id`, `email_domain`, plus plugin-specific detail columns

This matches `timeline`: `id` (text PK), `ts`, `entry_type_id`, `person_id`, optional `source_code_id` / `email_domain`. Once a file is in this shape it loads without further identity resolution. How `loadPeople` chooses `person_id` is [e9-person-id](../e9-person-id/SKILL.md).

### `InputWorker.id`

Takes a raw input file (CSV, JSONL, parquet, …) and produces `.idv1.parquet`:

1. **Output file** — `.idv1.parquet` beside the input or in a target directory. Skips if present and `overwriteIdFile` is false.
2. **`inputId` / `pluginId`** — UUID for this stream; plugin from the `input` table or directory.
3. **Analyze** — `FileWorker.analyze`; schema includes `id`, `ts`, `entry_type_id`, `person_id`, `source_code_id`, derived `email_domain` when `email` exists, plus extra input fields (excluding metadata like `remote_entry_uuid`, `input_id`, `entry_type`).
4. **`PersonWorker.loadPeople`** per batch — creates/updates `person` rows and stamps `person_id`.
5. **`appendTimelineId`** — `appendEntryTypeId` from `entry_type` or a default; then for rows without `id`, uses `remote_entry_id` / `remote_transaction_id` or `(ts, person_id, entry_type_id, source_code_id)` plus `pluginId` via `getTimelineEntryUUID`.
6. **Write parquet** — temp file, then rename; S3 inputs may upload `<original>.idv1.parquet`.

### `idFiles` and `idAllFiles`

- **`idFiles`**: one or many files; resolves `inputId` / `pluginId`; calls `id`; upserts `input` (`min_timeline_ts` / `max_timeline_ts`).
- **`idAllFiles`**: `idFiles` across accounts (serialized to avoid DB deadlocks). When `loadTimeline` is true, runs `loadTimelineTables` on the ID files.

## Loading ID files into tables

### `loadTimelineFile`

Lowest-level loader. Expects `id`, `ts`, `person_id`, `entry_type_id`, `source_code_id`. Validates UUID `id` and required fields. Upserts into `timeline`, setting `input_id` from `inputId` when provided.

### `loadTimeline`

One file (`filename`) or a directory of `.idv1.parquet` / legacy `.timeline.parquet`. Engine: `clickhouse` → `ClickHouseWorker`, `sqlite` → `SQLiteWorker`, else `InputWorker`. Optional `.processed` postfix. Returns `sources`, `inputTable` (`input_<uuid>`), and `sqliteFile` when using SQLite.

### `loadTimelineTables`

Timeline **and** detail: `fileArray` with `idFilename`, or `idFilename` / `filenames`, or `inputId` / `directory` via `getIdFilenames`. For each file: `loadTimeline`, then `loadTimelineDetails` when `loadTimelineDetail` and `timelineDetailTable` are set. Then `ensureTimelineSummary` — a view joining `timeline`, `input`, `plugin`, `source_code_dictionary`, and the detail table (`<detail>_summary`).

### `loadTimelineDetails`

Upserts extra columns into the plugin detail table (UUID `id` PK). Native parquet/csv path for DuckDB/ClickHouse; stream upsert otherwise. Core timeline columns are not duplicated on the detail table (`ts`, `person_id`, `entry_type_id`, `input_id`, `source_code_id`, …).

## Timeline Raw vs ID (producers)

Email/SMS/form workers often start from **Raw**-like vendor activity: `ts`, `entry_type` (`EMAIL_OPEN`, …), `email`, maybe no `person_id`. They map with `getEntryTypeId` / `getTimelineEntryUUID`, write Raw (needs person resolution) or ID (if `person_id` is already known), then hand off to `loadTimeline` / `loadTimelineTables`. Once loaded, each row is a timeline **entry**, never an event.

- **Prefer ID files** when you already have `person_id`, `input_id`, and `entry_type_id`.
- **Use Raw** when you only have an email or external person id and will resolve people later.

## Which method

| Method | Use |
|--------|-----|
| `InputWorker.id` | One raw file → `.idv1.parquet` (resolves `person_id` and `id`) |
| `idFiles` / `idAllFiles` | Bulk `id` across files / accounts |
| `loadTimelineFile` | One ID file → `timeline` only |
| `loadTimeline` | One or many ID files → engine table + stats |
| `loadTimelineTables` | `timeline` + detail table + summary view |

Classic flow: **`id` → `loadTimelineTables`**. Keep plugin output close to the input-tools timeline schema so ID conversion is mechanical.
