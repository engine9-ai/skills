# EQL expression samples

Fragments for `columns[].eql`, `conditions[].eql`, `groupBy`, `join_eql`, or MCP `sql` `command: "compile_eql"`.

## Operators and predicates

| EQL | Notes |
|-----|--------|
| `x < y` | Comparisons: `=`, `<>`, `!=`, `<`, `<=`, `>`, `>=` |
| `x and y and not z` | Boolean; also `or`, `&&`, `\|\|` |
| `x between 1 and 2` | Bounds may be expressions: `x between 1+3 and 2/x` |
| `1 in (x, y, 4, 5, x*3)` | `not in` supported |
| `"test" like "t%1"` | `not like` supported |
| `x is null` / `x is not null` | Null checks |

## Functions and math

| EQL | Compiles / notes |
|-----|------------------|
| `concat("test", x)` | Function call |
| `sum(total)/count(*)` | Aggregates in expressions |
| `test1.x + test2.y * 2` | Precedence: `*` before `+` |
| `(test1.x + test2.y) * 2` | Parentheses |
| `cast(x as char)` | Also `signed`, `unsigned`, `date`, `datetime`, `time`, `binary` |
| `distinct x` | Distinct operand |

## Conditionals

| EQL | Compiles roughly to |
|-----|---------------------|
| `if(x, y, z)` | `case when x then y else z end` |
| `if(1=1, null, 1)` | `case when (1 = 1) then null else 1 end` |
| `ifnull(x, y)` | `case when x is not null then x else y end` |
| `ifnull(x, y, z)` | Chain of non-null `when`s |
| `case when x=1 then "test" else "hello" end` | Searched CASE |
| `case x when 1 then "one" when 2 then "two" else "other" end` | Simple CASE |

## Dates

| EQL | Notes |
|-----|--------|
| `now()` / `getdate()` | Both → `now()` |
| `date_add("2019-01-01", interval 1 day)` | Add interval |
| `date_sub(now(), interval 1 day)` | Subtract |
| `now() + interval 1 day` | Same as `date_add` |
| `now() - interval 1 day` | Same as `date_sub` |
| `date_add(x, interval y day)` | Interval value may be a column/expr |
| `YEAR(modified_at)` | Dialect date part helpers (common in columns/groupBy) |

## Full MCP `eql` payloads

### Recent high-value transactions

```json
{
  "account_id": "test",
  "eql": {
    "table": "transaction",
    "columns": [
      { "column": "id" },
      { "column": "person_id" },
      { "column": "amount" },
      { "column": "ts" }
    ],
    "conditions": [
      { "eql": "amount > 10" },
      { "eql": "ts >= date_sub(now(), interval 30 day)" }
    ],
    "orderBy": [{ "column": "ts", "orderByDirection": "desc" }],
    "limit": 25
  }
}
```

### Person emails with join

```json
{
  "account_id": "test",
  "eql": {
    "table": "person",
    "joins": [
      {
        "table": "person_email",
        "join_eql": "person.id = person_email.person_id"
      }
    ],
    "columns": [
      { "eql": "person.id", "name": "id" },
      { "eql": "person_email.email", "name": "email" }
    ],
    "conditions": [{ "eql": "person_email.email like '%@gmail.com'" }],
    "limit": 50
  }
}
```

### Counts by year

```json
{
  "account_id": "test",
  "eql": {
    "table": "person",
    "columns": [
      { "eql": "YEAR(created_at)", "name": "year" },
      { "eql": "count(id)", "name": "people" }
    ],
    "conditions": [{ "eql": "id > 0" }],
    "groupBy": [{ "eql": "YEAR(created_at)" }],
    "orderBy": [{ "eql": "YEAR(created_at)", "orderByDirection": "asc" }]
  }
}
```

### `compile_eql` fragment

```json
{
  "account_id": "test",
  "command": "compile_eql",
  "table": "transaction",
  "eql": "sum(amount) / count(*)"
}
```

## Amount-band CASE (reporting)

```
case when amount<=20 then '$0-$20'
when amount>20 and amount<=50 then '$20-$50'
when amount>50 and amount<=100 then '$50-$100'
when amount>100 and amount<=250 then '$100-$250'
when amount>250 and amount<=500 then '$250-$500'
when amount>500 and amount<=1000 then '$500-$1000'
when amount>1000 and amount<=5000 then '$1000-$5000'
when amount>5000 then '$5000+' end
```

Use as a named column:

```json
{
  "eql": "case when amount<=20 then '$0-$20' when amount>20 and amount<=50 then '$20-$50' when amount>50 then '$50+' end",
  "name": "amount_band"
}
```
