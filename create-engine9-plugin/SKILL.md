---
name: create-engine9-plugin
description: >-
  Implement and extend engine9 interface packages (`@engine9/interfaces/*`) and
  native plugins (`@engine9/plugins/*`), including metadata, schemas, transforms,
  search, segments, metrics, reports, UI configuration, and worker classes. Use
  when building plugin capabilities, wiring deployment schemas, or documenting
  an engine9 interface or native plugin.
---

# Create an engine9 plugin or interface

engine9 separates shared data contracts, called interfaces, from deployable integrations, called native plugins. Both are Node ESM modules resolved by package path from `node_modules`, a monorepo sibling checkout, or an optional install `source`; they never require a `local$` path prefix. Use this skill when adding or extending an interface, native plugin, transform, search handler, segment, report, or deployment schema.

## Quick reference

| Need | Contract or location |
| --- | --- |
| Shared schema or reusable behavior | `@engine9/interfaces/<name>` |
| Deployable integration | `@engine9/plugins/<prefix>` |
| Transform capability | `<package>:transforms:<name>` |
| Search capability | `<package>:search:<handler>` |
| Segment definition | `<package>:segments:<key>` |
| Report definition | `@engine9/plugins/reports/<area>:reports:<key>` |
| Package documentation | Package-root `README.md` |
| Resolver and registration details | [reference.md](reference.md) |

## Concepts

### Interfaces and install uniqueness

`PluginWorker.install` determines whether a package path may have more than one `plugin` row:

1. Use `metadata.unique` when set. Native plugins typically set it to `true`; `person_custom` sets it to `false`.
2. Otherwise, packages under `@engine9/interfaces/*` default to unique, with one row per account.
3. Otherwise, use `options.unique`, which defaults to `false`; third-party plugins may therefore be installed multiple times.

When a package is unique, a second install reuses the existing row or errors if duplicate rows already exist. When it is not unique, every install without an `id` creates a row; `person_custom` receives a new `person_custom_<n>_` table prefix.

**Rule:** Export only standard feature modules and the default aggregate object from an interface. Server-only helpers, such as custom `resolveSegmentPluginId` functions, belong in server code keyed by `plugin.path`, not in the public interface API.

### Stacks

A stack at `@engine9/interfaces/stacks/<name>` is a metadata-only interface with `name`, `description`, `include`, and `exclude`. Core `PluginWorker`, not `SchemaWorker`, installs stacks: it deploys the plugin table, records the stack as a plugin row, walks `include`, and rejects paths forbidden by an installed plugin's `metadata.exclude`. `SchemaWorker` installs only one plugin's schema and row.

`installStandard({ path })` defaults to the account's default stack, typically `@engine9/interfaces/stacks/standard`. Pass another stack, such as `@engine9/interfaces/stacks/limited-pii`, instead of hard-coding stack behavior into `SchemaWorker`. Installing limited-pii and then standard without Server must throw.

Server `accounts.d` values `defaultStack` and `stacks[]` are options passed to `PluginWorker`; they do not belong in core. Inherited, child-first `settings.exclude_pii` forces limited-pii when the default would otherwise be standard and refuses standard, `person_email`, `person_phone`, and `person_address`, even when those plugins are already installed.

### Inbound people pipeline

Core assembles the people pipeline from plugins installed in an account; it does not maintain a list of plugin paths. A plugin participates by mapping pipeline slots to keys from its `transforms` export:

| Slot | Purpose |
| --- | --- |
| `normalize` | Clean or normalize fields |
| `id` | Populate `identifiers[]` before person assignment |
| `assign` | Core-owned person assignment phase |
| `upsert` | Queue table rows after person assignment |

Install validates that each transform key exists and that transform `type` values such as `'id'` or `'upsert'` match their slots. It then snapshots the specification on the `plugin` row. Verify the result with `personWorker.getInboundTransforms({ pluginId, describe: true })`.

### Segments and reports

Segments are keyed saved-audience definitions. The export key is the final part of `<package>:segments:<key>`. The deployed `segment.plugin_id` identifies the owning package; it need not match each plugin supplying data. An optional `universe` narrows the input IDs whose timeline files participate in a build, while an empty `pluginId` in search options preserves universe scope.

Reports are composed dashboards owned by native report plugins at `@engine9/plugins/reports/<area>`, never by interfaces. Export a keyed `reports` map on the default plugin object. `ReportWorker` compiles report EQL/SQL.

## File format

### Interface package layout

| Path | Purpose |
| --- | --- |
| `README.md` | Required audience-facing package documentation |
| `index.js` | Named feature exports and default aggregate |
| `schema.js` | Default export `{ tables: [...] }` |
| `transforms/inbound/` | Normalize, identity, and upsert steps |
| `transforms/outbound/` | Enrichment and output steps |
| `search.js` | Optional UI-to-EQL handlers |
| `segments.js` | Optional saved-audience definitions |
| `metrics.js` | Optional aggregate cards |
| `ui.console.json5` | Optional console UI configuration |

**Rule:** Do not ship `reports/` on interfaces. Put dashboards in `@engine9/plugins/reports/<area>`.

At minimum, interface metadata identifies the package and version:

```javascript
const metadata = {
  name: "@engine9/interfaces/example",
  version: "1.0.0",
  dependencies: { "@engine9/interfaces/person": ">=1.0.0" }, // optional
  schemas: ["schema.js"], // optional for a nonstandard schema filename
};
```

### Schema

Export `tables`. Each table has `name`, `columns`, and optional `indexes`; views add `type: 'view'` and `sql`. Column values may be shorthand types such as `'string'`, `'id'`, `'foreign_uuid'`, or `'created_at'`, or objects with `type`, `nullable`, `default_value`, `values`, `description`, and `length`.

```javascript
export const tables = [
  {
    name: "example_row",
    columns: {
      id: "id",
      person_id: "foreign_id",
      status: {
        type: "string",
        nullable: false,
        default_value: "active",
        values: ["active", "archived"],
      },
      payload: "json",
      created_at: "created_at",
      modified_at: "modified_at",
    },
    indexes: [
      { columns: ["person_id"] },
      { columns: ["person_id", "status"], unique: true },
    ],
  },
  {
    name: "example_summary",
    type: "view",
    sql: `select person_id, count(*) as cnt from example_row group by 1`,
  },
];
export default { tables };
```

### Search

The named `search` export is a map of handlers:

| Member | Purpose |
| --- | --- |
| `title`, `description` | Optional catalog labels |
| `form` | Canonical JSON Schema object |
| `optionsToEQL(options)` | Returns `{ text, eql }` |
| `optionsToEQLContext(options)` | Optional pre-query lookup context |

Canonical forms use `{ title, type: 'object', properties, required? }`. The server still normalizes legacy flat property maps and single-key wrappers. EQL may contain `table`, `columns`, `conditions`, and `joins`; conditions may be structured (`EQUALS`, `LIKE`) or raw `{ eql: '...' }` fragments.

Account-scoped discovery through MCP `searchOptions`, `PersonWorker.searchOptions`, or `GET /data/search/options` returns standard filters and all installed-plugin handlers with normalized forms. A UI submits `{ and: [{ path, options }] }` to search.

### Segment definition

Export an object map, not an array. Each value has `name`, optional `universe` EQL objects whose rows yield `input_id`, and optional `search` trees using `and`, paths, or table/column clauses. Paths use `@engine9/interfaces/...:search:<handler>`.

### Report definition

Each report contains `name`, `description`, `tags`, optional `data_sources`, `filters` expressed as JSON Schema, optional `optionsToEQL`, and `sections`. A section has an optional `title` and `components` such as `StatCard`, `ComposedChart`, or `Table`.

### Native plugin layout

Native plugins follow the same package-root `README.md` convention and may provide integration behavior, account setup, schema, and classes:

```javascript
const metadata = {
  name: "Human Name",
  prefix: "e9myplugin",
  unique: true,
  version: "1.0.0",
  dependencies: { "@engine9/interfaces/message": ">1.0.0" },
};

export default {
  metadata,
  schema,  // optional table DDL
  install, // optional async setup
  // Optional feature classes
};
```

`install(context)` is asynchronous and receives `{ account, plugin, sqlWorker }` for one-time provisioning.

Worker-style classes follow `function Worker(args) { ... }`, static `Worker.metadata`, prototype methods, and method-level metadata such as `Worker.prototype.myMethod.metadata = { options: { ... } }`. Export each class as a named property on the plugin object. Concrete domain integrations may live in separate classes/files and attach to the default export.

Metadata-only plugins may declare dependencies without schema or handlers. Optional `ui.console.json5` may define top-level `menu` and `routes`, CRUD components such as `RecordTable`, `RecordForm`, or `RecordDisplay`, and `sidebar` or `main` tabs with path segments.

## Workflow

1. Choose an interface for reusable contracts or a native plugin for deployable integration behavior.
2. Create package metadata and declare deployment dependencies.
3. Define tables, columns, indexes, and views.
4. Add inbound or outbound transforms and their bindings.
5. Add search, segments, metrics, reports, UI configuration, or worker classes where appropriate.
6. Export named features and a default aggregate from `index.js`.
7. Document behavior and every predefined segment in the package `README.md`.
8. Register a new interface when deployment uses `deployAllSchemas` or `getActivePluginPaths`.
9. Install and verify compiled features and inbound transforms.

### Document the package

`README.md` is the audience-facing contract. Do not rely on comments in `segments.js`, `metadata.description`, or skill files as package documentation. A useful outline is:

```markdown
# Human Name Interface

One-paragraph purpose. Name the package path and `metadata.dependencies`.

## Data Model
## Inbound Behavior
## Outbound Behavior
## Search
## Segments
## Metrics
## Reports and UI
```

For every predefined segment, document its display name, export key, definition path, included and excluded people, search or table-condition implementation, and universe. Write `None` when membership is based on current table state.

## Rules

**Rule:** Package metadata names must match the package scope: `@engine9/interfaces/...` for interfaces and the intended display name for native plugins.

**Rule:** Declare every interface or schema dependency required at deployment time.

**Rule:** Index columns used for joins, filters, natural keys, and uniqueness.

**Rule:** Bind state-changing inbound transforms to `sql.tables.upsert`; bind query enrichments to `sql.query`.

**Rule:** Never queue duplicate natural keys in one upsert batch. Use `mergeIntoQueue` and plugin-owned merge semantics; without a `merge` callback, duplicates throw during transformation.

**Rule:** Search handlers must return human-readable `text` and valid `eql`.

**Rule:** Segment exports must be keyed objects, and every key must be documented in the package README with membership and universe behavior.

**Rule:** Wire only standard named exports and the default aggregate from interface `index.js`.

## Examples

### Declare inbound transforms

```javascript
const metadata = {
  name: "@engine9/interfaces/example",
  version: "1.0.0",
  inbound: {
    id: ["extractLoyaltyNumber"],
    upsert: ["upsertMembership"],
  },
};
```

Identifier extraction transforms set `export const type = 'id'` and mutate each batch row's `identifiers`.

### Merge inbound rows

```javascript
import { mergeIntoQueue } from "@engine9/input-tools";

export const bindings = {
  tablesToUpsert: { path: "sql.tables.upsert" },
};

function mergeExampleRow(existing, incoming) {
  return {
    ...existing,
    ...incoming,
    id: existing.id ?? incoming.id ?? null,
    status: incoming.status ?? existing.status,
    source_input_id: existing.source_input_id ?? incoming.source_input_id,
  };
}

export async function transform({ batch, tablesToUpsert }) {
  tablesToUpsert.example_row ||= [];
  for (const row of batch) {
    mergeIntoQueue(
      tablesToUpsert.example_row,
      { person_id: row.person_id, status: row.status, id: null },
      {
        keyFields: ["person_id"],
        merge: mergeExampleRow,
        label: "example_row",
      },
    );
  }
}

export default { bindings, transform };
```

For status changes in one file, `person_email` uses last-row-wins order. Unsubscribed then Subscribed preserves a resubscription; the reverse preserves Unsubscribed. Same-timestamp rows are ambiguous, so use explicit entry types or sort the source when order matters.

### Enrich outbound rows

```javascript
export const bindings = {
  rows: {
    path: "sql.query",
    options: {
      table: "example_row",
      columns: ["person_id", "status"],
      lookup: ["person_id"],
      conditions: [],
    },
  },
};

export const transform = ({ batch, rows }) => {
  const statusByPerson = Object.fromEntries(
    rows.map((row) => [row.person_id, row.status]),
  );
  batch.forEach((row) => {
    row.status = row.status ?? statusByPerson[row.person_id] ?? null;
  });
};

export default {
  description: "Attach latest status",
  bindings,
  transform,
};
```

### Map fields with Handlebars

An asynchronous `transform({ batch, options })` may compile `options.map` with Handlebars and build new objects. The `*` mapping copies remaining fields.

### Export an inline transform

An interface may expose `{ description?, bindings, transform }`, where `transform` is synchronous and consumes pre-resolved binding data such as `sql.query`.

### Define a metric

Metric functions return `{ label, description?, eql: { table, columns: [aggregations] } }`.

### Wire the interface

```javascript
import schema from "./schema.js";
import upsert from "./transforms/inbound/upsert_tables.js";

const metadata = {
  name: "@engine9/interfaces/example",
  version: "1.0.0",
};
export const transforms = { upsert };
export { metadata, schema };
export default { metadata, schema, transforms };
```

Thin, schema-first interfaces are also valid. `message/index.js` exports only metadata with `schemas: ['schema.js']`; `job/index.js` and `segment_stats/index.js` export metadata, schema, and the default aggregate; `report/index.js` is a metadata-only marker.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Second install creates or reuses the wrong row | Review `metadata.unique`, interface defaults, and `options.unique` |
| Installed transform is absent | Confirm `metadata.inbound` names an exported transform and its `type` matches the slot |
| Upsert fails on duplicate keys | Merge rows by the schema's unique key with `mergeIntoQueue` |
| Search does not appear in discovery | Confirm the plugin is installed and the handler uses a canonical form |
| Segment membership is unexpectedly broad | Inspect `universe`, search path, and optional `pluginId` scope |
| Interface report is not available | Move it to a native `@engine9/plugins/reports/<area>` package |
| Package cannot resolve | Use the package path and inspect resolver/registration rules; do not add `local$` |
| Stack installation conflicts | Inspect installed stack `exclude` metadata and inherited `exclude_pii` |

## Related documentation

- [Plugin resolver and registration reference](reference.md)
- [Report authoring](../e9-reports/SKILL.md)
- `@engine9/core/lib/peoplePipeline/README.md`
- `stacks/standard/index.js`
- `stacks/limited-pii/index.js`
- `message/schema.js`, `person_email/schema.js`, and `job/schema.js`
- `person_email/transforms/inbound/upsert_tables.js`
- `person/transforms/inbound/upsert_tables.js`
- `person_email/transforms/outbound/appendEmail.js`
- `person_email/transforms/inbound/extract_identifiers.js`
- `person/transforms/simpleMap.js`
- `person_remote/index.js`
- `person_email/search.js`, `person/index.js`, and `channels/email/search.js`
- `segment/search.js`
- `person_email/segments.js`, `transaction/core/segments.js`, and `channels/email/segments.js`
- `person/metrics.js` and `source_code/metrics.js`
- `plugins/reports/people/reports/subscription_status.js`
- `plugins/reports/messaging/reports/email.js`
- `job/ui.console.json5` and `person_address/ui.console.json5`
- `person_email/index.js`, `segment/index.js`, and `timeline/index.js`
- `e9email/install.js`, `e9email/Messages.js`, `e9email/index.js`, and `e9forms/FormTimeline.js`
- `e9stub/index.js`, `e9workers/index.js`, and `e9console/index.js`
