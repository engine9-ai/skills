---
name: e9-person-remote
description: >-
  Explains engine9 person_remote: the attribute table that ties a person_id to a
  plugin-scoped remote_person_id via source_input_id, how exports/search filter
  by input.plugin_id, and when rows are written (loadPeople / idFiles / id,
  not loadTimeline or plugin-prefixed people tables). Use when working with
  person_remote, remote_person_id, Tatango/EN/RENxt remotes, export audiences
  filtered by remote people, idFiles without loading bot tables, or empty
  person_remote for a plugin.
---

# engine9 `person_remote`

`person_remote` answers: **does this `person_id` have a vendor/CRM id for plugin X?** It is an **attribute** table written by the people pipeline — not a timeline load, and not `{table_prefix}person` (e.g. `tatango_kic_person`).

Person **identity** matching on remotes (lookup keys, first-wins): [e9-person-id](../e9-person-id/SKILL.md). Timeline ID files: [e9-timeline/loading.md](../e9-timeline/loading.md).

## Table shape

| Column | Role |
|--------|------|
| `person_id` | Warehouse person (`person.id`) |
| `remote_person_id` | Vendor id string as stored on the row (usually **without** the `{pluginId}.` prefix) |
| `source_input_id` | `input.id` for the stream that wrote this remote |
| `id` | Surrogate PK |

Unique: `(source_input_id, remote_person_id, person_id)`.

**Plugin scoping is not a column on `person_remote`.** It comes from joining `input`:

```sql
FROM person_remote
INNER JOIN input ON person_remote.source_input_id = input.id
  AND input.plugin_id = '<plugin uuid>'
```

That is exactly what `@engine9/interfaces/person_remote:search:all` compiles when given `pluginId` — the filter used by export universes (“remote people for plugin X”).

Empty export / segment for “Tatango remotes ∩ transactions” almost always means: **no `person_remote` rows whose `source_input_id` belongs to the Tatango `input.plugin_id`**, not “no transactions.”

## When rows are written

`person_remote` is upserted inside the **inbound people pipeline**, after `person_id` is assigned:

```
… → person_remote:transforms:id     # extract remote_person_id → identifiers
  → … appendInputId / appendPersonId …
  → person_remote:transforms:upsert # requires pluginId; needs remote_person_id + person_id + input_id
  → sql.tables.upsert
```

Built by `buildInboundTransforms` (`@engine9/core/lib/peoplePipeline/getInboundTransforms.js`). Upsert implementation: `@engine9/interfaces/person_remote/transforms/inbound/upsert_tables.js`.

### Triggers that write `person_remote`

| Entry point | Writes `person_remote`? | Notes |
|-------------|-------------------------|--------|
| `PersonWorker.loadPeople` | **Yes** (default) | Direct path. Needs `plugin_id`, `default_input_id` (or per-row `input_id` / `remote_input_id`), and rows with `remote_person_id`. |
| `InputWorker.id` | **Yes** | Each batch calls `loadPeople` with `plugin_id` + `default_input_id`. Also stamps timeline ids / writes `.idv1.parquet` when the file has timeline-shaped rows. |
| `InputWorker.idFiles` / `idAllFiles` | **Yes** | Resolves `input_id` + `plugin_id`, then calls `id` per file. Upserts the `input` row. Does **not** call `loadTimeline`. |
| `PersonWorker.import` / console import | **Yes** | Thin wrapper over `loadPeople` with import defaults. |
| `PersonWorker.populatePersonRemoteFromIdentifiers` | **Yes** (backfill) | Copies legacy `person_identifier` (`id_type=remote_person_id`) into `person_remote`. Migration/repair, not day-to-day loads. |
| `InputWorker.loadTimeline` / `loadTimelineTables` | **No** | Loads `timeline` (+ detail). Assumes people already identified. |
| Plugin-prefixed people tables (`*_person`) | **No** (by themselves) | e.g. `engaging_efr_person` / `tatango_kic_person` are bot warehouses. Remotes appear in `person_remote` only when those rows are run through `loadPeople` / `id` / `importPeople`-style flows. |

### What “without loading to the database” means here

`idFiles` **does** write warehouse identity/attribute tables (`person`, `person_remote`, `person_email`, …). It does **not** require:

- `loadTimeline` / `loadTimelineTables`
- A plugin bot table such as `tatango_kic_person`

So: use **`idFiles` (or `loadPeople`) to populate Tatango-scoped `person_remote` without a Tatango people-table load or a timeline load.** Skip `do_not_upsert` / `doNotUpsert` (those run identity lookup only and **skip** all upserts, including `person_remote`).

## Required options so a plugin-scoped search sees people

For `@engine9/interfaces/person_remote:search:all` with `pluginId = <Tatango uuid>` (or the equivalent export SQL) to return rows after an id/load:

1. **`plugin_id` / `pluginId`** = that plugin’s UUID (must be a UUID, not `remote_plugin_id` / `tatango_kic`).
2. **`input_id` / `default_input_id`** = an `input.id` whose **`input.plugin_id` is that same plugin**. `person_remote.source_input_id` is set from the row’s `input_id` (via `appendInputId` + upsert). If you stamp an Engaging Networks input while passing Tatango as `plugin_id`, search-by-Tatango will still miss the rows.
3. **File/stream rows** with a non-empty **`remote_person_id`**. Without it, extract and upsert no-op for remotes (email/phone-only rows still create people, but no `person_remote`).
4. Prefer an **email and/or phone** on the same row so new remotes attach to existing donors instead of creating orphan people — identity still follows [e9-person-id](../e9-person-id/SKILL.md).
5. **`do_not_upsert` must be false** (default).

### `idFiles` sketch

```js
await inputWorker.idFiles({
  filename: '/path/to/people.csv', // or file_array: [{ filename, input_id, ... }]
  input_id: '<uuid of an input owned by that plugin>',
  plugin_id: '<plugin uuid>',
  // optional: remote_input_id / remote_input_name / input_type when creating/healing input metadata
});
```

`idFiles` requires `input_id` (or `default_input_id`) on each file item. If `plugin_id` is omitted, it is loaded from that `input` row — so the input must already belong to the target plugin.

Equivalent without writing an ID parquet (people only):

```js
await personWorker.loadPeople({
  filename: '/path/to/people.csv', // or stream: [{ remote_person_id, email, ... }]
  plugin_id: '<plugin uuid>',
  default_input_id: '<uuid of an input owned by that plugin>'
});
```

### Extract / upsert details

- **Extract** (`person_remote:transforms:id`): builds lookup key `{pluginId}.{remote_person_id}` lowercased, unless the inbound value already starts with a UUID + `.`.
- **Upsert**: looks up existing remotes for **this `pluginId`** via `person_remote` ⨝ `input`; inserts with `source_input_id = row.input_id`, or updates the existing row for the same person+remote under that plugin.
- Rows lacking `remote_person_id` or `person_id` are skipped by upsert.

## Outbound / exports

`person_remote:transforms:appendRemotePersonId` joins remotes for a plugin onto an export file (the inverse of inbound upsert). Search `all` only filters existence by plugin; append is for writing the vendor id column back out.

## Debugging empty remotes for a plugin

1. Confirm plugin UUID: `account` plugins (or `SELECT id, name, path FROM plugin`).
2. Count remotes for that plugin (join `input`) — not a bare `COUNT(*)` on `person_remote`.
3. Confirm candidate `input` rows: `SELECT id, plugin_id, input_type, remote_input_name FROM input WHERE plugin_id = ?`.
4. If bot tables exist but remotes do not, those people were never run through `loadPeople`/`id` with that plugin’s `plugin_id` + matching `input_id`.
5. If remotes exist under the wrong plugin’s inputs, fix the input ownership / re-id with the correct `plugin_id` + `input_id` pair — do not only change the export filter unless that matches product intent.

## Code map

| Piece | Location |
|-------|----------|
| Schema / search / transforms export | `@engine9/interfaces/person_remote` |
| Extract identifiers | `…/transforms/inbound/extract_identifiers.js` |
| Upsert | `…/transforms/inbound/upsert_tables.js` |
| Pipeline wiring | `@engine9/core/lib/peoplePipeline/getInboundTransforms.js` |
| `loadPeople` | `server/workers/PersonWorker.js` |
| `id` / `idFiles` | `server/workers/InputWorker.js` |
| Backfill from `person_identifier` | `PersonWorker.populatePersonRemoteFromIdentifiers` |
