# Engine9 plugin reference

## Path resolution

`ServerBaseWorker.compilePlugin` translates:

| Config prefix | Filesystem (typical monorepo layout) |
|---------------|--------------------------------------|
| `local$@engine9/interfaces/<pkg>` | `interfaces/<pkg>/index.js` (via `../../interfaces/...` from server) |
| `local$@engine9/plugins/<pkg>` | `plugins/<pkg>/index.js` |

Transforms registered on a plugin get `path` set to the plugin path string after load.

## Transform path shape

`<pluginPath>:transforms:<exportName>`

Example: `local$@engine9/interfaces/person_email:transforms:appendEmail`

The middle segment must be literally `transforms` or resolution fails.

## Search path shape (segments, pipelines)

`local$@engine9/interfaces/<pkg>:search:<handlerKey>`

Example from `person_email/segments.js`: `local$@engine9/interfaces/person_email:search:emails`.

## Binding paths (transforms)

Documented in `ServerBaseWorker.prototype.resolveBindings`:

- `sql.query` — requires `options.lookup` as array (one column); builds `IN (...)` from batch.
- `sql.tables.upsert` — provides `tablesToUpsert` object for inbound upsert transforms.
- `remote.id` — hydrates remote id fields via `appendRemotePluginData`.

## Registration / deploy

Core server preloads a fixed set of interface modules (see `ServerBaseWorker.js` `require('../../interfaces/...')`).

`getActivePluginPaths` returns a subset used by `deployAllSchemas`; new interfaces may need to be added there (or deployed explicitly via `deploy({ schema: 'local$@...' })` in your process).

## Repository map (examples)

| Area | Repo path |
|------|-----------|
| Interface examples | `interfaces/*` (each subfolder) |
| Native plugins | `plugins/e9email`, `plugins/e9forms`, `e9stub`, `e9workers`, `e9console` |
| Runtime resolution | `server/workers/ServerBaseWorker.js` (`compilePlugin`, `resolveTransform`, `resolveBindings`) |
