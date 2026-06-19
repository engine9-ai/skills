---
name: create-engine9-plugin
description: >-
  Describes how to implement Engine9 interface packages (@engine9/interfaces/*)
  and native plugins (@engine9/plugins/*): metadata, schemas, transforms with
  bindings, search, segments, metrics, reports, optional ui.console.json5, and worker
  classes. Use when adding or extending plugins, interfaces, transforms, EQL
  search, or Engine9 deployment schemas.
---

# Create an Engine9 plugin or interface

Engine9 splits shared data contracts (**interfaces**) from deployable integrations (**native plugins**). Both are Node ESM modules resolved at runtime via `local$@engine9/interfaces/...` or `local$@engine9/plugins/...` (see [reference.md](reference.md)).

## Interface package (`@engine9/interfaces/<name>`)

**Deploy once:** The server treats every `@engine9/interfaces/*` path as **unique** (one `plugin` row per account). You cannot turn that off from interface metadata. Native plugins may still allow multiple instances per path unless their metadata sets `unique: true`.

**Exports:** Only expose the standard feature modules below (and the default aggregate object). Do **not** export non-standard helpers meant only for the server (for example custom `resolveSegmentPluginId`–style functions). If an interface needs special install behavior, it belongs in server code keyed by `plugin.path`, not in the published interface API.

Typical layout:

- `index.js` — exports `metadata`, optional `schema`, `transforms`, `search`, `segments`, `metrics`, `reports`, default aggregate object.
- `schema.js` — `export default { tables: [...] }`.
- `transforms/inbound/…`, `transforms/outbound/…` — pipeline steps.
- Optional: `search.js`, `segments.js`, `metrics.js`, `reports/…`, `ui.console.json5`.

`metadata` at minimum:

```javascript
const metadata = {
  name: '@engine9/interfaces/example',
  version: '1.0.0',
  dependencies: { '@engine9/interfaces/person': '>1.0.0' }, // optional
  schemas: ['schema.js'] // optional hint when schema file name is nonstandard
};
```

### 1. Schema — tables, columns, indexes, views

Export `tables`: each entry has `name`, `columns`, optional `indexes`, optional `type: 'view'` + `sql`.

Column values can be shorthand types (`'string'`, `'id'`, `'foreign_uuid'`, `'created_at'`, …) or objects with `type`, `nullable`, `default_value`, `values` (enum), `description`, `length`.

Minimal example:

```javascript
export const tables = [
  {
    name: 'example_row',
    columns: {
      id: 'id',
      person_id: 'foreign_id',
      status: {
        type: 'string',
        nullable: false,
        default_value: 'active',
        values: ['active', 'archived']
      },
      payload: 'json',
      created_at: 'created_at',
      modified_at: 'modified_at'
    },
    indexes: [{ columns: ['person_id'] }, { columns: ['person_id', 'status'], unique: true }]
  },
  {
    name: 'example_summary',
    type: 'view',
    sql: `select person_id, count(*) as cnt from example_row group by 1`
  }
];
export default { tables };
```

Reference: `message/schema.js` (views, many tables), `person_email/schema.js` (enums), `job/schema.js` (`type: 'enum'`).

### 2. Transforms — inbound upsert (accumulate rows)

Bind `sql.tables.upsert` and push rows onto `tablesToUpsert.<table>`.

```javascript
export const bindings = {
  tablesToUpsert: { path: 'sql.tables.upsert' }
};
export async function transform({ batch, tablesToUpsert }) {
  tablesToUpsert.example_row = tablesToUpsert.example_row || [];
  for (const row of batch) {
    tablesToUpsert.example_row.push({ person_id: row.person_id, status: row.status });
  }
}
export default { bindings, transform };
```

Reference: `person/transforms/inbound/upsert_tables.js`.

### 3. Transforms — outbound enrichment (`sql.query`)

Declare `description`, `bindings` with a SELECT shaped as EQL (`table`, `columns`, `lookup`, `joins`, `conditions`), and a `transform` function `( { batch, ...bound, options } ) => void`.

```javascript
export const bindings = {
  rows: {
    path: 'sql.query',
    options: {
      table: 'example_row',
      columns: ['person_id', 'status'],
      lookup: ['person_id'],
      conditions: []
    }
  }
};
export const transform = ({ batch, rows, options }) => {
  const map = Object.fromEntries(rows.map((r) => [r.person_id, r.status]));
  batch.forEach((b) => {
    b.status = b.status ?? map[b.person_id] ?? null;
  });
};
export default { description: 'Attach latest status', bindings, transform };
```

Reference: `person_email/transforms/outbound/appendEmail.js`.

### 4. Transforms — identifier extraction (`type: 'id'`)

Used to populate `identifiers` for matching; set `export const type = 'id'` and mutate `batch` in `transform`.

Reference: `person_email/transforms/inbound/extract_identifiers.js`.

### 5. Transforms — Handlebars field map

Async `transform({ batch, options })` compiles `options.map` with Handlebars and builds new objects (`*` copies remaining fields).

Reference: `person/transforms/simpleMap.js`.

### 6. Transforms — inline object with `bindings` + function

Expose an object `{ description?, bindings, transform }` where `transform` is synchronous and uses pre-resolved binding data (e.g. `sql.query`).

Reference: `person_remote` → `transforms.appendRemotePersonId` in `person_remote/index.js`.

### 7. Search — UI form → EQL

Named export is a map of handlers. Each handler provides `form` (JSON-Schema-style) and `optionsToEQL(options)` returning `{ text, eql }` where `eql` includes `table`, `columns?`, `conditions`, `joins?`. Conditions may be structured (`type: 'EQUALS' | 'LIKE'` with `ref` / `value`) or `{ eql: 'raw sql fragment' }`.

Reference: `person_email/search.js`, `person/index.js` (mixed raw `eql` strings).

### 8. Search — `optionsToEQLContext`

For lookups before building the main EQL (e.g. load segment rows), add `optionsToEQLContext(opts)` returning named context; `optionsToEQL(options, context)` consumes it.

Reference: `segment/search.js` (`segment` handler).

### 9. Segments — saved audience presets

Export keyed definitions: `name`, optional **`universe`** (array of MySQL EQL objects whose rows yield `input_id` values), optional `search` tree (`and` / paths / table+columns). Paths use `local$@engine9/interfaces/...:search:<handler>`.

The deployed `segment.plugin_id` identifies the **owning** package (often the interface). It does not have to match every plugin that supplies data: the **universe** narrows which inputs (possibly across plugins) feed timeline files for the build. Optional empty `pluginId` in search options keeps handlers universe-scoped; set it only when the search handler should filter by a specific data plugin.

Reference: `person_email/segments.js`, `transaction/core/segments.js`, `channels/email/segments.js` (`universe` + engagement search).

### 10. Metrics — aggregate cards

Functions return `{ label, description?, eql: { table, columns: [ aggregations ] } }`.

Reference: `person/metrics.js`, `source_code/metrics.js`.

### 11. Reports — composed dashboards

Object with `name`, `description`, `components` mapping keys to `{ component: 'ReportTable', query: { table, joins, columns, groupBy, orderBy } }`.

Reference: `person_email/reports/subscription_status.js`.

### 12. Thin / schema-first `index.js`

- `message/index.js` exports only `metadata` (with `schemas: ['schema.js']`); table DDL lives in `schema.js`.
- `job/index.js` and `segment_stats/index.js` export `metadata`, `schema`, and default `{ metadata, schema }` (optional `metadata.dependencies` on `segment_stats`).
- `report/index.js` is metadata-only (marker interface).

### 13. Optional `ui.console.json5`

Console UI can ship as JSON5: top-level `menu` + `routes` (e.g. `RecordTable`, `RecordForm`, `RecordDisplay`), or extra tabs on existing routes (`sidebar` / `main` with `path` segments).

Reference: `job/ui.console.json5` (menus + job CRUD), `person_address/ui.console.json5` (person detail tabs).

### 14. Wiring `index.js`

Export named and default aggregates so `compilePlugin` can read `transforms`, `schema`, etc.:

```javascript
import schema from './schema.js';
import upsert from './transforms/inbound/upsert_tables.js';
const metadata = { name: '@engine9/interfaces/example', version: '1.0.0' };
export const transforms = { upsert };
export { metadata, schema };
export default { metadata, schema, transforms };
```

Reference: `person_email/index.js` (full feature set), `segment/index.js`, `timeline/index.js`.

---

## Native plugin (`@engine9/plugins/<prefix>`)

Implements integration behavior and optional account setup. Convention:

```javascript
const metadata = {
  name: 'Human Name',
  prefix: 'e9myplugin',
  unique: true,
  version: '1.0.0',
  dependencies: { '@engine9/interfaces/message': '>1.0.0' }
};
export default {
  metadata,
  schema, // optional table DDL module
  install // optional async (context) => { message }
  // Plus feature classes, see below
};
```

### 15. `install(context)`

Async function receiving `{ account, plugin, sqlWorker }`; run one-time provisioning (e.g. default rows).

Reference: `e9email/install.js`.

### 16. Worker-style class (RPC methods)

`function Worker(args) { … }`, `Worker.metadata = {}`, `Worker.prototype.myMethod = async function (opts) { … }`, `Worker.prototype.myMethod.metadata = { options: { … } }`. Export as a named property on the plugin object (e.g. `Messages`).

Reference: `e9email/Messages.js`.

### 17. Domain modules

Export concrete integrations (email providers, timeline processors, form handlers) as separate classes/files and attach them to the default plugin export.

Reference: `e9email/index.js` (`SendGridEmail`, `SESTimeline`, …), `e9forms/FormTimeline.js`.

### 18. Metadata-only plugin

Dependency declaration only — no schema or handlers until extended.

Reference: `e9stub/index.js`, `e9workers/index.js`, `e9console/index.js`.

---

## Referencing capabilities from config

- **Transform path:** `local$@engine9/interfaces/<pkg>:transforms:<name>` (colon-separated triple).
- **Search path:** `local$@engine9/interfaces/<pkg>:search:<handler>` (used inside segment JSON).

Server resolution maps `local$@engine9/...` to the checked-out `interfaces/` or `plugins/` tree and loads the module with `createRequire`.

---

## Checklist

- [ ] Interface `index.js` exposes only the standard exports (no server-only hooks or extra named APIs).
- [ ] `metadata.name` matches package scope (`@engine9/interfaces/...` or display name for native).
- [ ] `dependencies` declare other interfaces/schemas required at deploy time.
- [ ] `schema.js` indexes cover join/filter columns; primary keys set where needed.
- [ ] Transforms that mutate SQL state use correct `bindings.path` (`sql.tables.upsert` / `sql.query`).
- [ ] Search handlers return both human `text` and valid `eql`.
- [ ] New interface is registered for deploy if using `deployAllSchemas` / `getActivePluginPaths` (see [reference.md](reference.md)).

For deeper path rules and registration, read [reference.md](reference.md).
