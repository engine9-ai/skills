---
name: e9-person-id
description: >-
  Explains engine9 person identity: how a quality person_id is chosen from
  many inputs (email, phone, remote_person_id, delegate), the lookup tables
  (person_id_* compact vs deprecated person_identifier), attribute tables
  (person, person_email, person_phone, person_remote), assignPersonIds /
  loadPeople / processPeople, and first-wins dedupe. Covers the old
  person_metadata / person_id_int / timeline_v3 / transaction_metadata model
  that did not support multiple emails or phones per person. Use when working
  with person_id, person_identifier, person_id_email_hash_v1,
  person_id_phone_hash_v1, person_id_remote_person_id, email_hash_v1,
  phone_hash_v1, remote_person_id, loadPeople, assignPersonIds, identity,
  dedupe, or duplicate people — not source_code_id or other non-person ids.
---

# engine9 person identity (`person_id`)

The overarching goal of the person identity framework is to identify a quality **`person_id`** from many inbound inputs. A row may arrive with an email, a phone, a CRM `remote_person_id`, a delegate unid, an existing `person_id`, hashes only, or several of those at once. engine9 extracts match keys, looks them up, and stamps one integer `person_id` on the row. Downstream tables (`timeline`, `transaction`, segments) never invent identity — they consume that `person_id`.

This skill is **exclusively about `person_id`**. It is not about `source_code_id`, message ids, timeline entry ids, or other identifiers. For source codes see [e9-source-code](../e9-source-code/SKILL.md). For the person activity log that consumes `person_id`, see [e9-timeline](../e9-timeline/SKILL.md).

Worked email / phone / `remote_person_id` examples: [examples.md](examples.md).

## Canonical identity

`person.id` **is** `person_id`: a numeric autoincrement on the `person` table. Every other person-related table stores that integer as `person_id`.

Names (`given_name`, `family_name`) and addresses are **attributes**, not match keys. Two people can share a name. Identity matching uses only the identifier types below.

One person may have **many** emails and **many** phones. That is the current model. The old one-email-equals-one-person world lives in [Old identity model](#old-identity-model-person_metadata-person_id_int-timeline_v3-transaction_metadata).

## Tables

Two layers: a **lookup** from match key → `person_id`, and **attribute** rows hanging off that `person_id`.

### Identity lookup (match key → `person_id`)

This is the core lookup. The current optimization is one compact table per identifier type:

| Table | `id_type` | Role |
|-------|-----------|------|
| `person_id_email_hash_v1` | `email_hash_v1` | Email hash → `person_id` |
| `person_id_phone_hash_v1` | `phone_hash_v1` | Phone hash → `person_id` |
| `person_id_remote_person_id` | `remote_person_id` | Plugin-scoped remote id → `person_id` |
| `person_id_delegate` | `delegate` | Delegate unid → `person_id` |
| `person_id_<id_type>` | any safe `id_type` | Same shape for additional types |

Compact schema: `value` (16-byte hash, primary key) + `person_id`. No plaintext, no `source_input_id`. First insert wins (`INSERT OR IGNORE` / `INSERT IGNORE`). Tables are created on demand when that `id_type` is first written.

`value` is **not** `email_hash_v1` itself. Extract already hashed the email/phone to a hex `id_value`; the compact store hashes that string again: first 16 bytes of SHA-256(`id_value` UTF-8). `id_type` is the table name, not mixed into the hash. Rare collisions are accepted. For `remote_person_id` the `id_value` is the plugin-prefixed string, then the same 16-byte truncate.

**`person_identifier` is fundamentally similar but lower performing, so it is being deprecated, but can still be used.** Same semantics (`id_type` + `id_value` string → `person_id`), plus `source_input_id`. MySQL accounts still default to it until migrated. SQLite / D1 default to compact `person_id_*` and leave `person_identifier` empty.

Do not assume `person_identifier` is populated. Check `identifier_store_kind` (core plugin setting) or which tables have rows. See [Identifier stores](#identifier-stores).

### Person attributes (many per person)

Written **after** `person_id` is assigned. Unique keys allow multiple emails/phones/remotes on one person.

| Table | Unique-ish key | Role |
|-------|----------------|------|
| `person` | `id` | Canonical row: `given_name`, `family_name`, `created_at` |
| `person_email` | `(email, person_id)` | Emails, subscription/confirmation, `email_hash_v1` |
| `person_phone` | `(phone, person_id)` | Phones, SMS/call status, `phone_hash_v1` |
| `person_remote` | `(source_input_id, remote_person_id, person_id)` | Per-plugin CRM / vendor person id |
| `person_address` | (not an identity key) | Postal address; upserted in the same pipeline, never used to match |

`person_hash_email` / `person_hash_phone` (optional `@engine9/interfaces/person_hash`) store hashes without plaintext. They are **not** in the default inbound chain.

### Downstream consumers

`timeline`, `transaction`, `person_segment`, and similar tables store `person_id` after the people pipeline has run. They are not identity sources.

## Identifier types

Inbound rows do not match on raw email/phone strings in the lookup tables. Extract transforms push `{ type, value, path }` onto `row.identifiers`. `assignPersonIds` matches those.

| `id_type` | `id_value` | Extracted from | Notes |
|-----------|------------|----------------|-------|
| `email_hash_v1` | SHA-256 hex of trimmed **lowercase** email | `email`, or inbound `email_hash_v1` | Skip if email shorter than 5 chars. Refuse the SHA-256 of `''`. |
| `phone_hash_v1` | SHA-256 hex of normalized phone | `phone`, or `cell` / `mobile` / `mobile_phone`, or inbound `phone_hash_v1` | Digits only; US 10-digit → `+1…`; 11-digit starting `1` → `+1…`; already-`+` kept. Min 8 digits. |
| `remote_person_id` | `{pluginId}.{remote_person_id}` lowercased | `remote_person_id` | If the value already starts with a UUID + `.`, it is used as-is (lowercased). Requires `pluginId`. |
| `delegate` | trimmed lowercase unid | `delegate_id` | Core `PersonWorker.extractDelegateIdentifiers`; not an interface package. |

`appendPersonId` lowercases and strips accents on every identifier `value` before lookup (`NFD` + combining marks). Matching is case- and accent-insensitive.

The blank SHA-256 `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` is never stored. Extract skips it; `assignPersonIds` throws if it appears.

Code: `@engine9/interfaces/person_email|person_phone|person_remote/transforms/inbound/extract_identifiers.js`, `@engine9/core/lib/id/index.js`.

## Inbound pipeline

Shared by server `PersonWorker.loadPeople` (streams/files) and core `PersonWorker.processPeople` (in-memory). Built by `buildInboundTransforms` in `@engine9/core/lib/peoplePipeline/getInboundTransforms.js`.

```
beforeAll (extra)
  → person.normalizeFieldNames          # lowercase keys, strip punctuation
  → person_remote:transforms:id         # remote_person_id identifier
  → person_email:transforms:id          # email_hash_v1
  → person_phone:transforms:id          # phone_hash_v1
  → beforeIdentity (delegate; optional person_hash)
  → person.appendInputId
  → person.appendPersonId               # assignPersonIds — THE identity step
  → person.appendEntryTypeId
  → person.validateSourceCodeAscii / appendSourceCodeId   # not person identity
  → beforeUpsert
  → upsert person, person_remote, person_email, person_phone, person_address
  → afterAll
```

`doNotUpsert` / `do_not_upsert`: run extracts + lookup only. Existing people resolve; unknown rows stay without `person_id`; no inserts.

Identity assignment is **blocking** (lookups and inserts must serialize across threads). `InputWorker.id` calls `loadPeople` so Timeline ID files already have `person_id` before they hit `timeline`.

Inspect store mode:

```sql
SELECT value FROM setting
WHERE plugin_id = '00000000-0000-4000-a000-000000000001'
  AND name = 'identifier_store_kind';
-- compact | legacy | missing (then SQLite/D1 → compact, MySQL → person_identifier)
```

## How `assignPersonIds` works

`@engine9/core/lib/id/index.js`. One batch, one store.

1. **Refuse** blank SHA-256 identifier values.
2. **Within-batch person_id**: if any row already has `person_id` (caller-supplied), copy it onto other rows that share an identifier.
3. **Within-batch temp_id**: rows that share any identifier get the same `temp_id`, so they become one person even if the store has never seen them.
4. **Store lookup** of every `{ id_type, id_value }`.
5. **Apply hits without overwriting**: `remote_person_id` matches first, then other types. If `person_id` is already set, a later hit is ignored. First assignment wins for that row.
6. **Insert `person`** rows for unmatched `temp_id`s (unless `doNotUpsert`). Optional `created_at` from `date_created` / `remote_date_created` / `frakture_date_created` / `created_at`.
7. **Insert identifier mappings** that the store did not already have. Existing keys are skipped (**first-wins**, never remapped).

Consequences:

- One inbound row with email + phone + remote id attaches **all three** keys to the same `person_id`. A later row matching **any** of those keys resolves to that person (unless a higher-priority already-set `person_id` / `remote_person_id` won first).
- Identifier mappings are immutable. A second person created via a new `remote_person_id` does not steal an email/phone key that already points at someone else. `person_email` uniqueness is `(email, person_id)`, so the plaintext email **can** appear on more than one person; the **lookup** key still points at the first `person_id`.
- Caller-supplied `person_id` is respected and seeds the store for new keys.

After assignment, upserts write attributes. Email upsert looks up existing `person_email` by address; same person+email updates (keeps original `source_input_id`); new pair inserts. Default new subscription is `Subscribed`; explicit unsubscribe (or `EMAIL_UNSUBSCRIBE`) updates **all** `person_email` rows with that address. Phone upsert is analogous (min 8 digits; default SMS `Not Subscribed`).

## Identifier stores

`createDefaultIdentifierStore` (`@engine9/core/lib/id/storeKind.js`):

1. Durable Object binding (`worker.personIds` / `PERSON_IDS`) always wins (Cloudflare).
2. Else setting `identifier_store_kind` on the core plugin (`00000000-0000-4000-a000-000000000001`): `compact` or `legacy` (`person_identifier` accepted as `legacy`).
3. Else dialect default: **SQLite / D1 → compact**; **MySQL → `person_identifier`**.

`findPersonIdentifierHashCollisions` scans the legacy table for 16-byte hash collisions before migrate. `PersonWorker.populatePersonRemoteFromIdentifiers` backfills `person_remote` from `person_identifier` rows with `id_type = 'remote_person_id'` (strips the `{pluginId}.` prefix into `person_remote.remote_person_id`).

**Migrate:** `PersonWorker.migratePersonIdentifiersToCompact` pages `person_identifier`, writes compact tables (first-wins), renames `person_identifier` to `person_identifier_legacy_<timestamp>`, sets `identifier_store_kind=compact`. `dry_run` counts only.

## Old identity model (`person_metadata`, `person_id_int`, `timeline_v3`, `transaction_metadata`)

`person_metadata`, `person_id_int`, `transaction_metadata`, and `timeline_v3` all use an **OLD identity model that did not account for multiple emails or phones per person**.

In that model the person key was a single integer `person_id_int`, and `person_metadata.person_id` was often the **email string** (legacy loaders filter `meta.person_id like '%@%'`). One email ≈ one person. Phones were not first-class match keys. `timeline_v3` joined `person_metadata` **using (`person_id_int`)**. `transaction_metadata` sat in the same integer-person space.

`PersonWorker.unwindPersonMetadata` is the bridge: copy `person_metadata.person_id_int` into `person.id`, then delete those legacy metadata rows. `LocalDatabasePersonWorker.importFromTimelineV3` still reads `timeline_v3` joined to `person_metadata` and feeds emails into the **current** people pipeline.

When you see `person_id_int` or `person_metadata.person_id` as an email, you are in the old model. Do not join it to current `person_email` as if it were `person.id`. Resolve through the current lookup (`person_id_*` / `person_identifier`) and attribute tables instead.

Legacy `source_code_summary.origin_*` rollups were computed in that old identity world. Current models write `{prefix}_*` tables with bigint `person_id` — [e9-model](../e9-model/SKILL.md).

## Debugging identity

Prefer attribute tables (plaintext) over compact BLOBs. `DESCRIBE` first; filter to one example; LIMIT.

```sql
-- Email → person_id (current model)
SELECT person_id, email, subscription_status, email_hash_v1
FROM person_email
WHERE LOWER(email) = 'alice@example.com'
LIMIT 20;

-- Phone → person_id (phones stored normalized, often +1…)
SELECT person_id, phone, phone_type, sms_status, phone_hash_v1
FROM person_phone
WHERE phone IN ('+12025550143', '2025550143')
LIMIT 20;

-- Remote CRM id → person_id (value is usually NOT plugin-prefixed in this table)
SELECT pr.person_id, pr.remote_person_id, pr.source_input_id, i.plugin_id
FROM person_remote pr
JOIN input i ON i.id = pr.source_input_id
WHERE LOWER(pr.remote_person_id) = '123425385'
LIMIT 20;

-- Canonical person
SELECT id, given_name, family_name, created_at FROM person WHERE id = ?;
```

On **legacy** (`person_identifier`) accounts:

```sql
SELECT id_type, id_value, person_id, source_input_id
FROM person_identifier
WHERE id_type IN ('email_hash_v1', 'phone_hash_v1', 'remote_person_id', 'delegate')
  AND person_id = ?
LIMIT 50;
```

On **compact** accounts, `person_identifier` may be empty or renamed. Lookup tables are `person_id_email_hash_v1`, `person_id_phone_hash_v1`, `person_id_remote_person_id`. The `value` column is a 16-byte hash of the identifier `id_value`, not the email string — use `person_email` / `person_phone` / `person_remote` for human debugging.

Duplicate-people symptoms: same email on two `person_id`s (lookup first-wins; attribute table allows both), missing `remote_person_id` extract (`pluginId` required), short phone dropped (< 8 digits), blank hash refused, or still reading `person_metadata` / `timeline_v3` as if they were current.

## Additional resources

- Worked dedupe examples: [examples.md](examples.md)
- Person activity log: [e9-timeline](../e9-timeline/SKILL.md)
- Troubleshoot People tickets: [e9-troubleshoot](../e9-troubleshoot/SKILL.md)
