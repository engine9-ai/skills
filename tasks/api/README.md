# Task API — end-user documentation

Guides for developers and integrators who **call the Engine9 Task API** over HTTP. You do not need server repo access — your administrator provides a base URL, credentials, and account id.

**Cursor agents:** start with [SKILL.md](./SKILL.md).

## Start here

1. [concepts.md](./concepts.md) — flows, runs, IDs, states, async execution
2. [authentication.md](./authentication.md) — required headers on every request
3. [getting-started.md](./getting-started.md) — environment variables and first calls
4. [echo-walkthrough.md](./echo-walkthrough.md) — create → list → poll → read Echo output
5. [endpoints.md](./endpoints.md) — per-route reference with multiple examples
6. [errors.md](./errors.md) — HTTP status codes (Prefect links for error semantics only)

## What you need from your administrator

| Item | Example |
|------|---------|
| Base URL | `https://api.example.com` or `https://127.0.0.1:8443` |
| Bearer token | OAuth access token, Firebase ID token, or dev token |
| Account id | `acme` — sent as `X-ENGINE9-ACCOUNT-ID` |
| Available flows | Slugs from `GET /flows`, e.g. `echo-flow`, `nightly-sync` |

**Server setup** (operators): in the `server` repository, see `api/task/docs/admin/`.

## API shape

Routes live at the **API origin root** — not under `/api/task`:

```
GET  /flows
GET  /flows/:id
POST /flows/filter
POST /flow_runs/
GET  /flow_runs/:id
POST /flow_runs/filter
POST /task_runs/filter
GET  /task_runs/:id
...
```

Request and response bodies follow a **Prefect-compatible** JSON shape. Engine9-specific behavior (async execution, artifact paths) is documented here; Prefect is referenced only in [errors.md](./errors.md) for HTTP status semantics.

## Implementation (server repo)

Router: `server/api/task/index.js`, mounted alongside `/mcp` on the Engine9 API.
