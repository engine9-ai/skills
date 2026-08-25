# Echo walkthrough

Complete example: schedule **Echo** as an **on-demand task** (`path` + `method`), poll with `task_runs/filter`, and read output. Echo lives on the **built-in Engine9 Workers plugin** that every bootstrapped account already has — you do **not** need an `echo-flow` flow, and you do **not** install a separate Echo plugin.

MCP uses the same on-demand names: `path: "@engine9/plugins/e9workers:EchoWorker"` and `method: "echo"`. See [e9-mcp](../e9-mcp/SKILL.md#on-demand-tasks).

To run a published multi-step workflow instead, see [getting-started.md](./getting-started.md#predefined-flow--flow_id-required) (`POST /flow_runs/` with `flow_id`).

## On-demand task names

On-demand tasks are identified by **`path` + `method`** (never a `flow_id`). The same strings are used by REST `POST /tasks/schedule` and MCP `task`.

| What | Value |
|------|--------|
| **Plugin path** | `@engine9/plugins/e9workers` — installed by `bootstrapAccount` on every account |
| **Echo path** | `@engine9/plugins/e9workers:EchoWorker` |
| **Echo method** | `echo` — returns `options` plus metadata (`last_run`, …) |

Other built-in on-demand examples: `@engine9/plugins/e9workers:SQLWorker` + `query`, `@engine9/plugins/e9workers:SegmentWorker` + a SegmentWorker method.

Do **not** use `workers/EchoWorker` (local file path) or a standalone Echo plugin for this smoke test.

## Prerequisites

You need (from your administrator):

- Running Task API at a known base URL
- Valid `e9key_…` API key with `tasks:read` and `tasks:schedule`
- Account id (examples use `test`) whose Engine9 Workers plugin is installed (normal bootstrap)

Set variables (see [authentication.md](./authentication.md)):

```bash
export BASE_URL="https://data.engine9.ai"
export AUTH="Authorization: Bearer e9key_<your-key>"
export ACCOUNT="X-ENGINE9-ACCOUNT-ID: test"
export CURL_TLS=""
```

## What Echo does

`EchoWorker.echo` returns its input `options` unchanged (plus metadata like `last_run`). It is the standard check that authentication, on-demand scheduling, and result retrieval work end to end.

---

## Step 1: Schedule the on-demand Echo task

**`path` and `method` are required.** Do not send `flow_id`. Scheduling goes through `scheduleTasks` (same as MCP `task`).

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "path": "@engine9/plugins/e9workers:EchoWorker",
    "method": "echo",
    "options": { "message": "hello from echo", "seconds": 1 },
    "label": "echo-demo-2026-06-08"
  }' \
  "$BASE_URL/tasks/schedule" | tee /tmp/echo-task-run.json | jq .
```

The server accepts that path without looking up installed plugins (`@engine9/plugins/e9workers` is well-known).

Save IDs (shape matches MCP `task` schedule — `flow_run_id` / `task_run_ids`):

```bash
export FLOW_RUN_ID=$(jq -r '.result.flow_run_id // .flow_run_id // .id' /tmp/echo-task-run.json)
export TASK_RUN_ID=$(jq -r '.result.task_run_ids[0] // .task_run_ids[0] // empty' /tmp/echo-task-run.json)
echo "FLOW_RUN_ID=$FLOW_RUN_ID"
echo "TASK_RUN_ID=$TASK_RUN_ID"
```

An on-demand task is still wrapped in a flow run so you can poll the same listing endpoints.

---

## Step 2: Check status

`POST /task_runs/filter` lists tasks for the run:

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d "{\"flow_run_id\": \"$FLOW_RUN_ID\"}" \
  "$BASE_URL/task_runs/filter" | jq .
```

Or GET the flow run:

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" \
  "$BASE_URL/flow_runs/$FLOW_RUN_ID" | jq .
```

---

## Step 3: List task runs

### All task runs for your flow run

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d "{
    \"task_runs\": { \"flow_run_id\": { \"eq_\": \"$FLOW_RUN_ID\" } },
    \"limit\": 10
  }" \
  "$BASE_URL/task_runs/filter" | jq '[.task_runs[] | {id, task_key, state_type}]'
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

## Step 4: Poll until complete

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
| `COMPLETED` | Read output (step 5) |
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

## Step 5: Read Echo output

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
  "message": "hello from echo",
  "seconds": 1,
  "last_run": "2026-06-08T12:00:00.000Z"
}
```

The `message` and `seconds` values match the `options` you sent on schedule.

---

## MCP equivalent

After MCP login and an `account_id`, call `task` with the same on-demand names. Do **not** call `account` first — `@engine9/plugins/e9workers:...` is well-known.

```json
{
  "account_id": "test",
  "path": "@engine9/plugins/e9workers:EchoWorker",
  "method": "echo",
  "options": { "message": "hello from echo", "seconds": 1 },
  "label": "echo-demo"
}
```

Poll with `action: "listTasks"` and the `flow_run_id` from the schedule response.

---

## Troubleshooting

| Symptom | What to check |
|---------|----------------|
| `422` on `POST /tasks/schedule` | Body missing `path` and `method`, or `flow_id` was sent (use `POST /flow_runs/` for a published flow) |
| `Unknown engine9 worker` | Path submodule is not an Engine9Workers class (use `EchoWorker`, not a separate plugin) |
| `could not find plugin for path @engine9/plugins/e9workers` | Account was not bootstrapped — Engine9 Workers plugin missing |
| `401` | Token or account header — [authentication.md](./authentication.md) |
| `503` | API not configured — ask administrator |
| Stuck `PENDING` | Background execution not running — ask administrator |

HTTP status details: [errors.md](./errors.md).
