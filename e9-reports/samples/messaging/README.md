# Sample messaging reports

These files are **portable report JSON** (`schema_version` 1). They are simplified versions of `@engine9/plugins/reports/messaging`, published in this public GitHub repo so a host can fetch them as a remote report URL. Use them to preview email, SMS, and ads/social dashboards, or as templates when authoring a hosted definition.

## Quick reference

Base URL (after these files are on `main`):

`https://raw.githubusercontent.com/engine9-ai/skills/main/e9-reports/samples/messaging/`

| File | Consumers | Developers | Remote URL |
| --- | --- | --- | --- |
| [email.json](email.json) | Email sends, opens, clicks, unsubscribes, bounces | `global_message_summary`, `channel='email'`, date = `publish_date` | [email.json](https://raw.githubusercontent.com/engine9-ai/skills/main/e9-reports/samples/messaging/email.json) |
| [email_transactions.json](email_transactions.json) | Email last-click revenue next to engagement | Same email slice; `attributed_revenue` / `attributed_transactions` | [email_transactions.json](https://raw.githubusercontent.com/engine9-ai/skills/main/e9-reports/samples/messaging/email_transactions.json) |
| [sms.json](sms.json) | SMS sends, clicks, click rate, bounces | `global_message_summary`, `channel='sms'`, date = `publish_date` | [sms.json](https://raw.githubusercontent.com/engine9-ai/skills/main/e9-reports/samples/messaging/sms.json) |
| [sms_fundraising.json](sms_fundraising.json) | SMS engagement plus last-click revenue | Same SMS slice; `attributed_*` | [sms_fundraising.json](https://raw.githubusercontent.com/engine9-ai/skills/main/e9-reports/samples/messaging/sms_fundraising.json) |
| [ads_social_persuasion.json](ads_social_persuasion.json) | Ad/social impressions, clicks, CTR, spend | `global_message_summary_by_date`, date = `date`; optional `channel` | [ads_social_persuasion.json](https://raw.githubusercontent.com/engine9-ai/skills/main/e9-reports/samples/messaging/ads_social_persuasion.json) |
| [ads_social_fundraising.json](ads_social_fundraising.json) | Spend vs attributed revenue, ROI, transactions | Same by-date view; default channels include `bing` | [ads_social_fundraising.json](https://raw.githubusercontent.com/engine9-ai/skills/main/e9-reports/samples/messaging/ads_social_fundraising.json) |

Conductor:

```
/report https://raw.githubusercontent.com/engine9-ai/skills/main/e9-reports/samples/messaging/email.json
```

Rule: Use the `raw.githubusercontent.com` URL (JSON), not the GitHub `blob` HTML page.

Rule: The artifact host fetches the definition. MCP does not fetch URLs. SQL stays in `ReportWorker`.

## Concepts

### Consumers and developers

Each sample `description` is written for both audiences: what the dashboard shows, and which warehouse view and filters run it. The tables below repeat that split so you can pick a file without opening the JSON.

### Samples vs installed plugins

| | This folder | Installed plugin |
| --- | --- | --- |
| Shape | Portable JSON, no functions | JS modules exported as `reports.<key>` |
| How to open | Remote URL → host `fetch` → `run` with `definition` | `path` such as `@engine9/plugins/reports/messaging:reports:email` |
| Layout | Three sections: KPIs, one chart, one table | Full dashboards (campaign rollups, extra charts, plugin nickname) |
| Catalog | Not listed by `report` `command: list` | Listed when the plugin is installed |

Rule: These samples do not replace `@engine9/plugins/reports/messaging`. Install the plugin for the full account catalog.

## File format

Each file is a report definition:

| Field | Role |
| --- | --- |
| `schema_version` | `1` (required for hosted `definition` runs) |
| `name` | Title in the report header |
| `description` | Subtitle for consumers and developers |
| `tags` | Includes `Sample` plus channel tags |
| `data_sources.default` | Table, `date_column`, static `conditions` |
| `filters` | Extra run variables (ads/social `channel` only) |
| `sections` | `StatCard`, `ComposedChart`, and `Table` only |

Rule: Definitions are JSON only. Static predicates belong in `data_sources.conditions`.

`start` / `end` are added automatically because every sample sets `date_column`. Relative values such as `-3M` and `now` are valid.

## Workflow

### Open as a remote report URL

1. Copy a **raw** URL from the table above (the file must be on `main`).
2. In Conductor, run `/report <url>`.
3. Set `start` and `end` (for example `-3M` and `now`).

Simple `GET` is enough. Do not add custom headers that trigger a CORS preflight; GitHub raw allows `Access-Control-Allow-Origin: *` on GET but not on OPTIONS.

jsDelivr (optional CORS/CDN mirror):

`https://cdn.jsdelivr.net/gh/engine9-ai/skills@main/e9-reports/samples/messaging/email.json`

### Run from MCP

The host fetches the JSON, then:

```json
{
  "command": "run",
  "account_id": "<account_id>",
  "definition": {},
  "start": "-3M",
  "end": "now"
}
```

Replace `{}` with the file body. Do not pass `path` in the same call.

## Reports

### Email Engagement (`email.json`)

| Audience | Notes |
| --- | --- |
| Consumers | How many emails went out, how they were opened and clicked, unsubscribes, and bounces. The chart is volume over publish date; the table is one row per message subject. |
| Developers | `global_message_summary`, `channel='email' and publish_date is not null`. Opens are `sum(impressions)`. Filters: `start`, `end` on `publish_date`. |

Plugin counterpart: `@engine9/plugins/reports/messaging:reports:email`.

### Email Fundraising (`email_transactions.json`)

| Audience | Notes |
| --- | --- |
| Consumers | Last-click revenue and transaction count next to sends and rates. Use this when the question is “did this email raise money?”, not only “did people open it?”. |
| Developers | Same email slice as Email Engagement. KPIs use `attributed_revenue` and `attributed_transactions`. Do not add platform `revenue`. |

Plugin counterpart: `@engine9/plugins/reports/messaging:reports:email_transactions`.

### SMS Summary (`sms.json`)

| Audience | Notes |
| --- | --- |
| Consumers | Text-message volume, clicks, click rate, and bounces, with a per-message table. |
| Developers | `global_message_summary`, `channel='sms'`. Filters: `start`, `end` on `publish_date`. |

Plugin counterpart: `@engine9/plugins/reports/messaging:reports:sms`.

### SMS Fundraising (`sms_fundraising.json`)

| Audience | Notes |
| --- | --- |
| Consumers | SMS engagement plus attributed revenue and transactions. |
| Developers | Same SMS slice as SMS Summary. Fundraising metrics are `attributed_*`. |

Plugin counterpart: `@engine9/plugins/reports/messaging:reports:sms_fundraising`.

### Ads & Social Persuasion (`ads_social_persuasion.json`)

| Audience | Notes |
| --- | --- |
| Consumers | Impressions, clicks, CTR, and spend across ads and organic social. Optional channel filter. |
| Developers | `global_message_summary_by_date`, `date_column` `date` (stats day). Static `channel in (…)` matches the plugin default set. Optional run filter `channel` (`=`). |

Plugin counterpart: `@engine9/plugins/reports/messaging:reports:ads_social_persuasion`.

### Ads & Social Fundraising (`ads_social_fundraising.json`)

| Audience | Notes |
| --- | --- |
| Consumers | Spend versus attributed revenue, ROI, and transactions, broken down by channel. |
| Developers | Same by-date view; default channels add `bing`. ROI is `sum(attributed_revenue)/sum(spend)`. On `_by_date`, `date` is engagement day and/or transaction day — a day can have revenue with zero impressions. |

Plugin counterpart: `@engine9/plugins/reports/messaging:reports:ads_social_fundraising`.

## Rules

Rule: Prefer `attributed_revenue` / `attributed_transactions` for fundraising truth. Do not add those to platform `revenue` / `transactions`.

Rule: Email opens are `impressions` on `global_message_summary`.

Rule: Email and SMS samples filter `publish_date`. Ads/social samples filter `date` on `global_message_summary_by_date`.

Rule: `report` `command: list` stays installed plugins only. Hosted samples do not appear in that catalog.

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| Host returns HTML, not JSON | You used a `github.com/.../blob/...` URL. Switch to `raw.githubusercontent.com`. |
| CORS / failed fetch | Use a simple GET to the raw URL, or the jsDelivr mirror. CORS is the publisher’s problem; GitHub raw allows GET CORS but not preflight. |
| Empty widgets | The account may have no rows for that `channel` in the date range. Confirm `@engine9/interfaces/message` data is loaded. |
| `Unsupported report schema_version` | Keep `schema_version` at `1`. |
| `path or definition, not both` | Pass only `definition` when using a hosted file. |
| Numbers look like “native conversions” | Fundraising samples use `attributed_*`. Platform `revenue` is a different family. |

## Related documentation

- [engine9 reports](../../SKILL.md)
- [Global message view grain and metrics](../../../e9-global-message/SKILL.md)
- [engine9 MCP](../../../e9-mcp/SKILL.md)
- Plugin reference: `@engine9/plugins/reports/messaging`
