# Key concepts

Read this before calling the API. Terms match the JSON fields you will see in responses.

## Flow (definition)

A **flow** is a named workflow template: an ordered list of tasks. Flows are provisioned for your account by your administrator — you discover them via the API; you do not create or upload flow definitions through these endpoints. Developers authoring JSON5 flow files should use the [e9-dev-tasks](../e9-dev-tasks/SKILL.md) skill instead.

You interact with definitions via:

- `GET /flows` — list all flows available to your account
- `GET /flows/:id` — read one flow by **slug**
- `POST /flows/filter` — search and paginate

**Flow slug** — the human-readable `id` on the flow object (e.g. `nightly-sync`). This is what you pass as `flow_id` when scheduling a **predefined flow**.

**Flow UUID (`flow_id` on listed flows)** — a stable UUID associated with the slug. You may see it on flow and flow-run objects; for scheduling use the **slug**, not this UUID.

### Example flow object (abridged)

```json
{
  "id": "nightly-sync",
  "name": "Nightly sync",
  "flow_id": "8f0a5e4d-b82b-5c8f-b4c7-46a39b2d9be0",
  "tags": ["etl"],
  "tasks": [
    {
      "task_key": "load-step",
      "name": "Load step",
      "options": { "table": "person" }
    }
  ]
}
```

## Two schedule endpoints

Pick the endpoint that matches the work. Do not send `flow_id` to `/tasks/schedule`.

| Endpoint | When to use | Required fields |
|----------|-------------|-----------------|
| **`POST /flow_runs/`** | Run a published workflow (`GET /flows`) | **`flow_id`** — the flow slug (e.g. `nightly-sync`) |
| **`POST /tasks/schedule`** | Invoke one **on-demand** plugin method | Plugin **`path`** + **`method`** (optional `options`) |

See also: [POST /flow_runs/](./endpoints.md#post-flow_runs) ↔ [POST /tasks/schedule](./endpoints.md#post-tasksschedule).

An on-demand task needs only plugin identifiers and the method — **no `flow_id`**. The server wraps that one task in a flow run so you can still poll `flow_run_id` / `task_run_ids`.

**Built-in Engine9 Workers** use `@engine9/plugins/e9workers:<Worker>`. Echo smoke test: `path: "@engine9/plugins/e9workers:EchoWorker"`, `method: "echo"`. No plugin-id lookup. See [echo-walkthrough.md](./echo-walkthrough.md#on-demand-task-names).

A predefined flow needs **`flow_id`** and does **not** take `path` or `method`. Worker path and method come from the flow definition.

Do not pass a server filesystem `flow_path` unless your administrator told you to; API consumers use the published slug on `/flow_runs/`.

## Flow run (instance)

A **flow run** is one execution of a predefined flow **or** the wrapper around an on-demand scheduled task. Created with `POST /flow_runs/` (`flow_id` required) or `POST /tasks/schedule`.

| Field | Meaning |
|-------|---------|
| `id` | UUID for this run — save it immediately |
| `account_id` | Account that owns this run |
| `parent_account_id` | First id in that account's `parent_ids` (`null` if the account has no parent) |
| `parent_ids` | Full `parent_ids` array from the account document |
| `flow_slug` | Which flow template (e.g. `nightly-sync`) |
| `flow_id` | UUID associated with the slug |
| `state_type` | Overall run state (`SCHEDULED`, `RUNNING`, `COMPLETED`, `FAILED`, …) |
| `last_completed` | When this flow run last finished |
| `dataflow_last_completed` | When any run of the same dataflow last finished |
| `completed_since` | See [Completed since](#completed-since) |
| `task_runs` | Array of task runs created with this flow run (on create response) |

One flow run typically creates **one task run per task** in the flow definition.

List those task runs with **`POST /task_runs/filter`** (`{ "flow_run_id": "…" }` or Prefect `task_runs.flow_run_id.eq_`). That is the Prefect listing surface.

### Account parent

`parent_account_id` and `parent_ids` are looked up from the **account** document (`account.parent_ids`), not stored on the flow run or task run. They are included on `GET /flow_runs/:id`, `GET /task_runs/:id`, `POST /flow_runs/filter`, and `POST /task_runs/filter`.

| Field | Meaning |
|-------|---------|
| `parent_account_id` | First id in `account.parent_ids`, or `null` if the account has no parent |
| `parent_ids` | Full `parent_ids` array (an account can have more than one parent) |

`POST /flow_runs/filter` also accepts `parent_account_id` as a **filter** (default **remote** listing via Frakture): it selects runs whose account contains that id anywhere in `parent_ids` (`"none"` = accounts with no parent). That filter is "has this parent", which is not always the same as "this is the first parent" on the response. Pass `"remote": false` to list local runs instead.

### Completed since

`completed_since` is **computed from timestamps**, not from the stored Mongo flag `dataflow_completed_since_last_update`.

It is `true` when `dataflow_last_completed` exists and this flow run either never completed, or completed **at or before** that dataflow completion:

| Condition | `completed_since` |
|-----------|-------------------|
| No `dataflow_last_completed` | `false` |
| This run never completed (`last_completed` missing) and the dataflow has completed | `true` |
| `last_completed` **equals** `dataflow_last_completed` (this is the current completion) | **`true`** |
| `last_completed` **before** `dataflow_last_completed` (a later run finished) | `true` |
| `last_completed` **after** `dataflow_last_completed` | `false` |

Use it on `GET /flow_runs/:id` / `POST /task_runs/filter` responses, or as a filter on `POST /flow_runs/filter` (`{ "completed_since": true }`).

Do **not** treat a stored `dataflow_completed_since_last_update` value as the source of truth.

## Task (step)

A **task** is a single worker method: either a step inside a predefined flow, or an **on-demand task** you schedule directly with plugin `path` + `method`. There is no separate API resource for task definitions.

**`task_key`** — stable string identifying which step a task run belongs to. It is a logical name within the flow, not an opaque database id.

**`dynamic_key`** — distinguishes multiple runs of the same task key (usually `"0"` for a single run).

## Task run (instance)

A **task run** is one execution of one task within a flow run.

| Field | Meaning |
|-------|---------|
| `id` | UUID — use in `GET /task_runs/:id` |
| `flow_run_id` | Parent flow run UUID |
| `account_id` | Account that owns this run |
| `parent_account_id` | First id in that account's `parent_ids` (`null` if none) |
| `parent_ids` | Full `parent_ids` array from the account document |
| `task_key` | Which step in the flow |
| `state_type` | `PENDING`, `RUNNING`, `COMPLETED`, `FAILED`, … |
| `task_inputs` | Inputs for this run (including `options` passed to the step) |
| `state` | Full state object; see output below |

### Example task run (after completion)

```json
{
  "id": "0196f1d1-2c55-7d89-9c24-6fe4f7fd93ea",
  "flow_run_id": "0196f1d0-87cf-7b6f-b9a0-8f5d2d6c2f9e",
  "task_key": "echo",
  "state_type": "COMPLETED",
  "task_inputs": {
    "options": { "message": "hello from echo", "seconds": 1 }
  },
  "state": {
    "type": "COMPLETED",
    "state_details": {
      "output_path": "…"
    }
  }
}
```

The `output_path` value is an opaque locator for the completed result. How you retrieve the payload depends on your deployment — ask your administrator if you need a download URL or API rather than a direct path.

## Run states

Both flow runs and task runs use string `state_type` values.

| `state_type` | Meaning for task runs |
|--------------|----------------------|
| `PENDING` | Scheduled, not yet executing |
| `RUNNING` | Step is executing |
| `COMPLETED` | Finished successfully |
| `FAILED` | Finished with error |
| `CANCELLED` | Stopped before completion |
| `CRASHED` | Unexpected failure |

**Typical integration pattern:** schedule (on-demand task or predefined flow) → poll `GET /task_runs/:id` until `state_type` is terminal (`COMPLETED` or `FAILED`).

## Scheduling vs execution

**Creating a run is not the same as executing it.**

| Action | What happens |
|--------|----------------|
| `POST /flow_runs/` with `flow_id` | Schedules a **predefined flow** (slug required). See also `POST /tasks/schedule`. |
| `POST /tasks/schedule` with `path` + `method` | Schedules an **on-demand task**. See also `POST /flow_runs/`. |
| Background processing | Picks up `PENDING` tasks and executes them |
| `POST /task_runs/filter` / `GET /task_runs/:id` | Reports current state |

If tasks stay `PENDING` for a long time, execution may not be running in your environment — contact your administrator.

There is no public REST endpoint that blocks until a task finishes. Poll or implement retry logic client-side.

## Task output

Completed task output is **not** returned inline in `GET /task_runs/:id`. Instead:

1. Poll until `state_type` is `COMPLETED`
2. Read `state.state_details.output_path` (or other fields your administrator documents)
3. Retrieve the result using the mechanism your deployment provides (direct file access, signed URL, follow-up API, etc.)

For the Echo on-demand smoke test (`@engine9/plugins/e9workers:EchoWorker` + `echo`), the result object echoes the task `options`:

```json
{
  "message": "hello from echo",
  "seconds": 1,
  "last_run": "2026-06-08T12:00:00.000Z"
}
```

If you only have HTTP access, ask your administrator how to fetch completed task output for your account.

## IDs cheat sheet

| Name | Format | Example | Used in |
|------|--------|---------|---------|
| Flow slug | string | `nightly-sync` | `flow_id` on `POST /flow_runs/` (predefined flow) |
| Flow UUID | UUID | `8f0a5e4d-...` | Filter fields, flow run `flow_id` |
| On-demand path | string | `@engine9/plugins/e9workers:EchoWorker` | `path` when scheduling an on-demand task |
| Method | string | `echo` | `method` when scheduling an on-demand task |
| Flow run id | UUID | `0196f1d0-...` | `GET /flow_runs/:id`, task run `flow_run_id` |
| Task run id | UUID | `0196f1d1-...` | `GET /task_runs/:id` |
| Task key | string | `echo` | Task run `task_key`, filters |

**Common mistakes:** passing the flow UUID as `flow_id` (use the **slug**); sending `flow_id` to `POST /tasks/schedule` (use `POST /flow_runs/`); expecting an on-demand task to need a `flow_id`; treating Echo as a separate installed plugin (it is `@engine9/plugins/e9workers:EchoWorker`).

## Request flow (diagram)

```
You                    Task API                 Background execution
 |                         |                            |
 |  GET /flows             |                            |
 |------------------------>|  list published flows      |
 |<------------------------|                            |
 |                         |                            |
 |  either:                |                            |
 |  POST /flow_runs/       |  predefined flow           |
 |  { "flow_id": "…" }     |  (slug required)           |
 |  or:                    |                            |
 |  POST /tasks/schedule   |  on-demand task            |
 |  { "path", "method" }   |  (no flow_id)              |
 |------------------------>|  create flow + task runs |
 |<------------------------|  (task runs: PENDING)      |
 |                         |--------------------------->| execute
 |                         |                            |
 |  GET /task_runs/:id     |                            |
 |------------------------>|  state_type: RUNNING…      |
 |<------------------------|                            |
 |                         |                            |
 |  GET /task_runs/:id     |                            |
 |------------------------>|  state_type: COMPLETED    |
 |<------------------------|  output reference in state |
```

## Next steps

- [authentication.md](./authentication.md) — API keys (`e9key_…`) and scopes
- [getting-started.md](./getting-started.md) — curl setup
- [echo-walkthrough.md](./echo-walkthrough.md) — full worked example
