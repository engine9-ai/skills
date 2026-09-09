---
name: e9-global-message
description: >-
  Explains how to read and consume engine9 global message warehouse views
  (global_message_summary, global_message_summary_by_date): message identity,
  plugin context (legacy bot_* columns), platform engagement, last-click
  attributed_* metrics, and primary-source-code dictionary fields. Use when
  querying message performance, attributed revenue, spend/impressions by day,
  message grids/reports, or choosing between lifetime vs by-date summary views.
  Not for building messaging plugins or the attribution pipeline (see
  e9-source-code).
---

# engine9 global message tables

Warehouse **views** that put one message’s identity, platform engagement, last-click attribution, and (via primary source code) dictionary / legacy origin fields on a single row. Most message performance reports and data grids read these views rather than joining the underlying tables by hand.

This skill is about **reading and consuming** those views. For how last-click attribution is computed, see [e9-source-code](../e9-source-code/SKILL.md). For acquisition / LTV models, see [e9-model](../e9-model/SKILL.md).

| View | Grain | Date column |
|------|--------|-------------|
| `global_message_summary` | One row per `message_id` | Use `publish_date` for “when the message went out” |
| `global_message_summary_by_date` | One row per `message_id` + `date` | `date` is the stats day (engagement day and/or transaction day — see below) |

Both sit under the account `global_table_prefix` (often empty). They are views over `global_message`, `global_message_stats` / `_by_date`, plugin metadata, and `source_code_summary` / `_by_date` — not base tables you write to.

Say **transaction**, never donation. Prefer **`attributed_*`** for cross-platform fundraising truth; platform `revenue` / `transactions` are native conversions only.

## Column categories

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

**Important:** `origin_*` (and element fields) on a message row are properties of the primary source code, not of the message alone. If several messages share one primary code, they each show the **same** code-level origin totals. Do not sum `origin_*` across messages that share a code.

## Which view and which metrics

| Question | Use |
|----------|-----|
| How did this email/ad perform overall? | `global_message_summary`, filter on `publish_date` / `channel` / `bot_*` (plugin) |
| Spend, impressions, clicks over a calendar range? | `global_message_summary_by_date`, filter on `date`, sum engagement / `spend` |
| Last-click revenue in a calendar range? | `global_message_summary_by_date`, sum `attributed_revenue` on `date` (transaction day) |
| Cross-channel fundraising truth? | `attributed_*`, not `revenue` / `transactions` |
| Acquisition / LTV by first touch or CRM origin? | Current model tables (`{prefix}_*`), not `origin_*` on these views |
| ROI for ads? | `sum(attributed_revenue) / sum(spend)` (reports usually do this) |

## Related objects

| Object | Role |
|--------|------|
| `global_message` | Message dimension only |
| `global_message_stats` / `_by_date` | Measures behind categories 3–4 |
| `source_code_summary` / `_by_date` | Per–source-code rollups; join key for category 5 |
| `transaction_summary` | Row-level last-click debug (`recommended_message_id`, `transaction_source_code`) |
| `union_global_message_summary*` | Account-union variants (multi-warehouse setups) |

Stale `attributed_*` with good `transaction_summary.recommended_message_id` usually means attribution stats have not run for that publish window — diagnose with [e9-source-code](../e9-source-code/SKILL.md) pipeline A–F, not by rewriting these views.

## Inventory statistics

[e9-inventory](../e9-inventory/SKILL.md) warehouse statistics read these views. Grain is **month** (`YYYY-MM`). `impressions` is platform opens.

| Inventory key | Source | Bucket |
|---------------|--------|--------|
| `statistics.messages` | `global_message_summary` | Plugin (`bot_*`) / submodule / **channel** / month of **`publish_date`**. `records` = message count; sums `sent`, `impressions`, `clicks`, `spend`, `attributed_*` when columns exist. Also `by_channel_month`. |
| `statistics.message_summary_by_date` | `global_message_summary_by_date` | **Active ads** (`spend > 0`) by plugin / submodule / channel / month of **`date`** (`count(distinct message_id)`). Coverage only — no engagement sums. |
| `statistics.message_activity` | `global_message_summary_by_date` | Calendar-day engagement, **no spend filter**, same grain and metric sums as `messages`. Prefer this for Home sends / opens / clicks by channel. |

Use the lifetime summary (`messages`) for publish-month totals; use `message_activity` for calendar-month engagement (opens/clicks after send day, ads, SMS). Do not use `message_summary_by_date` for email activity — it drops rows without spend.

Home mapping: [e9-inventory using statistics](../e9-inventory/SKILL.md#using-statistics).
