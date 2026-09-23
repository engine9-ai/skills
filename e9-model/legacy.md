# Legacy models

Legacy models are the old-identity scores: `person_model_source_code` (`model_id` + `source_code_id`) and `transaction_model_pivot`. Use this file when the user explicitly asked for legacy models — a legacy-vs-current person search, a Legacy tab, or a pivot comparison. Current `{prefix}_*` models and their person-search forms are in [SKILL.md](SKILL.md#person-search).

Rule: Do not open this file, recommend a legacy search, pass `legacy: true`, or call `timelinePersonLegacy` unless the user explicitly asked for legacy. `searchOptions` may still list `@engine9/plugins/models/legacy` when that plugin is installed; leave it out of the UI until legacy is requested.

## Quick reference

| Need | Use |
| --- | --- |
| People in a legacy model for a source code | `@engine9/plugins/models/legacy:search:personSourceCode` |
| Those people, and not in the matching current model | That clause, plus `exclude: true` on the current `personSourceCode` path |
| UI catalog | MCP `searchOptions` / `GET /data/search/options` — render `form`, submit `{ path, options, exclude? }` |
| Legacy person / timeline inspect | MCP `timelinePersonLegacy` |
| Pivot vs current stats | `timelinePerson` `compareSourceCodesLegacy` (opt-in) |

## Person search

Package `@engine9/plugins/models/legacy` exports one handler. It reads `person_model_source_code` and returns current `person.id` values. It is person-level credit, the same question as current `personSourceCode`, on the old identity.

The handler appears in `searchOptions` only after `@engine9/plugins/models/legacy` is installed. The UI renders the catalog entry as returned:

| Field | Value |
| --- | --- |
| `path` | `@engine9/plugins/models/legacy:search:personSourceCode` |
| `title` | `Legacy model source code` |
| `form.properties.modelId` | string, required. `enum` is `{ value, name }` objects (below) |
| `form.properties.sourceCode` | string, required. Exact code, or `LIKE` when the value contains `%` |
| `form.required` | `["modelId", "sourceCode"]` |

| `modelId` | Name | Current person search to exclude |
| --- | --- | --- |
| `1` | First Touch | `@engine9/plugins/models/first_touch:search:personSourceCode` |
| `2` | CRM Origin | `@engine9/plugins/models/crm_origin:search:personSourceCode` |
| `8` | Last Channel Acquisition | `@engine9/plugins/models/last_acquisition:search:personSourceCode` |

Other numeric `modelId` values are accepted (label falls through to `Model <id>`). Exclude a current handler only when the user names that current model; unknown legacy ids have no shipped twin.

`exclude` is not a form field. The UI sets it on the clause. `PersonWorker.getPersonSearchSQL` turns `exclude: true` into `id not in (...)`.

### In a legacy model, out of the current model

People credited to a source code by the legacy model, who are not credited to that same source code by the matching current model. Both clauses use the same `sourceCode`. The excluded handler is `personSourceCode` (person credit). `transactionSourceCode` answers a different question and is the wrong exclude target here.

```json
{
  "and": [
    {
      "path": "@engine9/plugins/models/legacy:search:personSourceCode",
      "options": { "modelId": "1", "sourceCode": "WEB_PET_2019" }
    },
    {
      "exclude": true,
      "path": "@engine9/plugins/models/first_touch:search:personSourceCode",
      "options": { "sourceCode": "WEB_PET_2019" }
    }
  ]
}
```

Submit that object as MCP `search` `search` (or `POST /data/search`). Swap `modelId` and the excluded path together using the table above (`2` with `crm_origin`, `8` with `last_acquisition`). A code containing `%` is `LIKE` on both clauses.

Catalog entry the UI binds the legacy control to:

```json
{
  "path": "@engine9/plugins/models/legacy:search:personSourceCode",
  "title": "Legacy model source code",
  "description": "People credited in person_model_source_code (legacy model_id). Bridged to current person.id via person_email / person_metadata.",
  "form": {
    "title": "Legacy model source code",
    "type": "object",
    "properties": {
      "modelId": {
        "title": "Model",
        "description": "Legacy model_id",
        "type": "string",
        "enum": [
          { "value": "1", "name": "First Touch" },
          { "value": "2", "name": "CRM Origin" },
          { "value": "8", "name": "Last Channel Acquisition" }
        ]
      },
      "sourceCode": {
        "title": "Source code",
        "description": "Exact source code, or a LIKE pattern when it contains %",
        "type": "string"
      }
    },
    "required": ["modelId", "sourceCode"]
  }
}
```

### Identity

Legacy rows use `person_id_int` from `person_metadata`. The search bridges `person_email.email` → `person_metadata.person_id` → `person_model_source_code.person_id_int` → `source_code_dictionary`.

Rule: `person_metadata.person_id` is usually the email string (not `email_hash_v1`). `person_id_int` is not `person.id`. People with no matching email on `person_metadata.person_id` do not match, even when the legacy row exists.

## Legacy inspect and pivot stats

These reads live in `server/workers/model/legacy.js` and `server/workers/model/compare.js`. They do not write `{prefix}_*`.

| Surface | Call |
| --- | --- |
| Person / account legacy timeline | MCP `timelinePersonLegacy` (`ModelWorker.inspectPersonLegacy`) |
| One person's legacy score | `summarizePeopleLegacy({ emails \| person_ids })` |
| Current vs legacy person, paired by email | `comparePeopleLegacy` |
| Pivot rows for source codes | `summarizeSourceCodesLegacy({ source_codes })` |
| Pivot vs current delta | `compareSourceCodesLegacy({ source_codes })` |
| Current compare that also includes pivot columns | `compareSourceCodes({ legacy: true })` |

Tables: `person_model_source_code`, `person_model_source_code_summary`, `timeline_v3_summary`, `transaction_model_source_code`, `transaction_model_pivot`, `person_metadata`, `transaction_metadata`, and `source_code_summary.origin_*`.

Rule: Legacy timeline inspect reads `timeline_v3_summary`. The type column there is `entry_type_label`. Do not select `entry_type` from that view.

`emails` is the identity bridge: `person_email` → current `person.id`; SHA-256 of the trimmed lowercase email (`email_hash_v1`) → `person_metadata.person_id`; if the hash misses, the email string on that column. `comparePeopleLegacy` can also take explicit `person_ids` plus `legacy_person_ids`. Person `match` is source-code equality (empty counts as missing). Missing legacy tables are skipped.

`compareSourceCodesLegacy` returns `{ legacy, current, delta, match }` per shared metric for stems `first_touch`, `crm_origin`, and `last_acquisition`, plus any extra `{stem}_*` columns present on `transaction_model_pivot` (custom legacy models; omitted when absent). Shared metrics: `person_count`, `transactions`, `revenue`, `refund_count`, `refund_amount`, `transaction_unique_person`. The pivot also has `incipient_*`. `source_codes` is required; a token containing `%` is `LIKE`.

`compareSourceCodes({ legacy: true })` adds those pivot stems beside current `model_*_stats`. When both current first touch and legacy first touch are deployed, an omitted `source_codes` list also unions the top 10 source codes by absolute person-count difference (`top` key `first_touch_vs_legacy`). Conductor opts in with `MODEL_COMPARE_INCLUDE_LEGACY` and, for person inspect, `TIMELINE_PERSON_INCLUDE_LEGACY` (a second MCP call, not a flag on `inspect`).

Future deployments drop the pivot table and `timelinePersonLegacy`.

## Related documentation

- Current models and current person-search forms: [SKILL.md](SKILL.md)
- Running `ModelWorker`: [developers.md](developers.md)
- Current table DDL: [schema.md](schema.md)
- Package contract for the legacy search plugin: `plugins/models/legacy/README.md`
- MCP `search` / `searchOptions`: [e9-mcp](../e9-mcp/SKILL.md#search)
