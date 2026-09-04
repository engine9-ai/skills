---
name: e9-export
description: >-
  Explains engine9 export packages for receivers: directory layout,
  inventory.json5, warehouse table parquet, input-store .idv1.parquet files,
  metadata.json fields, common entry_type_ids, person-search CSVs, and how to
  join the pieces. Use when reading or consuming an export, interpreting
  metadata.json or inventory.json5, joining person / transaction / timeline
  files, or asking what is in an export. Create, run, and debug with the e9
  CLI: [building.md](building.md).
---

# engine9 exports

An **export** is a self-contained snapshot of one account’s warehouse data. A receiver gets files — not a live database. Typical contents are **table parquet**, **idv1 file copies** with each store’s **`metadata.json`**, and sometimes **person-search CSVs**.

This document is for **reading** an export. Creating, running, and debugging one is [building.md](building.md).

Related: warehouse inventory [e9-inventory](../e9-inventory/SKILL.md); person identity [e9-person-id](../e9-person-id/SKILL.md); remotes [e9-person-remote](../e9-person-remote/SKILL.md); timeline entries [e9-timeline](../e9-timeline/SKILL.md); source codes [e9-source-code](../e9-source-code/SKILL.md).

## What’s in the package

Default root: `{store_path}/{account_id}/exports/{export_id}/{date}/`

Start with **`inventory.json5`**. It is the catalog of what was written. A definition may relocate artifacts with `relative_path`; trust the inventory, not assumed paths.

| Path | Contents |
|------|----------|
| `tables/{table}.parquet` | One warehouse table dump |
| `{input_type}/{input_id}/*.idv1.parquet` | Activity rows for one input (`message`, `person`, `timeline`, …) |
| `{input_type}/{input_id}/metadata.json` | Descriptor for that input store |
| `search/{export_name}.export.csv` + `.metadata.json5` | Named person-search extract |
| `inventory.json5` | Export **plan**: paths, counts, skipped items |

**Not included**

- Raw CSV/JSON from the source input store (even when `metadata.json` lists them)
- Monthly warehouse **statistics** — those live on a standalone [inventory](../e9-inventory/SKILL.md) run, not in the export’s `inventory.json5`
- The `setting` table (credentials / config)

`inventory.json5` and the export result both carry `source_directory`: the export root. Strip that prefix from any absolute `filename` to recover the path under the root.

## `inventory.json5`

Bundle exports write this at the export root (`format_version` **2**). It is a **plan**, not monthly statistics.

| Key | Purpose |
|-----|---------|
| `definition_path`, `plugin_path`, `source_directory` | Which bundle ran, and the export root |
| `universe` | Resolved bundle entries |
| `tables[]` | `{ table, relative_path, records }` |
| `files[]`, `directories[]` | Planned idv1 copies and `metadata.json` |
| `skipped_tables[]`, `skipped_files[]` | Omitted items (`does_not_exist`, selector mismatch, …) |
| `table_records`, `file_records`, `records` | Plan totals |

`directories[].files` is the **file list** (name, filename, records, source_directory), not a count.

Use it to confirm every expected parquet exists, to recover relative paths, and to see why a table or store was omitted.

## Warehouse tables

Table parquet is a dump of the named warehouse table at export time. Columns match the live table; `DESCRIBE` / parquet schema is the authority. Bundles choose a subset — a typical dump includes people, messages, source codes, and transactions, not every warehouse table.

| Table | Role | Join |
|-------|------|------|
| `plugin` | Installed integration (`id`, `path`, `name`) | `input.plugin_id` |
| `input` | One stream (message, form, extract): `id`, `plugin_id`, `remote_input_id`, `remote_input_name`, `input_type`, `min_timeline_ts`, `max_timeline_ts`, `records` | Folder `{input_id}`; `timeline.input_id`; `transaction.input_id` |
| `person` | Canonical person. `id` **is** `person_id` (`given_name`, `family_name`) | Every other `person_id` |
| `person_email` | Emails on a person (`email`, `subscription_status`, `email_hash_v1`) | `person_id` |
| `person_phone` | Phones (`phone`, `sms_status`, `call_status`) | `person_id` |
| `person_address` | Postal addresses | `person_id` |
| `person_remote` | Vendor/CRM id for a plugin (`remote_person_id`, `source_input_id`) | `source_input_id` → `input.id` → `plugin` — [e9-person-remote](../e9-person-remote/SKILL.md) |
| `transaction` | Payments: `id`, `ts`, `person_id`, `input_id`, `amount`, `entry_type_id`, `source_code_id`, `recommended_message_id` | Person, input, source code, message |
| `source_code_dictionary` | One row per code (`source_code_id`, `source_code`) | `transaction.source_code_id`, timeline `source_code_id` |
| `source_code_summary` / `_by_date` | Per-code rollups (attributed revenue/transactions; `origin_*` is **legacy**) | `source_code_id` — [e9-source-code](../e9-source-code/SKILL.md) |
| `global_message` / `message` | Message identity (`channel`, `publish_date`, primary source code) | `input.id` for a message input is often the message id |
| `global_message_summary` / `_by_date` | Per-message rollups (sends, attributed revenue; daily spend/impressions) | Message / plugin |
| `timeline` | Warehouse entry log (when the bundle includes it): `id`, `ts`, `person_id`, `entry_type_id`, `input_id` | Same keys as idv1 rows |

Identity: [e9-person-id](../e9-person-id/SKILL.md). One person may have many emails, phones, and remotes. Do not treat `email` as a person key.

Say **transaction**, never donation. Revenue questions use `transaction` (and summary tables), not timeline `TRANSACTION_*` rows.

Say **entry**, never event. A timeline / idv1 row is an **entry**.

## Input stores: `metadata.json` + idv1

Each selected **input** is one stream — usually one message, one form, or one named extract. The export copies:

1. Every `.idv1.parquet` in that store
2. That store’s `metadata.json`

Default layout: `{input_type}/{input_id}/{filename}`. Unknown `input_type` is stored as `timeline`. The folder name `{input_id}` is the same UUID as `input.id` and as `timeline.input_id` / `transaction.input_id`.

### `metadata.json`

Snake_case on disk. Older files may still use camelCase (`inputId`, `pluginId`, `entryTypes`); prefer snake_case.

```json
{
  "input_id": "c8ec58a6-a9d7-5277-a12a-030cc01d037f",
  "plugin_id": "a1b2c3d4-0000-4000-8000-000000000001",
  "input_type": "message",
  "remote_input_id": "msg-123",
  "remote_input_name": "April Appeal",
  "min_timeline_ts": "2024-06-01T10:00:00.000Z",
  "max_timeline_ts": "2024-12-31T23:59:59.000Z",
  "records": 152340,
  "distinct_people": 89000,
  "distinct_records": 152340,
  "files": [
    { "filename": "email-sends.idv1.parquet", "records": 152340 }
  ],
  "entry_types": {
    "EMAIL_SEND": {
      "min_timeline_ts": "2024-06-01T10:00:00.000Z",
      "max_timeline_ts": "2024-12-31T23:59:59.000Z",
      "records": 152340,
      "distinct_people": 89000,
      "distinct_records": 152340
    }
  }
}
```

| Field | Meaning |
|-------|---------|
| `input_id` | Store / `input.id` (matches the folder name) |
| `plugin_id` | Owning plugin (`plugin.id`) |
| `input_type` | `message`, `person`, `timeline`, form type, … |
| `remote_input_id` / `remote_input_name` | Vendor id and human name (message title, form name) |
| `min_timeline_ts` / `max_timeline_ts` | Timestamp span across idv1 rows |
| `records` | Row count across idv1 files |
| `distinct_people` | Distinct `person_id` |
| `distinct_records` | Distinct timeline entry `id` |
| `files[]` | `{ filename, records }` for files **in the source store**. Only `.idv1.parquet` names in this list were copied. Legacy entries may be a bare string. |
| `entry_types` | Map keyed by **string name** (not an array). Each value has `min_timeline_ts`, `max_timeline_ts`, `records`, `distinct_people`, `distinct_records`. |

`entry_types` names are exact (`EMAIL_OPEN`, not `EMAIL_*`). That map is how a bundle selected the store; it is also how a receiver knows which activity kinds are in the parquet.

`records: 0` in metadata can be stale. Prefer parquet `COUNT(*)` when the count matters.

### `.idv1.parquet` rows

These are **Timeline ID** files: already resolved to `person_id` and a stable entry `id`. They are ready to load or query. Shape: [inputs/timeline](../inputs/timeline/SKILL.md).

| Column | Meaning |
|--------|---------|
| `id` | Stable UUID for this entry (reload upserts; not a person key) |
| `ts` | When it happened (not when engine9 loaded it) |
| `person_id` | Canonical person (`person.id`) |
| `entry_type_id` | Integer type (names below) |
| `input_id` | Present on warehouse `timeline`; often implicit from the folder on a copied store |
| `source_code_id` | Optional last-click / origin code |
| `email_domain` | Optional lowercased domain |

Extra columns (URL, amount, user agent, …) are plugin **detail** fields. They vary by file.

A click is not automatically an open. Filter on `entry_type_id`; do not infer one type from another.

## Entry types

Parquet stores **`entry_type_id` (integer)**. `metadata.json` `entry_types` uses the **string name**. Full catalog: [e9-timeline](../e9-timeline/SKILL.md#entry-types).

| Group | Name | Id | Typical meaning |
|-------|------|----|-----------------|
| Origin | `CRM_ORIGIN` | 1 | Person’s origin in the CRM |
| Origin | `ACQUISITION` | 2 | Acquisition |
| Signup | `SIGNUP` / `SIGNUP_INITIAL` / `SIGNUP_SUBSEQUENT` | 3 / 4 / 5 | List / form signup |
| Signup | `UNSUBSCRIBE` | 6 | Channel-agnostic unsubscribe |
| Money | `TRANSACTION` | 10 | Generic payment; prefer a specific type |
| Money | `TRANSACTION_ONE_TIME` | 11 | One-time payment |
| Money | `TRANSACTION_INITIAL` / `TRANSACTION_SUBSEQUENT` / `TRANSACTION_RECURRING` | 12 / 13 / 14 | Recurring series |
| Money | `TRANSACTION_REFUND` | 15 | Refund |
| Message | `MESSAGE_CONVERSION` / `_ADVOCACY` / `_TRANSACTION` | 20 / 21 / 22 | Conversion credited to a message |
| SMS | `SMS_SEND` / `SMS_DELIVERED` / `SMS_CLICK` | 30 / 31 / 33 | SMS lifecycle |
| SMS | `SMS_UNSUBSCRIBE` / `SMS_BOUNCE` / `SMS_SPAM` / `SMS_REPLY` | 34 / 37 / 38 / 39 | SMS negative / reply |
| Email | `EMAIL_SEND` / `EMAIL_DELIVERED` | 40 / 41 | Send / delivery |
| Email | `EMAIL_OPEN` / `EMAIL_CLICK` | 42 / 43 | Engagement |
| Email | `EMAIL_UNSUBSCRIBE` / `EMAIL_SOFT_BOUNCE` / `EMAIL_HARD_BOUNCE` / `EMAIL_BOUNCE` / `EMAIL_SPAM` / `EMAIL_REPLY` | 44 / 45 / 46 / 47 / 48 / 49 | Email negative / reply |
| Forms | `FORM_SUBMIT` / `FORM_PETITION` / `FORM_ADVOCACY` / `FORM_SURVEY` | 60 / 61 / 66 / 67 | Actions |

`SOURCE_CODE_OVERRIDE` (0) is a dictionary override marker, not a person action.

Email-oriented bundles typically ship stores whose `entry_types` include `EMAIL_*`. SMS-oriented bundles ship `SMS_*`.

## How the pieces join

```
plugin.id  =  input.plugin_id
input.id   =  folder {input_id}
           =  metadata.json input_id
           =  timeline.input_id / transaction.input_id
           =  person_remote.source_input_id   (for remotes written by that stream)

person.id  =  person_email.person_id
           =  person_phone.person_id
           =  person_remote.person_id
           =  transaction.person_id
           =  idv1 / timeline.person_id

source_code_dictionary.source_code_id  =  transaction.source_code_id
                                       =  idv1 / timeline.source_code_id
```

Plugin scope for remotes is **not** a column on `person_remote`. Join `person_remote.source_input_id` → `input.id` → `input.plugin_id`.

A message input’s `input.id` is the handle for that send. Opens and clicks for that send live in stores whose `metadata.json` `input_id` (or `entry_types`) points at that activity; they share `person_id` with `person` / `transaction`.

## Person-search CSVs

A named search export is one CSV plus a sidecar:

- `search/{export_name}.export.csv` (or `.export.csv.gz`)
- `search/{export_name}.export.csv.metadata.json5`

Columns are whatever the search and its row transforms selected — not a fixed warehouse schema.

The sidecar records how the file was built:

| Field | Meaning |
|-------|---------|
| `configuration` | Options used for the run |
| `sql` | Compiled search SQL (also pasted as a trailing `/* … */` comment) |
| `records` | Row count |
| `people_searched` | People in the search universe |
| `sample_person_ids` | Sample of matched ids |
| `deduplicated` / `deduplicated_against` | Present when prior export files were used as a dedupe set |

## Reading the files

Parquet is the native format. DuckDB example:

```sql
-- Catalog
-- (open inventory.json5 next to the files)

SELECT id, given_name, family_name
FROM read_parquet('tables/person.parquet')
LIMIT 20;

SELECT id, ts, person_id, amount, source_code_id
FROM read_parquet('tables/transaction.parquet')
WHERE person_id = 12345
ORDER BY ts DESC
LIMIT 20;

-- One input store (entry_type_id 42 = EMAIL_OPEN)
SELECT id, ts, person_id, entry_type_id
FROM read_parquet('timeline/*/opens.idv1.parquet')
WHERE entry_type_id = 42
LIMIT 20;
```

Resolve email → `person_id` via `person_email`, then join other files on `person_id`. Do not treat timeline `id` as a person key.

## Building and debugging

Create, run, and debug with `e9 exportworker export`: [building.md](building.md). Pre-run warehouse snapshot: [e9-inventory](../e9-inventory/SKILL.md).
