---
name: e9-source-code
description: >-
  Explains engine9 source codes: the dictionary, auto-parsing and labels,
  last-click vs origin attribution, overrides, and source-code conversion
  tracking. Walks the last-click pipeline (outbound link → payment platform →
  message load → transaction load → attribution → global_message_summary). Use
  when working with source_code, source_code_dictionary, message_source_code,
  transaction_summary, attributed revenue, last click, origin people/revenue,
  recommended_message_id, or when a Troubleshoot session classifies a
  source-code attribution issue.
---

# engine9 source codes

engine9 indexes source codes from messages (email, ads, SMS, mail), transactions, and CRM person origins into a single **Source Code Dictionary**. Performance metrics, parsed elements, and attribution all hang off that hub.

Assume each source code is unique to a single message. In reporting, **source code** and **message** are interchangeable when that uniqueness holds.

Say **transaction**, never donation. Last-click debug uses `transaction_summary`; the base table is `transaction`.

For the element catalog, see [elements.md](elements.md). For attribution models and conversion-tracking methods, see [attribution.md](attribution.md). For a broken last-click number, walk [pipeline debug A–F](#last-click-pipeline-debug-af) in order.

## Warehouse tables

| Table / view | Role |
|--------------|------|
| `source_code_dictionary` | One row per unique source code; parsed elements, format match, last used |
| `message_source_code` | Source codes extracted from loaded messages (links / primary code), with `publish_date` |
| `source_code_summary` | Per-code rollups: attributed transactions/revenue, origin people/revenue, spend |
| `global_message_summary` | Per-message rollups including `attributed_revenue` and `attributed_transactions` |
| `transaction_summary` | Loaded transactions with `transaction_source_code`, `ts`, `amount`, `recommended_message_id` |
| `transaction` | Base conversion rows; `source_code_id`, overrides, `recommended_message_id` / `final_message_id` |
| `message` | `primary_source_code`, `primary_source_code_override`, `final_primary_source_code`, `publish_date` |

Attributed transaction counts and revenue appear on both `global_message_summary` (per message) and `source_code_summary` (per source).

## Last-click vs origin

engine9 reports both models out of the box. They answer different questions; do not mix them into one “value” number.

**Last click** (immediate message): the source code on the transaction, usually carried through the payment form URL. Match that string to a message’s outbound-link source codes. Those matches are **attributed transactions** / **attributed revenue**.

- **Initial revenue** — one-time transactions plus only the *first* installment of a recurring series. Closest to an integrated CRM’s native email revenue when email and payments share a platform.
- **Revenue** (last click) — initial revenue plus *subsequent* recurring installments traced to the same source code. Grows over time as pledges continue.
- Initial + subsequent = last-click transactions.

**Origin** (first engagement): the source code of the person’s earliest appearance — new constituents recruited by a message (transaction, signup, or action form). Tracked independently of last click.

- **Origin people** — person records whose first appearance traces to the code. Usually taken from the CRM origin source (treat as immutable). Some systems pick the earliest timeline action instead. Field lives on `source_code_summary`.
- **Origin revenue** — lifetime transaction total for those people, including later transactions under different last-click codes. Used for acquisition ROI.

Some codes are origin-only (list acquisition). Some are last-click-only (an appeal email). Some accrue both.

**Refunds** are not subtracted from raw revenue unless a report explicitly says so.

## Extracting source codes from message links

Scan message links for these query parameters (complete list). If several are present, keep them all; the first twelve are ranked in this order when choosing the message’s **primary** source code. `utm_term` and `utm_campaign` are used only when present **and** they are the longest candidate.

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
11. `utm_source`
12. `ref`
13. `utm_term` (longest-only)
14. `utm_campaign` (longest-only)

Example: `?src=EM_FR_20221215_donorappeal_3` → message source `EM_FR_20221215_donorappeal_3`. A later transaction with that exact code is attributed to the message.

No other link parameter is treated as a source code.

## Auto-parsing

Formats extract elements from the full source-code string. Multiple formats per account are normal (legacy vs new, mail vs digital). During parse, engine9 tries formats in order; the matching format and extracted elements land on the dictionary. Humans can override any non-revenue element afterward.

Robot-friendly codes have at least one of:

- A stable sequence of [elements](elements.md) separated by a stable delimiter (`-`, `_`, `|`). Example format: `{{campaign}}-{{channel_prefix}}-{{geo}}` for `EOQ-FB-TX`.
- A fixed-length string where character positions map to elements. Example: `21FEOQTX`.

Formats must be distinguishable. `Campaign-Channel-Targeting` and `Appeal-Fund-Goal` (both three hyphenated tokens) will collide. Prefer distinct token counts, prefixes, or delimiters.

The matched format name is stored with the dictionary row.

## Labels

Optional human names for parsed values (`EM` → `Email`). Configured in Source Codes → Dictionary Labels; applied on the next parser run.

- Exposed as `{element}_label` (e.g. `source_code_channel_label`).
- Unmapped values pass through (`XY` → `XY`).
- Hierarchical: engine9 defaults, then agency, then account. A more specific entry wins. Built-in example: `FB` → `Facebook`.

Same tokens with different spellings (`FB`, `ADS`, `Facebook`, `12`) should share a label so reports roll up as one channel.

## Dictionary columns (common)

Revenue (bot-calculated from attribution; may differ from a message system’s native revenue):

| Field | Meaning |
|-------|---------|
| Initial revenue | One-time + first recurring installment |
| Revenue | Last-click total, including later recurring installments |
| Origin people / origin revenue | First-touch people and their lifetime transactions |
| Refunds | Not netted from revenue unless marked |

Cost (engine9 does not auto-compute ROI; the reporting environment does):

| Field | Meaning |
|-------|---------|
| Spend | Uneditable. Captured from ad platforms (Facebook, Google, Bing). Email, SMS, organic, mail are typically `$0`. Platform-reported only — excludes creative, list cost, etc. |
| Acquisition cost | Set via **Acquisition Cost Override** only; copies into Acquisition Cost. Not used by native reports unless the account wires it in. |
| Budget & projections | Per code, per day, across a start/end range. Console: Source Code → Budget & Projections. |

Common parsed elements (full list in [elements.md](elements.md)): `source_code_channel`, `source_code_date` (`YYYYMMDD`, often ≠ message publish date), `goal`, `geo`.

Recommended message hierarchy: **Campaign → Message set → Message** (`campaign`, `message_set`, `variant`).

## Overrides

Override values are stored separately from parsed defaults. Clearing an override restores the bot value.

**Dictionary (per element):** type into the `* Override` column (e.g. Goal Override). The live element updates immediately; downstream reports follow. Deleting the override snaps back to the parse.

**Transaction — source override** (`override_source_code_id`): reassigns the transaction to that dictionary code (and thus `source_code_summary`). Does **not** change message attribution if a message id is also set.

**Transaction — message id override** (`override_message_id`): changes which message gets attributed-revenue credit. Use when last-click revenue is on the wrong email/ad/SMS.

**Message — primary source override** (`primary_source_code_override`): a message may have many attached codes; one is primary (categorizes the message). Heuristics pick the primary; override to change grouping or attribution. Entering a value (A) attaches that code for last-click matching and (B) makes it primary for reports. Both can wait for an overnight attribution cycle.

## Conversion tracking

Prefer **source-code** conversion tracking over native CRM tracking or tags/pixels. Source codes are auditable (every transaction has a code, including blank / white mail), channel-agnostic (mail through social), fail visibly, and need no JavaScript. Downsides: every message/link must be coded, codes must be unique and documented, and message stats (impressions, clicks, spend) must be joined to conversions — that join **is** attribution.

Native and tag/pixel metrics may still be loaded; they are estimates (cookies, JS, privacy, web-only) and do not replace source-code attribution.

Details and method comparison: [attribution.md](attribution.md).

## Last-click pipeline debug (A–F)

Use this when [e9-troubleshoot](../e9-troubleshoot/SKILL.md) classifies a **Source coding / attribution** issue (wrong/zero attributed revenue, message not credited, source code missing in engine9). Diagnose from the live link and payment platform first; SQL confirms a named code. Do **not** inspect application code.

The usual last-click path is two platforms joined by one string. Walk **A → F in order**. Stop at the first gap; later steps are meaningless until that gap is explained. `DESCRIBE` tables first if column names differ. Filter to **one** example source code (here `SC_EG_123`) and LIMIT probes.

```
Last-click pipeline:
- [ ] A: Outbound payment link carries SC_EG_123
- [ ] B: Payment-platform transaction carries SC_EG_123
- [ ] C: Message load → source_code_dictionary + message_source_code
- [ ] D: Transaction load → transaction_summary.transaction_source_code
- [ ] E: Attribution → transaction_summary.recommended_message_id
- [ ] F: updateAttributionStatistics → global_message_summary
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

### F) `updateAttributionStatistics` → `global_message_summary`

`updateAttributionStatistics` groups attributed transactions by `message_id` and writes `global_message_summary`: `sum(amount)` → `attributed_revenue`, `count(*)` → `attributed_transactions`.

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

**Gap:** E has `recommended_message_id` but summary is zero/stale/wrong → stats job has not run or is aggregating a different set (overrides, date filters, refunds). A UI report that still disagrees after F matches is a report/UI issue, not this pipeline.

Record the first failing step in the Troubleshoot ticket notes (other accounts; messaging vs payment remotes).
