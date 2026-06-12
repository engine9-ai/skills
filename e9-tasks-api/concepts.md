# Key concepts

Read this before calling the API. Terms match the JSON fields you will see in responses.

## Flow (definition)

A **flow** is a named workflow template: an ordered list of tasks. Flows are provisioned for your account by your administrator — you discover them via the API; you do not create or upload flow definitions through these endpoints. Developers authoring JSON5 flow files should use the [e9-dev-tasks](../e9-dev-tasks/SKILL.md) skill instead.

You interact with definitions via:

- `GET /flows` — list all flows available to your account
- `GET /flows/:id` — read one flow by **slug**
- `POST /flows/filter` — search and paginate

**Flow slug** — the human-readable `id` on the flow object (e.g. `echo-flow`). This is what you pass as `flow_id` when creating a run.

**Flow UUID (`flow_id` on listed flows)** — a stable UUID associated with the slug. You may see it on flow and flow-run objects; for `POST /flow_runs/` use the **slug**, not this UUID.

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
| `flow_id` | UUID associated with the slug |
| `state_type` | Overall run state (`SCHEDULED`, `RUNNING`, `COMPLETED`, `FAILED`, …) |
| `task_runs` | Array of task runs created with this flow run (on create response) |

One flow run typically creates **one task run per task** in the flow definition.

## Task (step within a flow)

A **task** is a single step in a flow. Tasks appear inside flow definitions and as `task_runs` when a flow runs. There is no separate API resource for task definitions.

**`task_key`** — stable string identifying which step a task run belongs to (e.g. `echo-step`). It is a logical name within the flow, not an opaque database id.

**`dynamic_key`** — distinguishes multiple runs of the same task key (usually `"0"` for a single run).

## Task run (instance)

A **task run** is one execution of one task within a flow run.

| Field | Meaning |
|-------|---------|
| `id` | UUID — use in `GET /task_runs/:id` |
| `flow_run_id` | Parent flow run UUID |
| `task_key` | Which step in the flow |
| `state_type` | `PENDING`, `RUNNING`, `COMPLETED`, `FAILED`, … |
| `task_inputs` | Inputs for this run (including `options` passed to the step) |
| `state` | Full state object; see output below |

### Example task run (after completion)

```json
{
  "id": "0196f1d1-2c55-7d89-9c24-6fe4f7fd93ea",
  "flow_run_id": "0196f1d0-87cf-7b6f-b9a0-8f5d2d6c2f9e",
  "task_key": "echo-step",
  "state_type": "COMPLETED",
  "task_inputs": {
    "options": { "message": "hello from echo-flow", "seconds": 1 }
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

**Typical integration pattern:** create a flow run → poll `GET /task_runs/:id` until `state_type` is terminal (`COMPLETED` or `FAILED`).

## Scheduling vs execution

**Creating a run is not the same as executing it.**

| Action | What happens |
|--------|----------------|
| `POST /flow_runs/` | Creates a flow run and task runs in `PENDING` state |
| Background processing | Picks up `PENDING` tasks and executes them |
| `GET /task_runs/:id` | Reports current state |

If tasks stay `PENDING` for a long time, execution may not be running in your environment — contact your administrator.

There is no public REST endpoint that blocks until a task finishes. Poll or implement retry logic client-side.

## Task output

Completed task output is **not** returned inline in `GET /task_runs/:id`. Instead:

1. Poll until `state_type` is `COMPLETED`
2. Read `state.state_details.output_path` (or other fields your administrator documents)
3. Retrieve the result using the mechanism your deployment provides (direct file access, signed URL, follow-up API, etc.)

For the Echo sample flow, the result object echoes the task `options`:

```json
{
  "message": "hello from echo-flow",
  "seconds": 1,
  "last_run": "2026-06-08T12:00:00.000Z"
}
```

If you only have HTTP access, ask your administrator how to fetch completed task output for your account.

## IDs cheat sheet

| Name | Format | Example | Used in |
|------|--------|---------|---------|
| Flow slug | string | `echo-flow` | `POST /flow_runs/` body `flow_id`, `GET /flows/:id` |
| Flow UUID | UUID | `8f0a5e4d-...` | Filter fields, flow run `flow_id` |
| Flow run id | UUID | `0196f1d0-...` | `GET /flow_runs/:id`, task run `flow_run_id` |
| Task run id | UUID | `0196f1d1-...` | `GET /task_runs/:id` |
| Task key | string | `echo-step` | Task run `task_key`, filters |

**Common mistake:** passing the flow UUID as `flow_id` on create. Use the **slug** (`echo-flow`).

## Request flow (diagram)

```
You                    Task API                 Background execution
 |                         |                            |
 |  GET /flows             |                            |
 |------------------------>|  list flow definitions     |
 |<------------------------|                            |
 |                         |                            |
 |  POST /flow_runs/       |                            |
 |  { "flow_id": "…" }     |                            |
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

- [authentication.md](./authentication.md) — headers and Google OAuth
- [getting-started.md](./getting-started.md) — curl setup
- [echo-walkthrough.md](./echo-walkthrough.md) — full worked example
