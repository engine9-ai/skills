# Inventory examples

## CLI

```bash
# Default warehouse inventory (no definition_path)
e9 inventoryworker inventory -a liftoff

# Full report with a bundle definition
e9 inventoryworker inventory -a liftoff \
  --definition_path=engine9-accounts/frakture/liftoff/hockeystick/export

# Plan only — skip monthly statistics (faster)
e9 inventoryworker inventory -a liftoff --statistics=false

# Explicit table list only (defaults not applied)
e9 inventoryworker inventory -a liftoff --tables=person,transaction
```

Read the full report from `options_filename` in the command output:

```bash
e9 fileworker json -a liftoff --filename=/path/from/options_filename
```

## Summary return value

```json
{
  "definition_path": "engine9-accounts/frakture/liftoff/hockeystick/export",
  "plugin_path": "engine9-accounts/frakture/liftoff/hockeystick/export",
  "format_version": 2,
  "table_count": 12,
  "file_count": 14,
  "directory_count": 14,
  "table_records": 125000,
  "file_records": 45000,
  "records": 170000,
  "statistics": {
    "month_range": { "min": "2020-04", "max": "2026-09" },
    "input_records": 890000,
    "message_records": 1200
  },
  "tables": [
    { "table": "person", "records": 50000 },
    { "table": "transaction", "records": 75000 }
  ],
  "directories": [
    {
      "input_id": "c8ec58a6-a9d7-5277-a12a-030cc01d037f",
      "input_type": "message",
      "file_count": 1,
      "records": 2
    }
  ],
  "options_filename": "/stored_inputs/liftoff/temp/2026-09-02/….inventory.json"
}
```

## Plan section (excerpt)

Tables omit empty `transforms`. Files carry export paths and source store roots.

```json
{
  "format_version": 2,
  "definition_path": "engine9-accounts/frakture/liftoff/hockeystick/export",
  "plugin_path": "engine9-accounts/frakture/liftoff/hockeystick/export",
  "table_records": 125000,
  "file_records": 45000,
  "records": 170000,
  "tables": [
    {
      "table": "person",
      "relative_path": "tables/person.parquet",
      "records": 50000
    },
    {
      "table": "transaction",
      "relative_path": "tables/transaction.parquet",
      "records": 75000
    }
  ],
  "skipped_tables": [
    { "table": "global_message_summary_by_date", "reason": "does_not_exist", "records": 0 }
  ],
  "files": [
    {
      "input_id": "c8ec58a6-a9d7-5277-a12a-030cc01d037f",
      "input_type": "message",
      "name": "email-sends.idv1.parquet",
      "records": 2,
      "relative_path": "message/c8ec58a6-a9d7-5277-a12a-030cc01d037f/email-sends.idv1.parquet",
      "source_directory": "/stored_inputs/liftoff/message/c8ec/…"
    }
  ],
  "directories": [
    {
      "input_id": "c8ec58a6-a9d7-5277-a12a-030cc01d037f",
      "input_type": "message",
      "records": 2,
      "files": [
        {
          "name": "email-sends.idv1.parquet",
          "records": 2,
          "relative_path": "message/c8ec58a6-a9d7-5277-a12a-030cc01d037f/email-sends.idv1.parquet"
        }
      ]
    }
  ]
}
```

## Statistics section (excerpt)

Each month with data gets a bucket. Use `month_range` for timeline axis bounds.

```json
{
  "statistics": {
    "version": 1,
    "generated_at": "2026-09-02T12:00:00.000Z",
    "month_range": { "min": "2023-06", "max": "2026-09" },
    "inputs": {
      "by_plugin_entry_type_month": [
        {
          "plugin_id": "a1b2c3d4-…",
          "plugin_name": "@frakture-com/plugins/Switchboard",
          "entry_type": "EMAIL_SEND",
          "month": "2024-01",
          "records": 152340
        },
        {
          "plugin_id": "a1b2c3d4-…",
          "plugin_name": "@frakture-com/plugins/Switchboard",
          "entry_type": "EMAIL_OPEN",
          "month": "2024-01",
          "records": 42100
        }
      ],
      "records": 890000,
      "scanned_inputs": 42,
      "skipped_inputs": 1
    },
    "tables": [
      {
        "table": "transaction",
        "date_column": "ts",
        "records": 75000,
        "months": [
          { "month": "2023-06", "records": 1200 },
          { "month": "2023-07", "records": 980 },
          { "month": "2024-01", "records": 2100 }
        ],
        "by_plugin_month": [
          {
            "plugin_id": "…",
            "plugin_name": "@frakture-com/plugins/ActBlue",
            "month": "2023-06",
            "records": 800
          },
          {
            "plugin_id": "…",
            "plugin_name": "@frakture-com/plugins/Stripe",
            "month": "2024-01",
            "records": 2100
          }
        ]
      },
      {
        "table": "person",
        "date_column": "frakture_date_created",
        "records": 50000,
        "months": [
          { "month": "2020-04", "records": 500 },
          { "month": "2024-03", "records": 12000 }
        ],
        "by_plugin_month": [
          {
            "plugin_id": "…",
            "plugin_name": "@frakture-com/plugins/Switchboard",
            "month": "2024-03",
            "records": 8400
          }
        ]
      }
    ],
    "messages": {
      "table": "message",
      "records": 1200,
      "by_plugin_submodule_month": [
        {
          "plugin_id": "a1b2c3d4-…",
          "plugin_name": "@frakture-com/plugins/Switchboard",
          "submodule": "Messages",
          "month": "2024-01",
          "records": 85
        }
      ]
    },
    "message_summary_by_date": {
      "table": "global_message_summary_by_date",
      "filter": "`spend` > 0 or `impressions` > 0",
      "records": 400,
      "by_plugin_submodule_month": [
        {
          "plugin_id": "fb-plugin-id",
          "plugin_name": "@frakture-com/plugins/Facebook",
          "submodule": "Ads",
          "month": "2024-02",
          "records": 28
        }
      ]
    }
  }
}
```

## Mapping statistics to a warehouse timeline

| UI row (example) | Statistics source |
|------------------|-------------------|
| Email → Sends | `inputs.by_plugin_entry_type_month` where `entry_type` is `EMAIL_SEND` |
| Email → Opens | same, `EMAIL_OPEN` |
| Transactions → plugin | `tables[]` where `table === 'transaction'` → `by_plugin_month` (`input_id` → `input.plugin_id`) |
| People → Distinct | `tables[]` where `table === 'person'` → overall `months` / `records` |
| People → platform | same table → `by_plugin_month` from `person_remote` ⨝ `input` (`count(distinct person_id)`) |
| Legacy imports | `inputs` grouped by `plugin_name` |
| Message sends | `messages.by_plugin_submodule_month` |
| Ad spend / impressions | `message_summary_by_date.by_plugin_submodule_month` |

## Bundle export `inventory.json5`

During `e9 exportworker export`, the written `inventory.json5` contains the **plan** only (`statistics` omitted). Collect statistics separately:

```bash
e9 inventoryworker inventory -a liftoff \
  --definition_path=engine9-accounts/frakture/liftoff/hockeystick/export
```

## Programmatic use (server)

```javascript
import { buildInventoryReport } from '../utilities/inventoryReport.js';

const report = await buildInventoryReport(worker, {
  definition_path: 'engine9-accounts/frakture/liftoff/hockeystick/export',
  statistics: true
});
// report.statistics.inputs.by_plugin_entry_type_month
// report.tables — export plan
```

For plan-only (mirrors export bundle):

```javascript
const plan = await buildInventoryReport(worker, {
  definition_path: '…',
  statistics: false
});
```
