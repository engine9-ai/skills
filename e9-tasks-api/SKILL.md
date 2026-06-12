---
name: e9-tasks-api
description: >-
  Use the Engine9 Prefect-compatible Task API (flows, flow runs, task runs) to
  list definitions, create runs, poll state, and read Echo task output. Same
  Bearer auth as /mcp — Google OAuth access token.
---

# Engine9 Task API

Use this skill when calling the Task API from scripts, curl, or integrations — or when troubleshooting auth, account scoping, and run state.

When the user is working through **MCP** (not REST/curl), do not discover flows or worker methods from local code — use MCP `account` and MCP `task` per [e9-mcp](../e9-mcp/SKILL.md). This skill's `GET /flows` applies to direct Task API calls against the running server.

For designing and building flow JSON5 files (developers only), see [e9-dev-tasks](../e9-dev-tasks/SKILL.md).

## Documentation

| Doc | Use when |
|-----|----------|
| [concepts.md](./concepts.md) | Terminology: flows, runs, IDs, async execution |
| [authentication.md](./authentication.md) | Google OAuth access token, `X-ENGINE9-ACCOUNT-ID` |
| [getting-started.md](./getting-started.md) | First requests in five minutes |
| [echo-walkthrough.md](./echo-walkthrough.md) | Full Echo create → list → poll → output |
| [endpoints.md](./endpoints.md) | Every route with curl examples |
| [errors.md](./errors.md) | HTTP status codes |

## Quick reference

- Routes at API origin root (`/flows`, `/flow_runs/`, `/task_runs/` — not under `/api/task`)
- Auth: `Authorization: Bearer` + `X-ENGINE9-ACCOUNT-ID`
- Token: Google OAuth access token — see [authentication.md](./authentication.md)
- `POST /flow_runs/` creates `PENDING` task runs — does not block until execution finishes
- Poll `GET /task_runs/:id` until `state_type` is `COMPLETED`
- Results: `state.state_details.output_path` (retrieve per your deployment)

## Echo workflow (summary)

1. `GET /flows` — confirm `echo-flow` slug exists
2. `POST /flow_runs/` — `{"flow_id":"echo-flow"}` → save `id`, `task_runs[0].id`
3. `POST /task_runs/filter` — `{"task_runs":{"flow_run_id":{"eq_":"<id>"}}}`
4. `GET /task_runs/<task_run_id>` — poll `state_type`
5. Retrieve output using your administrator's documented method

Full curl: [echo-walkthrough.md](./echo-walkthrough.md).

## Endpoints

| Method | Path |
|--------|------|
| GET | `/flows`, `/flows/:id` |
| POST | `/flows/filter` |
| POST | `/flow_runs/`, `/flow_runs/filter`, `/flow_runs/:id/set_state` |
| GET | `/flow_runs/:id` |
| POST | `/task_runs/`, `/task_runs/filter`, `/task_runs/:id/set_state` |
| GET | `/task_runs/:id` |

Details: [endpoints.md](./endpoints.md).

## Common errors

| Status | Fix |
|--------|-----|
| 401 | Wrong or expired token — [authentication.md](./authentication.md) |
| 404 | Wrong flow slug or run UUID |
| 422 | Missing `flow_id` on create |
| 503 | API not configured — contact administrator |

Full list: [errors.md](./errors.md).
