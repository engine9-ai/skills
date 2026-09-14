---
name: e9-tasks-api
description: >-
  Use the engine9 Task API (flows, schedule, task_runs/filter, flow runs) over HTTP with
  e9k API keys and task scopes. POST /flow_runs/ schedules a predefined flow (flow_id
  required); POST /tasks/schedule schedules an on-demand task (plugin path + method).
  Built-in workers use `@engine9/plugins/e9workers:<Worker>` (e.g. EchoWorker + echo).
  Scheduling uses the same scheduleTasks path as MCP task. MCP itself stays on Firebase
  OAuth — do not mix credentials. For deploying a built-in flow to an account (e.g.
  identity-rebuild), see deploy-flow.md.
---

# engine9 Task API

The engine9 Task API schedules predefined flows and on-demand worker methods over HTTP, then exposes asynchronous flow and task run state. Use this skill for scripts, curl, and integrations, or when troubleshooting direct-HTTP authentication, account scoping, scheduling, and run state.

## Quick reference

| Need | Route or requirement |
|------|----------------------|
| Production base URL | `https://data.engine9.ai` |
| Authentication | `Authorization: Bearer e9key_…` plus `X-ENGINE9-ACCOUNT-ID` |
| Discover flows | `GET /flows` with `tasks:read` |
| Schedule a predefined flow | `POST /flow_runs/` with `flow_id` |
| Schedule an on-demand task | `POST /tasks/schedule` with `path` + `method` |
| Poll task runs | `POST /task_runs/filter` |

**Rule:** Do not mix Task API key authentication with MCP session authentication.

## Concepts

Pick the endpoint that matches the work. They both call `scheduleTasks` and return `flow_run_id` / `task_run_ids`.

| Endpoint | Schedules | Required | See also |
|----------|-----------|----------|----------|
| **`POST /flow_runs/`** | A **predefined flow** from `GET /flows` | **`flow_id`** (the flow slug) | For one plugin method: [`POST /tasks/schedule`](./endpoints.md#post-tasksschedule) |
| **`POST /tasks/schedule`** | An **on-demand task** | Plugin **`path`** + **`method`** | For a published flow: [`POST /flow_runs/`](./endpoints.md#post-flow_runs) |

An on-demand task does **not** need a `flow_id`. A predefined flow does **not** need `path` or `method` — those come from the flow definition. Do not send `flow_id` to `/tasks/schedule`.

**On-demand names** (REST and MCP): built-in engine9 Workers use `@engine9/plugins/e9workers:<Worker>` — no plugin-id lookup. Echo smoke test: `path: "@engine9/plugins/e9workers:EchoWorker"`, `method: "echo"`. See [echo-walkthrough.md](./echo-walkthrough.md#on-demand-task-names).

Routes live at the API origin root (`/flows`, `/tasks/schedule`, `/task_runs/filter`, `/flow_runs/`, …), not under `/api/task`. Read operations require `tasks:read`; scheduling and control operations require `tasks:schedule`. Requests do not block until execution finishes, so poll `POST /task_runs/filter` until terminal. `completed_since` on flow runs is computed from timestamps (`last_completed` ≤ `dataflow_last_completed`; equal is true) as described in [concepts.md](./concepts.md#completed-since).

**Rule:** Use `POST /flow_runs/` only for a published `flow_id`, and use `POST /tasks/schedule` only for an on-demand `path` plus `method`.

## Examples

Follow [echo-walkthrough.md](./echo-walkthrough.md) end to end: **ask the user for an account id and `e9key_` API key** (offer the default base URL `https://data.engine9.ai`), then schedule the Echo task, poll until it completes, and report the output — explaining each call and each response as you go. Never print the full API key.

## Workflow

1. **Discover flows** — `GET /flows` (or `POST /flows/filter`) lists published slugs; `GET /flows/:id` shows each step's `task_key` and default `options`
2. `POST /flow_runs/filter` with `{"limit":20}` — list recent **remote** runs for the account (default `remote: true`; no `flow_run_id`)
3. Schedule — pick the matching endpoint:
   - **On-demand task:** `POST /tasks/schedule` with `{ "path": "@engine9/plugins/e9workers:EchoWorker", "method": "echo", "options"? }`. Built-in workers: `@engine9/plugins/e9workers:<Worker>` (no plugin lookup). Account plugins: path from discovery, then this endpoint. Optional flow-run **`tags`** (Prefect name for Frakture `tracking_code`, one string). See also `POST /flow_runs/`.
   - **Predefined flow:** `GET /flows` then `POST /flow_runs/` with `{ "flow_id": "<slug>", "options"? }` — optional top-level `options` (e.g. `start` / `end`) merges into every step; optional `tasks: [{ task_key, options }]` for per-step overrides. `flow_id` is required. Optional `tags` same as on-demand (not flow-definition tags). See also `POST /tasks/schedule`.
   → save `flow_run_id` / `task_run_ids`
4. `POST /task_runs/filter` — `{ "flow_run_id": "…" }` until complete (each `task_run` includes **`log_link`**; no `checkpoints`)
5. Remote output: `GET /task_runs/:id` (`output`, `resolved_options`, **`checkpoints`**) or `GET /task_runs/:id/output`. Logs: `GET /task_runs/:id/log` (`log`, `truncated`, optional `log_url`). Per-task **Run now**: `POST /task_runs/:id/retry`. Checkpoints are worker-written option snapshots and appear only on this single-task read — not on `POST /task_runs/filter`.

Full curl: [echo-walkthrough.md](./echo-walkthrough.md).

## Endpoints

| Method | Path | Scope |
|--------|------|--------|
| GET | `/flows`, `/flows/:id` | `tasks:read` |
| POST | `/flows/filter` | `tasks:read` |
| POST | `/tasks/schedule` | `tasks:schedule` |
| POST | `/tasks/describe` | `tasks:read` |
| POST | `/flow_runs/` | `tasks:schedule` |
| GET | `/flow_runs/:id` | `tasks:read` |
| POST | `/flow_runs/filter` | `tasks:read` |
| POST | `/flow_runs/archive`, `/flow_runs/retry` | `tasks:schedule` (bulk `flow_run_ids`; add `parent_account_id` to span children — [archive](./endpoints.md#post-flow_runsarchive)) |
| POST | `/flow_runs/:id/set_state` | `tasks:schedule` |
| GET | `/task_runs/:id`, `/task_runs/:id/log`, `/task_runs/:id/output` | `tasks:read` |
| POST | `/task_runs/filter` | `tasks:read` |
| POST | `/task_runs/:id/retry`, `/pause`, `/resume`, `/stop`, `/set_state` | `tasks:schedule` |
| PATCH | `/task_runs/:id` | `tasks:schedule` |

Details: [endpoints.md](./endpoints.md).

## Troubleshooting

| Status | Fix |
|--------|-----|
| 401 | Missing/invalid `e9key_` key or account header — [authentication.md](./authentication.md) |
| 403 | Wrong account or missing scope |
| 404 | Wrong flow slug or run id |
| 409 | `PATCH /task_runs/:id` on a RUNNING or terminal run; action not in `allowed_actions` |
| 410 | Removed listing path — use `POST /task_runs/filter` |
| 422 | Missing `flow_id` (predefined flow) or `path`+`method` (on-demand task); retry of a RUNNING task without `force`; legacy Mongo status token in a filter |
| 503 | API or `api_key` table not configured — administrator: MCP `apiKey` create, or `e9 sqlworker createApiKey` |

Full list: [errors.md](./errors.md).

## Related documentation

| Documentation | Use when |
|---------------|----------|
| [concepts.md](./concepts.md) | Terminology: flows, runs, IDs, async execution, `completed_since`, checkpoints |
| [authentication.md](./authentication.md) | API keys, task scopes, and account headers |
| [getting-started.md](./getting-started.md) | First requests in five minutes |
| [deploy-flow.md](./deploy-flow.md) | Check and schedule a built-in or account flow with MCP or CLI |
| [echo-walkthrough.md](./echo-walkthrough.md) | Run the end-to-end Echo schedule, poll, and completion demo |
| [endpoints.md](./endpoints.md) | Review every HTTP route and curl examples |
| [errors.md](./errors.md) | Interpret HTTP status codes |
| [e9-mcp](../e9-mcp/SKILL.md) | Schedule and inspect work through an MCP session |
| [e9-api-key](../e9-api-key/SKILL.md) | Create and manage Task API credentials |
| [e9-dev-tasks](../e9-dev-tasks/SKILL.md) | Design and build flow JSON5 files |
