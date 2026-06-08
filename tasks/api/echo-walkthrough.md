# Echo walkthrough

Complete example: list the **Echo** sample flow, create a flow run, list runs, poll for completion, and read output. Assumes your administrator has published `echo-flow` or `echo-flow-fast` to your account.

## Prerequisites

You need (from your administrator):

- Running Task API at a known base URL
- Valid Bearer token
- Account id (examples use `test`)
- `echo-flow` or `echo-flow-fast` in your account's flows directory

Set variables:

```bash
export BASE_URL="https://127.0.0.1:8443"
export AUTH="Authorization: Bearer localdev"
export ACCOUNT="X-ENGINE9-ACCOUNT-ID: test"
export CURL_TLS="-k"
```

## What the Echo flow does

The Echo flow has one task that calls `EchoWorker.echo`. The worker returns the task `options` object unchanged (plus metadata like `last_run` and `cwd`).

Definition (for reference — you do not POST this):

```json5
{
  id: 'echo-flow',
  name: 'Echo Flow',
  tasks: [
    {
      task_key: 'echo-step',
      name: 'Echo step',
      worker_path: 'workers/EchoWorker',
      worker_method: 'echo',
      options: { message: 'hello from echo-flow', seconds: 1 },
    },
  ],
}
```

Use `"flow_id": "echo-flow"` when creating runs. For a faster task, ask for `echo-flow-fast` (`seconds: 1`).

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

## Step 2: Create a flow run

**Important:** `flow_id` is the flow **slug** (`echo-flow`), not the UUID `flow_id` field on the flow object.

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "flow_id": "echo-flow",
    "name": "echo-demo-2026-06-08",
    "parameters": {}
  }' \
  "$BASE_URL/flow_runs/" | tee /tmp/echo-flow-run.json | jq .
```

Save IDs:

```bash
export FLOW_RUN_ID=$(jq -r '.id' /tmp/echo-flow-run.json)
export TASK_RUN_ID=$(jq -r '.task_runs[0].id' /tmp/echo-flow-run.json)
echo "FLOW_RUN_ID=$FLOW_RUN_ID"
echo "TASK_RUN_ID=$TASK_RUN_ID"
```

Example response fields:

```json
{
  "id": "0196f1d0-87cf-7b6f-b9a0-8f5d2d6c2f9e",
  "flow_slug": "echo-flow",
  "state_type": "SCHEDULED",
  "task_runs": [
    {
      "id": "0196f1d1-2c55-7d89-9c24-6fe4f7fd93ea",
      "task_key": "echo-step",
      "state_type": "PENDING"
    }
  ]
}
```

### Create with explicit account_id in body

Usually unnecessary if you send the header:

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{"flow_id":"echo-flow","account_id":"test"}' \
  "$BASE_URL/flow_runs/" | jq '{id, flow_slug, state_type}'
```

---

## Step 3: List flow runs

### By flow slug

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "flow_runs": { "flow_id": { "eq_": "echo-flow" } },
    "sort": "CREATED_DESC",
    "limit": 5
  }' \
  "$BASE_URL/flow_runs/filter" | jq '[.[] | {id, name, state_type, created}]'
```

### Read one flow run

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/flow_runs/$FLOW_RUN_ID" | jq '{id, flow_slug, state_type, start_time, end_time}'
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
    worker: .task_inputs.worker_path,
    method: .task_inputs.worker_method
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
    "Authorization": f"Bearer {os.environ['ENGINE9_TOKEN']}",
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

When `state_type` is `COMPLETED`:

```bash
OUTPUT_PATH=$(curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/task_runs/$TASK_RUN_ID" | jq -r '.state.state_details.output_path')

echo "Output file: $OUTPUT_PATH"
cat "$OUTPUT_PATH" | jq .
```

Expected content (abridged):

```json
{
  "message": "hello from echo-flow",
  "seconds": 1,
  "last_run": "2026-06-08T12:00:00.000Z",
  "cwd": "/path/to/server"
}
```

The `message` and `seconds` values come from the flow definition's `options`.

---

## Troubleshooting

| Symptom | What to check |
|---------|----------------|
| `404` on `POST /flow_runs/` | Wrong `flow_id` slug; flow not published to your account |
| `422 flow_id required` | POST body missing `flow_id` |
| `401` | Token or account header — [authentication.md](./authentication.md) |
| `503` | Server not configured — ask administrator |
| Stuck `PENDING` | Task executor not running — [admin execution](../../../server/api/task/docs/admin/execution.md) |

HTTP status details: [errors.md](./errors.md).
