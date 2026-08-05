# Flow authoring and TaskWorker internals

Use this when **authoring or editing flow definitions** (`.json5` files), **starting or debugging flow runs** via `TaskWorker`, `SQLTaskManager`, or MCP `task`, or **adding SQL/worker/ETL steps** with persisted task-run state.

For REST API usage (listing flows, creating runs, polling state), see [e9-tasks-api](../e9-tasks-api/SKILL.md) and [concepts.md](../e9-tasks-api/concepts.md). For operator deployment setup, see `engine9/server/api/task/docs/admin/`.

A **Flow** is a Prefect-style workflow definition: one JSON5 file per flow, containing metadata and an ordered list of **tasks**. Each task names a server **worker** (`worker_path`) and **method** (`worker_method`) plus an `options` object passed to that method. Tasks are a subset of a flow — flows are the orchestration unit, tasks are the executable unit. **TaskWorker** (`manager/TaskWorker.js`) stores definitions and run metadata, **executes tasks locally** via `runFlow()` / `executeTaskRun()`, creates pending task runs for SQLTaskManager, and **schedules** work via `scheduleTasks` (local `createFlowRun` or remote Frakture job-list). **SQLTaskManager** polls SQL-backed `task_run` rows and executes pending tasks through **Manager** (forked WorkerRunner processes).

This document is based on `manager/TaskWorker.js`, `manager/SQLTaskManager.js`, and the examples in `engine9/server/test/task/*.json5`. The Prefect-compatible REST routes for these flows are mounted on the **main API** (`api/index.js` via `api/mcp/mount.js`) so they share the same Firebase OAuth pipeline as `/mcp` — see [e9-tasks-api README](../e9-tasks-api/README.md) (API consumers) and `engine9/server/api/task/docs/admin/README.md` (operators).

Flow definition files can live **anywhere** — pass the path to `createFlowRun({ flow: '/any/path/to/flow.json5', ... })` or `loadFlowFromFile`. `listFlows` / `getFlowById` only scan a configured flows directory (`flowsDir`, `{ENGINE9_ACCOUNT_DIR}/{account_id}/flows`, or `TASK_API_FLOWS_DIR`); that directory is for discovery, not a requirement for where files are stored.

Flow run JSON is written under `TASK_API_RUNS_DIR` (default: OS temp `task-api-runs`). Task runs are stored in the account database `task_run` table (`originator='sql'` by default). Task-run output/error artifacts are written under `ENGINE9_TASK_RUN_LOG_DIR` (defaults to `${ENGINE9_LOG_DIR}/tasks`, which itself defaults to `/var/log/engine9/tasks`) — **not** under `ENGINE9_ACCOUNT_DIR`, so logs stay out of git-managed account directories.

## Naming (Prefect-style)

Prefect uses **snake_case** for variables, parameters, and task names in Python (PEP 8), and Prefect **Variables** names are restricted to lowercase letters, digits, and underscores. Engine9 flow JSON follows the same style: `task_key`, `worker_path`, `worker_method`, `flow_run_id`, etc. Use snake_case in flow files and in `options` keys unless a specific worker method documents otherwise.

## Flow file format (JSON5)

One flow per file. Minimal shape (see `engine9/server/test/task/example.json5`):

```json5
{
  id: 'my-flow',           // slug; also used to derive stable flow_id UUID
  name: 'Human-readable name',
  tags: ['repair', 'person'],
  labels: { env: 'prod' },
  tasks: [
    { task_key: 'step-one', name: 'First step' },
    { task_key: 'step-two', name: 'Second step' },
  ],
}
```

Executable tasks must include worker routing (see `engine9/server/test/task/echo-flow.json5`):

```json5
{
  task_key: 'echo-step',
  name: 'Echo step',
  worker_path: 'workers/EchoWorker',
  worker_method: 'echo',
  options: { message: 'hello', seconds: 1 },
}
```

| Field | Purpose |
|-------|---------|
| `id` | Flow slug (filename stem if omitted). Stable `flow_id` UUID is derived via `uuidv5(id, FLOW_ID_NAMESPACE)`. |
| `name` | Display name |
| `tags`, `labels` | Optional metadata |
| `schedule` | Optional scheduling overrides for `TaskWorker.scheduleTasks` / MCP `task`: `{ label, tracking_code, start_after_timestamp }` |
| `tasks[]` | Ordered steps |
| `task_key` | Unique key per task; auto-generated from id/name/worker if omitted |
| `worker_path` | Path under server `workers/` (e.g. `workers/EchoWorker`, `workers/PersonWorker`) |
| `worker_method` | Worker method name (e.g. `echo`, `loadPeople`) |
| `options` | Passed to the worker method (must match that method's metadata) |

Comments and trailing commas are allowed (JSON5). String values in `options` must be valid JSON5 (no template literals or `+` concatenation); escape inner quotes or use a single line.

## TaskWorker: definitions and runs

`TaskWorker` methods:

- **`listFlows` / `getFlowById`** — read and normalize all `.json5` files in the flows directory (`normalizeFlow` fills `task_key`, `flow_id`, timestamps).
- **`createFlowRun`** — create a flow run under `flow_runs/{uuid}/run.json` and **one `task_run` per task** (does not execute them).
- **`runFlow`** — `createFlowRun` then **`executeTaskRun`** for each task in order (blocking). Each task runs in a forked `WorkerRunner` child process, same as Manager. Updates flow and task run state (`RUNNING` → `COMPLETED` / `FAILED`). Set `stop_on_error: false` to continue after a failed task.
- **`executeTaskRun`** — run a single task run in-process and persist state/output artifacts.
- **`createTaskRun` / `getTaskRun` / `updateTaskRun` / `listTaskRunIds`** — persist task runs in SQL; large outputs/errors go to `{ENGINE9_TASK_RUN_LOG_DIR}/{account_id}/flows/{flowId}/{date}/task_runs/{taskRunId}/`.
- **`loadFlowFromFile`** — load a single JSON5 path without listing the flows directory.

### Starting a run (execute immediately)

```javascript
const taskWorker = new TaskWorker({ accountId: 'frakture' });
const flowRun = await taskWorker.runFlow({
  flow: '/path/to/personId.json5',
  body: { account_id: 'frakture' },
});
// flowRun.state_type === 'COMPLETED' when all tasks succeed
// flowRun.task_runs[] — each task run with final state and output artifacts
```

### Starting a run (schedule only)

```javascript
const flowRun = await taskWorker.createFlowRun({
  flow: '/path/to/personId.json5',
  body: { account_id: 'frakture' },
});
// flowRun.task_runs[] — PENDING rows for SQLTaskManager to pick up
```

Requirements:

- **`account_id`** in the body (or on the worker) — required for DB-backed task runs.
- **`flow` file path** (any location), or **`flowId`** if the flow is listed via `getFlowById` / `listFlows`.

`TaskWorker.createFlowRun` and `createTaskRun` deploy `@engine9/interfaces/task` uniquely for the account (`SchemaWorker.install` plus schema `deploy`), so `task_run` exists even when `installStandard` was not run. `runFlow()` calls `createFlowRun`, which performs this step.

Idempotency: pass `body.idempotency_key` to return an existing run for the same flow.

### Scheduling via TaskWorker / MCP `task`

Pass `flow_path` (or MCP `flow_path`) to load a flow file with `loadFlowFromFile`. Tasks come from `flow.tasks`; optional schedule fields come from `flow.schedule` (or top-level overrides on the call). Explicit call parameters override flow defaults. Local tasks use `worker_path` values under `workers/...`; remote/plugin tasks use dotted plugin paths (e.g. `engine9.Engine9Workers.EchoWorker`). Mixed local and remote tasks in one flow are not supported in a single `scheduleTasks` call.

## SQLTaskManager: execution

`SQLTaskManager` polls `task_run` where `originator='sql'` and `state_type='PENDING'`, maps `task_inputs` to a Manager task:

- `worker_path` / `worker_method` from `task_inputs`
- `options` from `task_inputs.options`

It sets the task to `RUNNING`, pushes `task_start` to Manager, and on completion/failure updates `task_run` state (`COMPLETED` / `FAILED`) and writes `output.json` or timestamped `error.*.json` under the account flows artifact path.

Run SQLTaskManager alongside workers (or as a long-lived process) after `createFlowRun`. See `engine9/server/test/task/sql-task-manager-e2e.test.js`.

## Designing tasks

Each task is the same shape: pick a worker, a method, and an `options` object. Options must match `Worker.prototype.{method}.metadata.options` (see worker source, or MCP `sql` with `command: "describe"` for table schema).

Example (echo):

```json5
{
  task_key: 'echo-step',
  name: 'Echo step',
  worker_path: 'workers/EchoWorker',
  worker_method: 'echo',
  options: { message: 'hello', seconds: 1 },
}
```

Example (SQL — single statement):

```json5
{
  task_key: 'count_rows',
  name: 'Count rows',
  worker_path: 'workers/SQLWorker',
  worker_method: 'query',
  options: { sql: 'select count(*) as n from temp_example' },
}
```

`SQLWorker.query` accepts one `sql` string (and optional `values`). **Do not** pass semicolon-separated batches to `query` — drivers may reject them or run statements in an undefined order. Use `SQLWorker.queries` when a flow step needs multiple statements (e.g. drop then create):

```json5
{
  task_key: 'build_staging_table',
  name: 'Build staging table',
  worker_path: 'workers/SQLWorker',
  worker_method: 'queries',
  options: {
    queries: {
      drop_temp: { sql: 'drop table if exists temp_example' },
      create_temp: { sql: 'create table temp_example as select ...' },
    },
  },
}
```

Each key in `options.queries` is a label; each value is a SQL string or `{ sql, values? }`. Statements run in object key order; the method returns `{ [label]: rows }`. Temp table naming helpers (`getTempTableName`, `dropTempTables`) live on SQLWorker if you need them; otherwise use explicit `temp_*` table names and `drop table if exists` when re-running a flow.

### Task ordering

Tasks in the `tasks` array are created in order when using `createFlowRun({ flow: filePath })`. SQLTaskManager does not enforce dependencies automatically; later tasks should assume earlier tables/artifacts exist, or you should gate runs manually until prior task runs reach `COMPLETED`.

### Task keys

If `task_key` is omitted, TaskWorker generates:

`{flowId}.{taskName}.{worker}.{method}` (sanitized). Prefer explicit `task_key` values for stable logs and reruns.

## Repair / patch flows (example pattern)

For data repairs (invalid identifiers, bad hashes, etc.):

1. **Discover** — CTAS or `select` into a `temp_*` staging table grouped by keys you need to inspect.
2. **Plan** — optional tasks to export samples or counts (SQL or FileWorker).
3. **Fix** — PersonWorker / SQL updates / re-ingest (add tasks as needed).
4. **Verify** — SQL checks that bad rows are gone.

The blank SHA-256 `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` is the hash of an empty string; it often indicates a blank `person_identifier.id_value`.

## Checklist for a new flow

1. Choose `id`, `name`, and tags.
2. List tasks in execution order with `task_key`, `name`, `worker_path`, `worker_method`, `options`.
3. Save the `.json5` file wherever it fits your repo layout; pass that path to `createFlowRun`.
4. Run `createFlowRun` with `account_id` for the target account.
5. Call `taskWorker.runFlow(...)` to execute inline, or ensure SQLTaskManager is running to execute `PENDING` task runs from `createFlowRun`.
6. Inspect `task_run` state in the DB and artifacts under `${ENGINE9_TASK_RUN_LOG_DIR:-/var/log/engine9/tasks}/{account_id}/flows/{flowId}/{date}/task_runs/{id}/output.json`.

## Related code

- `engine9/server/manager/TaskWorker.js` — definitions, runs, local task execution, task_run persistence, `scheduleTasks`
- `engine9/server/api/task/index.js` — Prefect-compatible REST router (see [endpoints.md](../e9-tasks-api/endpoints.md) and `engine9/server/api/task/docs/admin/`)
- `engine9/server/manager/SQLTaskManager.js` — poll and dispatch
- `engine9/server/test/task/echo-flow.json5` — minimal executable sample
- `engine9/server/test/task/sql-task-manager-e2e.test.js` — end-to-end execution test
- `engine9/server/test/task/flow-task-storage.test.js` — DB + artifact storage
