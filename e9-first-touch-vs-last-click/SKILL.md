---
name: e9-first-touch-vs-last-click
description: >-
  Troubleshoots why a source code's last-click revenue (transaction source
  code → `source_code_summary.revenue`) does not match its First Touch revenue
  or people (`model_first_touch_*`, or `origin_*` in current mode). Builds the
  cohort of people who gave last-click transactions on the codes, finds which
  code First Touch credited those people and transactions to, and profiles them
  (email only, phone only, both, earlier entries, override, CRM origin, model
  not yet run). Use when a client says "these codes should line up", "first
  touch is zero but the code raised money", "origin revenue is lower than
  revenue", or asks where a code's donors were credited.
---

# First touch vs last click on the same source codes

Last click and First Touch answer different questions, so a source code's two
numbers usually differ. Clients often expect them to match for acquisition-style
codes (P2P SMS, a new list, a launch email). They match only when every person
who gave on the code had **no earlier entry anywhere on their timeline**, under
any email, phone, or remote id that resolved to the same `person_id`. This skill
finds the people for whom that is not true and says why.

## Quick reference

| Need | Start here |
| --- | --- |
| Connect before querying | [Step 0 — Connect through MCP](#step-0--connect-through-mcp) |
| Why the numbers differ | [Concepts](#concepts) |
| Run the diagnosis | [Workflow](#workflow) |
| Name the reason per person | [Buckets](#buckets) |
| Write up the answer | [What to tell the requester](#what-to-tell-the-requester) |

## Step 0 — Connect through MCP

Rule: Run every query in this skill through the engine9 MCP server (`sql`,
`timelinePerson`). Log in with `mcp_auth` → `ok` → `user`, then confirm the
account with `account` search ([e9-mcp](../e9-mcp/SKILL.md#step-0--log-in-always-first)).

Rule: If engine9 MCP is not connected in this session, `mcp_auth` fails, or the
account is not found, **stop and ask the user** how to proceed. Do not fall back
to the `e9` CLI (`e9 sqlworker query`), `accounts.d`, `accounts.compiled.json5`,
or any other local config or shell route to find the account or run SQL.

## Concepts

| | Last click (attribution) | First Touch (model) |
| --- | --- | --- |
| Unit | one transaction | one person, then all their transactions |
| Picks | the code on the transaction | the code on the person's **earliest** timeline entry |
| Column on the source code grid | `Revenue`, `Initial Rev.` | `First Touch – person count / revenue`; `Origin People / Origin Revenue` only when `summary_model_kind = current` |
| Tables | `transaction_summary.transaction_source_code`, `source_code_summary.revenue` | `model_first_touch_person`, `model_first_touch_transaction`, `model_first_touch_*_stats` |

So a code's First Touch revenue drops below its last-click revenue whenever a
donor on that code was already on the file, and rises above it whenever people
it acquired later gave on other codes. Both directions are correct behavior.
Background: [e9-model — Rules](../e9-model/SKILL.md#rules),
[e9-source-code — Concepts](../e9-source-code/SKILL.md#concepts).

Identity is what usually surprises clients. One `person_id` can hold many
emails, phones, and remote ids ([e9-person-id](../e9-person-id/SKILL.md)). A
P2P SMS donor who gives with an email the file already had is the **same
person** as that earlier subscriber, so the earlier email entry is their first
touch, not the SMS transaction.

Rule: Say transaction, never donation, in field and schema talk. Never add
last-click revenue and First Touch revenue together.

## Workflow

Walk the steps in order. Filter to the requester's exact code strings.
`DESCRIBE` a table first if a column name differs on the account.

```text
First touch vs last click:
- [ ] 1: Which surface is the requester reading? (mode + model freshness)
- [ ] 2: Cohort — people with last-click transactions on the codes
- [ ] 3: Where First Touch credited those transactions and people
- [ ] 4: Profile the people (identifiers, earlier entries, entry type, reason)
- [ ] 5: Bucket each person and write it up
```

### 1. Surface and freshness

- Run the mode check in
  [e9-source-code — `origin_*` on `source_code_summary`](../e9-source-code/SKILL.md#origin_-on-source_code_summary-legacy-or-current-model).
  In legacy mode, `Origin People / Origin Revenue` is old-identity
  `source_code_stats` and is not First Touch at all. Say so before anything
  else.
- Confirm First Touch ran after the transactions loaded. If the model is older
  than the transactions, the cohort is simply not credited yet.

```sql
SELECT MAX(date_of_source) AS latest_credit FROM model_first_touch_person;

SELECT COUNT(*) AS cohort_transactions,
       SUM(mt.transaction_id IS NULL) AS not_in_model
FROM transaction_summary t
LEFT JOIN model_first_touch_transaction mt ON mt.transaction_id = t.transaction_id
WHERE t.transaction_source_code IN ('<CODE_1>', '<CODE_2>');
```

### 2. Cohort

```sql
SELECT t.person_id,
       t.transaction_source_code,
       COUNT(*)    AS transactions,
       SUM(t.amount) AS amount,
       MIN(t.ts)   AS first_ts
FROM transaction_summary t
WHERE t.transaction_source_code IN ('<CODE_1>', '<CODE_2>')
GROUP BY t.person_id, t.transaction_source_code;
```

Totals should match the grid's last-click `Revenue`. If they do not, the
question is last click, not First Touch: go to
[e9-source-code — Last-click pipeline](../e9-source-code/SKILL.md#last-click-pipeline-debug-af).

### 3. Where First Touch put them

Transaction credit (the dollars) and person credit (the people) are separate
rows. Read both.

```sql
-- Where the cohort's own last-click transactions were credited
SELECT t.transaction_source_code AS last_click_code,
       d.source_code             AS first_touch_code,
       mt.reason,
       COUNT(*)                  AS transactions,
       SUM(t.amount)             AS amount
FROM transaction_summary t
LEFT JOIN model_first_touch_transaction mt ON mt.transaction_id = t.transaction_id
LEFT JOIN source_code_dictionary d ON d.source_code_id = mt.source_code_id
WHERE t.transaction_source_code IN ('<CODE_1>', '<CODE_2>')
GROUP BY 1, 2, 3
ORDER BY amount DESC;

-- Where the cohort's people were credited
SELECT d.source_code AS first_touch_code, mp.reason,
       MIN(mp.date_of_source) AS min_date, MAX(mp.date_of_source) AS max_date,
       COUNT(DISTINCT mp.person_id) AS people
FROM (SELECT DISTINCT person_id FROM transaction_summary
      WHERE transaction_source_code IN ('<CODE_1>', '<CODE_2>')) c
LEFT JOIN model_first_touch_person mp ON mp.person_id = c.person_id
LEFT JOIN source_code_dictionary d ON d.source_code_id = mp.source_code_id
GROUP BY 1, 2
ORDER BY people DESC;
```

MCP `timelinePerson` with `source_codes` returns the same credit rollup as
`source_code_model_credit`, plus `source_code_inputs` (entry type × input per
code). Use it to cross-check; SQL gives the per-person detail.

### 4. Profile the people

One row per cohort person: identifiers, what came before the transaction, and
what First Touch picked.

```sql
WITH c AS (
  SELECT person_id, MIN(ts) AS gift_ts
  FROM transaction_summary
  WHERE transaction_source_code IN ('<CODE_1>', '<CODE_2>')
  GROUP BY person_id
)
SELECT c.person_id,
       c.gift_ts,
       (SELECT COUNT(*) FROM person_email e WHERE e.person_id = c.person_id) AS emails,
       (SELECT COUNT(*) FROM person_phone p WHERE p.person_id = c.person_id) AS phones,
       (SELECT COUNT(*) FROM person_remote r WHERE r.person_id = c.person_id) AS remotes,
       (SELECT COUNT(*) FROM timeline tl
         WHERE tl.person_id = c.person_id AND tl.ts < c.gift_ts) AS entries_before_gift,
       (SELECT MIN(tl.ts) FROM timeline tl WHERE tl.person_id = c.person_id) AS earliest_entry_ts,
       d.source_code AS first_touch_code,
       mp.date_of_source,
       mp.reason
FROM c
LEFT JOIN model_first_touch_person mp ON mp.person_id = c.person_id
LEFT JOIN source_code_dictionary d ON d.source_code_id = mp.source_code_id
ORDER BY c.gift_ts;
```

Then group the earlier entries by entry type and code, to see what got there
first:

```sql
WITH c AS (
  SELECT person_id, MIN(ts) AS gift_ts
  FROM transaction_summary
  WHERE transaction_source_code IN ('<CODE_1>', '<CODE_2>')
  GROUP BY person_id
)
SELECT tl.entry_type_id, d.source_code, i.input_type, i.name AS input_name,
       COUNT(*) AS entries, COUNT(DISTINCT tl.person_id) AS people,
       MIN(tl.ts) AS min_ts, MAX(tl.ts) AS max_ts
FROM c
JOIN timeline tl ON tl.person_id = c.person_id AND tl.ts < c.gift_ts
LEFT JOIN source_code_dictionary d ON d.source_code_id = tl.source_code_id
LEFT JOIN input i ON i.id = tl.input_id
GROUP BY 1, 2, 3, 4
ORDER BY people DESC
LIMIT 50;
```

Translate `entry_type_id` with the table in
[e9-timeline](../e9-timeline/SKILL.md). For one surprising person, call
`timelinePerson` with `person_ids` and read the entries next to the model row.

Rule: Do not show emails or phone numbers in the write-up. Report counts and
`person_id`s.

## Buckets

Assign every cohort person to exactly one bucket, first match wins.

| # | Bucket | Signal | Meaning |
| --- | --- | --- | --- |
| 1 | Not in model | no `model_first_touch_person` row, or transaction missing from `model_first_touch_transaction` | Model has not run since the load. Rerun First Touch. |
| 2 | Source code override | `reason` = `Source Code Override` | Someone put this person on an override file. Overrides beat the rule. |
| 3 | CRM origin first | credited entry is `CRM_ORIGIN` (`entry_type_id` 1) dated before the gift | The CRM already had them, with an earlier origin code. |
| 4 | Already on file, same channel | earlier entries on the same channel (an earlier SMS send or transaction on another code) | Not new to the program. First Touch is that earlier code. |
| 5 | Already on file, other identifier | earlier entries came from an email or phone the person also holds (`emails` > 0 and `phones` > 0, earliest entry is on the other channel) | Identity merged an existing subscriber or donor with the SMS donor. |
| 6 | Same-day tie | earliest entry is the same day as the gift, with a different code (e.g. the send carries a different code than the transaction) | Entry ordering or coding mismatch, not prior history. Check codes on the send vs the payment link. |
| 7 | Genuinely new | no earlier entries, credited to this code | Matches. These are the people the requester expected. |

Then add the other direction: First Touch revenue on a code can include
**later** transactions by people it acquired, on other codes. Those raise First
Touch above last click and are not an error.

## What to tell the requester

Lead with the counts per bucket and the dollars they move, for example: "Of 24
people who gave on these codes, 3 were new to the file and are credited here.
17 were already email subscribers before the text went out, so First Touch
credits the code that first brought them in. 4 had given to an earlier SMS
code." Then:

- The two columns answer different questions. Last click credits the text that
  produced this gift. First Touch credits whatever first brought the person to
  the file.
- They line up only for donors who were new to the whole file, across every
  email and phone on record.
- If the requester wants these codes to own their donors, that is a Source Code
  Override decision, not a fix. Point them to
  [e9-source-code — Origin revenue debug](../e9-source-code/SKILL.md#origin-first-touch-revenue-differs-between-source-codes-or-surfaces).

## Related documentation

| Topic | Documentation |
| --- | --- |
| Last click pipeline, `origin_*` mode, overrides | [e9-source-code](../e9-source-code/SKILL.md) |
| First Touch rules, tables, `reason` | [e9-model](../e9-model/SKILL.md) |
| Entry types and timeline queries | [e9-timeline](../e9-timeline/SKILL.md) |
| Why one person holds many emails and phones | [e9-person-id](../e9-person-id/SKILL.md) |
| MCP login, `sql`, `timelinePerson` | [e9-mcp](../e9-mcp/SKILL.md) |
