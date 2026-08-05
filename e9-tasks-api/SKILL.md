---
name: e9-tasks-api
description: >-
  Use the Engine9 Task API (flows, schedule/check, flow runs) over HTTP with
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
| [concepts.md](./concepts.md) | Terminology: flows, runs, IDs, async execution |
| [authentication.md](./authentication.md) | **API keys (`e9key_…`), scopes (`tasks:read` / `tasks:schedule`), account header** |
| [getting-started.md](./getting-started.md) | First requests in five minutes |
| [echo-walkthrough.md](./echo-walkthrough.md) | Full Echo schedule → check walkthrough |
| [endpoints.md](./endpoints.md) | Every route with curl examples |
| [errors.md](./errors.md) | HTTP status codes |

## Quick reference

- **Base URL:** `https://data.engine9.ai` (production default)
- Routes at API origin root (`/flows`, `/tasks/schedule`, `/tasks/check`, `/flow_runs/`, … — not under `/api/task`)
- Auth: `Authorization: Bearer e9key_…` (or `X-API-Key`) + `X-ENGINE9-ACCOUNT-ID`
- Scopes: `tasks:read` (discover/check), `tasks:schedule` (schedule) — see [authentication.md](./authentication.md)
- Prefer **`POST /tasks/schedule`** and **`POST /tasks/check`** (same as MCP `task`)
- `POST /flow_runs/` with `flow_id` also calls `scheduleTasks` for a published flow slug
- Does not block until execution finishes — poll check until terminal

## Typical workflow

1. `GET /flows` — confirm published flow slugs (needs `tasks:read`)
2. `POST /tasks/schedule` — `{ "path", "method", "options" }` or `{ "flow_path" }` → save `flow_run_id` / `task_run_ids`
   - or `POST /flow_runs/` with `{ "flow_id": "<slug>" }`
3. `POST /tasks/check` — `{ "flow_run_id": "…" }` until complete
4. Retrieve output using your administrator's documented method

Full curl: [echo-walkthrough.md](./echo-walkthrough.md).

## Endpoints

| Method | Path | Scope |
|--------|------|--------|
| GET | `/flows`, `/flows/:id` | `tasks:read` |
| POST | `/flows/filter` | `tasks:read` |
| POST | `/tasks/schedule` | `tasks:schedule` |
| POST | `/tasks/check` | `tasks:read` |
| POST | `/flow_runs/` | `tasks:schedule` |
| GET | `/flow_runs/:id` | `tasks:read` |
| POST | `/flow_runs/filter`, `/flow_runs/:id/set_state` | read / schedule |
| GET | `/task_runs/:id` | `tasks:read` |
| POST | `/task_runs/filter`, `/task_runs/:id/set_state` | read / schedule |

Details: [endpoints.md](./endpoints.md).

## Common errors

| Status | Fix |
|--------|-----|
| 401 | Missing/invalid `e9key_` key or account header — [authentication.md](./authentication.md) |
| 403 | Wrong account or missing scope |
| 404 | Wrong flow slug or run id |
| 422 | Missing `flow_id` / `path`+`method` / `flow_run_id` |
| 503 | API or `api_key` table not configured — contact administrator |

Full list: [errors.md](./errors.md).
