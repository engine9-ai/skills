# engine9 plugin reference

## Path identity vs load source

`plugin.path` is **package identity only**: `@engine9/interfaces/<pkg>`,
`@engine9/plugins/<pkg>`, or a third-party path such as `acme-plugins/loyalty`.
Never encode load location in the path.

Core loads plugins only from packages listed in the site’s
`package.json` → `engine9.pluginPackages` (Node also allows
`dynamicPluginPackages`). Discovery finds every `index.js` directory and
`*.plugin.js` file inside those packages under `node_modules`. Absolute paths
and `install({ source })` are not supported.

Legacy `local$@engine9/...` is accepted as an **input alias** and stripped to
the canonical package path. New rows and capability strings always use the
canonical form.

| Identity | Typical filesystem |
|----------|-------------------|
| `@engine9/interfaces/<pkg>` | `node_modules/@engine9/interfaces/<pkg>/index.js` |
| `@engine9/plugins/<pkg>` | `node_modules/@engine9/plugins/<pkg>/index.js` |
| `acme-plugins/<pkg>` | `node_modules/acme-plugins/<pkg>/index.js` |
| `@acme/engine9-plugins/<pkg>` | `node_modules/@acme/engine9-plugins/<pkg>/index.js` |

Transforms registered on a plugin get `path` set to the canonical package path after load.

## Versioning

| Source of truth | Meaning |
|-----------------|--------|
| npm `package.json` `"version"` | Only version that matters (e.g. `acme-plugins@1.4.0`) |
| Registry `packageVersion(name)` | Host’s installed package version (disk on Node; baked into Cloudflare `engine9.plugins.js`) |
| `metadata.version` | Do not set; ignored |
| Warehouse `deployed_version` | Removed; not used |

`metadata.dependencies` map **plugin paths** → semver ranges. At install:

1. The dependency path must already have a `plugin` row on the account.
2. The range must satisfy the host’s npm version of `packageNameOf(depPath)`
   (e.g. `"@engine9/interfaces/person": ">=1.7.0"` checks host
   `@engine9/interfaces`).

Same-package paths (two plugins inside `acme-plugins`) share one package
version; presence of the dependency path is the meaningful check. Cross-package
ranges (Acme → `@engine9/interfaces`) pin the interfaces release the host must
run.

See [Creating and versioning a package (Acme)](SKILL.md#creating-and-versioning-a-package-acme).

## Identical vs conflicting plugins

| Situation | What to do |
|-----------|------------|
| Same package, local checkout vs npm | Identical — one `plugin.path`; host pin chooses files |
| Two implementations of “person” | Different identity (e.g. `acme-plugins/person` vs `@engine9/interfaces/person`) |
| Multiple installs of same contract | `metadata.unique: false` (e.g. `person_custom`) plus `metadata.prefix` for isolated tables |
| Inline / not a package | `type: 'local'` + explicit `id` + schema object |

## Table naming (third-party)

**Rule:** Self-scope table names in the schema (`acme_blog_post`). Omit `metadata.prefix`. Install leaves `plugin.table_prefix` empty and deploys those names as written.

Do not set `metadata.prefix` for ordinary third-party plugins — that enables the hex allocator (`{prefix}_{hex}_`), which forces every SQL caller to discover tables via the plugin row. Reserve `metadata.prefix` for multi-instance / engine9-owned patterns. Core does not enforce stem uniqueness today; a stable company+plugin stem is still required practice because prefixes may need registration later. Details: [SKILL.md — Table naming](SKILL.md#table-naming-self-scope).

## Transform path shape

`<pluginPath>:transforms:<exportName>`

Example: `@engine9/interfaces/person_email:transforms:appendEmail`

The middle segment must be literally `transforms` or resolution fails.

## Search path shape (segments, pipelines)

`@engine9/interfaces/<pkg>:search:<handlerKey>`

Example from `person_email/segments.js`: `@engine9/interfaces/person_email:search:emails`.

Handlers export optional `title`/`description`, a JSON Schema `form` (`{ title, type: 'object', properties, required? }`), and `optionsToEQL`. Account-scoped discovery: MCP `searchOptions` / `PersonWorker.searchOptions` / `GET /data/search/options`.

## Settings (warehouse configuration)

Declared on the package (`export const settings` or sibling `settings.js`), stored per installed plugin row in `setting`. Distinct from marketplace `auth_fields` (authorization).

Account-scoped discovery uses the same compile-and-aggregate path as inbound weaving, search, and reports: list installed plugins, `compilePlugin`, read `settings`, group by plugin path.

| Surface | Call |
|---------|------|
| Worker | `PluginWorker.listSettings` / `updateSetting` |
| MCP | `plugin` `command: settings` / `setSetting` |
| HTTP | `GET /data/settings`, `POST /data/settings` |

`compilePlugin` (SchemaWorker and ServerBaseWorker) attaches `settings` from the default export, a named `settings` export, or sibling `settings.js`.

## Binding paths (transforms)

Documented in `ServerBaseWorker.prototype.resolveBindings`:

- `sql.query` — requires `options.lookup` as array (one column); builds `IN (...)` from batch.
- `sql.tables.upsert` — provides `tablesToUpsert` object for inbound upsert transforms.
- `remote.id` — hydrates remote id fields via `appendRemotePluginData`.

## Registration / deploy

Core discovers plugins from `engine9.pluginPackages` at process start (Node) or
build time (`e9core build-plugins` for Cloudflare). A path that is not in a
listed, installed package is not available to `listAvailable` or `install`.

On the private server, `getActivePluginPaths` may still list a subset used by
`deployAllSchemas`; new Engine9 interfaces may need to be added there (or
deployed explicitly via `deploy({ schema: '@engine9/interfaces/...' })`).

## Repository map (examples)

| Area | Repo / package path |
|------|---------------------|
| Interface examples | `interfaces/*` (each subfolder of `@engine9/interfaces`) |
| Native plugins | `@engine9/plugins/*` (e.g. e9email, e9forms, e9workers) |
| Third-party (Acme) | npm `acme-plugins` or `@acme/engine9-plugins` with subfolder plugins |
| Core registry | `@engine9/core` `lib/pluginRegistry.js`, `bin/buildPlugins.js`, `bin/nodePluginRegistry.js` |
| Server resolution | `server/utilities/resolvePluginModule.js`, `ServerBaseWorker` (`compilePlugin`, …) |
