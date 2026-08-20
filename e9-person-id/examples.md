# Person identity examples

These examples follow `assignPersonIds` in `@engine9/core/lib/id/index.js` after the extract transforms have filled `row.identifiers`. Compact accounts write `person_id_<id_type>`; legacy accounts write `person_identifier`. Semantics are the same.

## 1. Dedupe by email

**Input (two loads, hours apart):**

```js
{ email: 'Alice@Example.com', given_name: 'Alice', family_name: 'Anderson' }
// later:
{ email: 'alice@example.com', given_name: 'Alicia' }
```

**Extract** (`person_email:transforms:id`): trim, lowercase for hashing only (`alice@example.com`), SHA-256 → `email_hash_v1`. The stored `person_email.email` keeps the original trimmed spelling on first insert.

**First load:** no store hit → insert `person` (say `id = 41`) → insert lookup `email_hash_v1` → 41 → insert `person_email`.

**Second load:** same `email_hash_v1` hits 41. No new `person`. Name upsert updates `given_name` to `Alicia`. `person_email` finds the existing (email, 41) row and updates it (keeps original `source_input_id`).

```sql
SELECT person_id, email FROM person_email WHERE LOWER(email) = 'alice@example.com';
-- both loads → person_id 41
SELECT id, given_name FROM person WHERE id = 41;
-- given_name = 'Alicia'
```

A third row `{ email_hash_v1: '<same sha256>' }` with no plaintext also resolves to 41.

Emails shorter than 5 characters are not identifiers. The SHA-256 of an empty string is refused.

## 2. Dedupe by phone

**Input:**

```js
{ phone: '202-555-0143', given_name: 'Pat' }
{ cell: '12025550143', given_name: 'Patricia' }  // same person, later file
```

**Extract** (`person_phone:transforms:id`):

- Prefer `cell` / `mobile` / `mobile_phone` when `phone` is empty.
- Strip to digits/`+`. Length 10 → `+12025550143`. Length 11 starting with `1` → `+12025550143`.
- SHA-256 of that normalized string → `phone_hash_v1`.
- Persist `phone` as the normalized value.

Both rows share `phone_hash_v1` → one `person_id`. `person_phone` unique `(phone, person_id)` → one phone row (`+12025550143`).

```sql
SELECT person_id, phone, phone_hash_v1 FROM person_phone WHERE phone = '+12025550143';
```

**Short phones:** `1234567` (7 digits) is dropped — no `phone_hash_v1`, no `person_phone` row. Those records only dedupe if they have another identifier (email or `remote_person_id`).

Two inbound rows in the **same batch** with the same normalized phone share a `temp_id` and become one person even if the store is empty.

## 3. Dedupe by `remote_person_id`

**Input** (plugin `pluginId` = `aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee`):

```js
{ remote_person_id: '123425385', email: 'one@example.com' }
{ remote_person_id: '123425385', email: 'two@example.com' }  // later load
{ remote_person_id: 'AAAAAA', email: 'aaaaa@aaaaa.com' }
{ remote_person_id: 'aaaaaa', email: 'aaaaa@aaaaa.com' }     // same remote, case
```

**Extract** (`person_remote:transforms:id`): value becomes `{pluginId}.{remote_person_id}` lowercased, unless the inbound value already starts with a UUID + `.`.

`aaaaaaaa-…eeee.123425385` is one lookup key. Both email rows resolve to the same `person_id`. After assignment, **both** emails are upserted onto that person (`person_email` allows many emails per `person_id`). That is the current model: one CRM constituent, many addresses.

`AAAAAA` and `aaaaaa` are the same key after lowercasing → one person.

```sql
SELECT pr.person_id, pr.remote_person_id, pe.email
FROM person_remote pr
LEFT JOIN person_email pe ON pe.person_id = pr.person_id
WHERE LOWER(pr.remote_person_id) = '123425385';
-- one person_id, two emails
```

`pluginId` is required. A missing or non-UUID plugin id throws. Do not pass `remote_plugin_id` where `pluginId` is expected.

## 4. Many inputs, one `person_id`

One row can carry several keys. They are all attached to the person that wins:

```js
{
  remote_person_id: 'crm-99',
  email: 'pat@example.com',
  phone: '2025550143',
  given_name: 'Pat'
}
```

After extract, `identifiers` is roughly:

- `{ type: 'remote_person_id', value: '<pluginId>.crm-99' }`
- `{ type: 'email_hash_v1', value: '<sha256 of pat@example.com>' }`
- `{ type: 'phone_hash_v1', value: '<sha256 of +12025550143>' }`

If any key already maps to person 17, the row becomes 17 and the **new** keys are inserted for 17. Later, a transaction file with only `remote_person_id: 'crm-99'`, an email blast with only `pat@example.com`, or an SMS file with only `+12025550143` all resolve to 17.

Store lookup applies **`remote_person_id` hits first**, then email/phone. A row whose remote id already maps to 17 keeps 17 even if its email hash maps to someone else. The email lookup key is **not** remapped (first-wins).

## 5. Same email, different remotes (first-wins)

**Same batch:**

```js
{ remote_person_id: 'A', email: 'shared@example.com' }
{ remote_person_id: 'B', email: 'shared@example.com' }
```

They share `email_hash_v1`, so they share a `temp_id` → **one** `person_id`, two `person_remote` rows (`A` and `B`). Email overlap in-batch merges.

**Separate loads**, `A` first: `A` creates person 10 and owns `email_hash_v1`. Later `B` hits `remote_person_id` as new → creates person 11, tries to insert the email key, store ignores it (still 10). `person_email` may still grow a `(shared@example.com, 11)` attribute row because uniqueness is `(email, person_id)`, not email alone. Lookup by email continues to return **10**.

That is why “quality `person_id`” prefers loading a stable `remote_person_id` together with emails/phones on the first pass when the CRM knows they are one constituent.

## 6. `doNotUpsert` (identify only)

```js
await personWorker.processPeople({
  doNotUpsert: true,
  batch: [{ email: 'alice@example.com' }, { email: 'unknown@example.com' }]
});
// personIds: [41, null] — no new person, no new identifiers
```

Use this to resolve ids without mutating the account.
