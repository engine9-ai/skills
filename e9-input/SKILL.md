---
name: e9-input
description: >-
  Produce engine9 input files and input-store metadata. An input is the generic
  wrapper for one named inbound stream owned by one plugin: a form, email blast,
  ad, SQL table, CRM extract, advocacy action, or other vendor object. Use when
  mapping vendor data into Timeline Raw rows, splitting so each remote_input_id
  is its own input, choosing remote_input_id vs remote_entry_id vs remote_person_id,
  writing files for InputWorker.id / idFiles, or deleting an input or its store
  files. Timeline activity with entry_type is the common case. Deleting requires
  a file list, a count, and a later explicit confirmation before anything is
  removed.
---

# Produce engine9 input files

An **input** is the generic wrapper for one named stream of inbound data owned by one plugin. Almost anything inbound can be an input: a form, an email blast, an ad, a SQL table or named query, a CRM people dump, an advocacy action, a payment extract, or a mixed activity log. Producers write files plus input identity; engine9 then derives `input_id`, resolves people, and (for activity rows) writes Timeline ID files. Use this skill when an outside agent or plugin must emit that shape.

The common case is **timeline** activity: each row is an **entry** with a string `entry_type` (`FORM_SUBMIT`, `EMAIL_SEND`, `EMAIL_OPEN`, `CRM_ORIGIN`, …). Those rows are not on the warehouse `timeline` until people are resolved and the ID files are loaded. Call them entries, never events.

## Quick reference

| Need | Guidance |
| --- | --- |
| What an input is | One named inbound stream (`remote_input_id`) owned by one plugin |
| Grain | One input per durable vendor object (form, blast, ad, table, extract) |
| `input_id` | `getInputUUID(plugin_id, remote_input_id)` — do not invent UUIDs |
| Row shape | Timeline Raw: `ts`, `entry_type`, identity, no `person_id` |
| Form / lead capture | `input_type: form`, `entry_type: FORM_SUBMIT`, one file per form |
| Email blast / SMS / ad | `input_type: message`, one input per send or ad |
| Person extract | `input_type: person`, `remote_input_id: remote_person` |
| SQL table / query | `input_type: sql` (or `person` / `timeline`), stable table or extract name |
| Split mixed streams | Stamp `remote_input_id` on every row, then write one file per id |
| Load path | `InputWorker.id` / `idFiles` → `.idv1.parquet` → optional `loadTimelineTables` |
| Delete an input or its files | List every file and the count, then stop. After a later confirmation, one `FileWorker.removeFiles` call removes that list |

Rule: A submission, payment, click, or lead id is a **row** id (`remote_entry_id`), not a person id (`remote_person_id`).

## Concepts

### The warehouse handle

The warehouse `input` row is the handle for the stream. Timeline entries, transactions, and `person_remote` rows point at `input.id`. The on-disk **input store** uses the same identity.

| Field | Meaning |
| --- | --- |
| `plugin_id` | Owning plugin UUID (`getPluginUUID` / MCP `plugin_id`) |
| `remote_input_id` | Stable vendor key for this stream (form id, message id, ad id, table name, or a named extract) |
| `remote_input_name` | Human label (form name, subject line, table title) |
| `input_type` | Folder and `input.input_type`: a string such as `form`, `message`, `person`, `advocacy`, `sql`, `timeline` |
| `input_id` | UUID derived from `plugin_id` + `remote_input_id` (same value as `input.id`) |

`input_type` is a label, not a closed enum. Use a conventional value from the table below so stores and exports group cleanly.

### Stream identity vs row identity

| Field | Meaning |
| --- | --- |
| `remote_input_id` | Which **stream** (the form, blast, ad, table, or extract) |
| `remote_entry_id` | Idempotency key for **this row** (submission, send, open, click, payment) |
| `remote_person_id` | Stable **person** in the vendor (constituent id, subscriber id) |
| `email` / `phone` | Match keys for `person_id` when the vendor has no person id, or in addition to it |
| `person_id` | Canonical warehouse person — producers of Timeline Raw must omit this |

Rule: Do not put a submission, payment, or click id in `remote_person_id`. That creates one person per row.

How `person_id` is chosen from those match keys is [e9-person-id](../e9-person-id/SKILL.md). Plugin-scoped vendor person ids land in `person_remote` via `source_input_id` → `input.id` ([e9-person-remote](../e9-person-remote/SKILL.md)).

### Timeline entries and `entry_type`

Most input files are person-activity rows that will become `timeline` entries:

| Warehouse | Producer file |
| --- | --- |
| `timeline.ts` | Row `ts` (when it happened, not when it was loaded) |
| `timeline.entry_type_id` | Row `entry_type` (string name; converted with `TIMELINE_ENTRY_TYPES`) |
| `timeline.person_id` | Assigned later from email / phone / `remote_person_id` |
| `timeline.input_id` | Derived from `plugin_id` + `remote_input_id` |
| `timeline.id` | Stable entry UUID; `remote_entry_id` keeps it stable across reloads |

Rule: Prefer the string `entry_type` on producer rows. Use numeric `entry_type_id` only after conversion via `TIMELINE_ENTRY_TYPES`.

Common names for producers (full catalog: [e9-timeline](../e9-timeline/SKILL.md#entry-types)):

| `input_type` | Typical `entry_type` |
| --- | --- |
| `form` | `FORM_SUBMIT`, `FORM_PETITION`, `FORM_ADVOCACY`, `FORM_SURVEY` |
| `message` | `EMAIL_SEND` / `EMAIL_OPEN` / `EMAIL_CLICK`, `SMS_SEND` / `SMS_CLICK`, … |
| `person` | `CRM_ORIGIN` (default for a people dump) |
| `advocacy` | `FORM_ADVOCACY` |
| payment extract | `TRANSACTION_*` — map the row with [transaction mapping](../inputs/transaction-mapping/SKILL.md), still on an input |

Until identity succeeds, a file that only has an email is not a timeline entry yet. After `InputWorker.id`, the store holds Timeline ID (`.idv1.parquet`). Loading those files into `timeline` is optional and separate — large stores often remain as idv1 only ([e9-inventory](../e9-inventory/SKILL.md), [timeline loading](../e9-timeline/loading.md)).

Row-level file shapes (Raw vs ID) are [inputs/timeline](../inputs/timeline/SKILL.md).

### When to split vs one extract

If the vendor has a durable object id, that object is the input. Do not dump every form, blast, or ad into one file when those ids exist.

| Source | `input_type` | `remote_input_id` |
| --- | --- | --- |
| Named forms, petitions, surveys, lead-gen forms | `form` | Vendor form / campaign id |
| Email blast, SMS send, or ad | `message` | Vendor message / send / ad id |
| Advocacy actions with a campaign or page id | `advocacy` | Vendor campaign / page id |
| CRM or ESP people dump | `person` | `remote_person` |
| SQL table or named query | `sql` | Stable table or extract name (not a freshly hashed query string) |
| Mixed activity with **no** durable form or message id | `timeline` | One named extract (for example `remote_person_signup`) |

A signup form may also appear as a `signup_form` **message** in the message catalog / `global_message`. That message input is the cataloged send, not the submission stream. Submission rows belong on the **form** input keyed by the vendor form id.

## File format

### Producer rows (Timeline Raw)

Emit CSV, JSONL, or parquet. Rows must not include `person_id`. Required and identity fields:

| Field | Required | Notes |
| --- | --- | --- |
| `ts` | Yes | Parseable `Date` (ISO-8601 preferred) |
| `entry_type` | Yes | String name (`FORM_SUBMIT`, `EMAIL_OPEN`, `CRM_ORIGIN`, …) |
| `remote_input_id` | Yes unless the whole file is one extract | Used to derive `input_id` |
| `remote_input_name` | Recommended | Copied onto the `input` row |
| `remote_entry_id` | Recommended | Stable vendor row id for timeline UUID / upsert |
| `email` and/or `phone` and/or `remote_person_id` | Yes | At least one match key |
| `given_name`, `family_name` | Optional | Attributes, not match keys |
| `source_code` | Optional | Last-click / origin string; do not invent one ([e9-source-code](../e9-source-code/SKILL.md)) |
| Other columns | Optional | Plugin detail (campaign ids, custom questions, URLs, …) |

Typical form row:

```javascript
{
  ts: "2026-03-01T15:04:05Z",
  entry_type: "FORM_SUBMIT",
  remote_input_id: "1234567890",
  remote_input_name: "Volunteer signup",
  remote_entry_id: "lead-aaa",
  email: "person@example.org",
  phone: "+15555550100",
  given_name: "Ada",
  family_name: "Lovelace",
  campaign_id: "111",
  ad_id: "222"
}
```

Person-extract row (`input_type: person`) uses `remote_input_id: "remote_person"` on the **file**, not per row. Default `entry_type` is `CRM_ORIGIN`. Include `remote_person_id` when the vendor has a durable constituent id.

Typical email-blast rows share one `remote_input_id` (the send) and differ by `entry_type`:

```javascript
{
  ts: "2026-04-01T12:00:00Z",
  entry_type: "EMAIL_SEND",
  remote_input_id: "msg-123",
  remote_input_name: "April Appeal",
  remote_entry_id: "send-aaa",
  email: "person@example.org"
}
```

Opens and clicks for that blast keep the same `remote_input_id` and use `EMAIL_OPEN` / `EMAIL_CLICK` with their own `remote_entry_id` and `ts`.

### Store metadata

Write one store per input:

```text
{account_id}/plugins/{plugin_id}/{input_type}/{input_id.slice(0,4)}/{input_id}/
  metadata.json
  {filename}
```

`metadata.json` (snake_case):

```json
{
  "input_id": "<uuid>",
  "plugin_id": "<plugin uuid>",
  "input_type": "form",
  "remote_input_id": "1234567890",
  "remote_input_name": "Volunteer signup"
}
```

Rule: `input_id` must equal `getInputUUID(plugin_id, remote_input_id)`. MCP `input_id` computes the same value. Do not use a random UUID.

After `InputWorker.id`, the store also has `.idv1.parquet` (Timeline ID). Export copies of those stores are [e9-export](../e9-export/SKILL.md).

### Handoff: one file per input

A producer returns a list of files (or writes them to the store above). Each item is **one** input:

| Field | Role |
| --- | --- |
| `filename` | CSV / JSONL / parquet of rows for **one** input |
| `input_id` | Derived UUID |
| `remote_input_id` / `remote_input_name` | Vendor stream identity |
| `input_type` | `form`, `message`, `person`, `sql`, … |
| `records` | Row count |

Stamp `remote_input_id` on every row of a mixed stream, then split. Do not concatenate forms, blasts, or ads into one stored file.

## Workflow

### Map a vendor feed

1. List the durable objects (forms, messages/sends, ads, tables, or a single extract name).
2. Choose `input_type` and `entry_type` from the tables above.
3. Map identity: vendor person id → `remote_person_id`; otherwise email/phone; submission / send / click id → `remote_entry_id`.
4. Map `ts` from the vendor event time, not load time.
5. Copy extra fields as detail; do not overload identity columns.
6. Split so each `remote_input_id` is its own file, then store with `metadata.json`.

### Produce files as an outside agent

1. Resolve `plugin_id` for the integration (MCP `plugin_id` or `getPluginUUID`).
2. For each durable object, set `remote_input_id` to the vendor object id (or a stable extract name).
3. Compute `input_id = getInputUUID(plugin_id, remote_input_id)`.
4. Write one Timeline Raw file per input plus `metadata.json`.
5. Run `InputWorker.id` / `idFiles` (people + Timeline ID), then `loadTimelineTables` if warehouse `timeline` load is in scope.

### Delete an input or its store files

A delete covers the input store (`metadata.json`, raw files, `.idv1.parquet`, and every other file in that input's directory) and, when the request includes it, the warehouse `input` row. Timeline or transaction rows that point at the input are a separate delete. They still require this same list, count, and confirmation when the request is to remove the input's files.

1. Resolve each input: `account_id`, plugin, `input_type`, `input_id`, and `data_path`.
2. List every file the delete would remove. When more than one input is in scope, say which input each file belongs to.
3. State the file count, and the input count when more than one input is in scope. Zero files is still a result: say there are none.
4. Stop. Show that list and those counts. Do not delete in that turn.
5. Delete only after a later message confirms that exact list. The original request to delete is not confirmation. A "yes" or "delete them" sent before the list was shown is not confirmation. A reply that does not refer to the listed set is not confirmation.
6. Remove the confirmed files with one `@engine9/plugins/e9workers:FileWorker` `removeFiles` call. Pass `filenames` (comma-delimited or an array), `file_array` (path strings or `{ filename }` objects), or `options_filename` / `optionsFilename` (a JSON file of those options, including an `idFiles` options file). Do not schedule one `remove` task per file. When the confirmed set is every file in one input’s store directory, MCP `input` `deleteFiles` is that single call. It resolves the directory from `input_id` and does not take a path.
7. If the store changes, or the user changes which inputs are in scope, list and count again and wait for a new confirmation.

## Rules

Rule: One named remote stream per input. If the vendor has a durable form, message, ad, or table id, that object is the input.

Rule: Call timeline records entries, never events.

Rule: Timeline Raw rows must not include `person_id`.

Rule: `remote_person_id` is a person in the vendor system. Submission, send, click, and payment ids go in `remote_entry_id`.

Rule: Do not set `input_id` by hand except via `getInputUUID(plugin_id, remote_input_id)`.

Rule: Do not mix `input_type` values in one file. Split first.

Rule: `FORM_SUBMIT` is the form-action type. Use `SIGNUP` / `SIGNUP_INITIAL` / `SIGNUP_SUBSEQUENT` only for list-subscription facts, not for “someone filled this form.”

Rule: Leave `source_code` empty when the vendor has no code; ad ids and campaign ids are detail, not source codes.

Rule: Never delete input store files or an `input` row in the same turn that lists them. Show the file list and the count, then wait for a later message that confirms that exact list.

Rule: Remove a confirmed set of input store files with one `FileWorker.removeFiles` call. Do not schedule one `remove` task per file.

Rule: MCP `input` `deleteFiles` removes every file directly in one input’s store directory. Use it only after the user confirms that folder. It does not delete a subset of the folder, the warehouse `input` row, or timeline rows.

Conductor’s Inputs artifact exposes the same folder delete: the Files card’s Delete button confirms, then calls `deleteFiles`.

## Examples

### Split a mixed form stream

```javascript
// Each row already has remote_input_id = vendor form id
const byRemoteInputId = new Map();
for (const row of rows) {
  const id = String(row.remote_input_id);
  if (!byRemoteInputId.has(id)) byRemoteInputId.set(id, []);
  byRemoteInputId.get(id).push(row);
}
// Write one Timeline Raw file + metadata.json per map entry
```

### Single person extract (do not split)

```javascript
{
  input_type: 'person',
  remote_input_id: 'remote_person',
  remote_input_name: 'Load People From Remote System'
}
```

Rows: `ts`, `remote_person_id`, `email`, `given_name`, `family_name`. File-level identity; rows do not need `remote_input_id`.

### SQL table extract

```javascript
{
  input_type: 'sql',
  remote_input_id: 'public.constituents',
  remote_input_name: 'Constituents table'
}
```

Use a stable table or extract name. Do not hash the SQL text as `remote_input_id` unless that string is itself durable — a changed query would mint a new `input_id`.

### Lead-gen form with no vendor person id (Facebook)

When the vendor issues a form id and a per-submission id, but no constituent id:

| Vendor | engine9 |
| --- | --- |
| Form `id` | `remote_input_id`, `input_type: form` |
| Form `name` | `remote_input_name` |
| Submission time | `ts` |
| Submission / lead `id` | `remote_entry_id` |
| Email / phone / names | `email`, `phone`, `given_name`, `family_name` |
| Ad / campaign / ad-set ids | Detail columns |
| — | No `source_code` unless the form has a real tracking code |

If a row has neither email nor phone, fall back to `remote_person_id` = submission id so the row can still resolve. That fallback is last-resort only.

Do **not** key the input by ad, ad set, or campaign when the object people filled in is the form. The ad can be its own **message** input for spend and catalog; submissions still belong on the form.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Every lead is a different person | Submission id was stored as `remote_person_id`; move it to `remote_entry_id` and match on email/phone |
| All forms (or blasts) landed in one `input` | File was stored with a single `remote_input_id`; stamp the vendor object id on each row and split |
| `input_id` changes across loads | `remote_input_id` is unstable (name instead of id, a new UUID each run, or a hashed query that changed) |
| `No input_id, and no remote_input_id` | Form/message/ad streams must set `remote_input_id` on every row; person extracts must pass file-level identity |
| Loader rejects identity | Row has no `email`, `phone`, `remote_person_id`, or `person_id` |
| Duplicate activity | Missing `remote_entry_id` / unstable `ts` so timeline UUIDs change |
| Message and form share an id | They are different streams; do not reuse a message uuid as the form `input_id` unless they are intentionally the same store |
| Entries missing on Person → Timeline | ID files may exist in the input store without a `timeline` load; see [e9-timeline](../e9-timeline/SKILL.md) |
| Need the store files and a row sample for one input | Conductor `/input`, or MCP `input` (`list`, `files`, `summary`). The tool takes `input_type` as a filter and does not special-case messages, transactions, or other types. See [e9-mcp](../e9-mcp/SKILL.md) |
| Delete every file in one input’s store folder | Conductor Inputs → Files → Delete, after the confirm. That calls MCP `input` `deleteFiles`. Same list-then-confirm rule when an agent does it. |
| A delete ran without a shown file list | Stop. Deleting an input requires the file list, the count, and a later confirmation of that exact list |

## Related documentation

- [Timeline Raw vs Timeline ID](../inputs/timeline/SKILL.md)
- [Timeline model and entry types](../e9-timeline/SKILL.md)
- [Timeline loading](../e9-timeline/loading.md)
- [Person identity](../e9-person-id/SKILL.md)
- [Person remotes](../e9-person-remote/SKILL.md)
- [Transaction mapping](../inputs/transaction-mapping/SKILL.md)
- [Export input stores](../e9-export/SKILL.md)
- [Inventory (input-store vs warehouse)](../e9-inventory/SKILL.md)
- MCP `plugin_id` / `input_id`
- `@engine9/input-tools` `getPluginUUID`, `getInputUUID`, `TIMELINE_ENTRY_TYPES`
