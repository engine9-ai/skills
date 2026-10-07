---

## name: create-engine9-plugin

description: >-
Author engine9 schema plugins (`@engine9/schemas/*`), native plugins
(`@engine9/plugins/*`), and third-party npm plugin packages (e.g. acme-plugins),
including package layout, versioning, metadata, schemas, transforms, search,
segments, metrics, reports, settings, UI, and worker classes. Use when building
or documenting a plugin package for one or more core deployments. Do not use
this skill to install onto accounts — use e9-plugin (MCP plugin install).

# Create an engine9 plugin or schema plugin

engine9 separates shared data contracts, called schema plugins (`@engine9/schemas/*`, published as `@engine9/interfaces` through 1.8.1), from deployable integrations, called native plugins. A schema plugin's table definitions are its `schema.js`; "schema plugin" is the whole directory. Both are Node ESM modules resolved by **npm package path** from `node_modules` (or a monorepo sibling linked into `node_modules`). They never require a `local$` path prefix, and core does not load plugins from absolute paths or `install({ source })`. Use this skill when adding or extending a schema plugin, native plugin, third-party package, transform, search handler, segment, report, settings, or deployment schema. To **install** an existing package onto accounts, use [e9-plugin](../e9-plugin/SKILL.md).

## Quick reference

| Need                                  | Contract or location                                                                     |
| ------------------------------------- | ---------------------------------------------------------------------------------------- |
| Install a package on accounts         | [e9-plugin](../e9-plugin/SKILL.md) — MCP `plugin` `install`, not this skill              |
| Shared schema or reusable behavior    | `@engine9/schemas/<name>`                                                             |
| Engine9-owned deployable integration  | `@engine9/plugins/<name>`                                                                |
| Third-party package (company example) | npm package `acme-plugins` → plugin paths `acme-plugins/<name>`                          |
| Third-party table names               | Self-scope in the schema: `acme_blog_post` — do **not** set `metadata.prefix`            |
| Package / release version             | npm `package.json` `"version"` only — never `metadata.version`                           |
| Host must load the package            | Site `package.json` → `engine9.pluginPackages` + `npm install`                           |
| Transform capability                  | `<package>:transforms:<name>`                                                            |
| Search capability                     | `<package>:search:<handler>`                                                             |
| Segment definition                    | `<package>:segments:<key>`                                                               |
| Report definition                     | `@engine9/plugins/reports/<area>:reports:<key>`                                          |
| Settings                              | `<package>` `settings` export or sibling `settings.js`; MCP `plugin` `command: settings` |
| Package documentation                 | Package-root `README.md`                                                                 |
| Resolver and registration details     | [reference.md](reference.md)                                                             |

## Creating and versioning a package (Acme)

A **package** is one npm artifact. A **plugin** is one discoverable module inside that package (a directory with `index.js`, or a `*.plugin.js` file). Version the package; install plugins by path on each account.

### Recommended layout for Acme

Prefer one npm package that holds many plugins (same pattern as `@engine9/schemas`):

```
acme-plugins/                 ← npm name: "acme-plugins" (or "@acme/engine9-plugins")
  package.json                ← ONLY place for the release version
  README.md
  loyalty/
    index.js                  ← identity: acme-plugins/loyalty
    schema.js
    settings.js
  gifts.plugin.js             ← identity: acme-plugins/gifts.plugin.js
```

| Recommendation                                                            | Why                                                          |
| ------------------------------------------------------------------------- | ------------------------------------------------------------ |
| Name the npm package `acme-plugins` or `@acme/engine9-plugins`            | Clear company ownership; path prefix matches the package     |
| Put plugins in subdirectories (or `*.plugin.js`), not one repo per plugin | One `npm install` / pin per host; shared deps; one changelog |
| Own repo or monorepo directory is fine                                    | Core only needs an installable npm package in `node_modules` |
| Bump `package.json` `"version"` on every release                          | That string is what hosts and dependency checks see          |
| Do **not** set `metadata.version`                                         | Ignored; confusing dual versioning                           |

Scoped packages work the same: `@acme/engine9-plugins` → identities `@acme/engine9-plugins/loyalty`.

### Versioning rules

| Layer               | What it is                                                                     | Who sets it                                     |
| ------------------- | ------------------------------------------------------------------------------ | ----------------------------------------------- |
| npm package version | `acme-plugins@1.4.0` in `package.json`                                         | Acme when publishing                            |
| Plugin path         | `acme-plugins/loyalty`                                                         | Acme in the package tree                        |
| Account install     | Warehouse `plugin` row for that path                                           | Operator / MCP `plugin` install on each account |
| Host pin            | Site depends on `acme-plugins@^1.4.0` and lists it in `engine9.pluginPackages` | Each core deployment                            |

There is **no** per-plugin version and **no** `deployed_version` column. Executing code is always whatever the host installed. Dependency ranges on `metadata.dependencies` are checked at install time against the **host’s** npm package version for `packageNameOf(depPath)` (plus “dependency path is installed on this account”).

```javascript
// acme-plugins/loyalty/index.js — no metadata.version, no metadata.prefix
const metadata = {
  name: "Acme Loyalty",
  unique: true,
  dependencies: {
    // path must be installed; range is satisfied by host @engine9/schemas version
    "@engine9/schemas/person": ">=1.7.0",
  },
};
```

**Rule:** Third-party plugins must self-scope table names in the schema (e.g. `acme_loyalty_member`). Do not set `metadata.prefix`. See [Table naming](#table-naming-self-scope).

### Ship to many core deployments

Each engine9 core site (Cloudflare Worker or Node host) pins and loads packages independently:

1. **Publish** `acme-plugins@1.4.0` (npm, git URL, or private registry).
2. **On each site**, add the dependency and declare the package:

```json
{
  "dependencies": {
    "@engine9/core": "...",
    "@engine9/schemas": "^1.7.4",
    "acme-plugins": "^1.4.0"
  },
  "engine9": {
    "pluginPackages": ["@engine9/schemas", "acme-plugins"]
  }
}
```

1. **Redeploy the host** (`npm install` + restart, or `wrangler deploy` so `build-plugins` bakes the package into the Worker). Until the package is on the host, `listAvailable` will not show Acme paths.
2. **Per account** on that host, install paths as needed: `acme-plugins/loyalty` (MCP [e9-plugin](../e9-plugin/SKILL.md)). Same path identity on every host; each host may pin a different package version.

**Upgrade:** bump the site’s pin (e.g. `1.5.0`), reinstall/redeploy the host, then reinstall account paths only when schema/settings/inbound snapshots need refreshing. Install-time dependency checks always use the new host package version immediately.

**Local development:** `"acme-plugins": "file:../acme-plugins"` (or npm link) is fine. Still list `acme-plugins` in `engine9.pluginPackages`.

## Concepts

### Schema plugins and install uniqueness

`PluginWorker.install` determines whether a package path may have more than one `plugin` row:

1. Use `metadata.unique` when set. Native plugins typically set it to `true`; `person_custom` sets it to `false`.
2. Otherwise, packages under `@engine9/schemas/*` default to unique, with one row per account.
3. Otherwise, use `options.unique`, which defaults to `false`; third-party plugins may therefore be installed multiple times.

When a package is unique, a second install reuses the existing row or errors if duplicate rows already exist. When it is not unique, every install without an `id` creates a row. Multi-instance plugins that need isolated tables (`person_custom`) opt into `metadata.prefix` so install allocates a hex suffix; third-party plugins should stay unique and self-scope table names instead.

**Rule:** Export only standard feature modules and the default aggregate object from a schema plugin. Server-only helpers, such as custom `resolveSegmentPluginId` functions, belong in server code keyed by `plugin.path`, not in the public schema plugin API.

### Table naming (self-scope)

**Rule:** For third-party plugins, put a brief, stable stem that is unique to the company and plugin into every table name you own. Prefer `acme_blog_post`, `acme_blog_comment` — not bare `post` / `comment`, and not `metadata.prefix`.

Core does **not** enforce this naming. Install with a schema and no `metadata.prefix` stores an empty `plugin.table_prefix` and deploys the table names exactly as written in the schema. SQL, transforms, metrics, and reports can then hard-code those names; they do not need to look up the plugin row first.

| Approach                                       | When                                                           | Result                                                                     |
| ---------------------------------------------- | -------------------------------------------------------------- | -------------------------------------------------------------------------- |
| **Self-scope (recommended for third parties)** | Omit `metadata.prefix`; name tables `acme_blog_<table>`        | Empty `table_prefix`; predictable SQL                                      |
| `metadata.prefix` **hex allocator**            | Multi-instance / engine9-owned cases that need isolated copies | `plugin.table_prefix` = `{prefix}_{hex}_`; every query must use that value |

Do not recommend `metadata.prefix` for ordinary third-party packages. The hex suffix makes static SQL and hand-written queries painful: callers must read `plugin.table_prefix` before they can name a table. Reserve `metadata.prefix` for cases that truly need multiple installs of the same schema on one account (engine9’s `person_custom` pattern).

Pick a stem you can keep forever (`acme_blog`, `acme_loyalty`). In the future, plugin table-name prefixes may need to be registered; a stable company+plugin stem is the right preparation. Core does not register or validate stems today.

### Stacks

A stack at `@engine9/schemas/stacks/<name>` is a metadata-only schema plugin with `name`, `description`, `include`, and `exclude`. Core `PluginWorker`, not `SchemaWorker`, installs stacks: it deploys the plugin table, records the stack as a plugin row, walks `include`, and rejects paths forbidden by an installed plugin's `metadata.exclude`. `SchemaWorker` installs only one plugin's schema and row.

`installDefaultPlugins({ path })` resolves an omitted path from (1)
`exclude_pii` on `@engine9/schemas/utilities/limited-pii` (when that plugin is
installed) → `@engine9/schemas/stacks/limited-pii`, (2) warehouse
`default_stack` on `@engine9/schemas/plugin`, or (3) the published core
person schema plugins — not automatically `@engine9/schemas/stacks/standard`.
Pass another stack, such as `@engine9/schemas/stacks/limited-pii`, instead
of hard-coding stack behavior into `SchemaWorker`. With `exclude_pii` set,
installing standard must throw.

Set `default_stack` on `@engine9/schemas/plugin` and `exclude_pii` on
`@engine9/schemas/utilities/limited-pii` via MCP `plugin` `setSetting` (or
`PluginWorker.updateSetting`). Empty `default_stack` means core person
schema plugins only. `exclude_pii: true` forces the limited-pii stack and refuses
standard, `person_email`, `person_phone`, and `person_address`, even when those
plugins are already installed.

### Inbound people pipeline

Core assembles the people pipeline from plugins installed in an account; it does not maintain a list of plugin paths. A plugin participates by mapping pipeline slots to keys from its `transforms` export:

| Slot        | Purpose                                           |
| ----------- | ------------------------------------------------- |
| `normalize` | Clean or normalize fields                         |
| `id`        | Populate `identifiers[]` before person assignment |
| `assign`    | Core-owned person assignment phase                |
| `upsert`    | Queue table rows after person assignment          |

Install validates that each transform key exists and that transform `type` values such as `'id'` or `'upsert'` match their slots. It then snapshots the specification on the `plugin` row. Verify the result with `personWorker.getInboundTransforms({ pluginId, describe: true })`.

### Segments and reports

Segments are keyed saved-audience definitions. The export key is the final part of `<package>:segments:<key>`. The deployed `segment.plugin_id` identifies the owning package; it need not match each plugin supplying data. An optional `universe` narrows the input IDs whose timeline files participate in a build, while an empty `pluginId` in search options preserves universe scope.

Reports are composed dashboards owned by native report plugins at `@engine9/plugins/reports/<area>`, never by schema plugins. Export a keyed `reports` map on the default plugin object. `ReportWorker` compiles report EQL/SQL.

## File format

### Schema plugin layout

| Path                   | Purpose                                               |
| ---------------------- | ----------------------------------------------------- |
| `README.md`            | Required audience-facing package documentation        |
| `index.js`             | Named feature exports and default aggregate           |
| `schema.js`            | Default export `{ tables: [...] }`                    |
| `transforms/inbound/`  | Normalize, identity, and upsert steps                 |
| `transforms/outbound/` | Enrichment and output steps                           |
| `search.js`            | Optional UI-to-EQL handlers                           |
| `segments.js`          | Optional saved-audience definitions                   |
| `metrics.js`           | Optional aggregate cards                              |
| `ui.console.json5`     | Optional console UI configuration                     |
| `settings.js`          | Optional warehouse `setting` rows inserted on install |

**Rule:** Do not ship `reports/` on schema plugins. Put dashboards in `@engine9/plugins/reports/<area>`.

At minimum, schema plugin metadata identifies the package path (and optional deployment dependencies):

```javascript
const metadata = {
  name: "@engine9/schemas/example",
  dependencies: { "@engine9/schemas/person": ">=1.0.0" }, // optional; range = host npm package version
  schemas: ["schema.js"], // optional for a nonstandard schema filename
};
```

Do not set `metadata.version`. See [Creating and versioning a package (Acme)](#creating-and-versioning-a-package-acme).

### Schema

Export `tables`. Each table has `name`, `columns`, and optional `indexes`; views add `type: 'view'` and `sql`. Column values may be shorthand types such as `'string'`, `'id'`, `'foreign_uuid'`, or `'created_at'`, or objects with `type`, `nullable`, `default_value`, `values`, `description`, and `length`.

**Rule:** Third-party table `name` values must include a company+plugin stem (`acme_blog_post`). Do not set `metadata.prefix` so install leaves `table_prefix` empty and deploys these names unchanged. See [Table naming](#table-naming-self-scope).

```javascript
export const tables = [
  {
    name: "acme_blog_post",
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
    name: "acme_blog_post_summary",
    type: "view",
    sql: `select person_id, count(*) as cnt from acme_blog_post group by 1`,
  },
];
export default { tables };
```

### Search

The named `search` export is a map of handlers:

| Member                         | Purpose                           |
| ------------------------------ | --------------------------------- |
| `title`, `description`         | Optional catalog labels           |
| `form`                         | Canonical JSON Schema object      |
| `optionsToEQL(options)`        | Returns `{ text, eql }`           |
| `optionsToEQLContext(options)` | Optional pre-query lookup context |

Canonical forms use `{ title, type: 'object', properties, required? }`. The server still normalizes legacy flat property maps and single-key wrappers. EQL may contain `table`, `columns`, `conditions`, and `joins`; conditions may be structured (`EQUALS`, `LIKE`) or raw `{ eql: '...' }` fragments.

Account-scoped discovery through MCP `searchOptions`, `PersonWorker.searchOptions`, or `GET /data/search/options` returns standard filters and all installed-plugin handlers with normalized forms. A UI submits `{ and: [{ path, options }] }` to search. Set `exclude: true` on a clause for people outside that handler's set; `exclude` is not a form field.

Current model membership (`personSourceCode` / `transactionSourceCode`, include and exclude) is [e9-model — Person search](../e9-model/SKILL.md#person-search). People in a legacy model for a source code and out of the matching current model are [e9-model/legacy.md](../e9-model/legacy.md). Read that file only when the user explicitly asked for legacy models.

### Segment definition

Export an object map, not an array. Each value has `name`, optional `universe` EQL objects whose rows yield `input_id`, and optional `search` trees using `and`, paths, or table/column clauses. Paths use `@engine9/schemas/...:search:<handler>`.

### Report definition

Each report contains `name`, `description`, `tags`, optional `data_sources`, `filters` expressed as JSON Schema, and `sections`. A section has an optional `title` and `components` such as `StatCard`, `ComposedChart`, or `Table`. Use declarative `filter: { column }` and static `data_sources.conditions`.

### Settings

Settings are per-install warehouse configuration. They are **not** marketplace authorization (`auth_fields` / vendor credentials on `account` plugin metadata).

Export `settings` from `index.js` and/or ship a sibling `settings.js`. Each entry is `{ name, type?, default?, values?, description?, label?, required?, secret?, hidden?, section? }` (or a name→def object). On `PluginWorker.install`, each name is inserted into `setting` for that plugin row when it is missing. Reinstall does not overwrite an existing value. Plugins with no `metadata.prefix` get an empty `table_prefix` (settings-only packages, and third-party schemas that self-scope table names).

| Field                   | Purpose                                                                            |
| ----------------------- | ---------------------------------------------------------------------------------- |
| `name`                  | Warehouse `setting.name`                                                           |
| `type`                  | `string` (default), `int`, `number`, `boolean`, `json`, `email`, `url`, `password` |
| `values`                | Enum of allowed values (UI + `setSetting` validation)                              |
| `default`               | Inserted on first install (`default_value` is also accepted)                       |
| `description` / `label` | UI copy                                                                            |
| `secret`                | Redact current value in catalogs                                                   |
| `hidden`                | Omit from UI catalogs unless `include_hidden` (internal allocators)                |

Account-scoped discovery uses the same compile-and-aggregate path as inbound weaving, `searchOptions`, and report list:

| Surface | Call                                                                                        |
| ------- | ------------------------------------------------------------------------------------------- |
| Worker  | `PluginWorker.listSettings({ path?, plugin_id?, include_hidden? })`                         |
| MCP     | `plugin` `command: "settings"`                                                              |
| HTTP    | `GET /data/settings`                                                                        |
| Update  | `PluginWorker.updateSetting` / MCP `plugin` `command: "setSetting"` / `POST /data/settings` |

`listSettings` groups by plugin path, includes JSON Schema `form` plus current warehouse values per instance, and puts per-plugin `compilePlugin` failures in `errors`. Do not read settings from marketplace metadata.

### Native plugin layout

Native plugins follow the same package-root `README.md` convention and may provide integration behavior, account setup, schema, and classes:

```javascript
// Inside acme-plugins/loyalty/index.js (identity acme-plugins/loyalty)
const metadata = {
  name: "Acme Loyalty", // display name
  unique: true,
  dependencies: { "@engine9/schemas/person": ">=1.7.0" },
};

export default {
  metadata,
  schema, // optional; table names self-scoped (acme_loyalty_member) — no metadata.prefix
  settings, // optional warehouse setting rows inserted on install
  install, // optional async setup
  // Optional feature classes
};
```

`install(context)` is asynchronous and receives `{ account, plugin, sqlWorker }` for one-time provisioning. With self-scoped tables, SQL may use the fixed names from the schema; `plugin.tablePrefix` is empty.

Worker-style classes follow `function Worker(args) { ... }`, static `Worker.metadata`, prototype methods, and method-level metadata such as `Worker.prototype.myMethod.metadata = { options: { ... } }`. Export each class as a named property on the plugin object. Concrete domain integrations may live in separate classes/files and attach to the default export.

Metadata-only plugins may declare dependencies without schema or handlers. Optional `ui.console.json5` may define top-level `menu` and `routes`, CRUD components such as `RecordTable`, `RecordForm`, or `RecordDisplay`, and `sidebar` or `main` tabs with path segments.

## Workflow

1. Choose a schema plugin (`@engine9/schemas/...`), an Engine9 native plugin (`@engine9/plugins/...`), or a third-party package (e.g. `acme-plugins/<name>`).
2. Create or extend the **npm package** (`package.json` name + `"version"`). Add plugin directories or `*.plugin.js` files inside it. Do not invent `metadata.version`.
3. Declare `metadata` (display name, `unique` as needed) and `metadata.dependencies` (paths + semver against host packages). Do **not** set `metadata.prefix` for third-party plugins.
4. Define tables, columns, indexes, and views. Self-scope every owned table name (`acme_blog_post`).
5. Add inbound or outbound transforms and their bindings.
6. Add search, segments, metrics, reports, settings, UI configuration, or worker classes where appropriate.
7. Export named features and a default aggregate from `index.js`.
8. Document behavior and every predefined segment in the package `README.md`.
9. Publish or link the package; on each core site add it to `dependencies` and `engine9.pluginPackages`, then redeploy the host.
10. Install plugin paths on accounts ([e9-plugin](../e9-plugin/SKILL.md)) and verify compiled features / inbound transforms.

### Document the package

`README.md` is the audience-facing contract. Do not rely on comments in `segments.js`, `metadata.description`, or skill files as package documentation. A useful outline is:

```markdown
# Human Name Schema Plugin

One-paragraph purpose. Name the package path and `metadata.dependencies`.

## Data Model

## Inbound Behavior

## Outbound Behavior

## Search

## Segments

## Metrics

## Reports and UI

## Settings
```

For every predefined segment, document its display name, export key, definition path, included and excluded people, search or table-condition implementation, and universe. Write `None` when membership is based on current table state.

## Rules

**Rule:** Version only the npm package (`package.json` `"version"`). Never set `metadata.version`. Never rely on a warehouse version column.

**Rule:** Third-party plugins self-scope every owned table name with a stable company+plugin stem (`acme_blog_post`). Do not set `metadata.prefix`. Core does not enforce stems; they may need registration later.

**Rule:** Plugin identity is the path under the npm package (`acme-plugins/loyalty`). `metadata.name` is the human display name.

**Rule:** For third-party packages, list the npm package in the site’s `engine9.pluginPackages` and install it into that site’s `node_modules` before account install can succeed.

**Rule:** `metadata.dependencies` keys are plugin **paths** that must already be installed on the account; semver ranges apply to the host npm version of each path’s package.

**Rule:** Schema plugin `metadata.name` values use `@engine9/schemas/...`. Native and third-party plugins use a human display name in `metadata.name`.

**Rule:** Declare every schema plugin or other plugin dependency required at deployment time.

**Rule:** Index columns used for joins, filters, natural keys, and uniqueness.

**Rule:** Bind state-changing inbound transforms to `sql.tables.upsert`; bind query enrichments to `sql.query`.

**Rule:** Never queue duplicate natural keys in one upsert batch. Use `mergeIntoQueue` and plugin-owned merge semantics; without a `merge` callback, duplicates throw during transformation.

**Rule:** Search handlers must return human-readable `text` and valid `eql`.

**Rule:** Segment exports must be keyed objects, and every key must be documented in the package README with membership and universe behavior.

**Rule:** Declare settings on the package (`settings` export or sibling `settings.js`). Do not put warehouse settings in marketplace `auth_fields`.

**Rule:** Wire only standard named exports and the default aggregate from a schema plugin's `index.js`.

## Examples

### Declare inbound transforms

```javascript
const metadata = {
  name: "@engine9/schemas/example",
  inbound: {
    id: ["extractLoyaltyNumber"],
    upsert: ["upsertMembership"],
  },
};
```

Identifier extraction transforms set `export const type = 'id'` and mutate each batch row's `identifiers`.

### Declare plugin settings

```javascript
export const settings = [
  {
    name: "summary_model_kind",
    type: "string",
    values: ["legacy", "current"],
    default: "legacy",
    description:
      "Use legacy origin/pivot models or current model_*_stats tables.",
  },
];
```

Discovery: MCP `plugin` `{ command: "settings", account_id }` or `PluginWorker.listSettings()`. Update: `command: "setSetting"` with `plugin_id`, `name`, `value`.

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

A schema plugin may expose `{ description?, bindings, transform }`, where `transform` is synchronous and consumes pre-resolved binding data such as `sql.query`.

### Define a metric

Metric functions return `{ label, description?, eql: { table, columns: [aggregations] } }`.

### Wire the schema plugin

```javascript
import schema from "./schema.js";
import upsert from "./transforms/inbound/upsert_tables.js";

const metadata = {
  name: "@engine9/schemas/example",
};
export const transforms = { upsert };
export { metadata, schema };
export default { metadata, schema, transforms };
```

Thin, schema-only schema plugins are also valid. `message/index.js` exports only metadata with `schemas: ['schema.js']`; `job/index.js` and `segment_stats/index.js` export metadata, schema, and the default aggregate; `report/index.js` is a metadata-only marker.

## Troubleshooting

| Symptom                                         | Check                                                                                                                          |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Second install creates or reuses the wrong row  | Review `metadata.unique`, schema plugin defaults, and `options.unique`                                                            |
| Installed transform is absent                   | Confirm `metadata.inbound` names an exported transform and its `type` matches the slot                                         |
| Upsert fails on duplicate keys                  | Merge rows by the schema's unique key with `mergeIntoQueue`                                                                    |
| Search does not appear in discovery             | Confirm the plugin is installed and the handler uses a canonical form                                                          |
| Settings do not appear in MCP `plugin` settings | Confirm the plugin is installed, `settings` is on the default export or sibling `settings.js`, and the setting is not `hidden` |
| Segment membership is unexpectedly broad        | Inspect `universe`, search path, and optional `pluginId` scope                                                                 |
| Schema plugin report is not available           | Move it to a native `@engine9/plugins/reports/<area>` package                                                                  |
| Package cannot resolve                          | Use the package path and inspect resolver/registration rules; do not add `local$`                                              |
| Stack installation conflicts                    | Inspect installed stack `exclude` metadata and utilities/limited-pii `exclude_pii`                                                         |

## Related documentation

- [Install plugins on accounts](../e9-plugin/SKILL.md)
- [Plugin resolver and registration reference](reference.md)
- [engine9 MCP](../e9-mcp/SKILL.md)
- [Report authoring](../e9-reports/SKILL.md)
- `@engine9/core/lib/peoplePipeline/README.md`
- `stacks/standard/index.js`
- `stacks/limited-pii/index.js`
- `utilities/limited-pii/index.js`
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
- `plugin/settings.js`
- `plugins/models/deployment/v2026_09_16/settings.js`
- `e9stub/index.js`, `e9workers/index.js`, and `e9console/index.js`
