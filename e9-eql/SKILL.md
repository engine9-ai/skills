---
name: e9-eql
description: >-
  Write and run Engine9 Query Language (EQL) — expression fragments and SELECT
  query objects — primarily via the MCP `eql` tool (and `sql` compile_eql).
  Use when the user mentions EQL, buildSqlFromEQLObject, query objects with
  table/columns/conditions, or asks how to query account data through MCP.
---

# Engine9 EQL

EQL is Engine9’s SQL-like expression language plus a JSON **query object** that compiles to a SELECT. The primary way to submit EQL from Cursor is the engine9 MCP **`eql`** tool.

For MCP login, account scope, and tool-error handling, follow [e9-mcp](../e9-mcp/SKILL.md). For `/e9` / `/e9a`, see [e9-cli](../e9-cli/SKILL.md).

## When to use which MCP tool

| Intent | Tool |
|--------|------|
| Run a SELECT from an EQL **query object** | **`eql`** |
| Convert one EQL **expression** to a SQL fragment | `sql` with `command: "compile_eql"` |
| Run hand-written SQL | `sql` with `command: "query"` (or `sql` set) |
| Profile / summarize a table | `analyze` |
| Schema only | `sql` `describe` / `indexes` / `tables` |

Prefer **`eql`** over raw SQL when the user is describing a structured query (table + columns + filters). Prefer **`analyze`** when they want a profile, not a custom SELECT.

## MCP workflow (primary)

1. **Log in** — `mcp_auth` → user completes prompt → `ok` / `user` ([e9-mcp Step 0](../e9-mcp/SKILL.md#step-0--log-in-always-first)).
2. **Know `account_id`** — from `/e9a`, session scope, or `account` search. Do not guess.
3. **Optional discovery** — `sql` `tables` / `describe` / `analyze` to learn table and column names before writing EQL.
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

5. **Read the response** — `{ ok: true, sql, data, columns }`. The generated `sql` is useful for debugging; `data` / `columns` are the rows.

On MCP errors (`isError`, unauthorized, DB unreachable), **stop** — do not retry with `sql`/`task` as a workaround. See [e9-mcp tool errors](../e9-mcp/SKILL.md#mcp-tool-errors--stop-immediately).

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

## Query object shape

`buildSqlFromEQLObject` / MCP `eql` accept a non-array object:

| Field | Required | Notes |
|-------|----------|-------|
| `table` | yes | Base table name, or alias when `subquery` is set |
| `columns` | yes* | Array of column specs. Alias: `fields`. At least `["*"]` |
| `conditions` | no | AND’d WHERE clauses |
| `groupBy` | no | Group expressions / columns |
| `orderBy` | no | Columns or `{ column, orderByDirection }` (`asc`/`desc`) |
| `limit` / `offset` | no | Pagination |
| `joins` | no | `{ table, join_eql, alias?, type? }` — `type`: `inner` (default), `left`, `right`, `outer` |
| `subquery` | no | Nested EQL object; outer `table` is the alias for the subquery |

\* Legacy callers may use `fields` instead of `columns`.

`having` is **not** part of this MCP query-object builder (segment search may use it elsewhere). Express filters with `conditions` / `groupBy` + column EQL, or use raw `sql`.

## Column specs

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
- `{ eql: "..." }` **requires** `name` (alias).
- `*` / `{ "column": "*" }` selects all columns from the table.

## Condition specs

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

## Joins

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

## Expression language (samples)

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

## Sample query objects

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

## Agent checklist

- [ ] Authenticated via MCP; `account_id` known
- [ ] Prefer MCP `eql` for query objects; `sql` `compile_eql` for fragments
- [ ] `columns` is a non-empty array (use `["*"]` if needed)
- [ ] Every `{ eql: "..." }` column has a `name`
- [ ] Joins use objects with `table` + `join_eql` (not bare strings)
- [ ] Discover table/column names via `sql` / `analyze`, not local schema guesses when MCP is connected
- [ ] On tool error, stop and report — do not invent SQL workarounds

## Additional resources

- Expression samples: [examples.md](examples.md)
- MCP tool strategy: [e9-mcp](../e9-mcp/SKILL.md)
