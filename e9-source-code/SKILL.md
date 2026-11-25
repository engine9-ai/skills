---
name: e9-source-code
description: >-
  Explains engine9 source codes: the dictionary, auto-parsing and labels,
  last-click attribution (transaction to message), overrides, and source-code
  conversion tracking. Walks the last-click pipeline (outbound link → payment
  platform → message load → transaction load → attribution →
  global_message_summary). Use when working with source_code,
  source_code_dictionary, message_source_code, transaction_summary, attributed
  revenue, last click, recommended_message_id, or diagnosing a source-code
  attribution issue. Explains that `source_code_summary.origin_*` is either
  legacy (`source_code_stats`) or the current first-touch model depending on
  `summary_model_kind`, how acquisition cost overrides change the view, and
  how to diagnose origin revenue that differs between surfaces.
---

# engine9 source codes

engine9 indexes source codes from messages, transactions, and CRM person
origins in a shared Source Code Dictionary. Use this skill to understand source
code parsing and overrides, interpret attribution metrics, or diagnose a
last-click attribution gap. Performance metrics, parsed elements, and
attribution all connect through the dictionary.

Rule: First Touch is the standard model, and every account should run it with
`summary_model_kind` set to `current`. Use CRM Origin or Last Acquisition only
when the user explicitly asks. Last click (`attributed_*`) is attribution to a
message, not a model ([e9-model — Default: First Touch](../e9-model/SKILL.md#default-first-touch)).

## Quick reference

| Need | Start here |
| --- | --- |
| Understand attribution versus origin models | [Concepts](#concepts) |
| Look up warehouse tables and fields | [File format](#file-format) |
| Parse codes, labels, and overrides | [Workflow](#workflow) |
| Apply safety constraints | [Rules](#rules) |
| Trace missing or wrong revenue | [Troubleshooting](#troubleshooting) |
| Older sends show $0 `attributed_*` after a history load | [F — `updateAllAttributionStatistics`](#f-updateallattributionstatistics--global_message_summary) (`start`/`end`; default `-270d`) |
| Is `source_code_summary.origin_*` legacy or first touch? | [`origin_*` on `source_code_summary`](#origin_-on-source_code_summary-legacy-or-current-model) |
| Why acquisition cost differs between dictionary and summary | [Acquisition cost overrides](#acquisition-cost-overrides-on-source_code_summary) |
| Origin / first touch revenue on one code but not another | [Origin revenue debug](#origin-first-touch-revenue-differs-between-source-codes-or-surfaces) |
| Last-click revenue and First Touch revenue differ on the same codes | [e9-first-touch-vs-last-click](../e9-first-touch-vs-last-click/SKILL.md) |

## Concepts

Rule: Assume each source code is unique to a single message. In reporting,
source code and message are interchangeable only when that uniqueness holds.

Rule: Say transaction, never donation, in schema and field discussions.

### Last-click attribution versus origin models

Last-click attribution rolls up on `source_code_summary`. Current origin and
other timeline models roll up on `{prefix}_person_stats` and
`{prefix}_transaction_stats`. They answer different questions.

Rule: Do not mix attributed revenue and model revenue into one “value” number.

## File format

### Warehouse tables

| Table / view | Role |
|--------------|------|
| `source_code_dictionary` | One row per unique source code; parsed elements, format match, last used |
| `message_source_code` | Source codes extracted from loaded messages (links / primary code), with `publish_date` |
| `source_code_summary` | Per-code rollups: last-click attributed transactions/revenue (attribution), spend, dictionary elements with acquisition overrides applied. `origin_*` comes from legacy `source_code_stats` **or** a current `{prefix}_*_stats` model, chosen by `summary_model_kind` — see [`origin_*` on `source_code_summary`](#origin_-on-source_code_summary-legacy-or-current-model) |
| `source_code_dictionary_override` | Hand-entered element overrides (acquisition cost/date, goal, …) keyed by the `source_code` **string**. Applied by `source_code_summary`, not written into `source_code_dictionary` |
| `global_message_summary` | Per-message rollups including `attributed_revenue` and `attributed_transactions` — how to read the views: [e9-global-message](../e9-global-message/SKILL.md) |
| `transaction_summary` | Loaded transactions with `transaction_source_code`, `ts`, `amount`, `recommended_message_id` |
| `transaction` | Base conversion rows; `source_code_id`, overrides, `recommended_message_id` / `final_message_id` |
| `message` | `primary_source_code`, `primary_source_code_override`, `final_primary_source_code`, `publish_date` |

Attributed transaction counts and revenue appear on both `global_message_summary` (per message) and `source_code_summary` (per source).

### Attribution and origin fields

**Last click** is **attribution**: it connects a transaction to a message. The source code on the transaction, usually carried through the payment form URL, is matched to a message’s outbound-link source codes. Those matches are **attributed transactions** / **attributed revenue**.

- **Initial revenue** — one-time transactions plus only the *first* installment of a recurring series. Closest to an integrated CRM’s native email revenue when email and payments share a platform.
- **Revenue** (last click) — initial revenue plus *subsequent* recurring installments traced to the same source code. Grows over time as pledges continue.
- Initial + subsequent = last-click transactions.

**Origin** is a **model**: it connects a person (and that person’s lifetime transactions) to an item in their [timeline](../e9-timeline/SKILL.md) history — not to the message last-click would credit. Usually the CRM origin source (treat as immutable); some systems pick the earliest timeline action instead.

- **Current** (new identity): `model_crm_origin_*`, `model_first_touch_*`, and other `{prefix}_*` tables — [e9-model](../e9-model/SKILL.md). `person_id` is bigint; `transaction_id` is UUID.
- **Legacy:** `source_code_stats.origin_*`, computed against the old identity (`person_metadata` / `person_id_int`). Do not use these for current model work.
- **`source_code_summary.origin_*` is either one**, depending on the account's `summary_model_kind`. The column names do not change when the source does. Rule: Before explaining any `origin_*` number, establish which mode the view is in — [`origin_*` on `source_code_summary`](#origin_-on-source_code_summary-legacy-or-current-model).

Some codes are origin-only (list acquisition). Some are last-click-only (an appeal email). Some accrue both.

**Refunds** are not subtracted from raw revenue unless a report explicitly says so.

### `origin_*` on `source_code_summary`: legacy or current model

`source_code_summary` is a view rebuilt by the source-code summary job. When it
is built, it reads one setting and conditionally maps the `origin_*` output
columns to either the legacy origin stats (old `person_metadata` identity
mapping) or the current first touch model's stats (when available). This is the
first thing to check when origin people or origin revenue look wrong.

| `summary_model_kind` | `origin_*` comes from | `trans_model_<stem>_*` comes from |
| --- | --- | --- |
| `current` | the preferred complete current model: `model_first_touch_*_stats`, else `model_crm_origin_*`, else `model_last_acquisition_*` (joined as `origin_model_person` / `origin_model_transaction`) | `model_<stem>_person_stats` / `_transaction_stats` for each model that has both |
| `legacy` | `source_code_stats.origin_*` (old identity) | `transaction_model_pivot` |
| no setting row | **same as `legacy`** | same as `legacy` |

The setting is `summary_model_kind` on the plugin at path
`@engine9/plugins/models/deployment/v2026_09_16` (join on `plugin.path`, never a
plugin UUID). Valid values are `current` and `legacy`.

In `current` mode, the mapped fields are:

| `source_code_summary` column | Current model column |
| --- | --- |
| `origin_person_count` | `{prefix}_person_stats.person_count` |
| `origin_transaction_count` | `{prefix}_transaction_stats.transactions` |
| `origin_transaction_revenue` | `{prefix}_transaction_stats.revenue` |
| `origin_refund_count` / `origin_refund_amount` | `{prefix}_transaction_stats.refund_count` / `refund_amount` |
| `origin_transaction_unique_person` | `{prefix}_transaction_stats.transaction_unique_person` |
| any other `origin_*` (`origin_initial_*`, `origin_subsequent_*`, …) | `0`; date-typed columns are `2000-01-01` |

Rule: In `current` mode, a `0` or `2000-01-01` on an unmapped `origin_*`
column is a placeholder, not data. Do not report it as "no initial revenue."

Rule: Zero-filled `trans_model_<stem>_*` columns are a compatibility
compromise for consumers that still select them out of the view. A `0` there
means that model's stats are not joined (not deployed or not run), not that the
model credited nothing. Confirm by checking for `model_<stem>_person_stats`.
Newer builds of the view omit undeployed standard stems instead of
zero-filling them, so the column set can differ between accounts.

Rule: The mode is read when the view is **rebuilt**, not when it is queried.
The live view can disagree with the setting until the next rebuild. Check both
the setting and the live definition. An account with no setting row whose live
view still joins current model stats will flip `origin_*` back to legacy on its
next rebuild.

**Check which mode applies:**

```sql
-- 1. The setting (no row = legacy on next rebuild)
SELECT s.value
FROM setting s
JOIN plugin p ON p.id = s.plugin_id
WHERE p.path = '@engine9/plugins/models/deployment/v2026_09_16'
  AND s.name = 'summary_model_kind';

-- 2. What the live view was actually built with
SELECT
  VIEW_DEFINITION LIKE '%origin_model_person%' AS origin_from_current_model,
  VIEW_DEFINITION LIKE '%source_code_stats%' AS reads_legacy_source_code_stats
FROM information_schema.VIEWS
WHERE TABLE_SCHEMA = DATABASE() AND TABLE_NAME = 'source_code_summary';

-- 3. Compare the three numbers for one code
SELECT
  d.source_code,
  scs.origin_person_count AS summary_origin_people,
  scs.origin_transaction_revenue AS summary_origin_revenue,
  ft.person_count AS first_touch_people,
  ftt.revenue AS first_touch_revenue,
  st.origin_person_count AS legacy_origin_people
FROM source_code_dictionary d
LEFT JOIN source_code_summary scs ON scs.source_code_id = d.source_code_id
LEFT JOIN model_first_touch_person_stats ft ON ft.source_code_id = d.source_code_id
LEFT JOIN model_first_touch_transaction_stats ftt ON ftt.source_code_id = d.source_code_id
LEFT JOIN source_code_stats st ON st.source_code_id = d.source_code_id
WHERE d.source_code = 'SC_EG_123';
```

If `summary_origin_*` matches `first_touch_*`, the view is current. If it
matches `legacy_origin_*`, the view is legacy. Legacy and current can credit
different codes for the same people, because legacy origin never saw the
current timeline's source code override entries.

### Acquisition cost overrides on `source_code_summary`

`source_code_summary` applies `source_code_dictionary_override`, joined on the
`source_code` string (`dict.source_code = override.source_code`). Acquisition
cost entered through the override folder or console therefore shows on the
summary view while `source_code_dictionary` keeps the parsed value, often `0`
or blank.

| Override field | Rule in the view |
| --- | --- |
| `acquisition_cost`, `acquisition_cost_per_person`, `acquisition_date` | override wins when it is **not null** (an explicit `0` overrides) |
| every other element (goal, channel, …) | override wins when it is a **non-empty** string |

Rule: When acquisition cost differs between `source_code_dictionary` and
`source_code_summary`, check `source_code_dictionary_override` for that exact
code string before suspecting a load problem. The view does not compute
`acquisition_cost_per_person`; it only passes through what was entered.

```sql
SELECT d.source_code, d.acquisition_cost AS dictionary_cost,
       o.acquisition_cost AS override_cost, scs.acquisition_cost AS summary_cost
FROM source_code_dictionary d
LEFT JOIN source_code_dictionary_override o ON o.source_code = d.source_code
LEFT JOIN source_code_summary scs ON scs.source_code_id = d.source_code_id
WHERE d.source_code LIKE 'ACQ_%'
LIMIT 50;
```

## Rules

### Extract source codes from message links

Scan message links for these query parameters (complete list). If several are present, keep them all; the first thirteen are ranked in this order when choosing the message’s **primary** source code. `utm_term` and `utm_campaign` are used only when present **and** they are the longest candidate.

1. `refcode`
2. `source`
3. `src`
4. `s_src`
5. `c_src`
6. `c_src2`
7. `ea.tracking.id`
8. `ms`
9. `en_txn_10`
10. `supporter.appealCode`
11. `appealCode`
12. `utm_source`
13. `ref`
14. `utm_term` (longest-only)
15. `utm_campaign` (longest-only)

Example: `?src=EM_FR_20221215_donorappeal_3` → message source `EM_FR_20221215_donorappeal_3`. A later transaction with that exact code is attributed to the message.

Rule: No other link parameter is treated as a source code.

Rule: Exact source-code string matches matter. Case changes, trailing spaces,
and other rewrites produce different codes.

## Workflow

### Auto-parsing

Formats extract elements from the full source-code string. Multiple formats per account are normal (legacy vs new, mail vs digital). During parse, engine9 tries formats in order; the matching format and extracted elements land on the dictionary. Humans can override any non-revenue element afterward.

Robot-friendly codes have at least one of:

- A stable sequence of [elements](elements.md) separated by a stable delimiter (`-`, `_`, `|`). Example format: `{{campaign}}-{{channel_prefix}}-{{geo}}` for `EOQ-FB-TX`.
- A fixed-length string where character positions map to elements. Example: `21FEOQTX`.

Formats must be distinguishable. `Campaign-Channel-Targeting` and `Appeal-Fund-Goal` (both three hyphenated tokens) will collide. Prefer distinct token counts, prefixes, or delimiters.

The matched format name is stored with the dictionary row.

### Labels

Optional human names for parsed values (`EM` → `Email`). Configured in Source Codes → Dictionary Labels; applied on the next parser run.

- Exposed as `{element}_label` (e.g. `source_code_channel_label`).
- Unmapped values pass through (`XY` → `XY`).
- Hierarchical: engine9 defaults, then agency, then account. A more specific entry wins. Built-in example: `FB` → `Facebook`.

Same tokens with different spellings (`FB`, `ADS`, `Facebook`, `12`) should share a label so reports roll up as one channel.

### Dictionary columns

Revenue from last-click **attribution** (may differ from a message system’s native revenue):

| Field | Meaning |
|-------|---------|
| Initial revenue | One-time + first recurring installment |
| Revenue | Last-click total, including later recurring installments |
| Refunds | Not netted from revenue unless marked |

Origin **model** on `source_code_summary`. The source is legacy or current depending on `summary_model_kind` ([details](#origin_-on-source_code_summary-legacy-or-current-model)). For model-specific work, read `{prefix}_*` directly — [e9-model](../e9-model/SKILL.md).

| Field | Meaning |
|-------|---------|
| Origin people / origin revenue | People the origin model credited to this code, and those people's lifetime transactions. Current mode: the first touch model when deployed. Legacy mode: `source_code_stats` |

Cost (engine9 does not auto-compute ROI; the reporting environment does):

| Field | Meaning |
|-------|---------|
| Spend | Uneditable. Captured from ad platforms (Facebook, Google, Bing). Email, SMS, organic, mail are typically `$0`. Platform-reported only — excludes creative, list cost, etc. |
| Acquisition cost | Set via **Acquisition Cost Override** only (`source_code_dictionary_override`). `source_code_summary` shows the override; `source_code_dictionary` keeps the parsed value. Not used by native reports unless the account wires it in. See [Acquisition cost overrides](#acquisition-cost-overrides-on-source_code_summary). |
| Budget & projections | Per code, per day, across a start/end range. Console: Source Code → Budget & Projections. |

Common parsed elements (full list in [elements.md](elements.md)): `source_code_channel`, `source_code_date` (`YYYYMMDD`, often ≠ message publish date), `goal`, `geo`.

Recommended message hierarchy: **Campaign → Message set → Message** (`campaign`, `message_set`, `variant`).

### Overrides

Override values are stored separately from parsed defaults. Clearing an override restores the parsed dictionary value.

**Dictionary (per element):** type into the `* Override` column (e.g. Goal Override). The live element updates immediately; downstream reports follow. Deleting the override snaps back to the parse.

**Transaction — source override** (`override_source_code_id`): reassigns the transaction to that dictionary code (and thus `source_code_summary`). Does **not** change message attribution if a message id is also set.

**Transaction — message id override** (`override_message_id`): changes which message gets attributed-revenue credit. Use when last-click revenue is on the wrong email/ad/SMS.

**Message — primary source override** (`primary_source_code_override`): a message may have many attached codes; one is primary (categorizes the message). Heuristics pick the primary; override to change grouping or attribution. Entering a value (A) attaches that code for last-click matching and (B) makes it primary for reports. Both can wait for an overnight attribution cycle.

### Conversion tracking

Prefer **source-code** conversion tracking over native CRM tracking or tags/pixels. Source codes are auditable (every transaction has a code, including blank / white mail), channel-agnostic (mail through social), fail visibly, and need no JavaScript. Downsides: every message/link must be coded, codes must be unique and documented, and message stats (impressions, clicks, spend) must be joined to conversions — that join **is** attribution.

Native and tag/pixel metrics may still be loaded; they are estimates (cookies, JS, privacy, web-only) and do not replace source-code attribution.

Details and method comparison: [attribution.md](attribution.md).

## Troubleshooting

### Last-click pipeline debug (A–F)

Use this when diagnosing a **Source coding / attribution** issue (wrong/zero attributed revenue, message not credited, source code missing in engine9). Diagnose from the live link and payment platform first; SQL confirms a named code. Do **not** inspect application code.

The usual last-click path is two platforms joined by one string.

Rule: Walk A → F in order and stop at the first gap; later steps are meaningless
until that gap is explained. `DESCRIBE` tables first if column names differ,
filter to one example source code (here `SC_EG_123`), and limit probes.

```text
Last-click pipeline:
- [ ] A: Outbound payment link carries SC_EG_123
- [ ] B: Payment-platform transaction carries SC_EG_123
- [ ] C: Message load → source_code_dictionary + message_source_code
- [ ] D: Transaction load → transaction_summary.transaction_source_code
- [ ] E: Attribution → transaction_summary.recommended_message_id
- [ ] F: updateAllAttributionStatistics → global_message_summary (default publish_date lookback -270d)
```

Exact string match matters (`SC_EG_123` ≠ `sc_eg_123` ≠ `SC_EG_123 `). Recognized link params are listed under [Extracting source codes from message links](#extracting-source-codes-from-message-links).

### A) Outbound payment link

A message (email / SMS / online ad) goes out with a payment URL whose query string includes `SC_EG_123` on a recognized param (`src`, `refcode`, `source`, …).

**Check:** the live link, message content, or ad URL — not engine9 SQL. If the send never included a recognized param, engine9 cannot attribute later.

**Gap:** coding on the messaging platform (wrong/missing param, truncated code, reused code). Remote issue.

### B) Transaction on the payment platform

The person completes a transaction on a **separate** payment platform. That platform’s transaction record stores source code `SC_EG_123`.

**Check:** the payment processor / CRM transaction (source / tracking / UTM field). engine9 can only load what that system stored.

**Gap:** code dropped or rewritten between click and payment (form default, different param name, landing-page strip). Remote issue.

### C) Message load → dictionary and `message_source_code`

engine9 loads messages from platform A. `SC_EG_123` is inserted into `source_code_dictionary` **and** `message_source_code`.

```sql
SELECT source_code_id, source_code
FROM source_code_dictionary
WHERE source_code = 'SC_EG_123'
LIMIT 5;

SELECT message_id, source_code, publish_date
FROM message_source_code
WHERE source_code = 'SC_EG_123'
LIMIT 20;
```

**Pass:** both queries return rows; `message_source_code.publish_date` is set.

**Gap:** dictionary missing → message (or its links) never loaded / param not extracted. Dictionary present but `message_source_code` empty → message row loaded without that code attached.

### D) Transaction load → `transaction_summary`

engine9 loads transactions from platform B. `SC_EG_123` appears on that row in `transaction_summary`.

**Note:** On `transaction_summary`, the last-click code is `transaction_source_code` — not a column named `source_code`. That value can be overridden per transaction via `transaction_source_code_override` (which then becomes the live `transaction_source_code`). Filter and join on `transaction_source_code`; if a row’s code looks wrong, check the override before blaming the payment-platform mapping.

```sql
SELECT transaction_id, ts, amount, transaction_source_code, transaction_source_code_override, recommended_message_id
FROM transaction_summary
WHERE transaction_source_code = 'SC_EG_123'
ORDER BY ts DESC
LIMIT 20;
```

**Pass:** the expected transaction is present with `transaction_source_code = 'SC_EG_123'`. `recommended_message_id` may still be null (that is E).

**Gap:** transaction missing → not loaded. Row present with blank/other code → mapping from the payment platform dropped or altered the source (or an override).

### E) Attribution → `recommended_message_id`

Attribution joins `transaction_summary` to `message_source_code` **on `transaction_source_code` = `message_source_code.source_code`**, and only when **`transaction_summary.ts` is after `message.publish_date`**. A match writes `recommended_message_id` on `transaction_summary`.

```sql
SELECT
  t.transaction_id,
  t.ts,
  t.transaction_source_code,
  t.transaction_source_code_override,
  t.recommended_message_id,
  m.message_id,
  m.publish_date
FROM transaction_summary t
JOIN message_source_code m ON t.transaction_source_code = m.source_code
WHERE t.transaction_source_code = 'SC_EG_123'
LIMIT 20;
```

**Pass:** a join row exists with `t.ts` after `m.publish_date`, and `t.recommended_message_id` is that message.

**Gap if join is empty:** string mismatch between C and D (not a missing load). **Gap if join exists but `ts` is not after `publish_date`:** date window (timezone, publish_date default, backdated transaction). **Gap if join and dates look valid but `recommended_message_id` is still null:** attribution has not run, or another row won / was skipped.

### F) `updateAllAttributionStatistics` → `global_message_summary`

Rule: The global `attributed_*` rollup is **`updateAllAttributionStatistics`** (TransactionBase). Message-load `updateAttributionStatistics` is a **no-op** — do not schedule it expecting a fix.

`updateAllAttributionStatistics` groups attributed transactions by final message id and writes the `attributed_*` columns of `global_message_stats` and `global_message_stats_by_date`, which `global_message_summary` shows: `sum(amount)` → `attributed_revenue`, `count(*)` → `attributed_transactions`. It is the only writer of those columns; the plugin that published the message writes only the platform columns.

Rule: Automatic runs only include messages with **`publish_date >= -270d`** (optional `end` = `publish_date < end`). Older history is **not** refreshed unless you pass `start` / `end` (or `message_ids`). This is intentional for performance — after a historical message backfill, or when tickets show $0 attributed revenue before ~270 days despite codes on both sides, the first fix to try is:

```text
task method updateAllAttributionStatistics
  options.start = <publish_date lower bound for the gap>
  options.end   = <optional publish_date upper bound>
```

If E is also empty for that old window, automatic `attribute` only covers transactions with **`ts >= -30d`** by default — widen `attribute` `start` first, then re-run `updateAllAttributionStatistics` for the message `publish_date` window.

```sql
SELECT
  recommended_message_id AS message_id,
  SUM(amount) AS attributed_revenue,
  COUNT(*) AS attributed_transactions
FROM transaction_summary
WHERE recommended_message_id IS NOT NULL
  AND transaction_source_code = 'SC_EG_123'
GROUP BY recommended_message_id
LIMIT 20;

SELECT message_id, attributed_revenue, attributed_transactions
FROM global_message_summary
WHERE message_id IN (
  SELECT recommended_message_id
  FROM transaction_summary
  WHERE transaction_source_code = 'SC_EG_123'
    AND recommended_message_id IS NOT NULL
)
LIMIT 20;
```

**Pass:** summary `attributed_revenue` / `attributed_transactions` match the grouped `transaction_summary` totals for that `message_id` (within the same filter the report uses).

**Gap:** E has `recommended_message_id` but summary is zero/stale/wrong → stats job has not run for that **publish_date** window (often outside `-270d`), or is aggregating a different set (overrides, date filters, refunds). A UI / BigQuery report that still disagrees after the account DB matches is a sync or report/UI issue, not this pipeline.

Record the first failing step (other accounts; messaging vs payment remotes).

### Origin (first touch) revenue differs between source codes or surfaces

Use this when a source code shows acquisition people or "original revenue" in
one place and zero in another. A common case is two codes for the same list,
one added through the override folder and one uploaded to the CRM with a
different date. This is a **model** question, not last click. Walk the steps in
order.

```text
Origin revenue debug:
- [ ] 1: Which mode is source_code_summary in? (setting + live view)
- [ ] 2: Is acquisition cost coming from an override?
- [ ] 3: Which entries carry each code? (entry type × input)
- [ ] 4: Which code did the model credit those people to, and why?
- [ ] 5: Do the stats tables agree with the person table?
```

**1. Mode.** Run the checks in
[`origin_*` on `source_code_summary`](#origin_-on-source_code_summary-legacy-or-current-model).
In legacy mode, `origin_*` cannot see current overrides. Stop and explain
that before comparing with the first touch model.

**2. Acquisition cost.** The summary view shows override cost; the dictionary
does not. A code can have cost on the summary, revenue in the model, and
neither on the dictionary row. See
[Acquisition cost overrides](#acquisition-cost-overrides-on-source_code_summary).

**3. Entries per code.** Call MCP `timelinePerson` with `source_codes` (one
call can cover all the codes, comma-delimited or with `%`). Read the
`source_code_inputs` table rather than the capped detail `timeline` rows. It
gives count, people, and min/max `ts` for each code × entry type × input, so
`SOURCE_CODE_OVERRIDE` (0), `CRM_ORIGIN` (1), and transaction entries are
separated per code. `source_code_overlap` shows how many people carry both
codes.

**4. Model credit.** In the same response, `source_code_model_credit` gives,
for the people on each code, the credited source code and `reason` per
`model_*_person`. People on a CRM-loaded code whose model credit is the
override code (`reason` = `Source Code Override`) are working as designed.
In every shipped model, a `SOURCE_CODE_OVERRIDE` entry beats the rule, so the
CRM code keeps only the people who are **not** on the override file.

**5. Stats.** `{prefix}_person_stats` / `_transaction_stats` should match the
credit counts from step 4. If `{prefix}_person_stats_by_date` is empty while
lifetime stats are populated, check whether `{prefix}_person.date_of_source`
is NULL ([e9-model](../e9-model/SKILL.md#troubleshooting)).

Rule: Trust the timeline **entry type** over the input or plugin join when
classifying an entry. On some older accounts, CRM origin entries were loaded
under a legacy override input. That input's `input_type` (`override`) and its
own min/max timeline ts do not describe the CRM rows. That is a legacy-data
artifact, not a model error. `source_code_inputs` shows it as
`entry_type_id = 1` on an input whose `input_min_timeline_ts` /
`input_max_timeline_ts` do not cover the entries' `min_ts` / `max_ts`.

**What to tell the requester** for the two-code case: the override-folder code
gets the people on its file plus their lifetime revenue. The CRM code keeps
only people not on the override file. Acquisition cost lives where the
override was entered. To consolidate on one code, put every person on the
override file, or remove the override and let the CRM origin stand.

## Related documentation

| Topic | Documentation |
| --- | --- |
| Full source-code element catalog | [Elements](elements.md) |
| Conversion tracking and transaction-to-message attribution | [Attribution](attribution.md) |
| Current person-level models and `{prefix}_*` tables | [e9-model](../e9-model/SKILL.md) |
| Last click vs First Touch on the same codes | [e9-first-touch-vs-last-click](../e9-first-touch-vs-last-click/SKILL.md) |
| Timeline entries used by models | [e9-timeline](../e9-timeline/SKILL.md) |
| Message performance summary views | [e9-global-message](../e9-global-message/SKILL.md) |
