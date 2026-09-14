---
name: e9-global-message
description: "Read and consume engine9 global message warehouse views, including message identity, plugin context in legacy bot_* columns, platform engagement, native conversions, last-click attributed_* metrics, and primary-source-code dictionary fields. Use when querying message performance, attributed revenue, spend or impressions by day, building message grids and reports, interpreting inventory message statistics, or choosing between lifetime and by-date summary views; do not use it to build messaging plugins or the attribution pipeline."
---

# engine9 global message tables

Global message warehouse views put a message’s identity, platform engagement, last-click attribution, and primary-source-code dictionary or legacy origin fields on a single row. Most message performance reports and data grids read these views instead of joining the underlying tables by hand. Use this skill to select a view, choose the correct metric family, and avoid double counting or incorrect attribution.

## Quick reference

| View | Grain | Date column |
|------|--------|-------------|
| `global_message_summary` | One row per `message_id` | Use `publish_date` for “when the message went out” |
| `global_message_summary_by_date` | One row per `message_id` + `date` | `date` is the stats day (engagement day and/or transaction day — see below) |

Both sit under the account `global_table_prefix` (often empty). They are views over `global_message`, `global_message_stats` / `_by_date`, plugin metadata, and `source_code_summary` / `_by_date`; they are not base tables you write to.

## Rules

Rule: Say **transaction**, never donation.

Rule: Prefer `attributed_*` for cross-platform fundraising truth; platform `revenue` and `transactions` are native conversions only.

Rule: Do not add platform `revenue` to `attributed_revenue`; the metric families overlap and would double count.

Rule: Do not sum `origin_*` across messages that share a primary source code; those values are code-level totals repeated on each message.

## Concepts

### Column categories

Treat columns as separate categories. Mixing them (e.g. adding `revenue` + `attributed_revenue`, or treating `origin_*` as message-level last click) double-counts or misattributes.

### 1. Message identity — from `global_message`

| Typical columns | Meaning |
|-----------------|--------|
| `message_id`, `id` | Frakture message key (int); optional UUID `id` |
| `label`, `subject`, `preview_url` | Platform-facing copy / preview |
| `channel`, `channel_label`, `type`, `status` | Channel and lifecycle |
| `publish_date` | When the message published |
| `campaign_id` / `campaign_name`, `adset_*`, `message_set_id` | Hierarchy on the platform |
| `from_address`, `from_name` | Sender (email) / from phone (SMS), etc. |
| `bot_id`, `submodule`, `remote_id` | Which plugin/submodule loaded it; remote key (`bot_id` is legacy naming — see below) |
| `primary_source_code`, `primary_source_code_override`, `final_primary_source_code` | Primary code used for dictionary join and reporting buckets |
| `do_not_attribute` | Message excluded from attribution when set |

### 2. Plugin context — legacy `bot_*` columns

These come from plugin / deployment metadata. Column names still use the older **bot** vocabulary; treat them as **plugin** fields.

| Columns | Meaning |
|---------|--------|
| `bot_id`, `bot_path`, `bot_label`, `bot_nickname` | Plugin that owns the message (reports often break down by `bot_nickname`) |

Filter or group by `bot_nickname` / `bot_path` the same way you would by plugin. Do not invent a separate “bot” concept when explaining results to users — say **plugin**.

### 3. Platform engagement and native conversions — from `global_message_stats` / `_by_date`

**Platform-reported** metrics from the messaging or ad system — not engine9 last-click attribution.

| Typical columns | Meaning |
|-----------------|--------|
| `sent`, `delivered` | Sends / deliveries (email-style) |
| `impressions`, `machine_impressions` | Opens / impressions (email opens often land in `impressions`) |
| `clicks`, `engagements`, `actions`, `leads`, `total_actions`, `unique_actions` | Engagement |
| `unsubscribes`, `complaints`, `soft_bounces`, `hard_bounces`, `blocked` | List hygiene / deliverability |
| `spend`, `spend_override` | Ad spend from the platform; optional override |
| `revenue`, `transactions` | **Native** platform conversion counts/amounts. Prefer `attributed_*` for fundraising truth |

On `_by_date`, these numbers are for activity on that `date` as reported by the platform.

### 4. Last-click attribution — also on `global_message_stats` / `_by_date`

Rollups of transactions whose final message id is this message. Pipeline details: [e9-source-code](../e9-source-code/SKILL.md) (last-click A–F).

| Columns | Meaning |
|---------|--------|
| `attributed_revenue`, `attributed_transactions` | Sum/count of attributed transaction amounts (`amount > 0`) |
| `attributed_revenue_day_1` / `_day_6`, `attributed_transactions_day_*` | Windows relative to `publish_date` (+1 / +6 days). On the **lifetime** summary; not the main by-date rollup |
| `attributed_recurring_*`, `attributed_initial_recurring_*`, `attributed_subsequent_recurring_*` | Recurring vs first installment vs later installments |
| `attributed_refund_*` (and recurring refund variants) | Refund counts/amounts (not netted out of `attributed_revenue` unless a report does so) |
| `attributed_soft_credit_*` | Soft credits when present |
| `attributed_min_amount` / `max` / `avg` | Amount distribution |
| `attributed_transaction_unique_person` | Distinct people (legacy `person_id_int`) on attributed txs |
| `attributed_actions` | Attributed non-monetary actions when populated |

On `_by_date`, `attributed_*` are grouped by **transaction calendar day** (`ts`), not by `publish_date`. A day can have attributed revenue with zero impressions (or the reverse): engagement days and conversion days are independent.

### 5. Source-code elements and legacy origin — from `source_code_summary` / `_by_date`

Joined on `message.final_primary_source_code = summary.source_code` (and matching `date` for the by-date view).

| Columns | Meaning |
|---------|--------|
| Parsed elements (`source_code_channel`, `goal`, `geo`, …) and `*_label` | Dictionary parse + labels for the **primary** source code |
| `source_code_date_parsed` / `source_code_parsed_date` | Date embedded in the code string (often ≠ `publish_date`) |
| `origin_person_count`, `origin_transaction_*`, `origin_initial_*`, `origin_subsequent_*`, `origin_refund_*` | **Legacy** origin-model rollups for that **source code**. Not message-scoped last click. Prefer current `{prefix}_*` model tables ([e9-model](../e9-model/SKILL.md)) |

`origin_*` and element fields on a message row are properties of the primary source code, not of the message alone. If several messages share one primary code, they each show the same code-level origin totals.

## Workflow

### Choose a view and metrics

| Question | Use |
|----------|-----|
| How did this email/ad perform overall? | `global_message_summary`, filter on `publish_date` / `channel` / `bot_*` (plugin) |
| Spend, impressions, clicks over a calendar range? | `global_message_summary_by_date`, filter on `date`, sum engagement / `spend` |
| Last-click revenue in a calendar range? | `global_message_summary_by_date`, sum `attributed_revenue` on `date` (transaction day) |
| Cross-channel fundraising truth? | `attributed_*`, not `revenue` / `transactions` |
| Acquisition / LTV by first touch or CRM origin? | Current model tables (`{prefix}_*`), not `origin_*` on these views |
| ROI for ads? | `sum(attributed_revenue) / sum(spend)` (reports usually do this) |

### Trace related objects

| Object | Role |
|--------|------|
| `global_message` | Message dimension only |
| `global_message_stats` / `_by_date` | Measures behind categories 3–4 |
| `source_code_summary` / `_by_date` | Per–source-code rollups; join key for category 5 |
| `transaction_summary` | Row-level last-click debug (`recommended_message_id`, `transaction_source_code`) |
| `union_global_message_summary*` | Account-union variants (multi-warehouse setups) |

Stale `attributed_*` with good `transaction_summary.recommended_message_id` usually means attribution stats have not run for that publish window — diagnose with [e9-source-code](../e9-source-code/SKILL.md) pipeline A–F, not by rewriting these views.

## Examples

### Query lifetime and daily metrics

```sql
-- Lifetime performance by publish month
SELECT
  date_trunc('month', publish_date) AS month,
  sum(sent) AS sent,
  sum(impressions) AS impressions,
  sum(clicks) AS clicks,
  sum(attributed_revenue) AS attributed_revenue
FROM global_message_summary
WHERE publish_date >= DATE '2026-01-01'
GROUP BY 1
ORDER BY 1;

-- Calendar-day attribution uses the transaction day
SELECT date, sum(attributed_revenue) AS attributed_revenue
FROM global_message_summary_by_date
WHERE date >= DATE '2026-01-01'
GROUP BY date
ORDER BY date;
```

### Interpret inventory statistics

[e9-inventory](../e9-inventory/SKILL.md) warehouse statistics read these views. Grain is **month** (`YYYY-MM`). `impressions` is platform opens.

These blocks are **aggregate** message stats (`kind: aggregate`) — the usual inventory audit. They are **not** per-person timeline entries. Per-person send/open/click rows live under **Timeline → Messages** (`statistics.inputs`, `subcategory: messages`). Audit this aggregate work more often than timeline entries.

| Inventory key | Source | Bucket |
|---------------|--------|--------|
| `statistics.messages` | `global_message_summary` | Plugin (`bot_*`) / submodule / **channel** / month of **`publish_date`**. `records` = message count; sums `sent`, `impressions`, `clicks`, `spend`, `attributed_*` when columns exist. Also `by_channel_month`. Inventory UI: **Messages**. |
| `statistics.message_summary_by_date` | `global_message_summary_by_date` | **Active ads** (`spend > 0`) by plugin / submodule / channel / month of **`date`** (`count(distinct message_id)`). Coverage only — no engagement sums. |
| `statistics.message_activity` | `global_message_summary_by_date` | Calendar-day engagement, **no spend filter**, same grain and metric sums as `messages`. Prefer this for Home sends / opens / clicks by channel. |

Use the lifetime summary (`messages`) for publish-month totals; use `message_activity` for calendar-month engagement (opens/clicks after send day, ads, SMS). Do not use `message_summary_by_date` for email activity — it drops rows without spend. Do not use `inputs.by_plugin_entry_type_month` as a substitute for these views.

## Troubleshooting

| Symptom | Check |
|---------|-------|
| `attributed_*` is stale but `recommended_message_id` is correct | Attribution statistics may not have run for the publish window |
| Revenue appears duplicated | Confirm native `revenue` was not added to `attributed_revenue` |
| `origin_*` totals repeat across rows | Check whether messages share `final_primary_source_code` |
| Email activity is missing from inventory | Use `message_activity`, not spend-filtered `message_summary_by_date` |

## Related documentation

- [Source codes and last-click attribution](../e9-source-code/SKILL.md)
- [Acquisition and LTV models](../e9-model/SKILL.md)
- [Inventory statistics and Home mapping](../e9-inventory/SKILL.md#using-statistics)
- [Timeline per-person message entries](../e9-timeline/SKILL.md)
