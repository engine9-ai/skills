# Per-model result tables

Read [SKILL.md](SKILL.md) for concepts and [developers.md](developers.md) for the API. There is no `@engine9/interfaces/model`. Each plugin’s `metadata.prefix` is the stem (`model_first_touch` → `model_first_touch_person`). Same suffix set on every account.

These tables are the **current** identity-aware model output (`person_id` bigint, `transaction_id` UUID). They are not `source_code_summary.origin_*` — those columns are the legacy origin implementation (old identity).

`ModelWorker.run` deploys them via SchemaWorker (`prefix: false` on each table so PluginWorker’s install counter is not applied). DDL source: `server/workers/model/schema.js` (`modelResultTables`). Stats are rebuilt from the detail tables by `replaceModelStats` in `server/workers/model/output.js`.

## Shipped prefixes

| Plugin | `metadata.prefix` | Tables created |
| --- | --- | --- |
| `@engine9/plugins/models/first_touch` | `model_first_touch` | `model_first_touch_{person,transaction,person_stats,transaction_stats}` |
| `@engine9/plugins/models/crm_origin` | `model_crm_origin` | `model_crm_origin_{person,transaction,person_stats,transaction_stats}` |
| `@engine9/plugins/models/last_acquisition` | `model_last_acquisition` | `model_last_acquisition_{person,transaction,person_stats,transaction_stats}` |

Account-specific models use the same `model_<name>` prefix rule and the same four suffixes. Tables exist only after that model has been `run` on the account — probe before joining.

## Four tables per prefix

| Table | Grain | Built when |
| --- | --- | --- |
| `{prefix}_person` | one row per `person_id` | `run` (person transform) |
| `{prefix}_transaction` | one row per `transaction_id` | `run` (transaction transform or person inheritance) |
| `{prefix}_person_stats` | one row per `source_code_id` | `run` / `loadStats` — `COUNT(*)` from `{prefix}_person` |
| `{prefix}_transaction_stats` | one row per `source_code_id` | `run` / `loadStats` — rollup of `{prefix}_transaction` (+ `transaction.amount` / refunds when present) |

No shared cross-model table. Join `source_code_dictionary` (or `source_code_summary`) for the code string; join `transaction` on `id = transaction_id` for amount / `ts`.

## `{prefix}_person`

One model source code per person. Unique on `person_id`; indexed on `source_code_id`.

| Column | Type | Notes |
| --- | --- | --- |
| `person_id` | `person_id` (bigint) | = `person.id`, unique |
| `source_code_id` | `source_code_id` | chosen entry’s code |
| `date_of_source` | datetime | chosen entry’s effective date (`timeline.ts`) |
| `reason` | text, nullable | why the rule chose this code |
| `created_at` | datetime | row write time |
| `modified_at` | datetime | row write time |

## `{prefix}_transaction`

One model source code per transaction. Unique on `transaction_id`; indexed on `person_id` and `source_code_id`.

| Column | Type | Notes |
| --- | --- | --- |
| `transaction_id` | UUID | = `transaction.id` (same as `timeline.id` for transaction entries) |
| `person_id` | `person_id` (bigint) | owner of the transaction |
| `source_code_id` | `source_code_id` | credit for this transaction (often inherited from the person row) |
| `date_of_source` | datetime | date of the credited entry |
| `reason` | text, nullable | why this code was chosen |
| `created_at` | datetime | row write time |
| `modified_at` | datetime | row write time |

Amount / `ts` / recurring fields stay on `transaction` — not duplicated here.

## `{prefix}_person_stats`

People acquired per source code. Unique on `source_code_id`.

| Column | Type | Notes |
| --- | --- | --- |
| `source_code_id` | `source_code_id` | unique |
| `person_count` | int | `COUNT(*)` from `{prefix}_person` for this code |
| `created_at` | datetime | stats rebuild time |
| `modified_at` | datetime | stats rebuild time |

Closest conceptual replacement for legacy `source_code_summary.origin_people` (when using first-touch / CRM-origin models), but current-identity and per-model — do not overwrite `origin_*` from these without an explicit product decision.

## `{prefix}_transaction_stats`

Lifetime giving of people (or transactions) credited to each source code. Unique on `source_code_id`.

| Column | Type | Notes |
| --- | --- | --- |
| `source_code_id` | `source_code_id` | unique |
| `transactions` | int | row count in `{prefix}_transaction` for this code |
| `revenue` | currency | `SUM(transaction.amount)` for those rows (0 if `transaction` is missing) |
| `refund_count` | int | rows with non-zero `refund_amount` |
| `refund_amount` | currency | `SUM(transaction.refund_amount)` |
| `transaction_unique_person` | int | `COUNT(DISTINCT person_id)` among credited transactions |
| `created_at` | datetime | stats rebuild time |
| `modified_at` | datetime | stats rebuild time |

Closest conceptual replacement for legacy `source_code_summary.origin_revenue` (and related origin transaction rollups). Again: per-model, current identity, optional when the tables exist.

## Optional use from `source_code_summary`

`source_code_summary` remains the hub for **attribution** (last-click `revenue` / transactions / spend) and dictionary labels. Model LTV / acquisition counts live in `{prefix}_*_stats`. When enriching or reading summary rows, join stats **only if the tables exist** on the account (models are not live and may not have been run).

Join key: `source_code_id`. Prefer the **stats** tables for per-code rollups — do not re-aggregate `{prefix}_person` / `{prefix}_transaction` inside the summary builder unless you need a metric stats does not store.

```sql
-- Example: first-touch model people + revenue beside attributed revenue
SELECT
  scs.source_code_id,
  scs.source_code,
  scs.revenue AS attributed_revenue,
  ft_p.person_count AS model_first_touch_person_count,
  ft_t.revenue AS model_first_touch_revenue,
  ft_t.transactions AS model_first_touch_transactions
FROM source_code_summary scs
LEFT JOIN model_first_touch_person_stats ft_p
  ON ft_p.source_code_id = scs.source_code_id
LEFT JOIN model_first_touch_transaction_stats ft_t
  ON ft_t.source_code_id = scs.source_code_id;
```

Rules for that work:

- Probe table presence (or catch missing-table) per prefix; skip missing models.
- Never treat model revenue as a substitute for attributed `scs.revenue` — different question ([SKILL.md §3](SKILL.md#3-attribution-is-not-a-model)).
- Do not write into legacy `origin_*` from current `{prefix}_*` unless product explicitly maps one model (usually first touch or CRM origin) into those columns for backward compatibility.
- After a manual SQL edit of detail tables, call `ModelWorker.loadStats({ model })` (or `prefix`) before trusting stats joins.

## Joins (detail grain)

```sql
SELECT p.person_id, d.source_code, p.date_of_source, p.reason
FROM model_first_touch_person p
JOIN source_code_dictionary d ON d.source_code_id = p.source_code_id
LIMIT 50;

SELECT tm.transaction_id, t.ts, t.amount, d.source_code
FROM model_first_touch_transaction tm
JOIN transaction t ON t.id = tm.transaction_id
JOIN source_code_dictionary d ON d.source_code_id = tm.source_code_id
ORDER BY t.ts DESC
LIMIT 50;
```

## summarizePeople (UI inspect)

`ModelWorker.summarizePeople({ person_ids | emails | search })` is the contract a user interface should consume to explain how stored models scored a person. It **reads** warehouse tables only — it does not run transforms.

```javascript
{
  person_ids: [123],
  models: [
    {
      prefix: 'model_first_touch',
      person_table: 'model_first_touch_person',
      transaction_table: 'model_first_touch_transaction'
    }
  ],
  people: [
    {
      person_id: 123,
      timeline: [
        {
          id: '…',
          ts: '2020-01-01T00:00:00.000Z',
          person_id: 123,
          input_id: '…',
          entry_type_id: 10,
          entry_type: 'TRANSACTION',
          source_code_id: 9,
          source_code: 'EM_FR_20200101_appeal',
          campaign: 'appeal',
          input_remote_id: '…',
          input_name: 'January appeal',
          input_type: 'message',
          plugin_id: '…',
          plugin_path: '@engine9/plugins/…',
          plugin_name: '…',
          transaction_id: '…',
          amount: 25
        }
      ],
      models: {
        model_first_touch: {
          person: {
            prefix: 'model_first_touch',
            person_id: 123,
            source_code_id: 9,
            source_code: 'EM_FR_20200101_appeal',
            date_of_source: '2020-01-01T00:00:00.000Z',
            reason: 'earliest timeline entry'
          },
          transactions: [
            {
              prefix: 'model_first_touch',
              transaction_id: '…',
              person_id: 123,
              source_code_id: 9,
              source_code: 'EM_FR_20200101_appeal',
              date_of_source: '2020-01-01T00:00:00.000Z',
              reason: '',
              amount: 25,
              transaction_ts: '2020-02-01T00:00:00.000Z'
            }
          ]
        }
      }
    }
  ]
}
```

`models` lists every `model_*_person` / `model_*_transaction` table that exists (stats suffixes are ignored). Each `people[].models[prefix]` key matches that catalog so the UI can render a column per model even when `person` is `null`.

Timeline context joins: `timeline.source_code_id` → `source_code_summary` (fallback `source_code_dictionary`); `timeline.input_id` → `input`; `input.plugin_id` → `plugin`; `timeline.id` → `transaction.id` for gift fields. Dictionary labels on model rows use the same `source_code_summary` join.

## Legacy inspect

`summarizePeopleLegacy` / `comparePeopleLegacy` live in `server/workers/model/legacy.js`. They read the old identity tables (`person_model_source_code_summary`, `timeline_v3_summary`, `transaction_model_source_code`, `person_metadata`, `transaction_metadata`) and do not write `{prefix}_*`.

Person inspect of the legacy activity log uses **`timeline_v3_summary`**, not base `timeline_v3` and not current `timeline`. The type column on that view is **`entry_type_label`**, not `entry_type`. Current `timeline` stores `entry_type_id`; current plugin `*_summary` views add string `entry_type`. Do not SELECT `entry_type` from `timeline_v3_summary`.

**Do not join `person_id_int` to `person.id`.** `person_metadata` generates `person_id_int` from the legacy `person_id` string. Current models use bigint `person.id`. Pair via `emails` (`person_email` + SHA-256 on `person_metadata.person_id`, email string as fallback) or explicit `person_ids` + `legacy_person_ids`.

Legacy `model_id` 1, 2, and 8 map to `model_first_touch`, `model_crm_origin`, and `model_last_acquisition`. Account-specific model ids the console CASE does not name fall through to the raw id.

## inspectPerson (conductor tables)

`ModelWorker.inspectPerson` (`server/workers/model/inspect.js`) is **current-identity only** (`timeline`, `model_*_person`). MCP tool **`timelinePerson`**. Timeline rows add **`effective_date`** = `timeline.ts` (resolved at load time — see [SKILL.md §5](SKILL.md#5-the-effective-date-of-an-entry)). Pass `legacy: true` to also load `timeline_v3_summary` / `person_model_source_code` from `legacy.js` (opt-in; not in future deployments). Missing tables are `skipped`. Emails bridge identity; `person.id` is never joined to `person_id_int`. Conductor currently sets `TIMELINE_PERSON_INCLUDE_LEGACY = true` in one place. Returns top-level **`sql`**: `[{ id, sql, error, table? }]` for every statement this request ran.

## compareSourceCodes (all current models)

`ModelWorker.compareSourceCodes` (`server/workers/model/compare.js`) lists every `model_*_stats` table and returns one row per source code with that model's person_count / revenue / transactions. MCP **`timelinePerson` `command: compareSourceCodes`**. Omit `source_codes` to union each model's top 10 by people and by revenue. When both the current first-touch model and legacy first touch are deployed, also unions the top 10 codes by absolute person_count difference (`top` key `first_touch_vs_legacy`). Pass `legacy: true` to also include `transaction_model_pivot` stems (opt-in; conductor currently sets `MODEL_COMPARE_INCLUDE_LEGACY = true`). Custom legacy models are extra `{stem}_*` columns on that table and are included only when present. Returns top-level **`sql`** for the top-N selection queries and each model's stats SELECT.

## transaction_model_pivot vs model_*_stats

`compareSourceCodesLegacy` / `summarizeSourceCodesLegacy` (`server/workers/model/compare.js`) are **opt-in** same-stem reads of the legacy pivot vs current `{prefix}_*_stats`. They are not the `/models` artifact. Shipped pivot stems: `first_touch`, `crm_origin`, `last_acquisition`. Some accounts also have **custom legacy models** as additional `{stem}_*` columns; those are discovered from the table and omitted when absent. Shared metrics: `person_count`, `transactions`, `revenue`, `refund_count`, `refund_amount`, `transaction_unique_person`. Pivot also has `incipient_*` (legacy only). `source_codes` is required; tokens with `%` use `LIKE`. Future deployments will drop the pivot table.
