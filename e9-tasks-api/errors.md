# Errors and HTTP status codes

How to interpret failed Task API responses. Response bodies use a `detail` field (string or array of objects with `msg` and optional `loc`).

## Status code summary

| Status | Meaning | Typical fix |
|--------|---------|-------------|
| **401** | Unauthorized | Send valid `e9key_` API key + account header — [authentication.md](./authentication.md) |
| **403** | Forbidden | Unknown/disabled account, or missing `tasks:read` / `tasks:schedule` scope |
| **404** | Not found | Wrong flow slug, flow run id, or task run id |
| **410** | Gone | Removed route — use the successor in `detail` (listing is `POST /task_runs/filter`) |
| **422** | Validation error | Fix request body (missing required field) |
| **500** | Server error | Unexpected failure — retry or contact administrator |
| **503** | Service unavailable | API not fully configured for your environment — contact administrator |

Prefect documents similar HTTP semantics for its REST API: [Prefect Server REST overview](https://docs.prefect.io/v3/api-ref/rest-api/server/).

## Response body shapes

### Validation error (422)

```json
{
  "detail": [
    {
      "msg": "flow_id required",
      "loc": ["body", "flow_id"]
    }
  ]
}
```

Prefect-style validation: [Create Flow Run](https://docs.prefect.io/v3/api-ref/rest-api/server/flow-runs/create-flow-run/) (422 on invalid body).

### Simple error (404)

```json
{
  "detail": "Flow not found"
}
```

### Configuration error (503)

```json
{
  "detail": [{ "msg": "Flow directory not configured" }]
}
```

Contact your administrator — the API is not fully provisioned for your account or environment.

### Server error (500)

```json
{
  "detail": [{ "msg": "..." }]
}
```

## Errors by endpoint

### `POST /flow_runs/` (predefined flow)

| Status | `detail` | Cause |
|--------|----------|-------|
| 422 | `flow_id required` | Body missing `flow_id` |
| 404 | Flow not found | Slug does not exist for your account |
| 503 | Flow directory not configured | Flows not provisioned — contact administrator |

To schedule an on-demand task instead, use [`POST /tasks/schedule`](./endpoints.md#post-tasksschedule) with `path` + `method`.

**Example — missing flow_id on a predefined flow:**

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{"name":"oops"}' \
  "$BASE_URL/flow_runs/"
# HTTP 422
```

**Example — wrong slug:**

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{"flow_id":"does-not-exist"}' \
  "$BASE_URL/flow_runs/"
# HTTP 404
```

### `POST /tasks/schedule` (on-demand task)

| Status | Cause |
|--------|-------|
| 422 | Body missing `path` and `method` |
| 422 | Body included `flow_id` — use `POST /flow_runs/` for a predefined flow |
| 500 / plugin error | `path` not installed on the account or method unknown |

To schedule a predefined flow instead, use [`POST /flow_runs/`](./endpoints.md#post-flow_runs) with `flow_id`. Built-in on-demand path is `@engine9/plugins/e9workers:<Worker>` (e.g. `@engine9/plugins/e9workers:EchoWorker` + `echo`). Do not send a flow slug as `path`.

### `GET /flows/:id`

| Status | Cause |
|--------|-------|
| 404 | No flow file with that slug |
| 503 | API not fully configured |

### `GET /flow_runs/:id` / `GET /task_runs/:id`

| Status | Cause |
|--------|-------|
| 404 | UUID not found (wrong id or different account scope) |

### `POST /flow_runs/:id/set_state` / `POST /task_runs/:id/set_state`

| Status | `detail` | Cause |
|--------|----------|-------|
| 404 | `Flow run not found` / `Task run not found` | Invalid id |
| 422 | `state.type required` | Body missing `state.type` |

Prefect task run state updates: [Set Task Run State](https://docs.prefect.io/v3/api-ref/rest-api/server/task-runs/set-task-run-state/).

### `POST /flow_runs/archive` / `POST /flow_runs/retry`

| Status | Cause |
|--------|-------|
| 422 | Body missing `flow_run_ids` (or `flow_runs.id`) |
| 503 | Archive/retry backend not reachable/configured on the server |

These mutate existing flow runs. The `result` field of the 200 body reflects which runs the remote server actually archived or retried.

### `POST /task_runs/filter`

| Status | Cause |
|--------|-------|
| 200 with `task_runs: []` | No matches (unknown ids do **not** 404) |
| 503 | API not fully configured |

### `GET /flows` / `POST /flows/filter`

| Status | Cause |
|--------|-------|
| 503 | API not fully configured |
| 401 | Missing auth |

## Authentication errors (401)

The body is a `detail` message. Common causes:

- `X-ENGINE9-ACCOUNT-ID` header omitted
- API key omitted (`Authorization: Bearer e9key_…` or `X-API-Key`)
- Key unknown, inactive, or issued for a different account (`Invalid API key (…)`)
- A non-`e9key_` bearer token (e.g. a Firebase/OAuth token meant for MCP) was sent

```bash
# This will 401 (no API key)
curl $CURL_TLS -sS -H "$ACCOUNT" "$BASE_URL/flows"
```

## Logical errors (HTTP 200 but wrong outcome)

These are not HTTP errors — handle in application logic.

| Observation | Meaning |
|-------------|---------|
| `state_type: PENDING` indefinitely | Background execution not running — contact administrator |
| `state_type: FAILED` | Task failed — inspect `state` and error artifacts |
| Empty `GET /flows` array | No flows published for your account |
| Unexpected task output | Wrong flow or task_key — verify `flow_id` and poll correct `task_run_id` |

## Retry guidance

| Status | Retry? |
|--------|--------|
| 401 | No — refresh token first |
| 404 | No — fix id/slug |
| 410 | No — switch to `POST /task_runs/filter` |
| 422 | No — fix request body |
| 500 | Yes — with backoff |
| 503 | No — administrator must fix configuration |

## Getting help

Include in support requests:

1. HTTP method and path
2. Status code and full `detail` body
3. `X-ENGINE9-ACCOUNT-ID` value used (not the token)
4. Flow slug or run UUIDs
5. Timestamp (UTC)
