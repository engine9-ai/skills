---
name: e9-timeline
description: Describes how engine9 server workers, especially InputWorker, read and write timeline files (Timeline ID vs Timeline Raw), and how those files move into the timeline and timeline detail tables.
---

# Timeline files in the server (InputWorker and friends)

Use this skill whenever you are:

- **Writing or modifying jobs** that call `InputWorker.id`, `InputWorker.idFiles`, `InputWorker.idAllFiles`, `InputWorker.loadTimeline`, `InputWorker.loadTimelineTables`, `InputWorker.loadTimelineFile`, or `InputWorker.loadTimelineDetails`.
- **Debugging how timeline data flows** from raw input files into the `timeline` and plugin-specific detail tables.
- **Designing new plugins or pipelines** that should produce Engine9-compatible timeline files.

Engine9 server workers treat timeline data in two main stages:

- **Raw input → Timeline ID files** (with `person_id` and `id`) – handled primarily by `InputWorker.id` and `PersonWorker.loadPeople`.
- **Timeline ID files → database tables** – handled by `InputWorker.loadTimeline`, `loadTimelineTables`, `loadTimelineFile`, and `loadTimelineDetails`, often with help from `SQLiteWorker` and `ClickHouseWorker`.

For library-level timeline file formats and `@engine9/input-tools` helpers, see [inputs/timeline/SKILL.md](../inputs/timeline/SKILL.md).

## Timeline ID files in InputWorker

### What a Timeline ID file looks like

A **Timeline ID file** is typically a parquet file whose name ends with `.idv1.parquet`. It contains rows that already have:

- **`id`**: UUID for the timeline entry (generated via `getTimelineEntryUUID`).
- **`ts`**: event timestamp (string/number convertible to `Date`).
- **`person_id`**: internal numeric person id.
- **`entry_type_id`**: integer from `TIMELINE_ENTRY_TYPES`.
- **Optional**: `source_code_id`, `email_domain`, plus arbitrary additional columns for plugin-specific detail.

This matches what `SQLiteWorker.ensureTimelineSchema` enforces for the `timeline` table:

- `id TEXT PRIMARY KEY`
- `ts INTEGER not null`
- `entry_type_id INTEGER not null`
- `person_id INTEGER not null`
- `source_code_id INTEGER null`
- `email_domain TEXT` (optional)

The **key idea**: once a file is in this shape, it can be loaded into the `timeline` table without further identity resolution.

### How `InputWorker.id` produces Timeline ID files

`InputWorker.id` takes a raw input file (CSV, JSONL, parquet, etc.) and produces a `.idv1.parquet` file with timeline-ready rows:

1. **Determine output file**:
   - Constructs an `.idv1.parquet` filename, either beside the input or in a target directory.
   - Skips work if an existing ID file is present and `overwriteIdFile` is false.
2. **Resolve `inputId` and `pluginId`**:
   - `inputId` is typically a UUID specific to this input stream (via `getInputId` / `getInputUUID`).
   - `pluginId` is resolved from the `input` table or the directory structure.
3. **Analyze the source file**:
   - Uses `FileWorker.analyze` to inspect fields and infer types.
   - Builds an initial parquet schema including:
     - `id`, `ts`, `entry_type_id`, `person_id`, `source_code_id`.
     - Derived `email_domain` when an `email` column exists.
     - Additional fields from the input (excluding known metadata like `remote_entry_uuid`, `input_id`, `entry_type`, etc.).
4. **Load people and attach `person_id`**:
   - For each batch, calls `PersonWorker.loadPeople` with:
     - The batch stream.
     - `pluginId`, `inputId`, `defaultSourceCode`, `defaultEntryType`, and any `extraTransforms`.
   - This ensures that:

     - `person` records exist or are updated.
     - Each row in the batch now has a valid `person_id`.

5. **Append timeline IDs**:
   - Calls `appendTimelineId` (from `ServerBaseWorker`) with:
     - `inputId` for the batch.
     - The batch array.
     - Options like `defaultTimestamp` and `timelineIdField: 'id'`.
   - `appendTimelineId`:
     - First calls `appendEntryTypeId` to ensure `entry_type_id` is set from `entry_type` or a default.
     - For each row without an `id`:
       - Ensures a `ts` exists (or uses `defaultTimestamp`).
       - Uses `remote_entry_id` / `remote_transaction_id` or the composite (`ts`, `person_id`, `entry_type_id`, `source_code_id`) plus `pluginId` to generate `id` via `getTimelineEntryUUID`.
6. **Write parquet**:
   - Turns each batch row into a parquet row using the schema and `parquetMap` functions.
   - Writes rows to a temp `.idv1.parquet` file, then renames into place.
   - For S3-based inputs, may move the local parquet file back to S3 as `<original>.idv1.parquet`.

The result is a **Timeline ID file** ready for loading by `loadTimeline` or `loadTimelineFile`.

### `InputWorker.idFiles` and `InputWorker.idAllFiles`

- **`idFiles`**:
  - Takes an array of file descriptors or a single `filename`.
  - For each file:
    - Ensures `inputId` and `pluginId` are resolved (via the database and/or metadata).
    - Calls `id` to generate the `.idv1.parquet` file.
    - Upserts `input` records and updates `min_timeline_ts` / `max_timeline_ts`.
  - Returns a summary JSON pointing to the created ID files.

- **`idAllFiles`**:
  - Orchestrates `idFiles` across many accounts in sequence (using `p-limit` to avoid DB deadlocks).
  - When `loadTimeline` is true, immediately runs `loadTimelineTables` on the resulting ID files.

Use these helpers when you need to bulk-convert many raw input files into Timeline ID files and optionally load them.

## Loading Timeline ID files into the database

Once you have Timeline ID files, there are three main ways to move them into engine tables.

### `InputWorker.loadTimelineFile`

This method is the **lowest-level loader** that assumes the file is already timeline-shaped:

- Expects columns: `id`, `ts`, `person_id`, `entry_type_id`, `source_code_id`.
- Validates:
  - `id` is a UUID.
  - All required fields (`id`, `ts`, `person_id`, `entry_type_id`) are present in each row.
- Calls `insertFromStream` to **upsert** rows into the canonical `timeline` table, setting `input_id` from `inputId` when provided (omitted if unset; DB must allow missing `input_id` if you skip it).

Use `loadTimelineFile` when:

- You have a single well-formed Timeline ID file, and
- You want to put it directly into `timeline` without extra statistics or detail table work.

### `InputWorker.loadTimeline`

`loadTimeline` is a higher-level entry point that:

- Accepts:
  - A single file via `filename`, or
  - A `directory`, in which case it:
    - Lists files with `FileWorker.list`.
    - Filters for `.idv1.parquet` and `.timeline.parquet` (legacy).
    - Optionally filters/moves files using `useProcessedPostfix`.
- Chooses a **stats worker** based on `options.engine`:
  - `clickhouse` → `ClickHouseWorker`.
  - `sqlite` → `SQLiteWorker`.
  - Otherwise uses `this` (the `InputWorker`) as the worker.
- For each file:
  - Calls the worker's `loadTimelineFile` (for SQLite/ClickHouse this is the implementation on `SQLiteWorker`).
  - Optionally moves files to a `.processed` variant when `useProcessedPostfix` is set.
- Returns:
  - A `sources` array summarizing each loaded file.
  - The `inputTable` name used for statistics (e.g. `input_<uuid>`).
  - The backing `sqliteFile` when using SQLite.

Use `loadTimeline` when:

- You have one or many ID files and want them:
  - Loaded into a database suitable for statistics and exploration.
  - Optionally tracked via a temporary `input_<uuid>` table.

### `InputWorker.loadTimelineTables`

`loadTimelineTables` orchestrates **both** timeline and detail table loading:

- Accepts:
  - A `fileArray` of items with `idFilename` (and optionally `inputId`), or
  - `idFilename` / `filenames`, or
  - `inputId` / `directory` from which it derives ID filenames via `getIdFilenames`.
- For each ID file:
  - Ensures a valid `inputId` (from options or inferred from the directory).
  - Calls `loadTimeline` to populate the engine-specific timeline table.
  - Optionally calls `loadTimelineDetails` when `loadTimelineDetail` is true and `timelineDetailTable` is provided.
- After all files are processed:
  - If a `timelineDetailTable` is provided, calls `ensureTimelineSummary` to build/refresh a view that joins `timeline`, `input`, `plugin`, `source_code_dictionary`, and the detail table.

Use `loadTimelineTables` when you have:

- A set of `.idv1.parquet` files, and
- A plugin-specific detail table that should be synchronized with the main `timeline` table.

## Timeline Raw files and server-side processing

While the **InputWorker** focuses on ID'd, person-resolved files, other workers (such as email timeline plugins) often start from **Timeline Raw**-like data:

- E.g. SES/SendGrid event streams that include:
  - `ts` (or equivalent),
  - `entry_type` (e.g. `'EMAIL_OPEN'`, `'EMAIL_CLICK'`),
  - `email`, `email_domain`,
  - `account_id`, `plugin_id`,
  - sometimes `person_id` (or an external person identifier).

These workers usually:

1. Map raw vendor events to a timeline-shaped object using:
   - `getEntryTypeId` and `TIMELINE_ENTRY_TYPES` from `@engine9/input-tools`.
   - `getTimelineEntryUUID` to generate `id` using the stable `plugin_id` as UUID namespace.
2. Write out either:
   - A **Timeline Raw** file that still needs `person_id` resolution, or
   - A **Timeline ID** file if they can set `person_id` directly.
3. Hand those files off to `InputWorker.loadTimeline` / `loadTimelineTables` or a downstream pipeline.

When designing new server-side timeline producers:

- **Prefer Timeline ID files** when:
  - You can reliably obtain `person_id`, `input_id`, and `entry_type_id`.
  - You want to rely on `InputWorker.loadTimeline` / `loadTimelineFile` directly.
- **Use Timeline Raw** when:
  - You only have partial identity (e.g., just an `email` or external person id).
  - You plan a separate person-resolution step (often in `PersonWorker`) before emitting final ID files.

## Summary: when to use which method

- **`InputWorker.id`**: Convert a single raw input file into a `.idv1.parquet` Timeline ID file; resolves `person_id` and `id`.
- **`InputWorker.idFiles` / `idAllFiles`**: Bulk version of `id` across many files and/or accounts.
- **`InputWorker.loadTimelineFile`**: Lowest-level loader; load a single Timeline ID file straight into the `timeline` table.
- **`InputWorker.loadTimeline`**: Load one or many Timeline ID files into an engine-specific table (SQLite/ClickHouse/native) with statistics support.
- **`InputWorker.loadTimelineTables`**: End-to-end loader for both `timeline` and plugin-specific detail tables, plus optional summary views.

When in doubt:

- Use **`InputWorker.id` → `InputWorker.loadTimelineTables`** for classic "input → ID file → timeline + detail table" flows.
- Keep raw plugin outputs as close to the **input-tools timeline schema** as possible so they can be easily transformed into Timeline ID files.
