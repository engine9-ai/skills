---
name: e9-reports
description: "Author and consume engine9 reports: JSON dashboards with EQL-backed StatCard, ComposedChart, and Table widgets. Definitions are installed with plugins or hosted as portable JSON. ReportWorker compiles declarative filters and start/end into SQL. Use when adding reports/, defining filters, running via MCP/HTTP with path or definition, or rendering dashboards."
---

# engine9 reports

Reports are **JSON dashboards**. The usual source is an installed plugin; the same JSON can be hosted elsewhere and passed to `run` as `definition`. `ReportWorker` turns run options into EQL conditions and compiles/runs SQL. Consumers pass `path` **or** `definition` plus filter options and render `sections` data — they never write SQL.

Use this skill when defining a report, wiring filters to EQL, executing it through supported surfaces, or implementing a consumer for report results.

## Quick reference

| Task | Contract |
|------|----------|
| Install | Plugin exports `reports.<key>` and `default.reports` |
| List | Catalog of **installed plugin** reports plus merged JSON Schema `filters` |
| Get | Definition selected by report `path` |
| Run | `path` **or** portable JSON `definition` + options → `sections[].components[].data` and debug SQL |
| Hosted | Artifact (or any host) fetches HTTPS JSON, then MCP `run` with `definition` |
| Samples | Simplified messaging JSON in [samples/messaging](samples/messaging/README.md), usable as a remote report URL |
| Render | Preserve section and component order |

## Workflow

### Install with a plugin

Path format: `@engine9/plugins/reports/<area>:reports:<key>` (same triple as transforms/search). Interfaces do not ship reports.

Put files under `reports/` and export a **keyed object on the default plugin export** (`compilePlugin` reads `mod.default.reports`):

```javascript
// reports/subscription_status.js
export default {
  name: 'Email Subscription Status',
  description: 'Counts by source plugin.',
  tags: ['Email'],
  data_sources: {
    default: { table: 'person_email' }
  },
  filters: {
    title: 'Filters',
    type: 'object',
    properties: {
      plugin_name: {
        type: 'string',
        title: 'Plugin name',
        filter: { column: 'plugin.name', operator: 'LIKE' }
      }
    }
  },
  sections: [
    {
      title: 'Breakdown by source plugin',
      components: [
        {
          id: 'by_plugin',
          component: 'Table',
          query: { table: 'person_email', columns: ['count(*) as emails'] }
        }
      ]
    }
  ]
};
```

```javascript
// plugins/reports/people/index.js
import subscription_status from './reports/subscription_status.js';
export const reports = { subscription_status };
export default { metadata, reports };
```

Ship only reports that belong on that plugin. Email send/engagement and fundraising dashboards live on `@engine9/plugins/reports/messaging` (`email`, `email_transactions`) because they query `global_message_summary` with `channel='email'`. Subscription counts and person-created charts live on `@engine9/plugins/reports/people`.

## Concepts

### Definition shape

| Field | Required | Notes |
|-------|----------|--------|
| `schema_version` | no | Portable JSON version. Omit or `1`. Unknown versions are rejected on `run` with `definition` |
| `name` | yes | Catalog title |
| `description` | no | Catalog subtitle |
| `tags` | no | String tags for catalog grouping |
| `data_sources.default` | for metric/stat/chart components | `{ table, date_column?, conditions? }` — `date_field` is accepted as an alias of `date_column` |
| `filters` | no | JSON Schema of accepted run variables. Merged with standard `start`/`end` (when a date column exists) and `limit`/`offset` |
| `sections` | yes | Ordered layout: `{ title?, components: [{ id, component, … }] }` |

Rule: Definitions are JSON only. Static predicates belong in `data_sources.conditions`. Run-time filters belong in `filters.properties` with `filter: { column, operator }`.

#### Layout: sections

Reports use an ordered **`sections`** array. Each section is a visual group; optional `title` is the heading. Widgets live under `components` with stable **`id`** values used for run-result binding (`sql[].id`, UI keys).

```javascript
sections: [
  {
    title: 'Key metrics',
    components: [
      { id: 'sent', component: 'StatCard', name: 'Sent', metric: { eql: 'sum(sent)' } },
      { id: 'open_rate', component: 'StatCard', name: 'Open Rate', metrics: [/* … */] }
    ]
  },
  {
    title: 'Emails by Date',
    components: [
      {
        id: 'opens_clicks',
        component: 'ComposedChart',
        isDate: true,
        dimension: { eql: 'publish_date' },
        metrics: [
          { name: 'Sent', eql: 'sum(sent)' },
          { name: 'Opened', eql: 'sum(impressions)' },
          { name: 'Clicked', eql: 'sum(clicks)' }
        ]
      }
    ]
  }
]
```

| Rule | Why |
|------|-----|
| Array order = render order | Explicit, stable |
| `section.title` for headings | Titles are layout, not widgets |
| Stable `id` per widget | Run `sql[]` and consumers key off `id` |
| One section per visual group | Stat cards that belong together share a section |

#### Components

| `component` | Role | Typical fields | `data` after run |
|-------------|------|----------------|------------------|
| `StatCard` | Single KPI | `name`, `metric` or `metrics[]` | One row |
| `ComposedChart` | Time series / bars / mixed | `isDate`, `dimension`, `metrics[]` | Many rows; x is `dimension.name` or `dimension_name` |
| `Table` | Grid | `dimensions[]`, `metrics[]`, or a full `query` | Many rows |

Metric / dimension objects:

```javascript
{ name: 'Open Rate', eql: 'sum(impressions)/sum(sent)', format: 'percent' }
```

`format` (display hint; SQL still returns numbers): `percent`, `currency`, `number`, `date`. Optional chart fields: `yaxis` (`left`/`right`), `type` (`bar`/`line`/`area`).

Two ways to specify the query:

1. **Data source + metrics** — worker builds `table` / `columns` / `groupBy` / date grouping from `metric(s)`, `dimension(s)`, `isDate`.
2. **Explicit `query`** — full EQL object (`table`, `joins`, `columns`, `groupBy`, `orderBy`). Date + filter options are **appended** to `query.conditions`.

Rule: Use only `StatCard`, `ComposedChart`, and `Table`.

## Rules

### Options → conditions

Run callers pass variables. `ReportWorker` turns them into EQL `conditions` on **every** component query. Nothing on the client invents WHERE clauses.

### 1. Declare accepted variables in `filters`

`list` / `get` return the **merged** JSON Schema (`mergeReportFilters`):

| Source | Keys |
|--------|------|
| Auto when `data_sources.*.date_column` is set | `start`, `end` (ISO or relative: `-3M`, `-30d`, `now`) |
| Always | `limit` (default 1000, max 10000), `offset` |
| Report author | Extra `filters.properties` |

Example — accept a channel and rely on the date column for range:

```javascript
data_sources: {
  default: {
    table: 'global_message_summary',
    date_column: 'publish_date'
  }
},
filters: {
  type: 'object',
  properties: {
    channel: {
      type: 'string',
      title: 'Channel',
      enum: ['email', 'sms', 'push'],
      filter: { column: 'channel', operator: '=' }
    }
  }
}
```

### 2. Call run with those variables

Surfaces accept the same bag of options (nested `options` and/or top-level aliases):

```json
{
  "path": "@engine9/plugins/reports/messaging:reports:email",
  "start": "-3M",
  "end": "now",
  "channel": "email",
  "limit": 1000
}
```

- **Worker:** `run({ path, options })` or `run({ definition, options })` or flattened `start` / `end` / `limit` / `offset`
- **HTTP:** `GET|POST /data/reports/run` — query string and/or JSON body (`path` **or** `definition` + options)
- **MCP:** `report` `command: run` — `path` **or** `definition`; extra keys (e.g. `plugin_name`) merge into options

Rule: `path` and `definition` are mutually exclusive.

`flattenRunOptions` merges nested `options` with top-level `start`/`end`/`limit`/`offset`/`days`.

### 3. Worker builds conditions (per component)

For each widget, `buildComponentQuery` concatenates, in order:

1. **Static** `data_sources.*.conditions` (and optional component `conditions`)
2. **Dates** — if `start`/`end` are set and a `date_column` is known:  
   `publish_date >= '<start ISO>'` (inclusive) and `publish_date < '<end ISO>'` (exclusive). Relative values go through `relativeDate`.
3. **Declarative filters** — for each `filters.properties` entry that has `filter: { column, operator }` and a non-empty option value, emit an EQL condition.  
   `operator`: `=` (default), `LIKE` (wraps `%value%` unless `%` already present), `IN`, `GREATER_THAN` / `>`, `LESS_THAN` / `<`.  
   Reserved / not mapped this way: `start`, `end`, `limit`, `offset`, `days` (dates use step 2; limit/offset set query paging).

Then SQL is compiled with `buildSqlFromEQLObject` and executed.

**Walkthrough:** `channel=email`, `start=-3M`, `end=now` on a report with `date_column: 'publish_date'` and `filter: { column: 'channel' }`:

| Option | Mechanism | Condition |
|--------|-----------|-----------|
| `start` / `end` | `getDateConditions` + `date_column` | `publish_date>='…'` / `publish_date<'…'` |
| `channel` | `conditionsFromFilterSchema` | `channel='email'` |
| (none) | static `data_sources.conditions` if any | e.g. `publish_date is not null` |

If a predicate is not a run option (always true for that dashboard), put it in `data_sources.conditions` instead of `filters`.

Rule: Use `filter: { column }` for run-time option predicates.

Reports only append conditions; each component already owns its table and columns.

`days` (optional on run) picks date-chart grouping: year (>1200d), month (>365), week (>180), else day. If omitted, `start`+`end` span is used when both are set.

### Date-aware engagement example

```javascript
export default {
  name: 'Email Engagement',
  description: 'Sends, opens, clicks, unsubs, bounces.',
  tags: ['Email'],
  data_sources: {
    default: {
      table: 'global_message_summary',
      date_column: 'publish_date',
      conditions: [{ eql: "channel='email' and publish_date is not null" }]
    }
  },
  sections: [
    {
      title: 'Key metrics',
      components: [
        { id: 'sent', component: 'StatCard', name: 'Sent', metric: { eql: 'sum(sent)' } },
        {
          id: 'open_rate',
          component: 'StatCard',
          name: 'Open Rate',
          metrics: [
            { name: 'Open Rate', eql: 'sum(impressions)/sum(sent)', format: 'percent' },
            { name: 'Opened', eql: 'sum(impressions)' }
          ]
        }
      ]
    },
    {
      title: 'Emails by Date',
      components: [
        {
          id: 'opens_clicks',
          component: 'ComposedChart',
          isDate: true,
          dimension: { eql: 'publish_date' },
          metrics: [
            { name: 'Sent', eql: 'sum(sent)' },
            { name: 'Opened', eql: 'sum(impressions)' },
            { name: 'Clicked', eql: 'sum(clicks)' }
          ]
        }
      ]
    }
  ]
};
```

(Email channel is fixed in `data_sources.conditions` here. To make channel a run variable, move it into `filters.properties` with `filter: { column: 'channel' }` and drop the hardcoded predicate.)

### List, get, and run

SQL is generated only in `ReportWorker` (`compileReport` → `buildSqlFromEQLObject`).

| Surface | List | Definition | Execute |
|---------|------|------------|---------|
| Worker | `list()` | `get({ path })` | `run({ path, options })` or `run({ definition, options })` |
| HTTP (`X-ENGINE9-ACCOUNT-ID` + session) | `GET /data/reports` | `GET /data/reports/get?path=` | `GET` or `POST /data/reports/run` (`path` or POST `definition`) |
| MCP | `report` `command: list` (default) | `command: get` | `command: run` with `path` or `definition` |

Rule: Do not hardcode report maps. Prefer native MCP `report` over `task` for interactive queries.

### Hosted (third-party) definitions

Plugin `path` is a locator for installed modules. A host can also load the same JSON from HTTPS and pass it as `definition`.

Rule: The artifact host fetches the definition. MCP does not fetch URLs. SQL stays in `ReportWorker`.

| Step | Who | What |
|------|-----|------|
| Load | Host (`fetch` HTTPS, or http on localhost) | Portable JSON (`schema_version` 1, no functions) |
| Run | MCP `report` `command: run` | `{ definition, options, start, end }` |
| Render | `@engine9/components` `Report` | Same `sections` + `data` as a plugin run |

Conductor: `/report https://…` sets `definitionUrl`. CORS is the publisher’s problem. `list` stays installed plugins only.

Published samples (simplified `@engine9/plugins/reports/messaging` dashboards) live in this skill so they can be fetched from GitHub after they land on `main`:

```
https://raw.githubusercontent.com/engine9-ai/skills/main/e9-reports/samples/messaging/email.json
```

```
/report https://raw.githubusercontent.com/engine9-ai/skills/main/e9-reports/samples/messaging/email.json
```

Use the `raw.githubusercontent.com` URL (JSON), not a GitHub `blob` HTML page. Consumer and developer notes for every file: [samples/messaging/README.md](samples/messaging/README.md).

Rule: `run` with `definition` rejects unknown `schema_version` and component types other than `StatCard` / `ComposedChart` / `Table`. Caps: 32 sections, 32 components, 200KB.

### List payload

```json
{
  "account_id": "<account_id>",
  "reports": [
    {
      "path": "@engine9/plugins/reports/messaging:reports:email",
      "name": "Email Engagement",
      "description": "…",
      "tags": ["Email"],
      "plugin": { "path": "@engine9/plugins/reports/messaging", "instances": [{ "id": "…", "name": "…" }] },
      "filters": {
        "title": "Filters",
        "type": "object",
        "properties": {
          "start": { "type": "string", "format": "date-time", "title": "Start" },
          "end": { "type": "string", "format": "date-time", "title": "End" },
          "limit": { "type": "integer", "default": 1000 },
          "offset": { "type": "integer", "default": 0 }
        },
        "required": []
      },
      "sections": [
        {
          "title": "Key metrics",
          "components": [
            { "id": "sent", "component": "StatCard", "name": "Sent" },
            { "id": "open_rate", "component": "StatCard", "name": "Open Rate" }
          ]
        }
      ]
    }
  ],
  "errors": []
}
```

`errors[]` is per-plugin `compilePlugin` failure; the rest of the catalog still returns.

### Run payload

Each widget gains `data`, `sql`, `query`, `seconds`. Top-level `sql` is `[{ id, table, sql, error, seconds }]`. Section titles are not executed.

Rule: Unknown component values must render as placeholders. Do not invent SQL outside `ReportWorker`.

### Widget contract for consumers

| Widget | Input | Output |
|--------|--------|--------|
| Catalog | `list.reports[]` | Links by `path` |
| Filter form | `filters` JSON Schema | `options` object |
| Section | `{ title?, components[] }` | Optional heading + children |
| StatCard | `{ name, metric(s), data[0], format }` | KPI; if two metrics, primary + secondary |
| ComposedChart | `{ isDate, dimension, metrics, data }` | X = `dimension.name` or `dimension_name`; series = `metrics[].name` |
| Table | `{ dimensions, metrics, data }` or column names from `data[0]` | Table; apply `format` when present |

Rule: Preserve section order and component order within each section.

## Examples

### MCP

```json
{ "account_id": "<account_id>" }
```

```json
{
  "command": "run",
  "account_id": "<account_id>",
  "path": "@engine9/plugins/reports/people:reports:subscription_status",
  "plugin_name": "email"
}
```

```json
{
  "command": "run",
  "account_id": "<account_id>",
  "definition": {
    "name": "People count",
    "data_sources": { "default": { "table": "person" } },
    "sections": [
      {
        "components": [
          { "id": "people", "component": "StatCard", "name": "People", "metric": { "eql": "count(*)" } }
        ]
      }
    ]
  }
}
```

Hosted sample (host fetches the URL, then `run` with `definition`):

```
https://raw.githubusercontent.com/engine9-ai/skills/main/e9-reports/samples/messaging/email.json
```

### Authentication constraints

Today reports run as:

- **MCP `report`** — Firebase / session / `localdev`
- **HTTP `/data/reports*`** — same user session as the rest of `/data`

Rule: An `e9key_` cannot run reports through these paths today. Do not implement or document API-key execution as if it were supported.

Recommendation only: add a future `reports:read` or `data:read` scope on `GET/POST /data/reports*` for machine clients while keeping SQL generation in `ReportWorker`.

## Troubleshooting

### Checklist for a new report

- [ ] File under `reports/<key>.js`; key is the path tail
- [ ] `name`, `description`, `tags`, `sections` with stable component `id`s
- [ ] `data_sources.default.table` (and `date_column` if callers should pass `start`/`end`)
- [ ] Extra run variables as JSON Schema + `filter: { column, operator }`
- [ ] Component names `StatCard` / `ComposedChart` / `Table` only
- [ ] Exported on `reports` **and** `default.reports`
- [ ] Documented in the package `README.md` (path, who it is for, filters)

## Related documentation

- [engine9 MCP](../e9-mcp/SKILL.md)
- [engine9 EQL](../e9-eql/SKILL.md)
- [Global message view grain and metrics](../e9-global-message/SKILL.md)
- [API key capabilities](../e9-api-key/SKILL.md)
- [Sample messaging reports](samples/messaging/README.md) (GitHub-hosted JSON, remote report URLs)
- Reference implementation: `plugins/reports/people/reports/subscription_status.js`
- Reference implementation: `plugins/reports/messaging/reports/email.js`
