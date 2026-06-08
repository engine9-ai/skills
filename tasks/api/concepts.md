# Key concepts

Read this before calling the API. Terms match the JSON fields you will see in responses.

## Flow (definition)

A **flow** is a named workflow template: an ordered list of tasks. It is stored as a JSON5 file on the server. You do not upload flows via the API — your administrator publishes them to the account flows directory.

You interact with definitions via:

- `GET /flows` — list all flows available to your account
- `GET /flows/:id` — read one flow by **slug**
- `POST /flows/filter` — search and paginate

**Flow slug** — the human-readable `id` in the flow file (e.g. `echo-flow`). This is what you pass as `flow_id` when creating a run.

**Flow UUID (`flow_id` on listed flows)** — a deterministic UUID derived from the slug. You may see it on flow and flow-run objects; for `POST /flow_runs/` use the **slug**, not this UUID.

### Example flow object (abridged)

```json
{
  "id": "echo-flow",
  "name": "Echo Flow",
  "flow_id": "8f0a5e4d-b82b-5c8f-b4c7-46a39b2d9be0",
  "tags": ["echo", "test"],
  "tasks": [
    {
      "task_key": "echo-step",
      "name": "Echo step",
      "worker_path": "workers/EchoWorker",
      "worker_method": "echo",
      "options": { "message": "hello from echo-flow", "seconds": 1 }
    }
  ]
}
```

## Flow run (instance)

A **flow run** is one execution of a flow. Created with `POST /flow_runs/`.

| Field | Meaning |
|-------|---------|
| `id` | UUID for this run — save it immediately |
| `flow_slug` | Which flow template (e.g. `echo-flow`) |
| `flow_id` | UUID derived from slug |
| `state_type` | Overall run state (`SCHEDULED`, `RUNNING`, `COMPLETED`, `FAILED`, …) |
| `task_runs` | Array of task runs created with this flow run (on create response) |

One flow run typically creates **one task run per task** in the flow definition.

## Task (step within a flow)

A **task** is a single step in a flow: a worker path, method, and options. Tasks are defined inside the flow file, not as separate API resources.

**`task_key`** — stable string identifying which step a task run belongs to (e.g. `echo-step`). It is a logical name, not a database foreign key.

**`dynamic_key`** — distinguishes multiple runs of the same task key (usually `"0"` for a single run).

## Task run (instance)

A **task run** is one execution of one task within a flow run.

| Field | Meaning |
|-------|---------|
| `id` | UUID — use in `GET /task_runs/:id` |
| `flow_run_id` | Parent flow run UUID |
| `task_key` | Which step in the flow |
| `state_type` | `PENDING`, `RUNNING`, `COMPLETED`, `FAILED`, … |
| `task_inputs` | Worker path, method, options copied from the flow |
| `state` | Full state object; see output below |

### Example task run (after completion)

```json
{
  "id": "0196f1d1-2c55-7d89-9c24-6fe4f7fd93ea",
  "flow_run_id": "0196f1d0-87cf-7b6f-b9a0-8f5d2d6c2f9e",
  "task_key": "echo-step",
  "state_type": "COMPLETED",
  "task_inputs": {
    "worker_path": "workers/EchoWorker",
    "worker_method": "echo",
    "options": { "message": "hello from echo-flow", "seconds": 1 }
  },
  "state": {
    "type": "COMPLETED",
    "state_details": {
      "output_path": "/var/log/engine9/tasks/test/flows/echo-flow/2026-06-08/task_runs/0196f1d1-.../output.json"
    }
  }
}
```

## Run states

Both flow runs and task runs use string `state_type` values.

| `state_type` | Meaning for task runs |
|--------------|----------------------|
| `PENDING` | Scheduled, not yet executing |
| `RUNNING` | Worker is executing |
| `COMPLETED` | Finished successfully |
| `FAILED` | Finished with error |
| `CANCELLED` | Stopped before completion |
| `CRASHED` | Unexpected failure |

**Typical integration pattern:** create a flow run → poll `GET /task_runs/:id` until `state_type` is terminal (`COMPLETED` or `FAILED`).

## Scheduling vs execution

**Creating a run is not the same as executing it.**

| Action | What happens |
|--------|----------------|
| `POST /flow_runs/` | Creates flow run + `PENDING` task runs in the database |
| Server background (TaskManager) | Picks up `PENDING` tasks and runs workers |
| `GET /task_runs/:id` | Reports current state |

If tasks stay `PENDING`, the server executor may not be running — contact your administrator ([execution.md](../../../server/api/task/docs/admin/execution.md)).

There is no public REST endpoint that blocks until a task finishes. Poll or implement retry logic client-side.

## Task output

Completed task output is **not** returned inline in `GET /task_runs/:id`. Instead:

1. Poll until `state_type` is `COMPLETED`
2. Read `state.state_details.output_path`
3. Fetch that file (filesystem path on server) or ask your platform for a download URL

For the Echo sample, `output.json` contains the echoed `options` object:

```json
{
  "message": "hello from echo-flow",
  "seconds": 1,
  "last_run": "2026-06-08T12:00:00.000Z",
  "cwd": "/path/to/server"
}
```

If your integration cannot read server filesystem paths, coordinate with your administrator on how to expose artifacts.

## IDs cheat sheet

| Name | Format | Example | Used in |
|------|--------|---------|---------|
| Flow slug | string | `echo-flow` | `POST /flow_runs/` body `flow_id`, `GET /flows/:id` |
| Flow UUID | UUID | `8f0a5e4d-...` | Filter fields, flow run `flow_id` |
| Flow run id | UUIDv7 | `0196f1d0-...` | `GET /flow_runs/:id`, task run `flow_run_id` |
| Task run id | UUIDv7 | `0196f1d1-...` | `GET /task_runs/:id` |
| Task key | string | `echo-step` | Task run `task_key`, filters |

**Common mistake:** passing the flow UUID as `flow_id` on create. Use the **slug** (`echo-flow`).

## Request flow (diagram)

```
You                          Task API                    Server executor
 |                               |                              |
 |  GET /flows                   |                              |
 |----------------------------->|  list JSON5 definitions      |
 |<-----------------------------|                              |
 |                               |                              |
 |  POST /flow_runs/             |                              |
 |  { "flow_id": "echo-flow" }   |                              |
 |----------------------------->|  create flow + task runs     |
 |<-----------------------------|  (task runs: PENDING)        |
 |                               |----------------------------->| execute
 |                               |                              |
 |  GET /task_runs/:id (poll)    |                              |
 |----------------------------->|  state_type: RUNNING...      |
 |<-----------------------------|                              |
 |                               |                              |
 |  GET /task_runs/:id           |                              |
 |----------------------------->|  state_type: COMPLETED       |
 |<-----------------------------|  output_path in state       |
```

## Next steps

- [authentication.md](./authentication.md) — headers
- [getting-started.md](./getting-started.md) — curl setup
- [echo-walkthrough.md](./echo-walkthrough.md) — full worked example
