---
name: e9-funnel
description: >-
  Tentative skill: build a people funnel — a sequence of progressively smaller
  sets (segment/query → exported → batched/pushed → remote outcome) to find who
  should have been processed but was not. Use when diagnosing gaps in export →
  batch → push flows, “who is still missing a flag,” or audience drip pipelines.
  Not product code; operational analysis only.
---

# People funnel (tentative)

A **funnel** is a named sequence of progressively smaller sets of people (or emails). Each stage is a filter on the previous stage. The goal is to locate the first stage where expected people drop out — for example who matched a query, who landed in an export dataset, who was batched for upload, and who shows the remote outcome.

This skill is **tentative**. It documents a working pattern, not a finished product surface.

## Quick reference

| Stage idea | Typical source |
| --- | --- |
| A — universe | Warehouse EQL/SQL (segment, source code, transactions, …) |
| B — still needs action | A minus remote/warehouse outcome already true |
| C — exported | Input store `*.export.csv` for the flow’s `remote_input_id` |
| D — batched / pushed | Input store `batched/` via MCP `files` + `path: "batched"` |
| E — remote outcome | Vendor table / custom field / CRM flag |

Rule: Report each stage as a count (and grain: `person_id` vs `email`). Do not jump to the bottom stage without stating the intermediates.

## Concepts

### Grain

Export datasets often unique on **email**. Warehouse queries often unique on **person_id**. One person may have several emails; one export row is usually one email. Always state which grain each stage uses and whether a “missing” person still has another email that already passed.

### Export → batch → push

A common drip flow:

1. **ExportWorker.export** writes new unique keys into an `input_type: export` store (`unique_fields`, `dedupe_against`).
2. **InputWorker.batchUnprocessed** reads unprocessed rows and writes a batch file under `batched/` (ledger / `seen_records`).
3. A vendor method (for example `pushPeopleFile`) uploads that batch.

Funnel stages map onto those artifacts, then onto the remote field that should change after a successful push.

### MCP-only file access

Rule: For input-store files, use MCP `input` only. Do **not** use local `aws s3`, client AWS credentials, or shell S3 sync.

```json
{ "command": "files", "account_id": "<account_id>", "input_id": "<uuid>" }
{ "command": "files", "account_id": "<account_id>", "input_id": "<uuid>", "path": "batched" }
{ "command": "download", "account_id": "<account_id>", "input_id": "<uuid>", "filename": "batched/foo.batch.csv" }
```

Rule: Listed `name` is relative to the input store root (`batched/seen_records.txt`, not bare `seen_records.txt`). Pass that string as `filename` to `download` / `summary`.

Rule: `path` is a directory `name` from a prior listing (or omit / `""` for root). Response includes `path` + `directory`. Rejects `..`, absolute paths, and `s3://` URIs.

Rule: If MCP still cannot supply a stage after using `path`, stop. Do not invent child paths or fall back to AWS.

## Workflow

### 1. Name the stages

Write a short table before querying:

| Stage | Definition | Grain | How you will measure |
| --- | --- | --- | --- |
| A | … | person / email | EQL/SQL |
| B | A minus outcome | … | … |
| C | In export store | email | MCP `input` `files` + `download` / `summary` |
| D | Batched / pushed | email | MCP `files` with `path: "batched"`, then `download` |
| E | Remote outcome true | email | Warehouse vendor table / field |

### 2. Resolve the export input

```json
{ "command": "list", "account_id": "<account_id>", "input_type": "export", "q": "<remote_input_id fragment>" }
```

Use `input.id` for later `files` / `download`.

### 3. Measure A and B (warehouse)

Use MCP `eql` or `sql`. Prefer one query that returns distinct keys at the chosen grain. For “should have flag X but doesn’t,” compute A, then B = A anti-joined to the outcome table on the same grain (usually email).

### 4. Measure C (exported)

1. `input` `files` for that `input_id`.
2. For each `*.export.csv` (skip `.metadata.json5`), `download` or `summary`.
3. Union emails (or unique keys). Compare to A/B.

Metadata sidecars often include `people_searched`, `records`, and `deduplicated` — useful for stage totals when you do not need the full email list.

### 5. Measure D (batched / pushed)

1. Root `files` — note the `batched` directory entry.
2. `files` with `path: "batched"`.
3. `download` / `summary` each listed store-relative `name` (e.g. `batched/*.batch.csv`, `batched/seen_records.txt`).
4. Union keys for the stage count.

If `path` is unsupported or listing fails, stop for stage D and report C/E without inventing push membership.

### 6. Measure E (remote outcome)

Query the vendor/warehouse field that the push is supposed to set. Reconcile with B: people still in B after C (and D when available) are the operational gap.

### 7. Report the funnel

```markdown
| Stage | Definition | Count | Grain |
| A | … | n | person_id |
| B | … | n | email |
| C | … | n | email |
| D | … | n or blocked | email |
| E | … | n | email |

Gap: B − E (or B − C, …) with a one-line hypothesis.
```

## Rules

Rule: A funnel is analysis, not a new engine9 worker or schema.

Rule: Never name specific clients or accounts in this skill’s examples; use `<account_id>`, `<remote_input_id>`, `<input_id>`.

Rule: Stop when MCP cannot supply a stage. Partial funnels are valid.

Rule: Do not use local AWS/S3 tooling for account input stores.

Rule: Keep stages nested: each stage ⊆ previous stage at the agreed grain (after normalizing email case/trim).

## Examples

### Export drip vs remote custom field

1. **A** — people with a transaction matching a source-code pattern.  
2. **B** — emails in A whose vendor person row does not have `custom_… = 'y'`.  
3. **C** — emails in the export input store for that drip (`*.export.csv`).  
4. **D** — emails in `batched/` (`files` + `path: "batched"`, then download).  
5. **E** — emails with `custom_… = 'y'` on the vendor person table.

Typical reading: C ≈ A (export caught up) but B still large → push or remote apply lag; compare D to B/E for upload vs vendor apply.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| C ≪ A | Export not run, bad search EQL, or dedupe path wrong |
| C ≈ A but B large | Batches not pushed, wrong upload column, or vendor table stale |
| `download` rejects `batched/…` | Call `files` with `path: "batched"` first; use the listed store-relative `name` |
| Bare `seen_records.txt` rejected | Use `batched/seen_records.txt` (root-relative `name`) |
| Person counts ≠ email counts | Multi-email people; state grain explicitly |

## Related documentation

| Topic | Doc |
| --- | --- |
| MCP `input` `files` / `download` | [e9-mcp](../e9-mcp/SKILL.md#input) |
| Export packages and input stores | [e9-export](../e9-export/SKILL.md) |
| Input identity and stores | [e9-input](../e9-input/SKILL.md) |
| EQL / SQL queries | [e9-eql](../e9-eql/SKILL.md) |
