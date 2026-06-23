# Getting started

Minimal steps to make your first Task API calls once your administrator has given you a base URL and credentials.

## 1. Collect connection details

From your administrator:

```
Base URL:     https://api.example.com
Account id:   acme
Token:        <Firebase ID token via Engine9 OAuth, or dev token for local use — see authentication.md>
TLS notes:    self-signed? use curl -k
```

## 2. Set shell variables

```bash
export BASE_URL="https://api.example.com"
export AUTH="Authorization: Bearer <your-token>"
export ACCOUNT="X-ENGINE9-ACCOUNT-ID: acme"
export CURL_TLS=""    # set to "-k" for self-signed dev certs
```

## 3. Confirm the API is reachable

List flows:

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" "$BASE_URL/flows" | jq .
```

**Success:** HTTP 200, JSON array (may be `[]` if no flows published yet).

**Empty array?** Ask your administrator to provision flows for your account.

## 4. Inspect one flow

Pick a slug from the list (e.g. `echo-flow`):

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/flows/echo-flow" | jq .
```

Note the `tasks[]` array — each task will become one task run when you create a flow run.

## 5. Create your first flow run

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "flow_id": "echo-flow",
    "name": "my-first-run"
  }' \
  "$BASE_URL/flow_runs/" | tee /tmp/my-flow-run.json | jq .
```

Save identifiers from the response:

```bash
export FLOW_RUN_ID=$(jq -r '.id' /tmp/my-flow-run.json)
export TASK_RUN_ID=$(jq -r '.task_runs[0].id' /tmp/my-flow-run.json)
echo "flow_run=$FLOW_RUN_ID"
echo "task_run=$TASK_RUN_ID"
```

## 6. Poll task state

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/task_runs/$TASK_RUN_ID" | jq '{state_type, task_key, flow_run_id}'
```

Repeat until `state_type` is `COMPLETED` or `FAILED`. If it stays `PENDING`, see [concepts — scheduling vs execution](./concepts.md#scheduling-vs-execution).

## 7. Read results

When `state_type` is `COMPLETED`:

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/task_runs/$TASK_RUN_ID" | jq '.state.state_details.output_path'
```

Retrieve the result using the method your administrator documents (the API returns a reference in `state.state_details.output_path`).

## Full walkthrough

For a step-by-step Echo example with filter queries and poll loops, see [echo-walkthrough.md](./echo-walkthrough.md).

## Reference

- [concepts.md](./concepts.md) — terminology
- [endpoints.md](./endpoints.md) — all routes with examples
- [errors.md](./errors.md) — when something fails
