# Endpoints reference

All routes are on the **API origin root**. Every request requires an **`e9key_` API key** and `X-ENGINE9-ACCOUNT-ID`. See [authentication.md](./authentication.md) for keys and scopes.

Unless noted, responses are JSON. Successful reads return **200**; schedule returns **200** with the result object. `POST /task_runs/filter` returns `{ ok: true, task_runs: [ … ] }`.

---

## Schedule

Two endpoints, both call `TaskWorker.scheduleTasks`. Listing belongs on `POST /task_runs/filter`.

| Endpoint | Schedules | Required |
|----------|-----------|----------|
| [`POST /flow_runs/`](#post-flow_runs) | **Predefined flow** | **`flow_id`** (published slug) |
| [`POST /tasks/schedule`](#post-tasksschedule) | **On-demand task** | Plugin **`path`** + **`method`** |

### `POST /tasks/schedule`

**Scope:** `tasks:schedule`

Schedule an **on-demand task** — one plugin worker method. Requires `path` and `method`. Do not send `flow_id`.

Built-in engine9 Workers (every account; no plugin lookup): `@engine9/plugins/e9workers:<Worker>`. Echo smoke test: `@engine9/plugins/e9workers:EchoWorker` + `echo`.

To run a published workflow instead, use [`POST /flow_runs/`](#post-flow_runs) with `flow_id`.

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "path": "@engine9/plugins/e9workers:EchoWorker",
    "method": "echo",
    "options": { "message": "hello from echo", "seconds": 1 },
    "label": "echo smoke"
  }' \
  "$BASE_URL/tasks/schedule"
```

Account plugin example (`path` from MCP `account` / installed plugins):

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "path": "@frakture-com/channelbots/RENxtBot:People",
    "method": "listCustomFields",
    "label": "renxt/people listCustomFields"
  }' \
  "$BASE_URL/tasks/schedule"
```

| Field | Required | Description |
|-------|----------|-------------|
| `path` + `method` | **Yes** | On-demand plugin path + worker method. Built-in: `@engine9/plugins/e9workers:EchoWorker` + `echo`. |
| `options` | No | Options object for the worker method |
| `label`, `tracking_code`, `start_after_timestamp` | No | Schedule metadata |
| `remote` | No | Default `true` (remote execution); `false` for local workers |

**Response:** `{ ok: true, action: "schedule", result: { flow_run_id, task_run_ids, … } }`

**422** — missing `path`+`method`, or `flow_id` sent here (use `POST /flow_runs/` for a predefined flow).

---

## Flows

### `GET /flows`

List all flow definitions for your account.

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" "$BASE_URL/flows"
```

```bash
# Compact summary with jq
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" "$BASE_URL/flows" \
  | jq '[.[] | {id, name, tags, tasks: (.tasks | length)}]'
```

**Response:** JSON array of flow objects.

---

### `GET /flows/:id`

Read one flow by **slug** (`id` from the flow file).

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" "$BASE_URL/flows/nightly-sync"
```

**404** — no flow with that slug for your account.

---

### `POST /flows/filter`

Search and paginate flows.

**Filter by name:**

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "flows": { "name": { "eq_": "Nightly sync" } },
    "limit": 20,
    "offset": 0,
    "sort": "CREATED_DESC"
  }' \
  "$BASE_URL/flows/filter"
```

**Filter by tags:**

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "flows": { "tags": { "any_": ["etl"] } },
    "limit": 10
  }' \
  "$BASE_URL/flows/filter"
```

**Pagination:**

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{"limit": 5, "offset": 10, "sort": "CREATED_ASC"}' \
  "$BASE_URL/flows/filter"
```

Supported `sort`: `CREATED_DESC` (default), `CREATED_ASC`.

---

## Flow runs

### `POST /flow_runs/`

**Scope:** `tasks:schedule`

Schedule a **predefined** (published) flow. **`flow_id` is required** — the flow slug from `GET /flows`, not a plugin path. Resolves the account flows directory JSON5 file and calls `scheduleTasks`.

To invoke a single plugin method instead, use [`POST /tasks/schedule`](#post-tasksschedule) with `path` + `method` (no `flow_id`). Echo smoke test is on-demand (`@engine9/plugins/e9workers:EchoWorker` + `echo`), not a flow.

**Minimal body:**

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{"flow_id": "nightly-sync"}' \
  "$BASE_URL/flow_runs/"
```

**With label and remote flag:**

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "flow_id": "nightly-sync",
    "name": "nightly-sync-check",
    "remote": true
  }' \
  "$BASE_URL/flow_runs/"
```

| Field | Required | Description |
|-------|----------|-------------|
| `flow_id` | **Yes** | Flow slug (e.g. `nightly-sync`), not the UUID |
| `name` / `label` | No | Display label for the run |
| `tracking_code`, `start_after_timestamp` | No | Schedule metadata |
| `remote` | No | Default `true` |

**Response:** `scheduleTasks` result (`flow_run_id`, `task_run_ids`, …).

**422** — missing `flow_id`. **404** — unknown flow slug.

See also: [`POST /tasks/schedule`](#post-tasksschedule) for a specific plugin method.

---

### `GET /flow_runs/:id`

**Scope:** `tasks:read` — lists the run (default `remote=true`; pass `?remote=false` for local).

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/flow_runs/$FLOW_RUN_ID"
```

Prefer `POST /task_runs/filter` with `{ "flow_run_id": "…" }` to list that run's tasks.

---

### `POST /flow_runs/filter`

**Scope:** `tasks:read`

**Default `remote: true`** — lists remote flow runs via `TaskWorker.listRemoteFlowRuns` (same as MCP `task` `action: "list"`). Pass `"remote": false` (or `?remote=false`) for local SQL/file runs.

**List recent remote runs for this account** (after creating a key; no `flow_run_id` required):

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{"limit":20}' \
  "$BASE_URL/flow_runs/filter"
```

One-liner:

```bash
curl -sS -H "Authorization: Bearer $ENGINE9_API_KEY" -H "X-ENGINE9-ACCOUNT-ID: <account_id>" -H "Content-Type: application/json" -d '{"limit":20}' https://data.engine9.ai/flow_runs/filter
```

Local listing:

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{"limit":20,"remote":false}' \
  "$BASE_URL/flow_runs/filter"
```

`POST /task_runs/filter` lists task runs (optionally scoped to a `flow_run_id`). `POST /flow_runs/filter` lists flow runs for the account.

**Recent runs for a flow:**

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "flow_runs": { "flow_id": { "eq_": "nightly-sync" } },
    "sort": "CREATED_DESC",
    "limit": 20,
    "offset": 0
  }' \
  "$BASE_URL/flow_runs/filter"
```

**Filter by state:**

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "flow_runs": {
      "state": { "type": { "any_": ["COMPLETED", "FAILED"] } }
    },
    "limit": 50
  }' \
  "$BASE_URL/flow_runs/filter"
```

`flow_id.eq_` matches slug **or** flow UUID.

**Filter by parent account:**

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "parent_account_id": "frakture_master",
    "status": ["FAILED"],
    "limit": 50
  }' \
  "$BASE_URL/flow_runs/filter"
```

Use `"parent_account_id": "none"` for accounts with no parent. Prefect-shaped alias: `"flow_runs": { "parent_account_id": { "eq_": "frakture_master" } }`.

**Filter by `completed_since`:**

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "completed_since": true,
    "limit": 50
  }' \
  "$BASE_URL/flow_runs/filter"
```

Prefect-shaped alias: `"flow_runs": { "completed_since": { "eq_": true } }`.

Each flow run in the response includes `account_id`, **`parent_account_id`**, **`parent_ids`**, `completed_since`, `last_completed`, and `dataflow_last_completed`.

- `parent_account_id` is the first id in the owning account's `parent_ids` (`null` if the account has no parent).
- `parent_ids` is the full array from the account document (an account can have more than one parent).
- The `parent_account_id` **filter** matches accounts that contain that id anywhere in `parent_ids` (or `'none'` for accounts with no parent). It is the same field name as the response, but the filter is "has this parent" rather than "this is the first parent".

Computation: `completed_since` is `true` when `dataflow_last_completed` exists and `last_completed` is missing or **≤** `dataflow_last_completed` (equal counts as true). See [concepts.md](./concepts.md#completed-since).

---

### `POST /flow_runs/archive`

**Scope:** `tasks:schedule`

**Auth (same as listing):** `user_id` is **not** required. Callers that can `POST /flow_runs/filter` can archive with the same credentials.

Bulk-archive flow runs.

Ids are sent as `flow_run_ids` (or the Prefect-style `flow_runs.id.any_` filter shape).

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "flow_run_ids": ["6a82fed56813e4e2a0a0144e"]
  }' \
  "$BASE_URL/flow_runs/archive"
```

**Response** (`flow_run_ids` echoes the requested ids; `result` is the remote outcome):

```json
{
  "ok": true,
  "action": "archive",
  "flow_run_ids": ["6a82fed56813e4e2a0a0144e"],
  "result": { "ok": true }
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `flow_run_ids` | One of | Array of flow run ids (preferred) |
| `flow_runs.id.any_` / `eq_` | One of | Prefect-style filter shape |

**422** — none of those id fields provided. **503** — archive is not reachable/configured on the server.

---

### `POST /flow_runs/retry`

**Scope:** `tasks:schedule`

**Auth (same as listing):** `user_id` is **not** required.

Bulk-retry flow runs.

For each id, the server retries the failed task run, or the most recent completed one if nothing failed. The `result` field of the response reflects which runs the remote server actually retried.

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "flow_run_ids": ["6a82fed56813e4e2a0a0144e"]
  }' \
  "$BASE_URL/flow_runs/retry"
```

**Response:**

```json
{
  "ok": true,
  "action": "retry",
  "flow_run_ids": ["6a82fed56813e4e2a0a0144e"],
  "result": { "ok": true }
}
```

Body fields match `POST /flow_runs/archive`.

**422** — none of those id fields provided. **503** — retry is not reachable/configured on the server.

---

### `POST /flow_runs/:id/set_state`

Update flow run state (advanced — orchestration integrations).

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "state": {
      "type": "RUNNING",
      "id": "0196f1d2-0000-7000-8000-000000000001",
      "name": "Running"
    }
  }' \
  "$BASE_URL/flow_runs/$FLOW_RUN_ID/set_state"
```

**422** — missing `state.type`.

---

## Task runs

Task runs are created by scheduling — an on-demand task (`POST /tasks/schedule` with `path` + `method`) or a predefined flow (`POST /flow_runs/` with `flow_id`). There is no standalone task-run create endpoint.

### `GET /task_runs/:id`

**Scope:** `tasks:read` — default `remote=true`. Remote responses include `resolved_options`, `output`, display fields (`bot`, `submodule`, `method`, `bot_location_id`, `errors`, `records`, `expected_start_time`, `updated`), and `log_url` when the job server can sign one.

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/task_runs/$TASK_RUN_ID"
```

**Key fields:**

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/task_runs/$TASK_RUN_ID" | jq '{
    id: .task_run.id,
    state_type: .task_run.state_type,
    task_key: .task_run.task_key,
    flow_run_id: .task_run.flow_run_id,
    options: .task_run.options,
    resolved_options: .task_run.resolved_options,
    output: .task_run.output,
    records: .task_run.records
  }'
```

**404** — unknown task run id.

---

### `GET /task_runs/:id/log`

**Scope:** `tasks:read`

Returns the job-server log text for a remote task run.

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/task_runs/$TASK_RUN_ID/log"
```

**200:**

```json
{
  "ok": true,
  "task_run_id": "…",
  "log": "<text>",
  "truncated": false
}
```

When the log cannot be fetched inline, `log` may be empty and `log_url` (signed, short-lived) is still returned. **404** — no log available.

---

### `GET /task_runs/:id/output`

**Scope:** `tasks:read`

Returns `{ "ok": true, "task_run_id": "…", "output": { … } }` for remote runs (same `output` as `GET /task_runs/:id`).

---

### `POST /task_runs/filter`

Prefect **Read Task Runs**. Primary way to list or poll task runs. Returns `{ ok: true, task_runs: [ … ] }`. When a **single** `flow_run_id` is supplied, also includes `flow_run` (extension so one call can poll the parent run).

**Default `remote: true`** — lists remote task runs via `TaskWorker.listRemoteTaskRuns`. Pass `"remote": false` (or `?remote=false`) for local SQL task runs.

Each `task_run` / `flow_run` includes `account_id`, `parent_account_id`, and `parent_ids`.

**All task runs for a flow run:**

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d "{\"flow_run_id\": \"$FLOW_RUN_ID\"}" \
  "$BASE_URL/task_runs/filter"
```

Prefect-shaped equivalent:

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d "{
    \"task_runs\": { \"flow_run_id\": { \"eq_\": \"$FLOW_RUN_ID\" } },
    \"sort\": \"ID_DESC\"
  }" \
  "$BASE_URL/task_runs/filter"
```

**Filter by ids or state:**

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "task_runs": {
      "id": { "any_": ["task-run-id-1"] },
      "state": { "type": { "any_": ["FAILED"] } }
    },
    "limit": 50
  }' \
  "$BASE_URL/task_runs/filter"
```

**Paginate** (no flow-run scope — recent task runs for the account):

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{"limit": 10, "offset": 0}' \
  "$BASE_URL/task_runs/filter"
```

| Field | Description |
|-------|-------------|
| `flow_run_id` / `task_runs.flow_run_id.eq_` / `flow_runs.id.eq_` | Scope to one flow run (also sets `flow_run` on the response) |
| `task_run_ids` / `task_run_id` / `task_runs.id.any_` | Restrict to those task run ids |
| `status` / `task_runs.state.type.any_` | Prefect state types (`FAILED`, `RUNNING`, `COMPLETED`, …) |
| `account_ids` / `parent_account_id` | Same multi-account filters as flow-run listing |
| `limit` | Default **500** when scoped to a flow/task run id, otherwise **20** (max 500) |
| `offset` | Default 0 |
| `sort` | `ID_DESC` (default) or `ID_ASC` |
| `remote` | Default `true` (Frakture). Pass `false` for local SQL task runs |

Does **not** 404 when ids are missing — unmatched filters return `task_runs: []` (`flow_run: null` if a single unknown `flow_run_id` was sent).

Each remote `task_run` includes display fields used by the flow-run UI: `bot.path`, `submodule`, `method`, `bot_location_id`, `errors` (`{ level, message, ts }[]`), `records`, `expected_start_time`, `updated`, plus `options` / `output`.

---

### `POST /task_runs/:id/retry`

**Scope:** `tasks:schedule`

Retry a **specific** remote task run (unlike `POST /flow_runs/retry`, which retries the failed task or the most recent completed one).

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{ "force": false }' \
  "$BASE_URL/task_runs/$TASK_RUN_ID/retry"
```

Queue behind another task in the same flow:

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{ "start_after": "previous" }' \
  "$BASE_URL/task_runs/$TASK_RUN_ID/retry"
```

Or pass an explicit sibling: `{ "start_after_task_run_id": "<previous task_run_id>" }`.

**200:** `{ "ok": true, "action": "retry", "task_run_id": "…", "flow_run_id": "…" }`

| Field | Description |
|-------|-------------|
| `force` | Re-run even if `COMPLETED`. Required to retry a `RUNNING` run |
| `start_after` | `"previous"` — wait for the preceding task in the flow |
| `start_after_task_run_id` | Wait for that sibling task run |

**422** — task run is `RUNNING` and `force` is not set.

---

### `POST /task_runs/:id/pause`

**Scope:** `tasks:schedule`

Pause a pending/scheduled remote task run (`Job.pause`).

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/task_runs/$TASK_RUN_ID/pause"
```

---

### `POST /task_runs/:id/resume`

**Scope:** `tasks:schedule`

Resume a paused remote task run (same as retry without `force`).

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/task_runs/$TASK_RUN_ID/resume"
```

---

### `POST /task_runs/:id/stop`

**Scope:** `tasks:schedule`

Kill a remote task run.

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{ "reason": "user_request" }' \
  "$BASE_URL/task_runs/$TASK_RUN_ID/stop"
```

---

### `POST /task_runs/:id/set_state`

On **remote** runs, `state.type` maps to the first-class actions above:

| `state.type` | Action |
|--------------|--------|
| `PAUSED` | pause |
| `SCHEDULED` / `PENDING` | resume |
| `CANCELLED` | stop |

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{ "state": { "type": "PAUSED" } }' \
  "$BASE_URL/task_runs/$TASK_RUN_ID/set_state"
```

**422** — missing `state.type`, or an unsupported type for remote runs. Pass `remote: false` for local file/SQL state updates.

---

### `PATCH /task_runs/:id`

**Scope:** `tasks:schedule`

Merge `options` onto a **pending or paused** remote task run before it executes.

```bash
curl $CURL_TLS -sS -X PATCH \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "options": {
      "model": "@engine9/plugins/models/first_touch",
      "limit": 100
    }
  }' \
  "$BASE_URL/task_runs/$TASK_RUN_ID"
```

**200:** `{ "ok": true, "task_run_id": "…", "options": { …merged… } }`

**409** — task run is already `RUNNING` or terminal. **422** — body missing `options` object.

---

## Diagnostics

### `GET /flows_dir`

Returns account context for the current request (useful when `GET /flows` is unexpectedly empty). Exact response shape may vary by deployment; confirm with your administrator.

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" "$BASE_URL/flows_dir"
```

---


## Typical integration sequence

```bash
# 1. On-demand Echo (path + method — no flow_id)
curl -X POST ... -d '{"path":"@engine9/plugins/e9workers:EchoWorker","method":"echo","options":{"message":"hello from echo","seconds":1}}' \
  "$BASE_URL/tasks/schedule"

# 2. Optional: predefined flow (flow_id required)
curl -X POST ... -d '{"flow_id":"nightly-sync"}' "$BASE_URL/flow_runs/"

# 3. Poll (lists tasks for the run; includes flow_run)
curl -X POST ... -d '{"flow_run_id":"'"$FLOW_RUN_ID"'"}' \
  "$BASE_URL/task_runs/filter"

# 4. Optional: list history
curl -X POST ... -d '{"flow_runs":{"flow_id":{"eq_":"nightly-sync"}},"limit":10}' \
  "$BASE_URL/flow_runs/filter"

# 5. Optional: retry a specific task, or the flow's failed task
curl -X POST ... -d '{"force":false}' "$BASE_URL/task_runs/$TASK_RUN_ID/retry"
curl -X POST ... -d '{"flow_run_ids":["'"$FLOW_RUN_ID"'"]}' \
  "$BASE_URL/flow_runs/retry"

# 6. Optional: pause / resume / edit options of a pending task
curl -X POST ... "$BASE_URL/task_runs/$TASK_RUN_ID/pause"
curl -X PATCH ... -d '{"options":{"limit":100}}' "$BASE_URL/task_runs/$TASK_RUN_ID"

# 7. Optional: log and output
curl ... "$BASE_URL/task_runs/$TASK_RUN_ID/log"
curl ... "$BASE_URL/task_runs/$TASK_RUN_ID/output"

# 8. Optional: archive finished runs
curl -X POST ... -d '{"flow_run_ids":["'"$FLOW_RUN_ID"'"]}' \
  "$BASE_URL/flow_runs/archive"
```

Full narrative: [echo-walkthrough.md](./echo-walkthrough.md).

Errors: [errors.md](./errors.md).
