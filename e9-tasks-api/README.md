# Task API — end-user documentation

Guides for developers and integrators who **call the Engine9 Task API** over HTTP. Your administrator provides a base URL, credentials, and account id.

**Cursor agents:** start with [SKILL.md](./SKILL.md).

## Start here

1. [concepts.md](./concepts.md) — flows, runs, IDs, states, async execution
2. [authentication.md](./authentication.md) — required headers and Google OAuth tokens
3. [getting-started.md](./getting-started.md) — environment variables and first calls
4. [echo-walkthrough.md](./echo-walkthrough.md) — create → list → poll → read Echo output
5. [endpoints.md](./endpoints.md) — per-route reference with multiple examples
6. [errors.md](./errors.md) — HTTP status codes (Prefect links for error semantics only)

Developers authoring JSON5 flow files: see [e9-dev-tasks](../e9-dev-tasks/SKILL.md) (not for normal API users).

## What you need from your administrator

| Item | Example |
|------|---------|
| Base URL | `https://api.example.com` |
| Bearer token | Google OAuth access token (or approved dev token for local use) — see [authentication.md](./authentication.md) |
| Account id | `acme` — sent as `X-ENGINE9-ACCOUNT-ID` |
| Available flows | Slugs from `GET /flows`, e.g. `echo-flow`, `nightly-sync` |
| Output retrieval | How to fetch completed task results for your environment |

Operators and deployment setup are documented separately (ask your administrator).

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

Request and response bodies follow a **Prefect-compatible** JSON shape. Engine9-specific behavior (async execution, result references) is documented here; Prefect is referenced only in [errors.md](./errors.md) for HTTP status semantics.
