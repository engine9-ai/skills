# Endpoints reference

All routes are on the **API origin root**. Every request requires an **`e9k_` API key** and `X-ENGINE9-ACCOUNT-ID`. See [authentication.md](./authentication.md) for keys and scopes.

Unless noted, responses are JSON. Successful reads return **200**; schedule/check return **200** with the result object.

---

## Schedule and check (MCP parity)

These routes mirror the MCP `task` tool and call `TaskWorker.scheduleTasks` / `checkTasks`.

### `POST /tasks/schedule`

**Scope:** `tasks:schedule`

Provide `flow_path` **or** both `path` and `method` (same as MCP).

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
| `flow_path` | One of | Absolute/server path to a JSON5 flow file |
| `path` + `method` | One of | Plugin path + worker method (resolved like MCP) |
| `options` | No | Options object for the worker method |
| `label`, `tracking_code`, `start_after_timestamp` | No | Schedule metadata |
| `remote` | No | Default `true` (remote job-list); `false` for local workers |

**Response:** `{ ok: true, action: "schedule", result: { flow_run_id, task_run_ids, … } }`

---

### `POST /tasks/check`

**Scope:** `tasks:read`

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{"flow_run_id": "<id from schedule>"}' \
  "$BASE_URL/tasks/check"
```

| Field | Required | Description |
|-------|----------|-------------|
| `flow_run_id` | One of | From schedule response |
| `task_run_ids` | One of | Optional with `flow_run_id` |
| `remote` | No | Default `true` |
| `flow_path` | No | Routing hint when `remote` is false |

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
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" "$BASE_URL/flows/echo-flow"
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
    "flows": { "name": { "eq_": "Echo Flow" } },
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
    "flows": { "tags": { "any_": ["echo", "etl"] } },
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

Schedule a **published** flow by slug. Resolves the account flows directory JSON5 file and calls `scheduleTasks` (same engine as MCP / `POST /tasks/schedule`).

**Minimal body:**

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{"flow_id": "echo-flow"}' \
  "$BASE_URL/flow_runs/"
```

**With label and remote flag:**

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "flow_id": "echo-flow",
    "name": "nightly-echo-check",
    "remote": true
  }' \
  "$BASE_URL/flow_runs/"
```

| Field | Required | Description |
|-------|----------|-------------|
| `flow_id` | **Yes** | Flow slug (e.g. `echo-flow`), not the UUID |
| `name` / `label` | No | Job list label |
| `tracking_code`, `start_after_timestamp` | No | Schedule metadata |
| `remote` | No | Default `true` |

**Response:** `scheduleTasks` result (`flow_run_id`, `task_run_ids`, …).

**422** — missing `flow_id`. **404** — unknown flow slug.

---

### `GET /flow_runs/:id`

**Scope:** `tasks:read` — polls via `checkTasks` (default `remote=true`; pass `?remote=false` for local).

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/flow_runs/$FLOW_RUN_ID"
```

Prefer `POST /tasks/check` for MCP-shaped clients.

---

### `POST /flow_runs/filter`

**Recent runs for a flow:**

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "flow_runs": { "flow_id": { "eq_": "echo-flow" } },
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

### `POST /task_runs/`

Create a standalone task run (advanced). Most clients use `POST /flow_runs/` instead.

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "flow_run_id": "'"$FLOW_RUN_ID"'",
    "task_key": "echo-step",
    "dynamic_key": "0"
  }' \
  "$BASE_URL/task_runs/"
```

---

### `GET /task_runs/:id`

Primary endpoint for checking execution progress and results.

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/task_runs/$TASK_RUN_ID"
```

**Key fields to extract:**

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/task_runs/$TASK_RUN_ID" | jq '{
    id,
    state_type,
    task_key,
    flow_run_id,
    start_time,
    end_time,
    output_path: .state.state_details.output_path,
    options: .task_inputs.options
  }'
```

**404** — unknown task run id.

---

### `POST /task_runs/filter`

**All task runs for a flow run:**

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d "{
    \"task_runs\": { \"flow_run_id\": { \"eq_\": \"$FLOW_RUN_ID\" } },
    \"limit\": 20,
    \"offset\": 0
  }" \
  "$BASE_URL/task_runs/filter"
```

**Paginate:**

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{"limit": 10, "offset": 0}' \
  "$BASE_URL/task_runs/filter"
```

Without `flow_run_id` filter, returns recent task runs for your account (up to `limit`).

---

### `POST /task_runs/:id/set_state`

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "state": {
      "type": "COMPLETED",
      "id": "0196f1d3-0000-7000-8000-000000000002",
      "name": "Completed"
    }
  }' \
  "$BASE_URL/task_runs/$TASK_RUN_ID/set_state"
```

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
# 1. Discover
curl ... "$BASE_URL/flows"

# 2. Create
curl -X POST ... -d '{"flow_id":"echo-flow"}' "$BASE_URL/flow_runs/"

# 3. Poll
curl ... "$BASE_URL/task_runs/$TASK_RUN_ID"

# 4. Optional: list history
curl -X POST ... -d '{"flow_runs":{"flow_id":{"eq_":"echo-flow"}},"limit":10}' \
  "$BASE_URL/flow_runs/filter"
```

Full narrative: [echo-walkthrough.md](./echo-walkthrough.md).

Errors: [errors.md](./errors.md).
