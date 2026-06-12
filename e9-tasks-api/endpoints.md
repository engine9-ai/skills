# Endpoints reference

All routes are on the **API origin root**. Every request requires `Authorization: Bearer` and typically `X-ENGINE9-ACCOUNT-ID`. See [authentication.md](./authentication.md).

Unless noted, responses are JSON. Successful reads return **200**; creates return **200** with the created object.

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

Create a flow run and one task run per task in the flow.

**Minimal body:**

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{"flow_id": "echo-flow"}' \
  "$BASE_URL/flow_runs/"
```

**With name and parameters:**

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "flow_id": "echo-flow",
    "name": "nightly-echo-check",
    "parameters": { "env": "prod" },
    "account_id": "test"
  }' \
  "$BASE_URL/flow_runs/"
```

| Field | Required | Description |
|-------|----------|-------------|
| `flow_id` | **Yes** | Flow slug (e.g. `echo-flow`), not the UUID |
| `account_id` | No | Defaults from `X-ENGINE9-ACCOUNT-ID` |
| `name` | No | Label for this run |
| `parameters` | No | Key-value parameters for the flow |
| `state` | No | Initial state override (advanced) |

**Response:** Flow run object with `id`, `flow_slug`, `state_type`, `task_runs[]`.

**Note:** Does not execute tasks — task runs start as `PENDING`. See [concepts.md](./concepts.md#scheduling-vs-execution).

**422** — missing `flow_id`. **404** — unknown flow slug.

---

### `GET /flow_runs/:id`

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/flow_runs/$FLOW_RUN_ID"
```

**404** — unknown flow run id.

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
