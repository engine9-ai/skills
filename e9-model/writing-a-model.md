# Writing a model

Read [SKILL.md](SKILL.md) first. A model connects a person or transaction to timeline history and writes `{prefix}_*` tables (new identity). Do not write `source_code_summary.origin_*` — that is the legacy origin implementation. Last-click transaction ↔ message is attribution ([e9-source-code](../e9-source-code/SKILL.md)), not a model.

## Package

```
plugins/models/my_model/
  index.js
  person.transform.js
  transaction.transform.js   # optional
```

```javascript
import personTransform from './person.transform.js';

const metadata = {
  name: 'Transaction model: my_model',
  prefix: 'model_my_model',
  unique: true,
  version: '1.0.0'
};

export const transforms = { person: personTransform };
export default { metadata, transforms };
```

`prefix` **is** the warehouse stem. `run` creates `model_my_model_person`, `model_my_model_transaction`, and the two `_stats` tables. It must match `model_<name>` and stay stable across accounts. Do not rely on `plugin.table_prefix` (that gets a per-install counter).

Identity: `@engine9/plugins/models/my_model`, or `engine9-accounts/...` for account-specific models.

## Person transform

Input `batch[]`: `{ person_id, timelineEntries }`. Entries already have `entry_type` and dictionary labels when the source provided them.

Set `source_code`, `date_of_source`, `reason`. Do not set `source_code_id`.

## Transaction transform (optional)

Without one, every transaction inherits the person code. With one, each item has `transaction.transaction_id` (UUID) and `transaction.person_model_source_code`.

## File-only test

1. Tiny timeline file sorted by `person_id`.
2. `runPeople({ model, timeline: absPath })` — inspect the csv.
3. `runTransactions({ model, timeline: absPath, person_filename })` if needed.
4. Unit-test `transform({ batch })` with hand-built `timelineEntries` (see `server/test/workers/modelWorker.test.js`).
5. When the pick rule looks right: `run({ model, person_ids: '123' })` or `run({ model, people_limit: 25 })`, then `summarizeSourceCodes({ model })` and `summarizePeople({ person_ids: '123' })`.
