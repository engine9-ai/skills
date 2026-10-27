---
name: e9-pii-free
description: >-
  Make an engine9 account PII-free (limited-pii): no plaintext names, email,
  phone, or postal address in the warehouse, with SHA-256 hashes for matching
  instead. Covers what PII-free means, the exclude_pii setting (the one check
  core and the server read), turning it on for a new account, the server
  pii-removal flow for an existing account (plan, hash capture, uninstall,
  verify), hash capture alone via backfillPersonHash, and troubleshooting.
  Use when someone asks for a PII-free, limited-pii, hash-only, or no-plaintext
  account, exclude_pii, or asks why names, emails, or phones still appear on
  such an account.
---

# PII-free accounts (limited-pii)

A PII-free account stores no plaintext name, email, phone, or postal address in its warehouse. People are matched and found by SHA-256 hashes (`person_hash_email`, `person_hash_phone`) and plugin remote ids, so the account can still dedupe, segment, attribute, and report. Use this skill to turn the feature on, remove leftover plaintext from an existing account, verify it, and troubleshoot.

## Quick reference

| Question | Answer |
| --- | --- |
| What makes an account PII-free | `exclude_pii` is `true` on `@engine9/schemas/utilities/limited-pii`. That is the one check core and the server read for installs, imports, exports, and reports |
| What new-account setup installs | With the setting on, `installDefaultPlugins` installs the stack `@engine9/schemas/stacks/limited-pii` (an install list: `person_hash` and no plaintext plugins) |
| What is stored instead | `person_hash_email`, `person_hash_phone` (SHA-256 `*_hash_v1`, MD5 `*_hash_md5`), plus `person_remote` ids |
| Plugins that must not be installed | `person_name`, `person_email`, `person_phone`, `person_address`, `transaction/contact_details`, and `stacks/standard` |
| New account (MCP) | `plugin install` the utility, `plugin setSetting` `exclude_pii` `true`, then `installDefaultPlugins` (or `bootstrapAccount` with `exclude_pii: true` on the server) |
| Existing account | Schedule built-in flow `pii-removal`, review the plan, confirm, verify. See [Existing account](#existing-account-pii-removal) |
| Hash search without removal | Task `PIIWorker.backfillPersonHash` (installs `person_hash` and backfills) |
| Regular audit | Built-in flow `pii-audit` (`PIIWorker.buildPiiSummaryFile`) writes `<account root>/pii/summary.json.gz`; Conductor `/pii` renders it. See [Regular audit](#regular-audit-pii-audit) |
| Check it | `plugin settings`, `plugin upgrade` dry run (`pii_free: true`), `sql tables` filter `person_`, `PIIWorker.verifyRemoval` after a removal |
| Look people up | By hash, or by `remote_person_id`. `lookup` by name matches nobody |

| Plugin path | Role |
| --- | --- |
| `@engine9/schemas/utilities/limited-pii` | Holds `exclude_pii`. Installs no tables |
| `@engine9/schemas/stacks/limited-pii` | Install list only. Installs `plugin`, `person`, `person_remote`, `segment`, `source_code`, `timeline`, `transaction/core`, `person_hash`. Does not refuse installs by itself |
| `@engine9/schemas/person_hash` | Hash match keys. Part of the stack. Installable alone |

## Concepts

### The setting

`exclude_pii` on `@engine9/schemas/utilities/limited-pii` is the only answer to "is this account PII-free?" Core and the server read it for:

- Install refusal (plaintext plugins and `stacks/standard`)
- `installDefaultPlugins` choosing the limited-pii stack
- Skipping leftover plaintext plugin rows in the inbound weave
- Default export table lists and inventory

The limited-pii stack is what new-account setup installs when the setting is on. Libraries do not detect the stack, and the stack has no `exclude` list. Removing plaintext from an account that already has it is a reviewed server process, not flipping the setting alone.

**Different rows, not a flag in the load.** When a load starts it reads the `plugin` table and weaves the inbound steps of whatever is installed. A PII-free account has `person_hash` steps and no plaintext steps (and with the setting on, leftover plaintext rows are skipped). See [the inbound pipeline](../../core/lib/peoplePipeline/README.md).

**Hashing.** Email is trimmed and lowercased, then SHA-256 hashed to `email_hash_v1`. Phone is normalized to digits (`+1` for US ten-digit numbers), then hashed to `phone_hash_v1`. MD5 (`*_hash_md5`) is computed only from plaintext and is empty forever for hash-only sources. A source that already supplies `email_hash_v1` / `phone_hash_v1` is accepted as is. Hashes are pseudonymous match keys, not anonymous: anyone holding an email can compute its hash and test for a match.

**What the setting stops.** It stops new installs of plaintext plugins and removes plaintext steps from the load. It does not delete anything that already exists. Existing plaintext is hashed, then removed, by the `pii-removal` flow.

## What is PII-free and what is not

| Data | PII-free account |
| --- | --- |
| `person` (`id`, `created_at`, `modified_at`) | Kept |
| `person_hash_email`, `person_hash_phone` | Kept (hashes) |
| `person_id_email_hash_v1`, `person_id_phone_hash_v1`, `person_id_remote_person_id` | Kept (hashes and vendor ids) |
| `person_remote` | Kept (plugin-scoped vendor ids). Review if a vendor id is an email |
| `timeline`, `transaction` (core columns), source codes, models, global message tables | Kept |
| `person_name`, `person_email`, `person_phone`, `person_address` | Not installed |
| `transaction` contact columns (`given_name`, `family_name`, `street_1`, `street_2`, `city`, `region`, `postal_code`, `country`, `email`, `phone`, `employer`, `occupation`) | Not installed |
| `PersonWorker.search` `given_names` / `last_names` | Match nobody |
| `PersonWorker.search` `emails` / `phones` | Hash the query and match `person_hash_*` |
| Outbound `appendName`, `{{given_name}}` | Empty. There is nothing to merge |
| Default export | Omits the four plaintext person tables. Includes the hash tables |

Rule: Hashes are not a substitute for consent or policy review. Tell the account owner they are pseudonymous.

## Workflow

### New account

Two steps, in this order. Sign in first ([e9-mcp](../e9-mcp/SKILL.md)).

```
- [ ] plugin install  @engine9/schemas/utilities/limited-pii   (returns the plugin id)
- [ ] plugin setSetting  plugin_id=<that id>  name=exclude_pii  value=true
- [ ] installDefaultPlugins (MCP task, or e9 pluginworker installDefaultPlugins -a <account_id>)
- [ ] Verify
```

```json
{ "command": "install", "account_id": "<account_id>", "path": "@engine9/schemas/utilities/limited-pii" }
{ "command": "setSetting", "account_id": "<account_id>", "plugin_id": "<utility plugin id>", "name": "exclude_pii", "value": true }
```

```bash
e9 pluginworker installDefaultPlugins -a <account_id>
```

On the server, `bootstrapAccount({ exclude_pii: true })` does both steps (utility + setting, then default plugins).

Rule: Set `exclude_pii` before `installDefaultPlugins`. The setting is what steers the install and blocks plaintext plugins. The stack is only the install list.

### Existing account: `pii-removal`

Server only. Core sites that already hold plaintext are out of scope for this flow; they can start PII-free (new account) and respect the setting, but removal/uninstall/backfill live on `PIIWorker`.

Hashes are captured before any plaintext is dropped. MD5 exists only where plaintext existed. Hashes are pseudonymous.

```
- [ ] Schedule flow_id "pii-removal" (first task paused for review)
- [ ] Review the plan (names and counts only). Set owner choices on createRemovalPlan if needed:
      discard_names, drop_findings ("table" or "table.column")
- [ ] Confirm: un-pause / run removePii with the plan hash
- [ ] verifyRemoval
```

```json
{ "action": "schedule", "account_id": "<account_id>", "flow_id": "pii-removal" }
```

`createRemovalPlan` writes a plan file with `plan_hash`. Sections include plaintext plugin rows, `person` name columns, hash-capture counts, legacy tables, other scan findings, raw input-store files (report only), and blockers. Approval is the plan hash passed to `removePii` / `verifyRemoval`.

`removePii` order: install `person_hash` → turn on `exclude_pii` → migrate legacy `person_identifier` to compact if needed → `backfillPersonHash` → gate (nothing still missing) → `uninstallPlugin` each plaintext plugin with `drop` → `upgrade` with `drop_name_columns` (and `discard_names` when chosen) → drop legacy and approved findings → `installDefaultPlugins`.

MCP `plugin` `uninstall` calls `PIIWorker.uninstallPlugin` for a single plugin (account admin). Options: `id` or `path`, `drop`, `dry_run`.

Rule: Do not drop a table, column, or plugin row without the owner's approval of the plan hash. Report the plan first.

Rule: Never pass `discard_names` unless the owner chose it. Names on `person` have nowhere else to go on a PII-free account.

Rule: Raw input-store files are reported only. Deleting them breaks future identity rebuilds and is a separate owner decision. Backfilling hashes from raw files is only through a separate identity rebuild.

### Hash capture alone

Any server account can run `backfillPersonHash` without removing anything: it installs `person_hash`, backfills from `person_email` / `person_phone` / `transaction` contact columns / `person_identifier` hash rows, and search and segments by hash work immediately.

```json
{
  "action": "schedule",
  "account_id": "<account_id>",
  "path": "@engine9/plugins/e9workers:PIIWorker",
  "method": "backfillPersonHash",
  "options": { "dry_run": true }
}
```

Run again without `dry_run` to write. Re-runs write nothing new. `missing` is the gate the removal flow uses before drops.

### Verify

| Check | Call | Expect |
| --- | --- | --- |
| Setting | `plugin` `settings` `{ "account_id": "<account_id>" }` | `exclude_pii` `value: true` on the utility |
| Installed plugins | `account` `search` with `ids` and `include_plugins: true` | `person_hash` and the stack present. None of the five plaintext paths |
| Upgrade view | `plugin` `upgrade` with `dry_run: true` | `pii_free: true`, no `refresh.failed` for refused plaintext plugins |
| Tables | `sql` `tables` with `filter: "person_"` | `person`, `person_remote`, `person_hash_email`, `person_hash_phone`, `person_id_*`. No `person_name` / `person_email` / `person_phone` / `person_address` |
| `person` columns | `sql` `describe` `table: "person"` | `id`, `created_at`, `modified_at` only |
| `transaction` columns | `sql` `describe` `table: "transaction"` | None of the contact columns above |
| After removal | `PIIWorker.verifyRemoval` with the plan path and hash | Scan clean for planned drops; hash counts at least the plan's hashable distinct people; identity lookup by a known email still finds the same `person_id` |

Rule: `pii_free: true` means the setting is on. It does not prove the data is gone. Always run the table and column checks (and `verifyRemoval` after a removal).

### Regular audit (`pii-audit`)

`PIIWorker.buildPiiSummaryFile` scans every base table with the same sampling and classifier as `scanTables` and writes `<account root>/pii/summary.json.gz`. It is read only and works on any account, PII-free or not. Schedule the built-in flow `pii-audit` on a recurring basis to catch PII that new plugin loads bring in.

```
task schedule { "account_id": "<account_id>", "flow_id": "pii-audit" }
```

| Field | Meaning |
| --- | --- |
| `exclude_pii` | The account setting at audit time. When `true`, every `pii` column is a problem |
| `summary` | `table_count`, `tables_by_status`, `tables_with_pii`, `columns_with_pii`, `columns_to_review`, `pii_types` (columns per type) |
| `tables_with_pii` | Table name to its PII column names, the same shape as `scanTables` |
| `tables[].status` | `error` (scan failed; the audit continues), `pii`, `review`, `empty` (no rows sampled), `clean` |
| `tables[].columns[].status` | `pii` (counted as PII), `review` (a weaker non-identifier finding), `insufficient_data`, `clean` |
| `tables[].columns[].findings` | Classifier type, status, confidence, `examined`, `matching_rows`, reason codes. Never examples or values |

Read it with `GET /data/pii/summary` (204 until the first audit), MCP `file` `pii/summary.json.gz`, or Conductor `/pii`. Run the audit on demand with `POST /data/pii/build` or the Run audit button.

Rule: The summary holds table and column names, statuses, and counts only. Do not add sample values to it.

## Rules

Rule: `exclude_pii` is the one check. The limited-pii stack is only what new-account setup installs.

Rule: Never install `@engine9/schemas/stacks/standard`, `person_name`, `person_email`, `person_phone`, `person_address`, or `transaction/contact_details` on a PII-free account. With `exclude_pii` on, core refuses them.

Rule: Do not read names, emails, or phones to prove a leak. Report column names, counts, and a masked pattern, not the values.

Rule: Do not add a plugin that writes plaintext (custom fields, vendor tables) to a PII-free account without the owner's review.

## Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| Plaintext still lands after the stack was installed | `exclude_pii` is off, or a plaintext plugin row still exists and the setting is off. Turn the setting on (weave skips leftovers) and remove the row/table via `pii-removal` or `plugin` `uninstall` |
| `Cannot install <path>: excluded by plugin setting exclude_pii` | Working as designed. Do not install it. Use hashes |
| `exclude_pii` is not in `plugin settings` | The utility is not installed. Install `@engine9/schemas/utilities/limited-pii` |
| `lookup` by name returns nothing | Expected. Names are not stored. Look up by email hash, phone hash, or `remote_person_id` |
| `{{given_name}}` renders empty | Expected. Nothing to merge |
| Default export still has `person_email` | `exclude_pii` is off, or the caller passed an explicit `tables` list |
| `drop_name_columns` is blocked | The account is PII-free and people have a name on `person`. The owner must choose `discard_names` |
| `upgrade` dry run shows `pii_free: false` after the stack was installed | The setting is `false` or the utility is missing. The stack alone is not PII-free |
| `removePii` stopped at the hash gate | `backfillPersonHash` dry run still reports `missing` rows. Re-run the backfill; do not drop plaintext until `missing` is zero |
| Hash search finds nobody after a manual table drop | `person_hash_*` was never backfilled. Run `backfillPersonHash` before dropping plaintext, or use `pii-removal` |

## Related documentation

- [Person hash plugin](../../schemas/person_hash/README.md)
- [Inbound pipeline and the weaver](../../core/lib/peoplePipeline/README.md)
- [Install plugins on accounts](../e9-plugin/SKILL.md)
- [Upgrades and `drop_name_columns`](../e9-upgrade/SKILL.md)
- [Person identity](../e9-person-id/SKILL.md)
- [Exports](../e9-export/SKILL.md)
- [Custom fields](../e9-custom-fields/SKILL.md)
- [Task API / flows](../e9-tasks-api/SKILL.md)
