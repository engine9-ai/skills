# engine9 models — developer guide

Read [SKILL.md](SKILL.md) first for what a model is and why. This file is for
running `ModelWorker`, authoring a model plugin, and the internals behind both.

Reminder that survives into code review: a **model** connects a person (and
their transactions) to a timeline entry. **Attribution** connects a transaction
to a **message** and is not a model ([e9-source-code](../e9-source-code/SKILL.md)).
Never write `source_code_summary.origin_*` — those are the legacy origin
implementation on the old identity.

## ModelWorker

`ModelWorker` (alias `model`) loads a model plugin, streams timeline rows, runs
the plugin transforms, and writes **that plugin's** tables. The table stem is
`metadata.prefix` and is the same on every account (like `{prefix}segment`).

```javascript
const model = new ModelWorker(accountWorker);
await model.run({ model: '@engine9/plugins/models/first_touch' });
await model.summarizeSourceCodes({ model: '@engine9/plugins/models/first_touch' });
await model.summarizePeople({ emails: 'a@example.com' });
```

| Method | Writes | Use when |
| --- | --- | --- |
| `runPeople` | Result **file** (csv) | Test the person transform |
| `runTransactions` | Result **file** (csv) | Test the transaction transform. Pass `person_filename` if there is no `transforms.transaction` |
| `run` | Files **and** `{prefix}_*` tables + stats | Production / account load. Installs the plugin row, then deploys tables from `metadata.prefix` (not the plugin counter prefix) |
| `summarizeSourceCodes({ model })` | — | Read `{prefix}_person_stats` and `{prefix}_transaction_stats` (rollup by source code). Alias: `summarize` |
| `summarizePeople({ emails / person_ids })` | — | UI inspect: timeline + stored rows from every available `model_*` table. Does **not** run models |
| `inspectPerson({ emails / person_ids })` | — | Conductor / MCP `timelinePerson`: **current** `timeline` / `model_*` only. Pass `legacy: true` to also load `timeline_v3_summary` / `person_model_source_code` (opt-in; future deployments will drop this). SQL lives in `workers/model` |
| `compareSourceCodes({ source_codes? })` | — | All current `model_*_stats` by source code. Omit `source_codes` to union each model's top 10 by people and by revenue. Pass `legacy: true` to also include `transaction_model_pivot` |
| `loadStats({ model })` | Rebuilds those stats tables | After a manual SQL edit |
| `summarizePeopleLegacy` / `comparePeopleLegacy` / `summarizeSourceCodesLegacy` / `compareSourceCodesLegacy` | — | Legacy identity only — see [Legacy (old identity)](#legacy-old-identity) |

`summarizeSourceCodes` / `loadStats` also accept `prefix: 'model_first_touch'`.

Common options: `model` (plugin path), `timeline` (`sql` / `.duckdb` / file),
`person_ids` / `emails` / `search`, `output_filename`.

Test caps (stop after N rows, then write whatever was produced):

| Option | `runPeople` | `runTransactions` | `run` |
| --- | --- | --- | --- |
| `people_limit` | max people | max people | both stages |
| `transaction_limit` | — | max transactions | transaction stage |
| `limit` | alias for `people_limit` | alias for `transaction_limit` | both aliases (10 people **and** 10 transactions) |

```javascript
await model.runPeople({ model: '…', people_limit: 25 });
await model.runTransactions({ model: '…', transaction_limit: 100, person_filename });
await model.run({ model: '…', people_limit: 25 }); // all transactions for those 25 people
```

File-only tests do not install tables. `metadata.prefix` is still required (the
path must compile).

## Timeline source

`openTimelineSource` (`workers/model/source.js`) is the only place that knows SQL
vs DuckDB vs file. Pass `timeline`; get `{ stream, close, mode }`.

| `timeline` | Reads |
| --- | --- |
| omitted / `'sql'` | Account `timeline` + dictionary labels. Transaction runs join `transaction` on `timeline.id = transaction.id` |
| `*.duckdb` | Same SQL against that file |
| `.parquet` / `.csv` / `.json` | File rows, ordered by `person_id` |

Entries arrive decorated with the parsed source-code columns from
`source_code_summary` (falling back to `source_code_dictionary`) —
`SOURCE_CODE_DICTIONARY_COLUMNS` in `source.js`, including `acquisition_cost`,
`acquisition_cost_per_person`, `acquisition_date`, `source_code_date_parsed`,
`source_code_channel`, and the parsed label fields. Transaction runs also carry
`amount`, `refund_amount`, `recurring_number`, `recurs`, `recurs_id`.

Transaction identity is `transaction.id` (UUID).

## Effective date resolution

`timeline.ts` is the entry's effective date; it is resolved at **load** time, not
at model time. Conceptual cascade: [SKILL.md §5](SKILL.md#5-the-effective-date-of-an-entry).
Field-level rules:

**Per row — `appendTimelineId` (`workers/ServerBaseWorker.js`).** The row's `ts`
is kept as-is, except that the input's `defaultTimestamp` replaces it when `ts`
is `null` / empty, or when the entry is `SOURCE_CODE_OVERRIDE` **and** `ts` is
exactly `'1970-01-01'` (the "no date given" placeholder). A row with no `ts` and
no default gets no timeline id and is not loaded as an entry.

**Per input — `effectiveTimestampFromParsedSourceCodeRow`
(`workers/OverrideWorker.js`).** `loadTimelineOverrides` derives one default
timestamp per override file from the source code parsed out of the file name:

1. `extract_timestamp` option — `'source_code_date'` or `'acquisition_date'`.
   Explicit and strict: if that field is missing, zero, or unparseable, the file
   **throws** instead of falling through.
2. No `extract_timestamp`: use `source_code_date` when it produced a usable
   `source_code_date_parsed`; otherwise use `acquisition_date` when present.
   (`acquisition_date` is re-parsed through `parseSourceCodeDate`.)
3. Otherwise the literal `default_timestamp` option.
4. `default_timestamp` of `'now'`, `null`, or omitted → today, UTC.

`finalizeTimelineOverrideTimestamp` then normalizes to UTC `YYYY-MM-DD` and
throws on missing / empty / zero / unparseable values, or any day before
`1980-01-01` UTC. This mirrors `Timeline3Base#getEffectiveTimestamp` in
frakture-workerbots.

**Parsing a date out of a code — `parseSourceCodeDate`
(`workers/SourceCodeWorker.js`).** Input priority is
`source_code_date_override` → `source_code_date` → `acquisition_date`. Format
detection, in order: explicit `date_format`; `MMYY` (4 digits); `YYYYMMDD` (`2`
plus 7 digits); `MMDDYYYY` (8 digits); `YYMMDD` (6 digits); `YYYY_MM_DD` (10
digit/underscore chars); `YY_MM_DD` (8); else `new Date(...)`. Unparseable →
`source_code_date_parsed = null`. In `parseSourceCodeWithMatchers`, a
non-ISO `acquisition_date` is parsed separately (honoring
`acquisition_date_override`) and only fills in when `source_code_date` did not
parse — source code date keeps historical priority.

**Model-side ordering.** The shipped transforms sort entries by `ts`, tie-broken
by `date_created`. The current SQL timeline source selects `created_at`, not
`date_created`, so ties fall back to source order; do not depend on the
tie-break for same-`ts` entries.

## Output

**Files:** csv with `person_id`, `prefix`, `source_code`, `source_code_id`,
`date_of_source`, `reason` (plus `transaction_id` / `amount` / `ts` on the
transaction file).

**SQL** (`run` only) — one set of tables per model, named from `metadata.prefix`:

| Table | Grain |
| --- | --- |
| `{prefix}_person` | one row per `person_id` (bigint) |
| `{prefix}_transaction` | one row per `transaction_id` (UUID) |
| `{prefix}_person_stats` | people per `source_code_id` |
| `{prefix}_transaction_stats` | tx / revenue / refunds / unique people per `source_code_id` |

No shared `person_model` interface table and no `@engine9/interfaces/model`.
Join `source_code_dictionary` for the code string; join `transaction` on `id`
for amount / `ts`. `run` deploys these through SchemaWorker with `prefix: false`
per table so PluginWorker's install counter is not applied. Columns:
[schema.md](schema.md).

## Writing a model

```
plugins/models/my_model/
  index.js
  person.transform.js
  transaction.transform.js   # optional
```

```javascript
import personTransform from './person.transform.js';

const metadata = {
  name: 'Transaction model: my_model',
  prefix: 'model_my_model',
  unique: true,
  version: '1.0.0'
};

export const transforms = { person: personTransform };
export default { metadata, transforms };
```

`metadata.prefix` is required, must match `model_<name>` (letters, digits,
underscores), **is** the warehouse stem, and must stay stable across accounts.
`run` creates `model_my_model_person`, `model_my_model_transaction`, and the two
`_stats` tables. Do not rely on `plugin.table_prefix` — that gets a per-install
counter.

Plugin identity: `@engine9/plugins/models/my_model`, or `engine9-accounts/...`
for an account-specific model.

**Person transform.** Input `batch[]` is `{ person_id, timelineEntries }`.
Entries already carry `entry_type` and dictionary labels when the source
provided them.

```javascript
const model = { model_id: 9, label: 'My Model' }; // optional label only

async function transform({ batch }) {
  for (const row of batch) {
    row.source_code = /* pick */;
    row.date_of_source = /* the chosen entry's ts */;
    row.reason = ['why'];
  }
}
```

Set `source_code`. Do **not** set `source_code_id` — the worker resolves it
(`doNotUpsert`). Honor `SOURCE_CODE_OVERRIDE` entries the way the shipped models
do, and always write a `reason` naming the branch that fired; `reason` is what
support reads first.

**Transaction transform (optional).** Without one, every transaction inherits
the person code. With one, each item has `transaction.transaction_id` (UUID) and
`transaction.person_model_*` (including `person_model_source_code`).

**Test loop.**

1. Tiny timeline file sorted by `person_id`.
2. `runPeople({ model, timeline: absPath })` — inspect the csv.
3. `runTransactions({ model, timeline: absPath, person_filename })` if needed.
4. Unit-test `transform({ batch })` with hand-built `timelineEntries` (see
   `server/test/workers/modelWorker.test.js`).
5. When the pick rule looks right: `run({ model, person_ids: '123' })` or
   `run({ model, people_limit: 25 })`, then `summarizeSourceCodes({ model })`
   and `summarizePeople({ person_ids: '123' })`.

## summarizePeople (UI inspect)

Read-only payload for a console or debugger. Does not call transforms and does
not write tables. Requires `person_ids`, `emails`, or `search` (same filters as
`run`).

```javascript
const { person_ids, models, people } = await model.summarizePeople({
  emails: 'a@example.com,b@example.com'
});
```

| Field | Meaning |
| --- | --- |
| `person_ids` | Resolved bigint ids (after optional `people_limit`) |
| `models[]` | Warehouse catalog: `{ prefix, person_table, transaction_table }` for every `model_*_person` / `model_*_transaction` that exists (stats tables omitted). Empty if no model has been `run` yet |
| `people[]` | One object per requested id, even when that person has no timeline or model row |

Each `people[]` item has `person_id`, `timeline[]` (rows ordered by `ts`, plus
`entry_type`; joined to `source_code_summary` / `source_code_dictionary` for
labels, `input` for `input_name` / `input_type` / `input_remote_id`, `plugin`
for `plugin_path` / `plugin_name`, and `transaction` for gift fields), and
`models[prefix]` (`{prefix}_person` or `null`, plus `{prefix}_transaction[]`
carrying `amount` / `transaction_ts`).

Timeline `source_code_id` is **as stored** — this method does not apply the
run-time source-code override stream. Stored `reason` / `date_of_source` are
what `run` last wrote. Full field list:
[schema.md](schema.md#summarizepeople-ui-inspect).

## inspectPerson (conductor / MCP)

`inspectPerson` (`workers/model/inspect.js`, `summarize.js`) is the payload for
MCP `timelinePerson` and the conductor Timeline & Models artifact. **Current
identity only**: `timeline` and `model_*_person`. Timeline rows expose
`effective_date` (= `ts`).

Pass `legacy: true` to also run `legacy.js` (`timeline_v3_summary` /
`person_model_source_code`) and append `section: 'legacy'` tables. Default off;
future deployments will not support it. Conductor opts in via
`TIMELINE_PERSON_INCLUDE_LEGACY` in `defs/timelinePerson.ts` — delete that flag
to drop the UI.

```javascript
const { queried, tables, person_ids, emails } = await model.inspectPerson({
  emails: 'a@example.com'
});
```

`tables[]` entries have `status` `ok` / `skipped` / `error`. Current tables:
`timeline`, `models`. Emails are the only identity bridge; `person.id` is never
joined to `person_id_int`.

The payload includes top-level **`sql`**: `[{ id, sql, error, table? }]` for
every warehouse statement the request ran. Same field on `compareSourceCodes`.
This is the MCP executed-SQL standard — see
[e9-mcp](../e9-mcp/SKILL.md#executed-sql-sql).

## compareSourceCodes

`compareSourceCodes` (`workers/model/compare.js`) is the payload for MCP
`timelinePerson` `command: compareSourceCodes` and the conductor `/models`
artifact. It reads **every** current `model_*_person_stats` /
`model_*_transaction_stats` table. Each `rows[]` item is one source code with
`{prefix}_{person_count|revenue|transactions}` columns.

When `source_codes` is omitted, each model contributes its **top 10 source codes
by `person_count` and top 10 by `revenue`**; the comparison uses the union.
Tokens containing `%` use SQL `LIKE`.

```javascript
const auto = await model.compareSourceCodes();
const specified = await model.compareSourceCodes({ source_codes: 'EM_%,MAIL', legacy: true });
```

## Legacy (old identity)

Legacy reads live only in `workers/model/legacy.js` — keep them out of the
current `{prefix}_*` path. These tables were computed against the old identity
model ([e9-person-id](../e9-person-id/SKILL.md#old-identity-model-person_metadata-person_id_int-timeline_v3-transaction_metadata)):
`person_model_source_code_summary`, `timeline_v3_summary`,
`transaction_model_source_code`, `transaction_model_pivot`, `person_metadata`,
`transaction_metadata`, and the `source_code_summary.origin_*` columns.

Rules that cause real bugs when ignored:

- **`person_id_int` is not `person.id`.** `person_metadata` generates
  `person_id_int` from the legacy `person_id` string (a hash, sometimes an
  email). Never join it to current `person.id`.
- Legacy timeline inspect reads **`timeline_v3_summary`**, where the type column
  is **`entry_type_label`** — current `timeline` stores `entry_type_id` and
  current plugin summaries add `entry_type`. Do not SELECT `entry_type` there.
- `emails` is the only bridge: `person_email` → current `person.id`; SHA-256 of
  the trimmed lowercase email (`email_hash_v1`) → `person_metadata.person_id`;
  if the hash misses, try the email string on that column.

```javascript
const legacy = await model.summarizePeopleLegacy({
  person_ids: '1d0cbfeb8a0d6e5606317f9493460d1fbcd4b531f583eba01ba5c7eed9e2e292'
});
const compared = await model.comparePeopleLegacy({ emails: 'user@example.com' });
```

`comparePeopleLegacy` pairs current vs legacy by email (hash first), or takes
explicit `person_ids` + `legacy_person_ids`. Person `match` is source_code
equality (empty ≡ missing). Missing legacy tables are skipped.

`summarizeSourceCodesLegacy({ source_codes })` returns `transaction_model_pivot`
rows only. `compareSourceCodesLegacy({ source_codes })` is the older same-stem
pivot-vs-current **delta** (`{ legacy, current, delta, match }` per metric) for
the `first_touch`, `crm_origin`, and `last_acquisition` stems; `source_codes` is
required there. Comma-delimited, `%` is `LIKE`. Future deployments will drop the
pivot table.

Legacy `model_id` labels match the console CASE: 1 First Touch, 2 CRM Origin,
8 Last Channel Acquisition. Account-specific model ids that the CASE does not
name fall through to the raw id.

## Additional resources

- Concepts, LTV / ROI reads, attribution boundary: [SKILL.md](SKILL.md)
- Table DDL and payload shapes: [schema.md](schema.md)
- Timeline entries and loading: [e9-timeline](../e9-timeline/SKILL.md)
- Source codes and attribution: [e9-source-code](../e9-source-code/SKILL.md)
