---
name: task-api
description: >-
  Use the Engine9 Prefect-compatible Task API (flows, flow runs, task runs) to
  list definitions, create runs, poll state, and read Echo task output. Mounted
  on the main API alongside /mcp with shared OAuth auth.
---

# Engine9 Task API

Use this skill when calling the Task API from scripts, curl, or integrations — or when troubleshooting auth, account scoping, and run state.

## Documentation

| Doc | Use when |
|-----|----------|
| [concepts.md](./concepts.md) | Terminology: flows, runs, IDs, async execution |
| [authentication.md](./authentication.md) | Bearer token, `X-ENGINE9-ACCOUNT-ID` |
| [getting-started.md](./getting-started.md) | First requests in five minutes |
| [echo-walkthrough.md](./echo-walkthrough.md) | Full Echo create → list → poll → output |
| [endpoints.md](./endpoints.md) | Every route with curl examples |
| [errors.md](./errors.md) | HTTP status codes |

**Administrators** (server setup, env vars, TaskManager): `server/api/task/docs/admin/`.

## Quick reference

- Router: `server/api/task/index.js`, mounted at API root (`/flows`, not `/api/task/flows`)
- Auth: `Authorization: Bearer` + `X-ENGINE9-ACCOUNT-ID`
- `POST /flow_runs/` creates PENDING task runs — does not execute workers
- Poll `GET /task_runs/:id` until `state_type` is `COMPLETED`
- Output: `state.state_details.output_path` → `output.json`

## Echo workflow (summary)

1. `GET /flows` — confirm `echo-flow` slug exists
2. `POST /flow_runs/` — `{"flow_id":"echo-flow"}` → save `id`, `task_runs[0].id`
3. `POST /task_runs/filter` — `{"task_runs":{"flow_run_id":{"eq_":"<id>"}}}`
4. `GET /task_runs/<task_run_id>` — poll `state_type`
5. Read `output.json` at `state.state_details.output_path`

Full curl: [echo-walkthrough.md](./echo-walkthrough.md).

## Endpoints

| Method | Path |
|--------|------|
| GET | `/flows`, `/flows/:id`, `/flows_dir` |
| POST | `/flows/filter` |
| POST | `/flow_runs/`, `/flow_runs/filter`, `/flow_runs/:id/set_state` |
| GET | `/flow_runs/:id` |
| POST | `/task_runs/`, `/task_runs/filter`, `/task_runs/:id/set_state` |
| GET | `/task_runs/:id` |

Details: [endpoints.md](./endpoints.md).

## Common errors

| Status | Fix |
|--------|-----|
| 401 | Bearer token — [authentication.md](./authentication.md) |
| 404 | Wrong flow slug or run UUID |
| 422 | Missing `flow_id` on create |
| 503 | Admin: flows directory — `server/api/task/docs/admin/configuration.md` |

Full list: [errors.md](./errors.md).

## Code references (server repo)

- `api/task/index.js`, `api/mcp/mount.js`
- `manager/TaskWorker.js`, `manager/TaskManager.js`
- `test/task/echo-flow.json5`, `workers/EchoWorker.js`
- `server/skills/flows/SKILL.md` — flow authoring
