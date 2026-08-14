# Task API — end-user documentation

Guides for developers and integrators who **call the Engine9 Task API** over HTTP. The default production base URL is **`https://data.engine9.ai`**; your administrator may provide a different host for staging or local use.

**Cursor agents:** start with [SKILL.md](./SKILL.md).

## Start here

1. [concepts.md](./concepts.md) — flows, runs, IDs, states, async execution
2. [authentication.md](./authentication.md) — **API keys (`e9key_…`) and scopes** (`tasks:read` / `tasks:schedule`)
3. [getting-started.md](./getting-started.md) — environment variables and first calls
4. [echo-walkthrough.md](./echo-walkthrough.md) — schedule → listTasks Echo walkthrough
5. [endpoints.md](./endpoints.md) — per-route reference with multiple examples
6. [errors.md](./errors.md) — HTTP status codes (Prefect links for error semantics only)

Developers authoring JSON5 flow files: see [e9-dev-tasks](../e9-dev-tasks/SKILL.md) (not for normal API users).

## What you need from your administrator

| Item | Example |
|------|---------|
| Base URL | `https://data.engine9.ai` |
| API key | `e9key_…` with `tasks:read` and/or `tasks:schedule` — see [authentication.md](./authentication.md) |
| Account id | `acme` — sent as `X-ENGINE9-ACCOUNT-ID` |
| Available flows | Slugs from `GET /flows`, e.g. `echo-flow`, `nightly-sync` |
| Output retrieval | How to fetch completed task results for your environment |

Operators and deployment setup are documented separately (ask your administrator).

## API shape

Routes live at the **API origin root** — not under `/api/task`:

```
GET  /flows
POST /tasks/schedule
POST /tasks/listTasks
POST /flow_runs/
GET  /flow_runs/:id
GET  /task_runs/:id
...
```

Primary integration path matches MCP `task`: **schedule** then **listTasks**. `POST /flow_runs/` schedules a published flow slug through the same `scheduleTasks` entry point. Prefect-shaped filter/set_state helpers remain for orchestration tooling.
