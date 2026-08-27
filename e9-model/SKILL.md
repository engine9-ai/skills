---
name: e9-model
description: >-
  Explains engine9 models: a model reads a person's timeline of entries and
  decides which source code deserves credit for that person (first touch, CRM
  origin, last acquisition), writing {prefix}_* tables
  (model_first_touch_person, …) that answer lifetime value (LTV) and acquisition
  ROI questions. Models connect people and their transactions to PEOPLE-level
  origin; attribution connects a transaction to a MESSAGE and is never a model
  (see e9-source-code). Also explains the timeline effective date cascade
  (timeline.ts / effective_date, default_timestamp, extract_timestamp,
  source_code_date, acquisition_date, SOURCE_CODE_OVERRIDE) that every model
  sorts by. Use when working with ModelWorker, model_*_person,
  model_*_transaction, lifetime value, LTV, acquisition ROI, first touch, CRM
  origin, or reading/running a model. Authoring and internals: developers.md.
---

# engine9 models

This document explains **what a model is, why it exists, and how to read what it
produces**. It assumes no prior engine9 knowledge. Building or running one in
code is [developers.md](developers.md).

## 1. The timeline: what a model reads

Every person in engine9 has a **timeline**: a list of things that happened to
them or that they did, in time order. One row is one **entry** — one fact about
one person at one moment.

| effective date (`ts`) | entry | source code on the entry |
| --- | --- | --- |
| 2019-03-02 | signed a petition | `WEB_PET_2019_climate` |
| 2019-03-05 | email send | `EM_WEL_20190305` |
| 2020-01-14 | email click | `EM_FR_20200114_appeal` |
| 2020-01-14 | transaction, $25 | `EM_FR_20200114_appeal` |
| 2023-06-01 | transaction, $50 | `MAIL_JUN2023` |

A **source code** is the campaign/effort label carried on an entry ("the June
2023 mail piece", "the climate petition"). Details: [e9-timeline](../e9-timeline/SKILL.md)
and [e9-source-code](../e9-source-code/SKILL.md).

The timeline is raw history. It does not say which of those five entries
*explains* why this person is on the file, and nothing in it lets you total up
"revenue from people the climate petition brought us." That is the gap a model
fills.

## 2. What a model is

A **model** is a rule that walks one person's entries and picks the single entry
that deserves credit for **that person**. It stores four things per person:

| Stored | Example |
| --- | --- |
| the source code from the chosen entry | `WEB_PET_2019_climate` |
| the date of that entry | 2019-03-02 |
| the reason the rule chose it | `First Touch` |
| the person | `person_id` 12345 |

It then stamps that same credit onto **every transaction that person ever
makes** — the $25 in 2020 and the $50 in 2023 both count toward the climate
petition. A model may optionally use a different rule for transactions than for
people, but the default is "the person's credit applies to their transactions."

That one step is what makes person-level economics computable:

| Question | How the model answers it |
| --- | --- |
| **Lifetime value (LTV)** — what is a person acquired by this effort worth over time? | Sum every transaction stamped with that source code, divide by the number of people stamped with it |
| **Acquisition ROI** — did this effort pay for itself? | That revenue against the acquisition cost recorded for the source code |
| How many people did this effort bring onto the file? | Count of people stamped with that source code |
| Which channel produces donors who keep giving? | Compare LTV across the source codes of each channel |

A model is a *judgment call made explicit and stored*. Two models can disagree
about the same person and both be correct, because they answer different
questions ("who first touched them?" vs. "what most recently reactivated
them?"). This is why engine9 runs several models side by side instead of
choosing one, and why every stored row carries a `reason`.

## 3. Attribution is NOT a model

This is the most important distinction in this document, and the easiest to get
wrong.

| | **Attribution** | **Model** |
| --- | --- | --- |
| Connects a transaction to | a **message** | — |
| Connects a person (and all their transactions) to | — | an **entry in that person's timeline** |
| Question answered | "Which email/mail piece produced *this* gift?" | "What brought *this person* onto the file?" |
| Scope | one transaction at a time | a person's whole history |
| Method | last click on the transaction's source code | a rule over the timeline |
| Lives in | [e9-source-code](../e9-source-code/SKILL.md) | this document |
| Output | `recommended_message_id`, `attributed_revenue`, `source_code_summary.revenue` | `{prefix}_person`, `{prefix}_transaction` |

**Attribution is exclusively transaction → message.** It is never a model. Do
not call last click "the last-click model", do not list it among the models, and
do not put it in a model comparison as if it were a peer.

Both are true at once, and they are not alternatives:

> The $50 gift in June 2023 was **attributed** to the June mail piece — that
> mail piece is what produced that gift. The **first touch model** says this
> person came from the 2019 climate petition — that petition is why this person
> exists at all. Credit the mail piece for the gift; credit the petition for the
> donor.

Consequences to respect:

- Never add attributed revenue and model revenue together; they are the same
  dollars counted for two different questions.
- Model revenue for a source code will not equal that source code's attributed
  revenue, and that is not a bug. A petition that never asks for money can have
  huge model revenue and near-zero attributed revenue.
- `source_code_summary.revenue` / `attributed_*` are attribution. The
  `source_code_summary.origin_*` columns are a **legacy** model implementation
  (old identity) and are not current model output — see
  [developers.md](developers.md#legacy-old-identity).

## 4. The models engine9 ships

Each model is a plugin with a stable `prefix`. The prefix is the table name stem
and is identical on every account.

| Model | `prefix` | The rule, in words |
| --- | --- | --- |
| First Touch (`@engine9/plugins/models/first_touch`) | `model_first_touch` | The **earliest** entry of any kind. "Where did this person first show up?" |
| CRM Origin (`@engine9/plugins/models/crm_origin`) | `model_crm_origin` | The entry the CRM itself labels as the origin (`CRM_ORIGIN`). "What does the system of record say?" Blank when the CRM says nothing. |
| Last Acquisition (`@engine9/plugins/models/last_acquisition`) | `model_last_acquisition` | The **most recent** entry marked `ACQUISITION`, falling back to `CRM_ORIGIN`. "What most recently brought this person in?" |

In all three, an explicit `SOURCE_CODE_OVERRIDE` entry on the timeline wins over
the rule, and the stored `reason` says `Source Code Override`. That is the
supported way to hand-correct one person without changing the model.

Accounts can ship additional models of their own with the same contract and the
same `model_<name>` prefix rule; they appear alongside these in every read and
comparison.

## 5. The effective date of an entry

Every rule above is a statement about **time** — *earliest* entry, *most recent*
acquisition. So a model's answer depends entirely on which date each entry
carries. That date is the entry's **effective date**: when the thing actually
happened, not when engine9 loaded it. It is stored as `timeline.ts`, and
`timelinePerson` shows it as `effective_date`.

The effective date is decided **once, at load time**, and models only read it. A
model never recomputes it. So if a model picked the "wrong" entry, check the
dates before questioning the rule — an entry with a bad effective date sorts to
the wrong end of history and quietly changes both first touch and last
acquisition.

**Cascade for a single entry**, first usable value wins:

| Order | Date used | Applies when |
| --- | --- | --- |
| 1 | the `ts` the source system gave the row | The system of record knows when it happened — send time, click time, gift date. Nearly every entry stops here. |
| 2 | the input's **default timestamp** (below) | The row has no `ts`, or it is a `SOURCE_CODE_OVERRIDE` entry whose `ts` is the placeholder `1970-01-01` (treated as "no date given") |
| 3 | none — the row does not become a timeline entry | Neither of the above produced a date |

Rows loaded from a file often have no per-row date. A **source code override**
file is the common case: it names a set of people and says "credit this source
code," with no date in the file at all. The date it should land on is the date of
the effort itself, so the default timestamp is derived from the source code.

**Cascade for the default timestamp**, first usable value wins:

| Order | Date used | Applies when |
| --- | --- | --- |
| 1 | the field named by the `extract_timestamp` option (`source_code_date` or `acquisition_date`) | Someone stated explicitly which date to use. If that field is missing or unparseable the load **fails** rather than guessing. |
| 2 | the source code's parsed `source_code_date` | No explicit choice, and the code contains a date that parsed cleanly — `EM_FR_20200114_appeal` → 2020-01-14 |
| 3 | the source code's `acquisition_date` | The code has no usable date of its own, but the format's parsed values or the dictionary record when the effort acquired people |
| 4 | the literal `default_timestamp` option | Given, and nothing above resolved |
| 5 | today's date | `default_timestamp` is `now` or omitted |

Two override values in the source code dictionary sit above the parsed value:
`source_code_date_override` and `acquisition_date_override` replace what was
parsed out of the code string. Use them when a code's embedded date is wrong or
unparseable, rather than editing history.

A date resolved this way is normalized to a UTC calendar day (`YYYY-MM-DD`) and
must be a real date on or after 1980-01-01. Missing, zero, or unparseable values
raise an error and the file fails to load. That strictness is deliberate: a
mis-parsed code should fail loudly rather than land silently at 1970 and become
everyone's "first touch".

Field-level detail, date format heuristics, and the exact option names:
[developers.md](developers.md#effective-date-resolution).

## 6. What a model writes

One `run` of one model produces four tables, named from its prefix:

| Table | One row per | Holds |
| --- | --- | --- |
| `{prefix}_person` | person | chosen `source_code_id`, `date_of_source`, `reason` |
| `{prefix}_transaction` | transaction | the credit for that transaction |
| `{prefix}_person_stats` | source code | `person_count` — people acquired |
| `{prefix}_transaction_stats` | source code | `transactions`, `revenue`, `refund_count`, `refund_amount`, `transaction_unique_person` |

So `model_first_touch_person`, `model_first_touch_transaction`,
`model_first_touch_person_stats`, `model_first_touch_transaction_stats`.

Amounts and dates stay on `transaction`; the readable code string lives in
`source_code_dictionary`. There is no shared cross-model table — one set per
model, on purpose, so models never overwrite each other. Column types:
[schema.md](schema.md).

## 7. Reading the answers

**Which efforts brought the most people?**

```sql
SELECT d.source_code, s.person_count
FROM model_first_touch_person_stats s
JOIN source_code_dictionary d ON d.source_code_id = s.source_code_id
ORDER BY s.person_count DESC
LIMIT 20;
```

**Lifetime value per person acquired** — all revenue those people have ever
given, divided by how many of them there are:

```sql
SELECT
  d.source_code,
  p.person_count,
  t.revenue,
  t.revenue / NULLIF(p.person_count, 0) AS lifetime_value_per_person
FROM model_first_touch_person_stats p
JOIN model_first_touch_transaction_stats t ON t.source_code_id = p.source_code_id
JOIN source_code_dictionary d ON d.source_code_id = p.source_code_id
ORDER BY t.revenue DESC
LIMIT 20;
```

**Acquisition ROI** — the same revenue against what the effort cost. Requires
`acquisition_cost` to be populated on the source code dictionary for that code;
where it is not, ROI cannot be computed for that code:

```sql
SELECT
  d.source_code,
  p.person_count,
  t.revenue,
  d.acquisition_cost,
  t.revenue / NULLIF(d.acquisition_cost, 0) AS roi
FROM model_first_touch_transaction_stats t
JOIN model_first_touch_person_stats p ON p.source_code_id = t.source_code_id
JOIN source_code_dictionary d ON d.source_code_id = t.source_code_id
WHERE d.acquisition_cost > 0
ORDER BY roi DESC
LIMIT 20;
```

**Model revenue next to attributed revenue** — two different questions in two
columns. Read them side by side; never sum them:

```sql
SELECT
  d.source_code,
  t.revenue AS model_revenue,        -- people this code acquired, all their giving
  scs.revenue AS last_click_revenue  -- gifts attributed to this code's messages
FROM model_first_touch_transaction_stats t
JOIN source_code_dictionary d ON d.source_code_id = t.source_code_id
LEFT JOIN source_code_summary scs ON scs.source_code_id = t.source_code_id
ORDER BY t.revenue DESC
LIMIT 20;
```

**Every model, one source code per row:** `compareSourceCodes` returns
`{prefix}_person_count` / `_revenue` / `_transactions` columns for each model
that has been run, so disagreement between models is visible in one grid. With
no arguments it picks each model's top codes by people and by revenue. Available
over MCP as `timelinePerson` with `command: compareSourceCodes`.

## 8. Checking one person

To see why a person got the credit they got, read their entries and each model's
stored conclusion together. MCP tool **`timelinePerson`** returns exactly that:
the person's `timeline` entries (with input, plugin, source code, and
transaction detail) plus each `model_*_person` row and its `reason`.

Read it in this order:

1. Are the entries you expect present at all? If an acquisition entry is
   missing, no model can pick it — that is a timeline loading question
   ([e9-timeline](../e9-timeline/SKILL.md)), not a model question.
2. Does the entry carry the source code you expect? If not, that is a source
   code question ([e9-source-code](../e9-source-code/SKILL.md)).
3. Is each entry's `effective_date` right, and do the entries sort the way you
   expect? A date resolved from a default or a mis-parsed source code puts an
   entry at the wrong end of history — see §5.
4. Given those entries in that order, did each model's rule pick correctly? The
   `reason` column says which branch fired. Only now is it a model question.

Stored `reason` and `date_of_source` reflect the last `run`; a model does not
re-decide when you read it.

## 9. Running a model

A model is not live — someone has to run it, and the tables hold whatever the
last run decided.

```javascript
const model = new ModelWorker(accountWorker);
await model.run({ model: '@engine9/plugins/models/first_touch' });
```

`run` streams the timeline, applies the rule, and writes that model's four
tables. Method list, test options, authoring a new model, and legacy tables:
[developers.md](developers.md).

## Vocabulary

| Term | Meaning |
| --- | --- |
| **entry** | One row on the timeline: one fact about one person at one time. Never say "event". |
| **transaction** | A payment. Never say "donation" in schema or field talk. |
| **source code** | The campaign/effort label on an entry. |
| **effective date** | When an entry actually happened (`timeline.ts`), resolved once at load time. What every model sorts by. |
| **model** | A rule that credits a person (and their transactions) to one timeline entry. |
| **attribution** | Connecting one transaction to one message. Not a model. |
| **prefix** | A model's table stem, e.g. `model_first_touch`. |
| **reason** | Why the model chose what it chose, stored per row. |

## Additional resources

- Running, authoring, testing, internals, legacy: [developers.md](developers.md)
- Table columns and payload shapes: [schema.md](schema.md)
- The timeline the model reads: [e9-timeline](../e9-timeline/SKILL.md)
- Source codes and **attribution** (transaction ↔ message): [e9-source-code](../e9-source-code/SKILL.md)
- How `person_id` is decided: [e9-person-id](../e9-person-id/SKILL.md)
