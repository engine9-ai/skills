---
name: e9-tasks-api
description: >-
  Use the engine9 Task API (flows, schedule, task_runs/filter, flow runs) over HTTP with
  e9k API keys and task scopes. POST /flow_runs/ schedules a predefined flow (flow_id
  required); POST /tasks/schedule schedules an on-demand task (plugin path + method).
  Built-in workers use `@engine9/plugins/e9workers:<Worker>` (e.g. EchoWorker + echo).
  Scheduling uses the same scheduleTasks path as MCP task. MCP itself stays on Firebase
  OAuth — do not mix credentials.
---

# engine9 Task API

Use this skill when calling the Task API from scripts, curl, or integrations — or when troubleshooting auth, account scoping, and run state.

When the user is working through **MCP** (not REST/curl), use MCP `account` and MCP `task` per [e9-mcp](../e9-mcp/SKILL.md). This skill is for direct HTTP against the Task API.

For designing and building flow JSON5 files (developers only), see [e9-dev-tasks](../e9-dev-tasks/SKILL.md).

## Two schedule endpoints

Pick the endpoint that matches the work. They both call `scheduleTasks` and return `flow_run_id` / `task_run_ids`.

| Endpoint | Schedules | Required | See also |
|----------|-----------|----------|----------|
| **`POST /flow_runs/`** | A **predefined flow** from `GET /flows` | **`flow_id`** (the flow slug) | For one plugin method: [`POST /tasks/schedule`](./endpoints.md#post-tasksschedule) |
| **`POST /tasks/schedule`** | An **on-demand task** | Plugin **`path`** + **`method`** | For a published flow: [`POST /flow_runs/`](./endpoints.md#post-flow_runs) |

An on-demand task does **not** need a `flow_id`. A predefined flow does **not** need `path` or `method` — those come from the flow definition. Do not send `flow_id` to `/tasks/schedule`.

**On-demand names** (REST and MCP): built-in engine9 Workers use `@engine9/plugins/e9workers:<Worker>` — no plugin-id lookup. Echo smoke test: `path: "@engine9/plugins/e9workers:EchoWorker"`, `method: "echo"`. See [echo-walkthrough.md](./echo-walkthrough.md#on-demand-task-names).

## Documentation

| Doc | Use when |
|-----|----------|
| [concepts.md](./concepts.md) | Terminology: flows, runs, IDs, async execution, `completed_since` |
| [authentication.md](./authentication.md) | **API keys (`e9key_…`), scopes (`tasks:read` / `tasks:schedule`), account header** |
| [getting-started.md](./getting-started.md) | First requests in five minutes |
| [echo-walkthrough.md](./echo-walkthrough.md) | **Agent-runnable** end-to-end Echo demo: schedule → poll → completion, with expected responses |
| [endpoints.md](./endpoints.md) | Every route with curl examples |
| [errors.md](./errors.md) | HTTP status codes |

## Quick reference

- **Base URL:** `https://data.engine9.ai` (production default)
- Routes at API origin root (`/flows`, `/tasks/schedule`, `/task_runs/filter`, `/flow_runs/`, … — not under `/api/task`)
- Auth: `Authorization: Bearer e9key_…` (or `X-API-Key`) + `X-ENGINE9-ACCOUNT-ID`
- Scopes: `tasks:read` (discover/list), `tasks:schedule` (schedule) — see [authentication.md](./authentication.md)
- **`POST /tasks/schedule`** — on-demand task (`path` + `method`, e.g. `@engine9/plugins/e9workers:EchoWorker` + `echo`). **`POST /flow_runs/`** — predefined flow (`flow_id` required). **`POST /task_runs/filter`** — list/poll task runs.
- Does not block until execution finishes — poll `POST /task_runs/filter` until terminal
- `completed_since` on flow runs is computed from timestamps (`last_completed` ≤ `dataflow_last_completed`; **equal is true**) — [concepts.md](./concepts.md#completed-since)

## Agent demo (prove an account/key works)

Follow [echo-walkthrough.md](./echo-walkthrough.md) end to end: **ask the user for an account id and `e9key_` API key** (offer the default base URL `https://data.engine9.ai`), then schedule the Echo task, poll until it completes, and report the output — explaining each call and each response as you go. Never print the full API key.

## Typical workflow

1. `POST /flow_runs/filter` with `{"limit":20}` — list recent **remote** runs for the account (default `remote: true`; no `flow_run_id`)
2. Schedule — pick the matching endpoint:
   - **On-demand task:** `POST /tasks/schedule` with `{ "path": "@engine9/plugins/e9workers:EchoWorker", "method": "echo", "options"? }`. Built-in workers: `@engine9/plugins/e9workers:<Worker>` (no plugin lookup). Account plugins: path from discovery, then this endpoint. See also `POST /flow_runs/`.
   - **Predefined flow:** `GET /flows` then `POST /flow_runs/` with `{ "flow_id": "<slug>" }` — `flow_id` is required. See also `POST /tasks/schedule`.
   → save `flow_run_id` / `task_run_ids`
3. `POST /task_runs/filter` — `{ "flow_run_id": "…" }` until complete
4. Remote output: `GET /task_runs/:id` (`output`, `resolved_options`) or `GET /task_runs/:id/output`. Logs: `GET /task_runs/:id/log`. Per-task **Run now**: `POST /task_runs/:id/retry`.

Full curl: [echo-walkthrough.md](./echo-walkthrough.md).

## Endpoints

| Method | Path | Scope |
|--------|------|--------|
| GET | `/flows`, `/flows/:id` | `tasks:read` |
| POST | `/flows/filter` | `tasks:read` |
| POST | `/tasks/schedule` | `tasks:schedule` |
| POST | `/flow_runs/` | `tasks:schedule` |
| GET | `/flow_runs/:id` | `tasks:read` |
| POST | `/flow_runs/filter` | `tasks:read` |
| POST | `/flow_runs/archive`, `/flow_runs/retry` | `tasks:schedule` |
| POST | `/flow_runs/:id/set_state` | `tasks:schedule` |
| GET | `/task_runs/:id`, `/task_runs/:id/log`, `/task_runs/:id/output` | `tasks:read` |
| POST | `/task_runs/filter` | `tasks:read` |
| POST | `/task_runs/:id/retry`, `/pause`, `/resume`, `/stop`, `/set_state` | `tasks:schedule` |
| PATCH | `/task_runs/:id` | `tasks:schedule` |

Details: [endpoints.md](./endpoints.md).

## Common errors

| Status | Fix |
|--------|-----|
| 401 | Missing/invalid `e9key_` key or account header — [authentication.md](./authentication.md) |
| 403 | Wrong account or missing scope |
| 404 | Wrong flow slug or run id |
| 409 | `PATCH /task_runs/:id` on a RUNNING or terminal run; action not in `allowed_actions` |
| 410 | Removed listing path — use `POST /task_runs/filter` |
| 422 | Missing `flow_id` (predefined flow) or `path`+`method` (on-demand task); retry of a RUNNING task without `force`; legacy Mongo status token in a filter |
| 503 | API or `api_key` table not configured — administrator: `e9 sqlworker createApiKey` |

Full list: [errors.md](./errors.md).
