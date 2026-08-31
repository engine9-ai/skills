# Echo walkthrough

Complete example: schedule **Echo** as an **on-demand task** (`path` + `method`), poll with `task_runs/filter`, and read output. Echo lives on the **built-in engine9 Workers plugin** that every bootstrapped account already has — you do **not** need an `echo-flow` flow, and you do **not** install a separate Echo plugin.

MCP uses the same on-demand names: `path: "@engine9/plugins/e9workers:EchoWorker"` and `method: "echo"`. See [e9-mcp](../e9-mcp/SKILL.md#on-demand-tasks).

To run a published multi-step workflow instead, see [getting-started.md](./getting-started.md#predefined-flow--flow_id-required) (`POST /flow_runs/` with `flow_id`).

## Running this walkthrough as an agent

An agent with shell access can run this end to end. Follow these rules:

1. **Collect credentials first.** Ask the user for three values (do not guess):
   - **Account id** (goes in `X-ENGINE9-ACCOUNT-ID`)
   - **API key** (`e9key_…` with `tasks:read` and `tasks:schedule` scopes)
   - **Base URL** — offer the production default `https://data.engine9.ai` and only ask if the user's deployment differs
2. **Protect the key.** Put it in a shell variable or environment variable. Never repeat the full key back in chat, logs, or summaries — refer to it as `e9key_…<last 4>`.
3. **Explain every step.** Before each request, state in one or two sentences what you are about to call and why (e.g. "Scheduling the Echo task with `POST /tasks/schedule` — this creates a flow run and one task run"). After each response, show the relevant fields and say what they mean before moving on. Do not run the whole sequence silently and dump a final answer.
4. **Run the sequence in order:**
   - Step 1 — schedule Echo, save `flow_run_id` and `task_run_ids[0]` from the response
   - Steps 2–4 — poll `POST /task_runs/filter` (or `GET /task_runs/:id`), reporting each observed `state_type`, until it is terminal (`COMPLETED`, `FAILED`, `CANCELLED`, `CRASHED`). Wait a couple of seconds between polls; Echo with `"seconds": 1` normally completes within a few polls.
   - Step 5 — on `COMPLETED`, report the output reference and confirm the echoed `options` match what was sent
5. **Finish with a summary:** the ids involved, how long the run took (`start_time` → `end_time`), the final state, and the echoed output. If the run ends `FAILED` or stays `PENDING` for more than a minute or two, stop polling and report the state and error details instead of retrying forever (see [Troubleshooting](#troubleshooting)).

Each step below shows the call **and** the response shape to expect, so you can verify as you go.

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

You need (from your administrator — or, if you are an agent, from the user):

- Running Task API at a known base URL (production default: `https://data.engine9.ai`)
- Valid `e9key_…` API key with `tasks:read` and `tasks:schedule`
- Account id (examples use `test`) whose engine9 Workers plugin is installed (normal bootstrap)

Set variables once and reuse them in every call (see [authentication.md](./authentication.md)):

```bash
export BASE_URL="https://data.engine9.ai"
export AUTH="Authorization: Bearer e9key_<your-key>"
export ACCOUNT="X-ENGINE9-ACCOUNT-ID: test"
export CURL_TLS=""    # "-k" only for self-signed dev certs
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

**Expected response** (abridged — your ids will differ):

```json
{
  "ok": true,
  "action": "schedule",
  "result": {
    "flow_run_id": "6a82fed56813e4e2a0a0144e",
    "task_run_ids": ["6a82fed66813e4e2a0a0144f"]
  }
}
```

What it means: the task was **accepted and queued**, not executed — `flow_run_id` identifies the wrapping flow run, and `task_run_ids[0]` is the Echo task run you will poll. Scheduling never blocks until completion.

Save the ids (shape matches MCP `task` schedule — `flow_run_id` / `task_run_ids`):

```bash
export FLOW_RUN_ID=$(jq -r '.result.flow_run_id // .flow_run_id // .id' /tmp/echo-task-run.json)
export TASK_RUN_ID=$(jq -r '.result.task_run_ids[0] // .task_run_ids[0] // empty' /tmp/echo-task-run.json)
echo "FLOW_RUN_ID=$FLOW_RUN_ID"
echo "TASK_RUN_ID=$TASK_RUN_ID"
```

An on-demand task is still wrapped in a flow run so you can poll the same listing endpoints.

---

## Step 2: Check status

`POST /task_runs/filter` lists tasks for the run (and includes the parent `flow_run` when a single `flow_run_id` is sent — one call polls both):

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d "{\"flow_run_id\": \"$FLOW_RUN_ID\"}" \
  "$BASE_URL/task_runs/filter" | jq .
```

**Expected response** (abridged):

```json
{
  "ok": true,
  "task_runs": [
    {
      "id": "6a82fed66813e4e2a0a0144f",
      "flow_run_id": "6a82fed56813e4e2a0a0144e",
      "task_key": "echo",
      "state_type": "PENDING"
    }
  ],
  "flow_run": {
    "id": "6a82fed56813e4e2a0a0144e",
    "state_type": "RUNNING"
  }
}
```

`state_type` starts at `PENDING` (queued), moves to `RUNNING` when an executor picks it up, and ends at `COMPLETED`. Seeing `PENDING` right after scheduling is normal.

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

Agents: report each observed state as you poll (e.g. "still `RUNNING` after 4s"), wait ~2 seconds between polls, and stop with an explanation if the run ends `FAILED` or stays `PENDING` for more than a minute or two.

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
    start_time,
    end_time,
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

The `message` and `seconds` values match the `options` you sent on schedule — that round trip is the success check.

## Step 6: Summarize

Close out with a short report (agents: give this to the user):

- `flow_run_id` and `task_run_id`
- Final `state_type` and elapsed time (`start_time` → `end_time`)
- The echoed output (or the `output_path` reference if the payload is not directly retrievable)
- Confirmation that scheduling, execution, and result retrieval all worked — the account and key are good for real tasks

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
| `could not find plugin for path @engine9/plugins/e9workers` | Account was not bootstrapped — engine9 Workers plugin missing |
| `401` | Token or account header — [authentication.md](./authentication.md) |
| `503` | API not configured — ask administrator |
| Stuck `PENDING` | Background execution not running — ask administrator |

HTTP status details: [errors.md](./errors.md).
