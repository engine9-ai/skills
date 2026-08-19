---
name: e9-tasks-api
description: >-
  Use the Engine9 Task API (flows, schedule/listTasks, flow runs) over HTTP with
  e9k API keys and task scopes. Scheduling uses the same scheduleTasks path as
  MCP task. MCP itself stays on Firebase OAuth — do not mix credentials.
---

# Engine9 Task API

Use this skill when calling the Task API from scripts, curl, or integrations — or when troubleshooting auth, account scoping, and run state.

When the user is working through **MCP** (not REST/curl), use MCP `account` and MCP `task` per [e9-mcp](../e9-mcp/SKILL.md). This skill is for direct HTTP against the Task API.

For designing and building flow JSON5 files (developers only), see [e9-dev-tasks](../e9-dev-tasks/SKILL.md).

## Documentation

| Doc | Use when |
|-----|----------|
| [concepts.md](./concepts.md) | Terminology: flows, runs, IDs, async execution, `completed_since` |
| [authentication.md](./authentication.md) | **API keys (`e9key_…`), scopes (`tasks:read` / `tasks:schedule`), account header** |
| [getting-started.md](./getting-started.md) | First requests in five minutes |
| [echo-walkthrough.md](./echo-walkthrough.md) | Full Echo schedule → listTasks walkthrough |
| [endpoints.md](./endpoints.md) | Every route with curl examples |
| [errors.md](./errors.md) | HTTP status codes |

## Quick reference

- **Base URL:** `https://data.engine9.ai` (production default)
- Routes at API origin root (`/flows`, `/tasks/schedule`, `/tasks/listTasks`, `/flow_runs/`, … — not under `/api/task`)
- Auth: `Authorization: Bearer e9key_…` (or `X-API-Key`) + `X-ENGINE9-ACCOUNT-ID`
- Scopes: `tasks:read` (discover/listTasks), `tasks:schedule` (schedule) — see [authentication.md](./authentication.md)
- Prefer **`POST /tasks/schedule`** and **`POST /tasks/listTasks`** (same as MCP `task`)
- `POST /flow_runs/` with `flow_id` also calls `scheduleTasks` for a published flow slug
- Does not block until execution finishes — poll listTasks until terminal
- `completed_since` on flow runs is computed from timestamps (`last_completed` ≤ `dataflow_last_completed`; **equal is true**) — [concepts.md](./concepts.md#completed-since)

## Typical workflow

1. `GET /flows` — confirm published flow slugs (needs `tasks:read`)
2. `POST /flow_runs/filter` with `{"limit":20}` — list recent **remote** runs for the account (default `remote: true`; no `flow_run_id`)
3. `POST /tasks/schedule` — `{ "path", "method", "options" }` or `{ "flow_path" }` → save `flow_run_id` / `task_run_ids`
   - or `POST /flow_runs/` with `{ "flow_id": "<slug>" }`
4. `POST /tasks/listTasks` — `{ "flow_run_id": "…" }` until complete
5. Retrieve output using your administrator's documented method

Full curl: [echo-walkthrough.md](./echo-walkthrough.md).

## Endpoints

| Method | Path | Scope |
|--------|------|--------|
| GET | `/flows`, `/flows/:id` | `tasks:read` |
| POST | `/flows/filter` | `tasks:read` |
| POST | `/tasks/schedule` | `tasks:schedule` |
| POST | `/tasks/listTasks` | `tasks:read` |
| POST | `/flow_runs/` | `tasks:schedule` |
| GET | `/flow_runs/:id` | `tasks:read` |
| POST | `/flow_runs/filter` | `tasks:read` |
| POST | `/flow_runs/archive`, `/flow_runs/retry` | `tasks:schedule` |
| POST | `/flow_runs/:id/set_state` | `tasks:schedule` |
| GET | `/task_runs/:id` | `tasks:read` |
| POST | `/task_runs/filter`, `/task_runs/:id/set_state` | read / schedule |

Details: [endpoints.md](./endpoints.md).

## Common errors

| Status | Fix |
|--------|-----|
| 401 | Missing/invalid `e9key_` key or account header — [authentication.md](./authentication.md) |
| 403 | Wrong account or missing scope |
| 404 | Wrong flow slug or run id |
| 422 | Missing `flow_id` / `path`+`method` / `flow_run_id` / `flow_run_ids` |
| 503 | API or `api_key` table not configured — administrator: `e9 sqlworker createApiKey` |

Full list: [errors.md](./errors.md).
