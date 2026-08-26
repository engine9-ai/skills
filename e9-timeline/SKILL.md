---
name: e9-timeline
description: >-
  Explains the engine9 person activity timeline: one entry per row (send, open,
  click, transaction, signup), entry types, warehouse tables (timeline, input,
  detail, summary), how plugins load entries, the console Timeline tab,
  querying, segments, and missing-entry debug. Timeline rows are entries, never
  events. Use when working with timeline,
  entry_type_id, EMAIL_OPEN, EMAIL_CLICK, EMAIL_SEND, person's activity,
  person_entry_summary, timeline_detail, missing opens/clicks, or when a
  Troubleshoot session classifies a Timeline issue.
---

# engine9 timeline

The **timeline** is each person’s activity log. One row is one **entry**: one fact about one person at one time — an email send, open, or click; an SMS; a transaction; a signup; a form submit; a segment add.

Say **entry**, never event. A `timeline` row is an **entry** (`entry_type_id`, `person_entry_summary`, `getTimelineEntryUUID`). Vendor systems may call the same facts events; once they land in engine9 they are entries.

engine9 does not invent timeline rows from reports. Plugins load activity from the systems that already recorded it (ESP, SMS, CRM, payments, forms). Identity is stamped first (`person_id`); the **entry** then lands on `timeline`. How that `person_id` is chosen is [e9-person-id](../e9-person-id/SKILL.md). Source codes on entries are [e9-source-code](../e9-source-code/SKILL.md). **Attribution** joins a transaction to a message from those codes. A **model** connects a person or transaction to an item in this history and writes `{prefix}_*` tables — [e9-model](../e9-model/SKILL.md).

Say **transaction**, never donation. A payment can appear both as a `transaction` row and as a timeline `TRANSACTION_*` **entry**; revenue questions use the transaction tables, not this log.

For a missing open/click/send on a person, walk [missing-entry debug A–F](#missing-entry-debug-af) in order. Product surfaces first; do not inspect application code. Plugin file shapes: [inputs/timeline](../inputs/timeline/SKILL.md). Load jobs: [loading.md](loading.md).

## Warehouse tables

| Table / view | Role |
|--------------|------|
| `timeline` | Canonical entries. One row per entry: `id`, `ts`, `person_id`, `entry_type_id`, `input_id` |
| `input` | The stream that produced the entries (a message, form, CRM extract, …). Owns `plugin_id`, `remote_input_name`, `min_timeline_ts` / `max_timeline_ts` |
| `plugin` | Which integration owns the input |
| `source_code_dictionary` | Optional source code on the entry (`timeline.source_code_id`) |
| plugin **detail** tables | Extra columns (URL, amount, user agent, …) keyed by the same entry `id` |
| `<detail>_summary` | View: timeline + input + plugin + source code + detail extras, with a string `entry_type` |
| `person_entry_summary` | Per `(person_id, entry_type_id)`: `first_ts`, `last_ts`, `entry_count` |
| `timeline_v3` | Legacy entries (old identity, `person_id_int`). Not current `timeline`. |
| `timeline_v3_summary` | Legacy inspect view. **Type name is `entry_type_label`, not `entry_type`.** |

`timeline` is the hub. Detail tables and summaries hang off `timeline.id`. Segments and the person Timeline tab read `timeline`, often joined to `input`.

**Current vs legacy type names:** current `timeline` stores `entry_type_id` (integer); current plugin `*_summary` views add string `entry_type`. Legacy **`timeline_v3_summary` uses `entry_type_label`** for that string — do not query `entry_type` on it.

Detail table names vary by plugin (`…_timeline_detail`, `timeline_detail_email_open`, …). `DESCRIBE` / list tables; do not guess.

## Anatomy of an entry

Every `timeline` row has:

| Field | Meaning |
|-------|---------|
| `id` | Stable UUID for this entry. Reloading the same entry **upserts** (no duplicate). |
| `ts` | When it happened (not when engine9 loaded it). |
| `person_id` | Canonical person. Entries never invent identity. |
| `entry_type_id` | What kind of entry (integer; names below). |
| `input_id` | Which input (message / form / extract) it came from. Required. |
| `source_code_id` | Optional. Last-click / origin code on this entry. |
| `email_domain` | Optional. Lowercased domain, often derived from `email`. Used for domain rollups. |
| `created_at` | When engine9 stored the row. |

The console **Person → Timeline** tab is `timeline` filtered to that `person_id`.

Prefer a `*_summary` view when you want plugin name, input name, source code string, and `entry_type` (the name, not the integer). Fall back to joining `timeline` → `input` → `plugin` yourself.

## Entry types

`timeline` stores **`entry_type_id` (integer)**. Summary views add the string `entry_type`. Filter SQL on the integer; talk to people in the names.

| Group | Name | Id | Typical meaning |
|-------|------|----|-----------------|
| Origin | `CRM_ORIGIN` | 1 | Person’s origin in the CRM |
| Origin | `ACQUISITION` | 2 | Acquisition |
| Signup | `SIGNUP` / `SIGNUP_INITIAL` / `SIGNUP_SUBSEQUENT` | 3 / 4 / 5 | List / form signup |
| Signup | `UNSUBSCRIBE` | 6 | Channel-agnostic unsubscribe |
| Signup | `DATA_APPEND` | 7 | Appended attributes (often source-code related) |
| Money | `TRANSACTION` | 10 | Generic payment; prefer a specific type |
| Money | `TRANSACTION_ONE_TIME` | 11 | One-time payment |
| Money | `TRANSACTION_INITIAL` / `TRANSACTION_SUBSEQUENT` / `TRANSACTION_RECURRING` | 12 / 13 / 14 | Recurring series |
| Money | `TRANSACTION_REFUND` | 15 | Refund |
| Segment | `SEGMENT_PERSON_ADD` / `REMOVE` / `IN_SEGMENT` | 16 / 17 / 18 | Audience membership |
| Message | `MESSAGE_CONVERSION` / `_ADVOCACY` / `_TRANSACTION` | 20 / 21 / 22 | Conversion credited to a message |
| Message | `MESSAGE_DELIVERY_FAILURE_SHOULD_RETRY` / `_SHOULD_NOT_RETRY` | 25 / 26 | Delivery failure |
| SMS | `SMS_SEND` / `SMS_DELIVERED` / `SMS_CLICK` | 30 / 31 / 33 | SMS lifecycle |
| SMS | `SMS_UNSUBSCRIBE` / `SMS_BOUNCE` / `SMS_SPAM` / `SMS_REPLY` | 34 / 37 / 38 / 39 | SMS negative / reply |
| Email | `EMAIL_SEND` / `EMAIL_DELIVERED` | 40 / 41 | Send / delivery |
| Email | `EMAIL_OPEN` / `EMAIL_CLICK` | 42 / 43 | Engagement |
| Email | `EMAIL_UNSUBSCRIBE` / `EMAIL_SOFT_BOUNCE` / `EMAIL_HARD_BOUNCE` / `EMAIL_BOUNCE` / `EMAIL_SPAM` / `EMAIL_REPLY` | 44 / 45 / 46 / 47 / 48 / 49 | Email negative / reply |
| Phone | `PHONE_CALL_ATTEMPT` / `SUCCESS` / `FAIL` | 50 / 51 / 52 | Calls |
| Forms | `FORM_SUBMIT` / `FORM_PETITION` / `FORM_ADVOCACY` / `FORM_SURVEY` | 60 / 61 / 66 / 67 | Actions |
| Other | `FILE_IMPORT`, `EXPORT*` | 70, 80–82 | Loads and pushes |
| Other | `INFERRED_*` | 91–93 | Modeled, not observed |

`SOURCE_CODE_OVERRIDE` (0) is a dictionary override marker, not a person action.

Openers and clickers are **independent**. A click is not automatically an open.

## How entries get onto the timeline

1. A **plugin** extracts activity from a remote system (ESP activity, CRM actions, payment rows, form posts, …).
2. Each extract is an **input** — usually one message, one form, or one named stream (`input.remote_input_name`).
3. engine9 matches each row to a **person** (email / phone / `remote_person_id`). Unknown people are created; known keys reuse the existing `person_id`.
4. Each row gets a **stable `id`**. Same person + type + time + plugin (or a vendor `remote_entry_id` / `remote_entry_uuid`) → same UUID → upsert.
5. Core fields go to **`timeline`**. Extra fields go to the plugin **detail** table. A **summary** view is refreshed so reports can join names without repeating that SQL.

Until step 3 succeeds, the row is not on `timeline`. A file that only has an email is not a timeline entry yet.

## Related models (do not mix)

| Question | Where to look |
|----------|----------------|
| Who is this person? Duplicate people? | [e9-person-id](../e9-person-id/SKILL.md) — not `timeline.id` |
| Why is revenue on the wrong email? | [e9-source-code](../e9-source-code/SKILL.md) — `transaction_summary`, not timeline engagement |
| Did this person open / click / get sent a message? | `timeline` (`EMAIL_*`, `SMS_*`) |
| Did this person transact? Amount, recurring, refund? | `transaction` / `transaction_summary` (also mirrored as `TRANSACTION_*` timeline rows) |
| Is this person in an engagement audience? | Segment build: universe (which inputs) + search (which `entry_type_id` / window) |

**Email engagement segments** (30/60/90-day openers and clickers) read `timeline` joined to `input`. The **universe** chooses which sends can contribute (typically email messages published in the last 90 days). The **search** chooses people with `EMAIL_OPEN` or `EMAIL_CLICK` in the rolling window. An open on a message outside that universe does not count, even if the open itself is recent.

## Querying

`DESCRIBE` first if column names differ. Filter to **one** person (and one entry type when known). LIMIT.

```sql
-- Person → entries (use the integer type, or join a summary for the name)
SELECT id, ts, entry_type_id, input_id, source_code_id
FROM timeline
WHERE person_id = 12345
ORDER BY ts DESC
LIMIT 50;

-- Opens on that person
SELECT id, ts, input_id
FROM timeline
WHERE person_id = 12345
  AND entry_type_id = 42  -- EMAIL_OPEN
ORDER BY ts DESC
LIMIT 20;

-- Entry → which message / plugin
SELECT
  t.id,
  t.ts,
  t.entry_type_id,
  i.remote_input_name,
  i.input_type,
  p.name AS plugin_name,
  p.path AS plugin_path
FROM timeline t
JOIN input i ON i.id = t.input_id
JOIN plugin p ON p.id = i.plugin_id
WHERE t.person_id = 12345
ORDER BY t.ts DESC
LIMIT 50;

-- Ever / last activity of a type (if the rollup exists)
SELECT person_id, entry_type_id, first_ts, last_ts, entry_count
FROM person_entry_summary
WHERE person_id = 12345
LIMIT 50;
```

```sql
-- Legacy person inspect: timeline_v3_summary (not timeline, not base timeline_v3).
-- The type string is entry_type_label — current tables use entry_type / entry_type_id.
SELECT effective_date, person_id_int, entry_type_label, source_code
FROM timeline_v3_summary
WHERE person_id_int = 1517
ORDER BY effective_date DESC
LIMIT 100;
```

Resolve email → `person_id` via `person_email` ([e9-person-id](../e9-person-id/SKILL.md#debugging-identity)), then query `timeline`. Do not treat `timeline.id` as a person key. For legacy inspect, pair via email / `person_metadata` / `person_id_int` — never join `person_id_int` to current `person.id`.

## Missing-entry debug (A–F)

Use this when [e9-troubleshoot](../e9-troubleshoot/SKILL.md) classifies a **Timeline** issue (person Timeline tab empty/wrong, missing open/click/send, engagement segment empty, “we sent this but engine9 has no entry”). Find the person and input from the product first; SQL confirms one named entry. Do **not** inspect application code.

Walk **A → F in order**. Stop at the first gap. Pick **one** person and **one** expected entry (type + message/input if known).

```
Timeline pipeline:
- [ ] A: Person exists (person_id from email / phone / remote id)
- [ ] B: Input exists for that message / form / stream
- [ ] C: timeline row for that person_id
- [ ] D: entry_type_id is the expected type
- [ ] E: input_id joins to the expected input / plugin
- [ ] F: Detail, summary, or segment universe (complaint is extras or an audience, not the raw entry)
```

### A) Person exists

The entry cannot land without a `person_id`.

```sql
SELECT person_id, email FROM person_email
WHERE LOWER(email) = 'alice@example.com'
LIMIT 20;
```

**Gap:** no person → identity / load of the people file, not timeline. Switch to [e9-person-id](../e9-person-id/SKILL.md). **Gap:** several `person_id`s for one email → you may be looking at the wrong person; the entry can sit on the other id.

### B) Input exists

Every timeline row points at an `input` (the message, form, or extract).

```sql
SELECT i.id, i.remote_input_name, i.input_type, i.min_timeline_ts, i.max_timeline_ts, p.name, p.path
FROM input i
JOIN plugin p ON p.id = i.plugin_id
WHERE i.remote_input_name LIKE '%Appeal%'
LIMIT 20;
```

**Pass:** the expected message/form is an `input` row, with a plugin you recognize. **Gap:** no input → that stream was never loaded (remote extract or plugin install). Not a timeline query bug.

### C) A timeline row for that person

```sql
SELECT id, ts, entry_type_id, input_id
FROM timeline
WHERE person_id = ?
ORDER BY ts DESC
LIMIT 50;
```

**Pass:** any rows in the window you care about. **Gap:** person and input exist but this person has no rows → the vendor file had no corresponding activity for them, identity did not match this person, or the load did not run. Check `input.min_timeline_ts` / `max_timeline_ts` and `input.records`. Remote vs in-account: if the ESP/CRM also lacks the activity, it is remote.

### D) Correct entry type

```sql
SELECT entry_type_id, COUNT(*) AS n
FROM timeline
WHERE person_id = ?
GROUP BY entry_type_id
LIMIT 50;
```

**Pass:** the expected id is present (`42` open, `43` click, `40` send, …). **Gap:** sends exist but no opens → often real (they were sent, they did not open), or the ESP engagement file was not loaded. **Gap:** you filtered the UI/segment on `EMAIL_OPEN` while the row is `EMAIL_CLICK` (or the reverse). They are not interchangeable.

### E) Right input / plugin

```sql
SELECT t.id, t.ts, t.entry_type_id, i.remote_input_name, p.name AS plugin_name
FROM timeline t
JOIN input i ON i.id = t.input_id
JOIN plugin p ON p.id = i.plugin_id
WHERE t.person_id = ?
  AND t.entry_type_id = 42
ORDER BY t.ts DESC
LIMIT 20;
```

**Pass:** the row is attached to the message/plugin you expected. **Gap:** entries exist but on a different input (wrong message, another ESP, a test plugin). The person Timeline tab shows all inputs; a segment universe may hide this one.

### F) Detail, summary, or segment — not the raw entry

The `timeline` row is present and correctly typed/joined, but the complaint is a blank extra field, a summary view, or an audience.

- **Detail missing:** the core entry loaded; the plugin detail table did not. The Timeline tab can still show the entry.
- **Summary stale:** `*_summary` / `person_entry_summary` disagree with `timeline` → the summary job has not run. Trust `timeline`.
- **Segment empty:** check universe (message `publish_date` / channel) **and** search window. A recent open on a message published more than 90 days ago does not qualify for the shipped email-opener segments.
- **UI disagrees after F matches:** report/UI issue, not this pipeline.

Record the first failing step in the Troubleshoot ticket notes (other accounts; ESP/CRM/payment remotes).

## Additional resources

- Load path (ID files → tables): [loading.md](loading.md)
- File shapes for plugins: [inputs/timeline](../inputs/timeline/SKILL.md)
- Person identity: [e9-person-id](../e9-person-id/SKILL.md)
- Source codes / attribution (transaction ↔ message): [e9-source-code](../e9-source-code/SKILL.md)
- Models (timeline long-term value): [e9-model](../e9-model/SKILL.md)
- Troubleshoot (support tickets): [e9-troubleshoot](../e9-troubleshoot/SKILL.md)
