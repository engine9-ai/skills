# Errors and HTTP status codes

How to interpret failed Task API responses. Response bodies use a `detail` field (string or array of objects with `msg` and optional `loc`).

## Status code summary

| Status | Meaning | Typical fix |
|--------|---------|-------------|
| **401** | Unauthorized | Send valid `Authorization: Bearer` — [authentication.md](./authentication.md) |
| **403** | Forbidden | Authenticated but not permitted (uncommon on task routes) |
| **404** | Not found | Wrong flow slug, flow run id, or task run id |
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

### `POST /flow_runs/`

| Status | `detail` | Cause |
|--------|----------|-------|
| 422 | `flow_id required` | Body missing `flow_id` |
| 404 | Flow not found | Slug does not exist for your account |
| 503 | Flow directory not configured | Flows not provisioned — contact administrator |

**Example — missing flow_id:**

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

### `GET /flows` / `POST /flows/filter`

| Status | Cause |
|--------|-------|
| 503 | API not fully configured |
| 401 | Missing auth |

## Authentication errors (401)

No body shape is guaranteed. Common causes:

- Header omitted: `Authorization: Bearer ...`
- Expired or invalid Firebase ID token
- Dev token not valid for your account or environment

```bash
# This will 401
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
