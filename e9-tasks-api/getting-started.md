# Getting started

Minimal steps to make your first Task API calls. The default production base URL is **`https://data.engine9.ai`**.

## 1. Collect connection details

From your administrator:

```
Base URL:     https://data.engine9.ai
Account id:   acme
API key:      e9key_…  (scopes: tasks:read,tasks:schedule)
TLS notes:    production uses a public cert; local dev may need curl -k
```

## 2. Set shell variables

```bash
export BASE_URL="https://data.engine9.ai"
export AUTH="Authorization: Bearer e9key_<your-key>"
export ACCOUNT="X-ENGINE9-ACCOUNT-ID: acme"
export CURL_TLS=""    # set to "-k" for self-signed dev certs
```

## 3. Confirm the API is reachable

List recent **remote** runs for the account (`tasks:read`; `remote` defaults to `true`; no `flow_run_id`):

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{"limit":20}' \
  "$BASE_URL/flow_runs/filter"
```

Pass `"remote": false` to list local runs instead.

**401/403?** Check key, account header, and scopes — [authentication.md](./authentication.md).

Optional: `GET /flows` lists published flow slugs (may be `[]` if none are published). On-demand Echo does not need a flow.

## 4. Schedule work

Two mutually exclusive modes. Both enqueue via `scheduleTasks` and return `flow_run_id` / `task_run_ids`.

### On-demand task — plugin path and method (recommended first call)

No `flow_id`. Identify the worker with `path` + `method`. Built-in Engine9 Workers use **`@engine9/plugins/e9workers:<Worker>`** — no plugin-id lookup. Echo is the smoke test:

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "path": "@engine9/plugins/e9workers:EchoWorker",
    "method": "echo",
    "options": { "message": "hello from echo", "seconds": 1 },
    "label": "my-first-task"
  }' \
  "$BASE_URL/tasks/schedule" | tee /tmp/schedule.json | jq .
```

Other built-in examples: `@engine9/plugins/e9workers:SQLWorker` + `query`. Account plugins (RENxt, …) still need a path from MCP `account` — see [echo-walkthrough.md](./echo-walkthrough.md#on-demand-task-names).

For a published multi-step workflow, use [`POST /flow_runs/`](#predefined-flow--flow_id-required) instead.

### Predefined flow — `flow_id` required

Use a slug from `GET /flows`. Worker `path` and `method` come from the flow file; do not send them.

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{"flow_id": "nightly-sync", "name": "my-first-run"}' \
  "$BASE_URL/flow_runs/" | tee /tmp/schedule.json | jq .
```

`POST /tasks/schedule` takes `path` + `method` only — it is not equivalent. Use `POST /flow_runs/` for published flows.

Save identifiers:

```bash
export FLOW_RUN_ID=$(jq -r '.result.flow_run_id // .flow_run_id // .id' /tmp/schedule.json)
echo "flow_run=$FLOW_RUN_ID"
```

## 5. Poll status

`POST /task_runs/filter` lists the run's tasks (includes `flow_run` when a single `flow_run_id` is sent):

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d "{\"flow_run_id\": \"$FLOW_RUN_ID\"}" \
  "$BASE_URL/task_runs/filter" | jq .
```

Repeat until the run reaches a terminal state. See [concepts — scheduling vs execution](./concepts.md#scheduling-vs-execution).

## 6. Read results

How you retrieve completed output depends on your deployment — ask your administrator (often a path or follow-up API referenced in the check response).

## Full walkthrough

See [echo-walkthrough.md](./echo-walkthrough.md).

## Reference

- [concepts.md](./concepts.md) — terminology
- [authentication.md](./authentication.md) — keys and scopes
- [endpoints.md](./endpoints.md) — all routes
- [errors.md](./errors.md) — when something fails
