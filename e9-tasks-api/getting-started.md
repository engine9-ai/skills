# Getting started

Minimal steps to make your first Task API calls once your administrator has given you a base URL, API key, and account id.

## 1. Collect connection details

From your administrator:

```
Base URL:     https://api.example.com
Account id:   acme
API key:      e9k_…  (scopes: tasks:read,tasks:schedule)
TLS notes:    self-signed? use curl -k
```

## 2. Set shell variables

```bash
export BASE_URL="https://api.example.com"
export AUTH="Authorization: Bearer e9k_<your-key>"
export ACCOUNT="X-ENGINE9-ACCOUNT-ID: acme"
export CURL_TLS=""    # set to "-k" for self-signed dev certs
```

## 3. Confirm the API is reachable

List flows:

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" "$BASE_URL/flows" | jq .
```

**Success:** HTTP 200, JSON array (may be `[]` if no flows published yet).

**401/403?** Check key, account header, and scopes — [authentication.md](./authentication.md).

## 4. Schedule work (MCP-parity)

Prefer `POST /tasks/schedule` with a plugin `path` + `method`, or a `flow_path`:

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "flow_path": "/path/on/server/to/echo-flow.json5",
    "label": "my-first-run"
  }' \
  "$BASE_URL/tasks/schedule" | tee /tmp/schedule.json | jq .
```

Or schedule a **published** flow by slug:

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{"flow_id": "echo-flow", "name": "my-first-run"}' \
  "$BASE_URL/flow_runs/" | tee /tmp/schedule.json | jq .
```

Save identifiers:

```bash
export FLOW_RUN_ID=$(jq -r '.flow_run_id // .result.flow_run_id // .id' /tmp/schedule.json)
echo "flow_run=$FLOW_RUN_ID"
```

## 5. Poll status

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d "{\"flow_run_id\": \"$FLOW_RUN_ID\"}" \
  "$BASE_URL/tasks/check" | jq .
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
