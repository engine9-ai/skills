# Task API — end-user documentation

The engine9 Task API lets developers and integrators schedule work and inspect asynchronous runs over HTTP. Use this documentation when building a direct HTTP integration against the production service or an administrator-provided staging or local host.

## Quick reference

| Need | Value |
|------|-------|
| Production base URL | `https://data.engine9.ai` |
| Authentication | `e9key_…` with `X-ENGINE9-ACCOUNT-ID` |
| Read scope | `tasks:read` |
| Schedule and control scope | `tasks:schedule` |
| Poll runs | `POST /task_runs/filter` |

**Rule:** Task API routes live at the API origin root, not under `/api/task`.

## Concepts

Obtain these values from your administrator before integrating:

| Item | Example |
|------|---------|
| Base URL | `https://data.engine9.ai` |
| API key | `e9key_…` with `tasks:read` and/or `tasks:schedule` — see [authentication.md](./authentication.md) |
| Account id | `<account_id>` — sent as `X-ENGINE9-ACCOUNT-ID` |
| Available flows | Slugs from `GET /flows` (optional; on-demand Echo does not need a flow) |
| Output retrieval | How to fetch completed task results for your environment |

Operators and deployment setup are documented separately (ask your administrator).

## Workflow

Routes live at the **API origin root** — not under `/api/task`:

```
GET  /flows
POST /tasks/schedule
POST /task_runs/filter
POST /flow_runs/
GET  /flow_runs/:id
POST /flow_runs/filter
POST /flow_runs/count
POST /flow_runs/metrics
GET  /task_runs/:id
GET  /task_runs/:id/log
GET  /task_runs/:id/output
POST /task_runs/:id/retry
POST /task_runs/:id/pause
POST /task_runs/:id/reset_checkpoints
PATCH /task_runs/:id
...
```

Primary integration path: pick the **schedule endpoint**, then **list** via `POST /task_runs/filter`.

- **Predefined flow:** `POST /flow_runs/` with **`flow_id`** (published slug) — see also `POST /tasks/schedule`
- **On-demand task:** `POST /tasks/schedule` with plugin **`path`** + **`method`** — built-in: `@engine9/plugins/e9workers:EchoWorker` + `echo`; see also `POST /flow_runs/`

## Related documentation

| Documentation | Use when |
|---------------|----------|
| [SKILL.md](./SKILL.md) | Give a Cursor agent the complete Task API operating rules |
| [concepts.md](./concepts.md) | Learn flows, runs, IDs, states, and asynchronous execution |
| [authentication.md](./authentication.md) | Configure API keys and `tasks:read` / `tasks:schedule` scopes |
| [getting-started.md](./getting-started.md) | Set environment variables and make the first calls |
| [echo-walkthrough.md](./echo-walkthrough.md) | Schedule and poll the built-in Echo worker |
| [endpoints.md](./endpoints.md) | Review routes and request examples |
| [errors.md](./errors.md) | Interpret HTTP status codes |
| [e9-dev-tasks](../e9-dev-tasks/SKILL.md) | Author JSON5 flow files |
