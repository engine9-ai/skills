# Inventory examples

## CLI

```bash
# Read the cached inventory (does not generate)
e9 inventoryworker inventory -a <account_id>

# Build or refresh {account root}/cache/inventory.json
e9 inventoryworker buildInventoryReport -a <account_id>

# Full report with a bundle definition
e9 inventoryworker buildInventoryReport -a <account_id> \
  --definition_path=engine9-accounts/<org>/<account>/export

# Plan only — skip monthly statistics (faster)
e9 inventoryworker buildInventoryReport -a <account_id> --statistics=false

# Explicit table list only (defaults not applied)
e9 inventoryworker buildInventoryReport -a <account_id> --tables=person,transaction
```

Read the full report from `inventory_path` / `options_filename` (same path when ready):

```bash
e9 fileworker json -a <account_id> --filename=/path/from/inventory_path
```

## Summary return value

```json
{
  "ready": true,
  "definition_path": "engine9-accounts/<org>/<account>/export",
  "plugin_path": "engine9-accounts/<org>/<account>/export",
  "format_version": 2,
  "table_count": 12,
  "file_count": 14,
  "directory_count": 14,
  "table_records": 125000,
  "file_records": 45000,
  "records": 170000,
  "options_filename": "<store>/<account_id>/cache/inventory.json",
  "inventory_path": "<store>/<account_id>/cache/inventory.json",
  "cached_at": "2026-09-08T12:00:00.000Z",
  "statistics": {
    "version": 1,
    "month_range": { "min": "2020-04", "max": "2026-09" },
    "inputs": { "records": 890000, "by_plugin_entry_type_month": ["…"] },
    "tables": ["…"]
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
  ]
}
```

## Plan section (excerpt)

Tables omit empty `transforms`. Files carry export paths and source store roots.

```json
{
  "format_version": 2,
  "definition_path": "engine9-accounts/<org>/<account>/export",
  "plugin_path": "engine9-accounts/<org>/<account>/export",
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
      "source_directory": "/stored_inputs/<account_id>/message/c8ec/…"
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
          "channel": "email",
          "month": "2024-01",
          "records": 152340
        },
        {
          "plugin_id": "a1b2c3d4-…",
          "plugin_name": "@frakture-com/plugins/Switchboard",
          "entry_type": "EMAIL_OPEN",
          "channel": "email",
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
        "revenue": 217700,
        "months": [
          { "month": "2023-06", "records": 1200, "revenue": 48000 },
          { "month": "2023-07", "records": 980, "revenue": 41200 },
          { "month": "2024-01", "records": 2100, "revenue": 128450 }
        ],
        "sql": "select DATE_FORMAT(...) as month, count(*) as records, sum(coalesce(`amount`, 0)) as revenue from transaction where `ts` is not null group by 1 order by 1",
        "by_plugin_month": [
          {
            "plugin_id": "…",
            "plugin_name": "@frakture-com/plugins/ActBlue",
            "month": "2023-06",
            "records": 800,
            "revenue": 32000
          },
          {
            "plugin_id": "…",
            "plugin_name": "@frakture-com/plugins/Stripe",
            "month": "2024-01",
            "records": 2100,
            "revenue": 128450
          }
        ],
        "by_plugin_month_sql": "select i.plugin_id as plugin_id, p.path as plugin_name, ... from transaction t left join input i ... left join plugin p ..."
      },
      {
        "table": "person",
        "date_column": "frakture_date_created",
        "records": 50000,
        "months": [
          { "month": "2020-04", "records": 500 },
          { "month": "2024-03", "records": 12000 }
        ],
        "sql": "select DATE_FORMAT(...) as month, count(*) as records from person where ...",
        "by_plugin_month": [
          {
            "plugin_id": "…",
            "plugin_name": "@frakture-com/plugins/Switchboard",
            "month": "2024-03",
            "records": 8400
          }
        ],
        "by_plugin_month_sql": "select i.plugin_id ..., count(distinct pr.person_id) ... from person_remote pr join input i ... left join plugin p ...",
        "by_plugin_month_date_column": "modified_at"
      },
      {
        "table": "timeline",
        "date_column": "ts",
        "records": 890000,
        "months": [
          { "month": "2023-12", "records": 40000, "people": 45900 },
          { "month": "2024-01", "records": 52000, "people": 48210 }
        ],
        "sql": "select DATE_FORMAT(...) as month, count(*) as records, count(distinct `person_id`) as people from timeline where `ts` is not null group by 1 order by 1"
      }
    ],
    "messages": {
      "table": "global_message_summary",
      "date_column": "publish_date",
      "records": 1200,
      "sql": "select `bot_id` as plugin_id, `bot_path` as plugin_name, `submodule`, `channel` as channel, ... count(*) as records, sum(coalesce(m.`sent`, 0)) as sent, sum(coalesce(m.`impressions`, 0)) as impressions, sum(coalesce(m.`clicks`, 0)) as clicks ... from global_message_summary m where m.`publish_date` > '2000-01-01' group by 1, 2, 3, 4, 5",
      "by_plugin_submodule_month": [
        {
          "plugin_id": "a1b2c3d4-…",
          "plugin_name": "@frakture-com/plugins/Switchboard",
          "submodule": "Messages",
          "channel": "email",
          "month": "2024-01",
          "records": 85,
          "sent": 412880,
          "impressions": 98000,
          "clicks": 12000
        }
      ],
      "by_channel_month": [
        {
          "channel": "email",
          "month": "2024-01",
          "records": 85,
          "sent": 412880,
          "impressions": 98000,
          "clicks": 12000
        }
      ]
    },
    "message_summary_by_date": {
      "table": "global_message_summary_by_date",
      "date_column": "date",
      "filter": "`spend` > 0",
      "count": "count(distinct `message_id`)",
      "records": 40,
      "sql": "select `bot_id` as plugin_id, ... `channel` as channel, ... count(distinct `message_id`) ... where `date` is not null and (`spend` > 0) ...",
      "by_plugin_submodule_month": [
        {
          "plugin_id": "fb-plugin-id",
          "plugin_name": "@frakture-com/plugins/Facebook",
          "submodule": "Ads",
          "channel": "facebook",
          "month": "2024-02",
          "records": 28
        }
      ],
      "by_channel_month": [
        {
          "channel": "facebook",
          "month": "2024-02",
          "records": 28
        }
      ]
    },
    "message_activity": {
      "table": "global_message_summary_by_date",
      "date_column": "date",
      "count": "count(distinct `message_id`)",
      "records": 16,
      "sql": "select `bot_id` as plugin_id, ... `channel` as channel, ... count(distinct `message_id`) as records, sum(coalesce(`sent`, 0)) as sent, sum(coalesce(`impressions`, 0)) as impressions, sum(coalesce(`clicks`, 0)) as clicks ... from global_message_summary_by_date where `date` is not null group by 1, 2, 3, 4, 5",
      "by_plugin_submodule_month": [
        {
          "plugin_id": "a1b2c3d4-…",
          "plugin_name": "@frakture-com/plugins/Switchboard",
          "submodule": "Messages",
          "channel": "email",
          "month": "2024-01",
          "records": 12,
          "sent": 18000,
          "impressions": 6200,
          "clicks": 900
        },
        {
          "plugin_id": "sms-plugin-id",
          "plugin_name": "@frakture-com/plugins/Tatango",
          "submodule": "Messages",
          "channel": "sms",
          "month": "2024-01",
          "records": 4,
          "sent": 3200,
          "impressions": 0,
          "clicks": 180
        }
      ],
      "by_channel_month": [
        {
          "channel": "email",
          "month": "2024-01",
          "records": 12,
          "sent": 18000,
          "impressions": 6200,
          "clicks": 900
        },
        {
          "channel": "sms",
          "month": "2024-01",
          "records": 4,
          "sent": 3200,
          "impressions": 0,
          "clicks": 180
        }
      ]
    }
  }
}
```

## Mapping statistics to a warehouse timeline

| UI row (big 3) | Statistics source |
|----------------|-------------------|
| People → Distinct | `tables[]` where `table === 'person'` → overall `months` / `records` |
| People → platform | same table → `by_plugin_month` from `person_remote` ⨝ `input` (`count(distinct person_id)`) |
| Messages → plugin → published | `messages.by_plugin_submodule_month` (`global_message_summary` / `publish_date`; includes `channel`) |
| Messages → plugin → Active ads | `message_summary_by_date.by_plugin_submodule_month` (`spend > 0` on `date`) |
| Messages → plugin → entry type | `inputs.by_plugin_entry_type_month` (`EMAIL_SEND`, `EMAIL_OPEN`, …; `channel` from prefix) |
| Transactions → plugin | `tables[]` where `table === 'transaction'` → `by_plugin_month` (`input_id` → `input.plugin_id`; includes `revenue`) |

## Mapping statistics to Home

| Home surface | Statistics source |
|--------------|-------------------|
| Total revenue | `transaction` `months[].revenue` vs previous month |
| Donations | `transaction` `months[].records` |
| Active people | `timeline` `months[].people` |
| Emails sent / SMS sent | `message_activity.by_channel_month` filtered by `channel`, field `sent` |
| Channel activity (sends, opens, clicks) | `message_activity.by_channel_month`: `sent`, `impressions`, `clicks` |

Fallback if `message_activity` is skipped: `messages.by_channel_month`, then `inputs` entry types. Ad coverage (not Home activity) stays on `message_summary_by_date`.

## Bundle export `inventory.json5`

During `e9 exportworker export`, the written `inventory.json5` contains the **plan** only (`statistics` omitted). Collect statistics into the account cache separately:

```bash
e9 inventoryworker buildInventoryReport -a <account_id> \
  --definition_path=engine9-accounts/<org>/<account>/export
```

## Programmatic use (server)

```javascript
import { buildInventoryReport } from '../utilities/inventoryReport.js';

const report = await buildInventoryReport(worker, {
  definition_path: 'engine9-accounts/<org>/<account>/export',
  statistics: true
});
// report.statistics.message_activity.by_channel_month  — sent / impressions / clicks
// report.statistics.tables.find(t => t.table === 'transaction').months  — revenue + records
// report.tables — export plan
```

For plan-only (mirrors export bundle):

```javascript
const plan = await buildInventoryReport(worker, {
  definition_path: '…',
  statistics: false
});
```
