---
name: e9-eql
description: >-
  Write and run engine9 Query Language (EQL) — expression fragments and SELECT
  query objects — primarily via the MCP `eql` tool (and `sql` compile_eql).
  Use when the user mentions EQL, buildSqlFromEQLObject, query objects with
  table/columns/conditions, asks how to query account data through MCP, or
  wants a SQL table analysis (min, max, distinct, samples) via MCP `analyze`.
---

# engine9 EQL

EQL is engine9’s SQL-like expression language plus a JSON query object that
compiles to a `SELECT`. Use this skill to build structured account queries,
compile expression fragments, or choose between EQL and adjacent data tools.
The primary way to submit an EQL query object from Cursor is the engine9 MCP
`eql` tool.

## Quick reference

| Intent | Tool |
|--------|------|
| Run a SELECT from an EQL **query object** | **`eql`** |
| Convert one EQL **expression** to a SQL fragment | `sql` with `command: "compile_eql"` |
| Run hand-written SQL | `sql` with `command: "query"` (or `sql` set) |
| Analyze a table (min, max, distinct, samples, date buckets) | **`analyze`** |
| Date histogram only | `sql` with `command: "histo"` |
| Schema only | `sql` `describe` / `indexes` / `tables` |

Prefer **`eql`** over raw SQL when the user is describing a structured query (table + columns + filters). Prefer **`analyze`** when they want mins, maxes, distinct counts, and samples.

## Workflow

1. **Log in** — `mcp_auth` → user completes prompt → `ok` / `user` ([e9-mcp Step 0](../e9-mcp/SKILL.md#step-0--log-in-always-first)).
2. **Know `account_id`** — from `/e9a`, this chat’s session scope, or `account`
   search.
3. **Optional discovery** — `sql` `tables` / `describe`, or MCP `analyze`, to learn table shape before writing EQL. See [Analyzing a table](#analyzing-a-table).
4. **Call `eql`**:

```json
{
  "account_id": "<account_id>",
  "eql": {
    "table": "transaction",
    "columns": ["*"],
    "conditions": [{ "eql": "amount > 10" }],
    "orderBy": [{ "column": "ts", "orderByDirection": "desc" }],
    "limit": 5
  }
}
```

5. **Read the response** — `{ ok: true, sql, data, columns, max_rows, row_count, truncated }`. The generated `sql` is useful for debugging; `data` / `columns` are the rows. MCP hard-caps at **10000** rows.

Rule: On MCP errors (`isError`, unauthorized, or database unreachable), stop.
Do not retry with `sql` or `task` as a workaround.

Rule: Do not guess account scope or read leftover CLI files such as
`.e9_parameters`; ask or suggest `/e9a` instead.

### Expression-only: `compile_eql`

When you need a fragment (not a full SELECT), use `sql`:

```json
{
  "account_id": "<account_id>",
  "command": "compile_eql",
  "table": "person",
  "eql": "YEAR(modified_at)"
}
```

Returns a cleaned SQL fragment and `refsByTable`. Do **not** use the `eql` tool for fragments alone.

## File format

### Query object shape

`buildSqlFromEQLObject` / MCP `eql` accept a non-array object:

| Field | Required | Notes |
|-------|----------|-------|
| `table` | yes | Base table name, or alias when `subquery` is set |
| `columns` | yes* | Array of column specs. Alias: `fields`. At least `["*"]` |
| `conditions` | no | AND’d WHERE clauses |
| `groupBy` | no | Group expressions / columns |
| `orderBy` | no | Columns or `{ column, orderByDirection }` (`asc`/`desc`) |
| `limit` / `offset` | no | Pagination. MCP `eql` defaults `limit` to **10000** and will not exceed that |
| `joins` | no | `{ table, join_eql, alias?, type? }` — `type`: `inner` (default), `left`, `right`, `outer` |
| `subquery` | no | Nested EQL object; outer `table` is the alias for the subquery |

\* Legacy callers may use `fields` instead of `columns`.

`having` is **not** part of this MCP query-object builder (segment search may use it elsewhere). Express filters with `conditions` / `groupBy` + column EQL, or use raw `sql`.

### Column specifications

Any of:

```json
"id"
"person.email as email"
{ "column": "id" }
{ "column": "ts", "name": "event_ts" }
{ "eql": "YEAR(modified_at)", "name": "year_modified" }
{ "eql": "count(id)", "name": "count" }
```

Rules:

- Plain strings without `(` are treated as column names (optional `table.column`).
- Strings with `(` are parsed as EQL expressions.
- `{ eql: "..." }` requires `name` as an alias.
- `*` / `{ "column": "*" }` selects all columns from the table.

Rule: Every `{ eql: "..." }` column specification must include `name`.

### Condition specifications

AND’d together. Prefer EQL strings for clarity:

```json
{ "eql": "amount > 10" }
{ "eql": "id in (1,2,3)" }
{ "eql": "email like '%@example.com'" }
```

Typed conditions (UI-style):

```json
{
  "type": "EQUALS",
  "values": [
    { "ref": { "column": "id" } },
    { "value": { "value": "3" } }
  ]
}
```

Supported `type` values: `EQUALS`, `NOT_EQUALS`, `LESS_THAN`, `LESS_THAN_OR_EQUAL`, `GREATER_THAN`, `GREATER_THAN_OR_EQUAL`, `LIKE`, `NOT_LIKE`, `CONTAINS`, `DOES_NOT_CONTAIN`, `IS_NULL`, `IS_NOT_NULL`.

### Joins

```json
{
  "table": "person",
  "joins": [
    {
      "table": "person_email",
      "join_eql": "person.id=person_email.person_id"
    },
    {
      "table": "person_email",
      "alias": "work_emails",
      "type": "left",
      "join_eql": "person.id=work_emails.person_id and work_emails.email_type='Work'"
    }
  ],
  "columns": [
    { "eql": "person.id", "name": "id" },
    { "eql": "person_email.email", "name": "email" },
    { "eql": "work_emails.email", "name": "work_email" }
  ],
  "limit": 25
}
```

Each join needs `table` + `join_eql`. Optional `alias` (defaults to table name) and `type`.

## Concepts

### Expression language

EQL expressions appear inside `columns[].eql`, `conditions[].eql`, `groupBy`, and `join_eql`. Syntax is SQL-like; the server dialectizes functions.

### Comparisons and logic

```
x < y
x = 1
x and y and not z
x between 1 and 2
1 in (x, y, 4, 5)
1 not in (x, y)
"test" like "t%"
x is null
x is not null
```

### Math and casts

```
sum(total) / count(*)
(test1.x + test2.y) * 2
cast(x as char)
cast(x as signed)
```

### Conditionals

```
if(x, y, z)
ifnull(x, y)
ifnull(x, y, z)
case when x = 1 then "test" else "hello" end
case x when 1 then "one" when 2 then "two" else "other" end
```

`if(...)` compiles to `case when ...`. `ifnull` with multiple args chains non-null checks.

### Dates

```
now()
getdate()
date_add(now(), interval 1 day)
date_sub(now(), interval 1 day)
now() + interval 1 day
now() - interval 1 day
YEAR(modified_at)
```

### Aggregates and distinct

```
count(*)
count(id)
sum(amount)
distinct x
```

### Bucket example (amount bands)

```
case when amount <= 20 then '$0-$20'
when amount > 20 and amount <= 50 then '$20-$50'
when amount > 50 and amount <= 100 then '$50-$100'
when amount > 100 then '$100+' end
```

More expression round-trips live in [examples.md](examples.md).

## Examples

### Simple select

```json
{
  "table": "person",
  "columns": ["id"],
  "limit": 1
}
```

### Filter + order + limit (MCP integration style)

```json
{
  "table": "transaction",
  "columns": ["*"],
  "conditions": [{ "eql": "amount > 10" }],
  "orderBy": [{ "column": "ts", "orderByDirection": "desc" }],
  "limit": 5
}
```

### Aggregates + groupBy

```json
{
  "table": "person",
  "columns": [
    { "eql": "YEAR(modified_at)", "name": "year_modified" },
    { "eql": "count(id)", "name": "count" }
  ],
  "conditions": [
    { "eql": "YEAR(modified_at) > 2020" },
    { "eql": "id in (1,2,3)" }
  ],
  "groupBy": [{ "eql": "YEAR(modified_at)" }]
}
```

### Subquery

Outer `table` is the alias for the nested SELECT:

```json
{
  "table": "subquery_1",
  "subquery": {
    "table": "person_email",
    "columns": [
      { "eql": "person_id", "name": "person_id" },
      { "eql": "count(*)", "name": "emails" }
    ],
    "groupBy": [{ "eql": "person_id" }]
  },
  "columns": [
    {
      "eql": "case when subquery_1.emails > 1 then 'multi-email' else 'one-email' end",
      "name": "level"
    },
    { "eql": "count(person_id)", "name": "people" }
  ],
  "groupBy": [
    {
      "eql": "case when subquery_1.emails > 1 then 'multi-email' else 'one-email' end"
    }
  ]
}
```

## Analyzing a table

MCP **`analyze`** analyzes one or more SQL tables: inferred types, min, max, empty counts, distinct counts, frequent samples, and (when an indexed datetime column exists) a full-table date histogram. Use it when the question is about the shape of a table. Schema-only questions stay on `sql` `describe` / `indexes` / `tables`. A histogram without the per-column analysis is `sql` `command: "histo"` (any datetime column; index not required).

Rule: Call `analyze` for a table analysis. Do not hand-write `MIN`, `MAX`, or `COUNT(DISTINCT)` to rediscover the same stats.

### Call

```json
{
  "account_id": "<account_id>",
  "table": "transaction",
  "max_tables": 1,
  "target_buckets": 10
}
```

| Argument | Required | Notes |
|----------|----------|-------|
| `account_id` | yes | Account whose warehouse to analyze |
| `table` | yes | Exact name or filter. `tables({ filter })` tries a regex first, then language tokens (`"ROI transaction"`) |
| `max_tables` | no | How many matches to analyze. Default **3**, max **10** |
| `target_buckets` | no | Approximate date-histogram bucket count when an indexed datetime column exists. Default **10** |

Response: `{ ok, filter, matched_tables, analyzed_tables, truncated, analyses }`. Each `analyses[]` entry is `{ table, records, table_records, columns }`. `table_records` is `count(*)`. Column stats live on `columns`. Histogram buckets are merged onto those same column objects.

### Column stats

| Field | Meaning |
|-------|---------|
| `name` | Column name |
| `type` | Inferred from sampled values: `string`, `int`, `bigint`, `double`, `decimal`, `date`, `datetime`, `uuid` |
| `min` / `max` | Smallest and largest values in the sample. When a histogram runs, every datetime column is replaced with full-table SQL min/max (see below) |
| `empty` | Sampled rows where the value was null or missing |
| `distinct` | Distinct values seen in the sample |
| `sample` | Up to **32** most frequent values in the sample, stringified |
| `min_length` / `max_length` | Present on string columns |
| `buckets` | On datetime columns when a histogram runs: `{ range, start, end, records, min, max }` |
| `bucket_column` | `true` on the column the histogram is cut on |
| `bucket_unit` | `day`, `week`, `month`, `quarter`, or `year` when buckets exist |

The histogram column is chosen in this order: `ts`, `created_at`, `modified_at`, otherwise the first indexed datetime column that leads an index. Every datetime column receives the same buckets, with that column’s own `min` / `max` inside each bucket. `bucket_column` is true only on the column used to cut the ranges. `analyze` only auto-runs the histogram when an indexed datetime leading column exists; for unindexed dates use `sql` `command: "histo"` with an explicit `column`.

### Sample vs full table

Rule: `table_records` is `count(*)` for the whole table.

Rule: `min`, `max`, `distinct`, `empty`, `sample`, and `records` come from a row sample of up to **10000** rows: **5000** from the start of the table plus **5000** highest primary-key rows when the table has a primary key, or **10000** rows when it does not. They are not full-table aggregates. A row that appears in both reads can be counted twice.

Rule: When the table has an indexed datetime column, `analyze` also runs a SQL date histogram. That histogram’s `MIN` / `MAX` replace `min` and `max` on every datetime column. `buckets[].records` are full-table counts for those ranges. `records` on the analysis itself stays the sample size. `table_records` stays `count(*)`.

`target_buckets` only changes the histogram. It does not change the row sample.

## Rules

Rule: Stop and report MCP tool errors; do not invent SQL workarounds.

Rule: Analyze a table with MCP `analyze`. Sample min/max are not full-table aggregates; indexed datetime min/max from the histogram are.

- [ ] Authenticated via MCP; `account_id` known
- [ ] Prefer MCP `eql` for query objects; `sql` `compile_eql` for fragments
- [ ] `columns` is a non-empty array (use `["*"]` if needed)
- [ ] Every `{ eql: "..." }` column has a `name`
- [ ] Joins use objects with `table` + `join_eql` (not bare strings)
- [ ] Discover table/column names via `sql` / `analyze`, not local schema guesses when MCP is connected
- [ ] Table analyses (min, max, distinct, samples, date buckets) use MCP `analyze`, including `target_buckets` when the histogram grain matters
- [ ] On tool error, stop and report

## Related documentation

| Topic | Documentation |
| --- | --- |
| Additional expression round-trips | [Examples](examples.md) |
| MCP login, account scope, and tool-error handling | [e9-mcp](../e9-mcp/SKILL.md) |
| `/e9` and `/e9a` command-style interactions | [e9-cli](../e9-cli/SKILL.md) |
