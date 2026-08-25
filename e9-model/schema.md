# Per-model result tables

Read [SKILL.md](SKILL.md) first. There is no `@engine9/interfaces/model`. Each plugin’s `metadata.prefix` is the stem (`model_first_touch` → `model_first_touch_person`). Same suffix set on every account.

These tables are the **current** identity-aware model output (`person_id` bigint, `transaction_id` UUID). They are not `source_code_summary.origin_*` — those columns are the legacy origin implementation (old identity).

`ModelWorker.run` deploys these via SchemaWorker (`prefix: false` on each table so PluginWorker’s install counter is not applied).

## `{prefix}_person`

One model source code per person.

| Column | Type |
| --- | --- |
| `person_id` | `person_id` (bigint), unique |
| `source_code_id` | `source_code_id` |
| `date_of_source` | datetime |
| `reason` | text |

## `{prefix}_transaction`

One model source code per transaction.

| Column | Type |
| --- | --- |
| `transaction_id` | UUID = `transaction.id`, unique |
| `person_id` | bigint |
| `source_code_id` | `source_code_id` |
| `date_of_source` | datetime |
| `reason` | text |

Amount / `ts` / recurring fields stay on `transaction`.

## Stats

`{prefix}_person_stats`: `source_code_id` → `person_count`.

`{prefix}_transaction_stats`: `source_code_id` → `transactions`, `revenue`, `refund_count`, `refund_amount`, `transaction_unique_person`.

## Joins

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

**Do not join `person_id_int` to `person.id`.** `person_metadata` generates `person_id_int` from the legacy `person_id` string. Current models use bigint `person.id`. Pair via `emails` (`person_email` + SHA-256 on `person_metadata.person_id`, email string as fallback) or explicit `person_ids` + `legacy_person_ids`.

`model_id` 1/2/4/8/9/10 map to `model_first_touch` / `model_crm_origin` / `model_authentic_origin` / `model_last_acquisition` / `model_authentic_v2` / `model_authentic_v2025`. The console CASE leaves 10 as the raw id.
