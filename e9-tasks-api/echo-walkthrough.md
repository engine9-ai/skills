# Echo walkthrough

Complete example: list the **Echo** sample flow, create a flow run, list runs, poll for completion, and read output. Assumes your administrator has provisioned `echo-flow` or `echo-flow-fast` for your account.

## Prerequisites

You need (from your administrator):

- Running Task API at a known base URL
- Valid `e9k_…` API key with `tasks:read` and `tasks:schedule`
- Account id (examples use `test`)
- `echo-flow` or `echo-flow-fast` available via `GET /flows`

Set variables (see [authentication.md](./authentication.md)):

```bash
export BASE_URL="https://127.0.0.1:8443"
export AUTH="Authorization: Bearer e9k_<your-key>"
export ACCOUNT="X-ENGINE9-ACCOUNT-ID: test"
export CURL_TLS="-k"
```

## What the Echo flow does

The Echo flow has one task that returns its input `options` unchanged (plus metadata like `last_run`). It is useful for verifying that authentication, scheduling, and result retrieval work end to end.

Inspect the live definition (you do not POST this — use `GET /flows`):

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/flows/echo-flow" | jq '{id, name, tasks}'
```

Use `"flow_id": "echo-flow"` when creating runs. For a faster step, use `echo-flow-fast` if your administrator has provisioned it.

---

## Step 1: List available flows

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/flows" | jq '[.[] | {id, name, task_count: (.tasks | length)}]'
```

Expected:

```json
[
  {
    "id": "echo-flow",
    "name": "Echo Flow",
    "task_count": 1
  }
]
```

If the array is empty or missing `echo-flow`, contact your administrator.

### Filter by tag

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "flows": { "tags": { "any_": ["echo"] } },
    "limit": 10,
    "sort": "CREATED_DESC"
  }' \
  "$BASE_URL/flows/filter" | jq .
```

### Read full flow definition

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/flows/echo-flow" | jq .
```

---

## Step 2: Schedule the flow

**Important:** `flow_id` is the flow **slug** (`echo-flow`), not the UUID `flow_id` field on the flow object. Scheduling goes through `scheduleTasks` (same as MCP).

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "flow_id": "echo-flow",
    "name": "echo-demo-2026-06-08"
  }' \
  "$BASE_URL/flow_runs/" | tee /tmp/echo-flow-run.json | jq .
```

Save IDs (shape matches MCP `task` schedule — `flow_run_id` / `task_run_ids`):

```bash
export FLOW_RUN_ID=$(jq -r '.flow_run_id // .id' /tmp/echo-flow-run.json)
export TASK_RUN_ID=$(jq -r '.task_run_ids[0] // .task_runs[0].id // empty' /tmp/echo-flow-run.json)
echo "FLOW_RUN_ID=$FLOW_RUN_ID"
echo "TASK_RUN_ID=$TASK_RUN_ID"
```

---

## Step 3: Check status

Prefer MCP-parity check:

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d "{\"flow_run_id\": \"$FLOW_RUN_ID\"}" \
  "$BASE_URL/tasks/check" | jq .
```

Or GET:

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/flow_runs/$FLOW_RUN_ID" | jq .
```

### Only completed runs

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "flow_runs": {
      "flow_id": { "eq_": "echo-flow" },
      "state": { "type": { "any_": ["COMPLETED"] } }
    },
    "limit": 10
  }' \
  "$BASE_URL/flow_runs/filter" | jq 'length'
```

---

## Step 4: List task runs

### All task runs for your flow run

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d "{
    \"task_runs\": { \"flow_run_id\": { \"eq_\": \"$FLOW_RUN_ID\" } },
    \"limit\": 10
  }" \
  "$BASE_URL/task_runs/filter" | jq '[.[] | {id, task_key, state_type}]'
```

### Read one task run

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/task_runs/$TASK_RUN_ID" | jq '{
    id,
    task_key,
    state_type,
    flow_run_id,
    options: .task_inputs.options
  }'
```

---

## Step 5: Poll until complete

### Single check

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/task_runs/$TASK_RUN_ID" | jq '{
    state_type,
    start_time,
    end_time,
    output_path: .state.state_details.output_path
  }'
```

| `state_type` | Action |
|--------------|--------|
| `PENDING` | Wait — executor has not started the task |
| `RUNNING` | Wait — task in progress |
| `COMPLETED` | Read output (step 6) |
| `FAILED` | Inspect error details / contact support |

### Bash poll loop

```bash
while true; do
  RESP=$(curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
    "$BASE_URL/task_runs/$TASK_RUN_ID")
  STATE=$(echo "$RESP" | jq -r '.state_type')
  echo "$(date -Iseconds) state_type=$STATE"
  case "$STATE" in
    COMPLETED|FAILED|CANCELLED|CRASHED) break ;;
  esac
  sleep 2
done
echo "$RESP" | jq .
```

### Python poll loop

```python
import os, time, requests

base = os.environ["BASE_URL"]
headers = {
    "Authorization": f"Bearer {os.environ['TASK_API_TOKEN']}",
    "X-ENGINE9-ACCOUNT-ID": "test",
}
task_run_id = os.environ["TASK_RUN_ID"]

while True:
    r = requests.get(f"{base}/task_runs/{task_run_id}", headers=headers, verify=False)
    r.raise_for_status()
    data = r.json()
    state = data["state_type"]
    print(state)
    if state in ("COMPLETED", "FAILED", "CANCELLED", "CRASHED"):
        break
    time.sleep(2)

print("output_path:", data.get("state", {}).get("state_details", {}).get("output_path"))
```

---

## Step 6: Read Echo output

When `state_type` is `COMPLETED`, get the result reference:

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/task_runs/$TASK_RUN_ID" | jq '{
    state_type,
    output_path: .state.state_details.output_path
  }'
```

Retrieve the payload using the method your administrator documents (direct file access, signed URL, or another API). Expected content (abridged):

```json
{
  "message": "hello from echo-flow",
  "seconds": 1,
  "last_run": "2026-06-08T12:00:00.000Z"
}
```

The `message` and `seconds` values match the task `options` from the flow definition.

---

## Troubleshooting

| Symptom | What to check |
|---------|----------------|
| `404` on `POST /flow_runs/` | Wrong `flow_id` slug; flow not provisioned for your account |
| `422 flow_id required` | POST body missing `flow_id` |
| `401` | Token or account header — [authentication.md](./authentication.md) |
| `503` | API not configured — ask administrator |
| Stuck `PENDING` | Background execution not running — ask administrator |

HTTP status details: [errors.md](./errors.md).
