# Engine9 plugin reference

## Path identity vs load source

`plugin.path` is **package identity only**: `@engine9/interfaces/<pkg>` or
`@engine9/plugins/<pkg>`. Never encode load location in the path.

The server resolver (`utilities/resolvePluginModule.js`, used by
`compilePlugin` / SchemaWorker / UIWorker) loads modules in this order:

1. Explicit `source` (absolute dir / file / `file:` URL) on install/compile
2. Node package resolve (`node_modules` / npm link)
3. Monorepo sibling checkout (`../interfaces/...`, `../plugins/...`)

Legacy `local$@engine9/...` is accepted as an **input alias** and stripped to
the canonical package path. New rows and capability strings always use
`@engine9/...`.

| Identity | Typical filesystem |
|----------|-------------------|
| `@engine9/interfaces/<pkg>` | `interfaces/<pkg>/index.js` (or node_modules) |
| `@engine9/plugins/<pkg>` | `plugins/<pkg>/index.js` (or node_modules) |

Transforms registered on a plugin get `path` set to the canonical package path after load.

## Identical vs conflicting plugins

| Situation | What to do |
|-----------|------------|
| Same package, local checkout vs npm | Identical — one `plugin.path`; resolver picks files |
| Two implementations of “person” | Different identity (e.g. `@myorg/interfaces/person`) |
| Multiple installs of same contract | `metadata.unique: false` (e.g. `person_custom`) |
| Inline / not a package | `type: 'local'` + explicit `id` + schema object |

## Transform path shape

`<pluginPath>:transforms:<exportName>`

Example: `@engine9/interfaces/person_email:transforms:appendEmail`

The middle segment must be literally `transforms` or resolution fails.

## Search path shape (segments, pipelines)

`@engine9/interfaces/<pkg>:search:<handlerKey>`

Example from `person_email/segments.js`: `@engine9/interfaces/person_email:search:emails`.

## Binding paths (transforms)

Documented in `ServerBaseWorker.prototype.resolveBindings`:

- `sql.query` — requires `options.lookup` as array (one column); builds `IN (...)` from batch.
- `sql.tables.upsert` — provides `tablesToUpsert` object for inbound upsert transforms.
- `remote.id` — hydrates remote id fields via `appendRemotePluginData`.

## Registration / deploy

Core server preloads a fixed set of interface modules (see `ServerBaseWorker.js`).

`getActivePluginPaths` returns a subset used by `deployAllSchemas`; new interfaces may need to be added there (or deployed explicitly via `deploy({ schema: '@engine9/interfaces/...' })` in your process).

## Repository map (examples)

| Area | Repo path |
|------|-----------|
| Interface examples | `interfaces/*` (each subfolder) |
| Native plugins | `plugins/e9email`, `plugins/e9forms`, `e9stub`, `e9workers`, `e9console` |
| Runtime resolution | `server/utilities/resolvePluginModule.js`, `server/workers/ServerBaseWorker.js` (`compilePlugin`, `resolveTransform`, `resolveBindings`) |
