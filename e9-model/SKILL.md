---
name: e9-model
description: >-
  Run and author engine9 models via ModelWorker: timeline-based long-term
  value (LTV, first touch, CRM origin, last acquisition, …) written to
  {prefix}_* tables (model_first_touch_person, …) with bigint person_id and
  UUID transaction_id. Use when working with ModelWorker, model_*_person,
  model_*_transaction, or writing a new model — not last-click attribution
  (e9-source-code) and not source_code_summary.origin_* (legacy origin).
---

# engine9 models

**Attribution** connects a transaction to a message (`recommended_message_id`, `attributed_revenue`). That join is [e9-source-code](../e9-source-code/SKILL.md). Do not call last-click a model.

A **model** connects a person or transaction to an item in their [timeline](../e9-timeline/SKILL.md) history — not to a message. First touch, last acquisition, CRM origin, and account-specific variants are models. Current models use the new identity (`person.id` bigint, `transaction.id` UUID) and write `{prefix}_person` / `{prefix}_transaction` / stats.

`source_code_summary.origin_people` / `origin_revenue` (and other `origin_*` columns there) are the **legacy** origin implementation, computed against the old identity ([e9-person-id](../e9-person-id/SKILL.md#old-identity-model-person_metadata-person_id_int-timeline_v3-transaction_metadata)). Do not use them for current model work. Last-click `revenue` / `attributed_*` on that same table are attribution, not a model.

`ModelWorker` (alias `model`) loads a plugin, streams timeline rows, runs the plugin transforms, and writes **that plugin’s** tables. The stem is `metadata.prefix` and is the same on every account (like `{prefix}segment`).

| Plugin | `metadata.prefix` | Person table |
| --- | --- | --- |
| `@engine9/plugins/models/first_touch` | `model_first_touch` | `model_first_touch_person` |
| `@engine9/plugins/models/crm_origin` | `model_crm_origin` | `model_crm_origin_person` |
| `@engine9/plugins/models/last_acquisition` | `model_last_acquisition` | `model_last_acquisition_person` |
| `engine9-accounts/frakture/authentic/authentic_origin` | `model_authentic_origin` | `model_authentic_origin_person` |
| `engine9-accounts/frakture/authentic/authentic_v2` | `model_authentic_v2` | `model_authentic_v2_person` |
| `engine9-accounts/frakture/authentic/authentic_v2025` | `model_authentic_v2025` | `model_authentic_v2025_person` |

Suffixes: `_person`, `_transaction`, `_person_stats`, `_transaction_stats`.

## When to use this skill

- Authoring or testing a model plugin
- Running `ModelWorker.run` / `runPeople` / `runTransactions` / `summarizeSourceCodes` / `summarizePeople`
- Querying `{prefix}_person` / `{prefix}_transaction` / `{prefix}_*_stats` (current models)
- Comparing a current model to last-click attributed revenue on `source_code_summary` (`revenue` / attributed fields — never `origin_*`)
- Inspecting **legacy** model tables (`person_model_source_code_summary`, `timeline_v3_summary`, `transaction_model_source_code`) via `summarizePeopleLegacy`. Legacy `person_id_int` is generated from `person_metadata.person_id` and is **not** `person.id`. `timeline_v3_summary` names the type **`entry_type_label`**, not `entry_type`.

## Run

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
| `summarizePeopleLegacy({ person_ids / person_id_ints / emails })` | — | Legacy tables only. `emails` looks up `person_email` (current) and SHA-256 then email on `person_metadata.person_id`. **Not** `person.id` = `person_id_int` |
| `comparePeopleLegacy({ emails })` | — | Current vs legacy paired by email (hash first). Or pass `person_ids` + `legacy_person_ids` |
| `loadStats({ model })` | Rebuilds those stats tables | After a manual SQL edit |

`summarizeSourceCodes` / `loadStats` also accept `prefix: 'model_first_touch'`.

Common options: `model` (plugin path), `timeline` (`sql` / `.duckdb` / file), `person_ids` / `emails` / `search`, `output_filename`.

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

File-only tests do not install tables. `metadata.prefix` is still required (the path must compile).

## Timeline source

`openTimelineSource` is the only place that knows SQL vs DuckDB vs file. Pass `timeline`; get `{ stream, close, mode }`.

| `timeline` | Reads |
| --- | --- |
| omitted / `'sql'` | Account `timeline` + dictionary labels. Transaction runs join `transaction` on `timeline.id = transaction.id` |
| `*.duckdb` | Same SQL against that file |
| `.parquet` / `.csv` / `.json` | File rows, ordered by `person_id` |

Transaction identity is `transaction.id` (UUID).

## Output

**Files:** csv with `person_id`, `prefix`, `source_code`, `source_code_id`, `date_of_source`, `reason` (plus `transaction_id` / `amount` / `ts` on the transaction file).

**SQL** (`run` only) — one set of tables per model, named from `metadata.prefix`:

| Table | Grain |
| --- | --- |
| `{prefix}_person` | one row per `person_id` (bigint) |
| `{prefix}_transaction` | one row per `transaction_id` (UUID) |
| `{prefix}_person_stats` | people per `source_code_id` |
| `{prefix}_transaction_stats` | tx / revenue / refunds / unique people per `source_code_id` |

No shared `person_model` interface table. Join `source_code_dictionary` for the code string; join `transaction` on `id` for amount / `ts`.

Detail: [schema.md](schema.md).

## Plugin contract

`metadata.prefix` is required and must match `model_<name>` (letters, digits, underscores). That stem is the table name on every account.

```javascript
const metadata = { name: 'Transaction model: my_model', prefix: 'model_my_model', unique: true, version: '1.0.0' };
const model = { model_id: 9, label: 'My Model' }; // optional label only

async function transform({ batch }) {
  for (const row of batch) {
    row.source_code = /* pick */;
    row.date_of_source = /* ts */;
    row.reason = ['why'];
  }
}
```

Set `source_code`. Do not set `source_code_id` — the worker resolves it (`doNotUpsert`). Optional `transforms.transaction` sees `transaction.transaction_id` (UUID) and `transaction.person_model_*`.

Authoring: [writing-a-model.md](writing-a-model.md).

## Sample statistics

```javascript
const { person_stats, transaction_stats, tables } = await model.summarizeSourceCodes({
  model: '@engine9/plugins/models/first_touch'
});
```

## Inspect people (UI)

`summarizePeople` is a **read-only** payload for a console or debugger. It does not call person/transaction transforms and does not write tables. Require `person_ids`, `emails`, or `search` (same filters as `run`).

```javascript
const { person_ids, models, people } = await model.summarizePeople({
  emails: 'a@example.com,b@example.com'
});
```

Return shape (also on `summarizePeople.metadata.output`):

| Field | Meaning |
| --- | --- |
| `person_ids` | Resolved bigint ids (after optional `people_limit`) |
| `models[]` | Warehouse catalog: `{ prefix, person_table, transaction_table }` for every `model_*_person` / `model_*_transaction` that exists (stats tables are omitted). Empty if no model has been `run` yet |
| `people[]` | One object per requested id, even when that person has no timeline or model row |

Each `people[]` item:

| Field | Meaning |
| --- | --- |
| `person_id` | bigint |
| `timeline[]` | Warehouse `timeline` rows for that person, ordered by `ts`, plus `entry_type`. Joins: `source_code_summary` (or `source_code_dictionary`) for parsed labels; `input` (`input_name`, `input_type`, `input_remote_id`); `plugin` (`plugin_path`, `plugin_name`); `transaction` (`transaction_id`, `amount`, …) when the row is a gift |
| `models[prefix]` | Stored `{prefix}_person` (or `null`) and `{prefix}_transaction[]`. Same source-code labels as timeline. Transaction rows also carry `amount` / `transaction_ts` from `transaction` |

Timeline `source_code_id` is **as stored**. This method does not apply the run-time source-code override stream. Stored model `reason` / `date_of_source` are what `run` last wrote.

Full field list: [schema.md](schema.md#summarizepeople-ui-inspect).

## Legacy inspect (old identity)

Keep this out of the current `{prefix}_*` path. `workers/model/legacy.js` reads the old tables only. **`person_id_int` is not `person.id`.** `person_metadata` generates `person_id_int` from the legacy `person_id` string (hash, sometimes an email). Never join those integers to current `person.id`. Legacy timeline inspect reads **`timeline_v3_summary`**, where the type column is **`entry_type_label`** (current `timeline` / plugin summaries use `entry_type`). Legacy timeline inspect reads **`timeline_v3_summary`**, where the type column is **`entry_type_label`** (current `timeline` / plugin summaries use `entry_type`).

`emails` is the bridge: `person_email` → current `person.id`; SHA-256 of trimmed lowercase email (`email_hash_v1`) → `person_metadata.person_id`; if the hash misses, try the email string on that column.

```javascript
const legacy = await model.summarizePeopleLegacy({
  person_ids: '1d0cbfeb8a0d6e5606317f9493460d1fbcd4b531f583eba01ba5c7eed9e2e292'
});
const { current, legacy, comparison, email_map } = await model.comparePeopleLegacy({
  emails: 'user@example.com'
});
```

`model_id` labels match the console CASE: 1 First Touch, 2 CRM Origin, 4 Authentic First Touch, 8 Last Channel Acquisition, 9 Authentic V2; 10 (Authentic V2025) falls through to the raw id. Person `match` is source_code equality (empty ≡ missing). Missing legacy tables are skipped.

```sql
SELECT d.source_code, s.person_count
FROM model_first_touch_person_stats s
JOIN source_code_dictionary d ON d.source_code_id = s.source_code_id
ORDER BY s.person_count DESC
LIMIT 20;

SELECT
  d.source_code,
  tms.revenue AS model_revenue,
  scs.revenue AS last_click_revenue
FROM model_first_touch_transaction_stats tms
JOIN source_code_dictionary d ON d.source_code_id = tms.source_code_id
LEFT JOIN source_code_summary scs ON scs.source_code_id = tms.source_code_id
ORDER BY tms.revenue DESC
LIMIT 20;

SELECT d.source_code, COUNT(*) AS transactions, SUM(t.amount) AS revenue
FROM model_first_touch_transaction tm
JOIN transaction t ON t.id = tm.transaction_id
JOIN source_code_dictionary d ON d.source_code_id = tm.source_code_id
GROUP BY d.source_code
ORDER BY revenue DESC
LIMIT 20;
```

## Additional resources

- Table DDL: [schema.md](schema.md)
- Writing / testing: [writing-a-model.md](writing-a-model.md)
- Last-click attribution (transaction ↔ message): [e9-source-code](../e9-source-code/SKILL.md)
