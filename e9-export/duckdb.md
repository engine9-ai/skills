# Query an export with DuckDB

This guide is for people who receive an engine9 export and want to query it on their own machine. You do not deploy engine9 or connect to an engine9 server. You need a plain [DuckDB](https://duckdb.org/docs/installation/) install (1.3 or later) and a copy of the export folder.

## The DuckDB file

Everything in this guide reads and writes **one DuckDB database file**:

```
{export_root}/engine9.duckdb
```

`{export_root}` is your local copy of the export: the folder that contains `inventory.json5`, `tables/`, and the input-store folders (`message/`, `timeline/`, …). The build step below creates `engine9.duckdb` in that folder. After that, open the file and run the example queries:

```
duckdb /path/to/export/engine9.duckdb
```

The build **copies** the data into the file. The finished `engine9.duckdb` is self-contained: you can move it, share it, or open it from Python, R, DBeaver, or any other DuckDB client. Each new export date is a new folder; rebuild a fresh `engine9.duckdb` in that folder.

## 1. Download the export

Copy the whole export root, not single files. For an S3 export:

```
aws s3 sync s3://<bucket>/<account_id>/exports/<export_id>/<date>/ ./export/
cd ./export
ls   # inventory.json5  tables/  message/ ...
```

`inventory.json5` is written last. If it is missing, the export did not finish.

## 2. Build `engine9.duckdb`

Run these commands **from the export root**. The paths in them are relative.

Load every warehouse table (`tables/{table}.parquet`) as a table of the same name:

```
for f in tables/*.parquet; do
  duckdb engine9.duckdb -c "CREATE OR REPLACE TABLE \"$(basename "$f" .parquet)\" AS SELECT * FROM read_parquet('$f')"
done
```

Then load the input-store rows and create the helper views:

```
duckdb engine9.duckdb <<'SQL'
-- Every .idv1.parquet row from every copied input store.
-- input_type and input_id come from the folder: {input_type}/{input_id}/{file}.
CREATE OR REPLACE TABLE input_timeline AS
SELECT
  COLUMNS(lambda c: c NOT IN ('filename', 'input_id', 'input_type')),
  string_split(filename, '/')[-3] AS input_type,
  string_split(filename, '/')[-2] AS input_id,
  filename
FROM read_parquet('*/*/*.idv1.parquet', union_by_name = true, filename = true);

CREATE OR REPLACE TABLE entry_type AS
SELECT * FROM (VALUES
  (10, 'TRANSACTION'), (11, 'TRANSACTION_ONE_TIME'), (12, 'TRANSACTION_INITIAL'),
  (13, 'TRANSACTION_SUBSEQUENT'), (14, 'TRANSACTION_RECURRING'), (15, 'TRANSACTION_REFUND'),
  (30, 'SMS_SEND'), (31, 'SMS_DELIVERED'), (33, 'SMS_CLICK'), (34, 'SMS_UNSUBSCRIBE'),
  (37, 'SMS_BOUNCE'), (38, 'SMS_SPAM'), (39, 'SMS_REPLY'),
  (40, 'EMAIL_SEND'), (41, 'EMAIL_DELIVERED'), (42, 'EMAIL_OPEN'), (43, 'EMAIL_CLICK'),
  (44, 'EMAIL_UNSUBSCRIBE')
) AS t(id, name);

-- One row per message. remote_id is the vendor's id for the message (the "campaign id").
CREATE OR REPLACE VIEW message_lookup AS
SELECT
  g.id,
  g.message_id,
  g.channel,
  g.bot_path,
  g.label,
  g.publish_date,
  coalesce(i.remote_input_id, g.remote_id) AS remote_id,
  nullif(g.final_primary_source_code, '') AS source_code,
  i.id IS NOT NULL AS has_input_store
FROM global_message_summary g
LEFT JOIN input i ON i.id = g.id;

-- One row per person with the preferred email and phone (preference_order 0 is first).
CREATE OR REPLACE VIEW person_contact AS
WITH e AS (
  SELECT person_id, email FROM person_email
  QUALIFY row_number() OVER (PARTITION BY person_id ORDER BY preference_order, modified_at DESC, id) = 1
), ph AS (
  SELECT person_id, phone FROM person_phone
  QUALIFY row_number() OVER (PARTITION BY person_id ORDER BY preference_order, modified_at DESC, id) = 1
)
SELECT p.id AS person_id, p.given_name, p.family_name, e.email, ph.phone
FROM person p
LEFT JOIN e ON e.person_id = p.id
LEFT JOIN ph ON ph.person_id = p.id;
SQL
```

The build expects the tables a typical message export ships: `person`, `person_email`, `person_phone`, `input`, `global_message_summary`, `source_code_dictionary`, and `transaction`. If `read_parquet('*/*/*.idv1.parquet')` finds no files, the export has no input stores; skip `input_timeline` and the views that use it.

Check what loaded:

```sql
SHOW TABLES;
SELECT input_type, count(DISTINCT input_id) AS stores, count(*) AS entries,
       min(ts) AS first_entry, max(ts) AS last_entry
FROM input_timeline GROUP BY 1;
SELECT e.name, count(*) FROM input_timeline t JOIN entry_type e ON e.id = t.entry_type_id GROUP BY 1 ORDER BY 2 DESC;
```

## 3. Example lists

These examples answer three common requests from a texting program. In the receiver's language a **message** is a **campaign**, and the campaign id is the message's `remote_id`. Each list has one row per person per campaign (or per activity), with email, phone, campaign id, and source code. A query after each one reduces it to one row per person.

Create the views once in `engine9.duckdb`:

```
duckdb engine9.duckdb <<'SQL'
-- People who received a message (SMS_SEND 30, EMAIL_SEND 40).
CREATE OR REPLACE VIEW campaign_participants AS
SELECT
  t.person_id,
  c.email,
  coalesce(t.phone, c.phone) AS phone,
  m.remote_id AS campaign_id,
  m.label AS campaign_name,
  m.source_code,
  min(t.ts) AS sent_at
FROM input_timeline t
JOIN message_lookup m ON m.id = t.input_id
LEFT JOIN person_contact c ON c.person_id = t.person_id
WHERE t.entry_type_id IN (30, 40)
GROUP BY ALL;

-- Everyone who opted out of text messages.
-- SMS_UNSUBSCRIBE entries carry the date and campaign; person_phone.sms_status adds
-- opt-outs recorded by other systems (no date or campaign).
CREATE OR REPLACE VIEW sms_opt_outs AS
SELECT
  t.person_id,
  c.email,
  t.phone,
  m.remote_id AS campaign_id,
  m.label AS campaign_name,
  m.source_code,
  t.ts AS opted_out_at,
  'SMS_UNSUBSCRIBE entry' AS evidence
FROM input_timeline t
JOIN message_lookup m ON m.id = t.input_id
LEFT JOIN person_contact c ON c.person_id = t.person_id
WHERE t.entry_type_id = 34
UNION ALL BY NAME
SELECT
  ph.person_id,
  c.email,
  ph.phone,
  NULL AS campaign_id,
  NULL AS campaign_name,
  NULL AS source_code,
  NULL::TIMESTAMP AS opted_out_at,
  'person_phone.sms_status' AS evidence
FROM person_phone ph
LEFT JOIN person_contact c ON c.person_id = ph.person_id
WHERE ph.sms_status = 'Unsubscribed'
  AND NOT EXISTS (
    SELECT 1 FROM input_timeline t
    WHERE t.entry_type_id = 34 AND t.person_id = ph.person_id AND t.phone = ph.phone
  );

-- Everyone with a transaction (any source) or an SMS link click.
-- A transaction's campaign is the message it is credited to (final_message_id), on any channel.
CREATE OR REPLACE VIEW donors_and_clickers AS
SELECT
  tr.person_id,
  c.email,
  coalesce(c.phone, tr.phone) AS phone,
  m.remote_id AS campaign_id,
  m.channel AS campaign_channel,
  m.label AS campaign_name,
  nullif(sc.source_code, '') AS source_code,
  'transaction' AS activity,
  tr.ts,
  tr.amount
FROM "transaction" tr
LEFT JOIN message_lookup m ON m.id = tr.final_message_id
LEFT JOIN source_code_dictionary sc ON sc.source_code_id = tr.final_source_code_id
LEFT JOIN person_contact c ON c.person_id = tr.person_id
WHERE tr.entry_type_id BETWEEN 10 AND 14
UNION ALL BY NAME
SELECT
  t.person_id,
  c.email,
  coalesce(t.phone, c.phone) AS phone,
  m.remote_id AS campaign_id,
  m.channel AS campaign_channel,
  m.label AS campaign_name,
  m.source_code,
  'sms_click' AS activity,
  t.ts,
  NULL AS amount
FROM input_timeline t
JOIN message_lookup m ON m.id = t.input_id
LEFT JOIN person_contact c ON c.person_id = t.person_id
WHERE t.entry_type_id = 33;
SQL
```

`phone` on an SMS entry is the number that received the text. When a file has no `phone` column, remove `t.phone` and use `c.phone`.

### Campaign participants

```sql
-- Find the campaign id
SELECT remote_id AS campaign_id, label, publish_date, source_code
FROM message_lookup WHERE channel = 'sms' AND has_input_store
ORDER BY publish_date DESC LIMIT 20;

-- Everyone who received one campaign
SELECT * FROM campaign_participants WHERE campaign_id = '<campaign_id>';

-- One row per person, across every campaign they received
SELECT person_id, any_value(email) AS email, any_value(phone) AS phone,
       list(campaign_id ORDER BY sent_at) AS campaign_ids,
       list(DISTINCT source_code) AS source_codes,
       count(*) AS campaigns_received
FROM campaign_participants GROUP BY person_id;
```

### Opt-outs

```sql
-- One row per person, most recent dated opt-out first
SELECT * FROM sms_opt_outs
QUALIFY row_number() OVER (PARTITION BY person_id ORDER BY opted_out_at DESC NULLS LAST) = 1;
```

`SMS_SPAM` (38) entries are complaints. Add `38` to the filter if those people must also be suppressed.

### Donors and clickers

```sql
-- One row per person, most recent activity
SELECT * FROM donors_and_clickers
QUALIFY row_number() OVER (PARTITION BY person_id ORDER BY ts DESC) = 1;

-- Only activity tied to a texting campaign
SELECT * FROM donors_and_clickers WHERE campaign_channel = 'sms';
```

A transaction with no `campaign_id` was not credited to a message. It still counts as a transaction.

### Write a CSV

```sql
COPY (SELECT * FROM sms_opt_outs) TO 'sms_opt_outs.csv' (HEADER);
```

## Related

- [Export contents and joins](SKILL.md)
- [Timeline entries and entry types](../e9-timeline/SKILL.md)
- [Source codes and attribution](../e9-source-code/SKILL.md)
- [Message warehouse views](../e9-global-message/SKILL.md)
