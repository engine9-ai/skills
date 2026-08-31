# Authentication and scopes

The Task API authenticates with **engine9 API keys** (`e9key_…`) — the same layer-1 keys used by `@engine9/core` site APIs. Keys are stored (hashed) in the **account database** and scoped to that account.

MCP (`POST /mcp`) continues to use Firebase / engine9 OAuth / `localdev`. Do **not** send Firebase ID tokens to the Task API.

## Required headers

```http
Authorization: Bearer e9key_<key>
X-ENGINE9-ACCOUNT-ID: <account_id>
```

`X-API-Key: e9key_<key>` is also accepted instead of the Bearer form.

The account header selects which account database verifies the key. A key created for account `acme` only works when `X-ENGINE9-ACCOUNT-ID` is `acme`.

**Example:**

```http
GET /flows HTTP/1.1
Host: data.engine9.ai
Authorization: Bearer e9key_0123456789abcdef0123456789abcdef01234567
X-ENGINE9-ACCOUNT-ID: acme
```

## Authentication and scopes (canonical)

engine9 API keys require a **non-empty scopes list** at creation. A request is allowed when the key includes that route’s scope, or includes `admin`. Empty scopes deny access (they do **not** mean “all access”).

| Scope | Used by | Allows |
|-------|---------|--------|
| `people:write` | Core `POST /people` | Inbound people pipeline |
| `tables:write` | Core `POST /upsert/:table` | Allowlisted table upserts |
| `data:read` | Core `GET /read/:name` | Configured reads |
| `tasks:read` | Task API | List/read flows; check run status and logs (`GET /flows*`, `POST /task_runs/filter`, `GET /flow_runs/:id`, `GET /task_runs/:id`, `GET /task_runs/:id/log`, `GET /task_runs/:id/output`) |
| `tasks:schedule` | Task API | Schedule and control work (`POST /tasks/schedule`, `POST /flow_runs/`, retry/pause/resume/stop, `PATCH /task_runs/:id`, `*/set_state`) |
| `admin` | Any | All scopes (use this instead of the old `*` wildcard) |
| `public` | Inbound | Public forms / e9-inbound (`e9publickey_` prefix; same `api_key` table) |

Constants: `SCOPES` from `@engine9/core` / `@engine9/core/api` (`TASKS_READ`, `TASKS_SCHEDULE`, `ADMIN`, `PUBLIC`, …).

Prefixes: `e9key_…` for normal scopes; `e9publickey_…` when the key includes `public`.

### Task API scope matrix

| Route | Scope |
|-------|--------|
| `GET /flows`, `GET /flows/:id`, `POST /flows/filter`, `GET /flows_dir` | `tasks:read` |
| `GET /flow_runs/:id`, `POST /flow_runs/filter` | `tasks:read` |
| `POST /flow_runs/archive`, `POST /flow_runs/retry` | `tasks:schedule` |
| `GET /task_runs/:id`, `GET /task_runs/:id/log`, `GET /task_runs/:id/output`, `POST /task_runs/filter` | `tasks:read` |
| `POST /tasks/schedule` | `tasks:schedule` |
| `POST /flow_runs/` | `tasks:schedule` |
| `POST /flow_runs/:id/set_state`, `POST /task_runs/:id/set_state` | `tasks:schedule` |
| `POST /task_runs/:id/retry`, `/pause`, `/resume`, `/stop` | `tasks:schedule` |
| `PATCH /task_runs/:id` | `tasks:schedule` |

`POST /flow_runs/archive` and `POST /flow_runs/retry` use the **same identity as listing** (`POST /flow_runs/filter`). A `user_id` is **not** a request field and is **not** required — callers that can list can archive/retry.

For a partner that discovers flows and schedules tasks, issue a key with both:

```text
tasks:read,tasks:schedule
```

## Obtaining a key

Your administrator creates a key against the account database (plaintext shown once).

**engine9 server (preferred):** `SQLWorker.createApiKey` deploys `api_key` if missing, then inserts the hashed key. Account id comes from `accounts.d` (same as `e9 sqlworker ok`):

```bash
e9 sqlworker createApiKey -a <account_id> \
  --name partner-tasks --scopes tasks:read,tasks:schedule
```

Do not schedule this method via MCP `task` — the plaintext key must not land in task run output.

**Core-only / D1 sites** (no `e9` CLI):

```bash
npx e9core create-api-key \
  --db "<account database_connection>" \
  --name "partner-tasks" \
  --scopes tasks:read,tasks:schedule
```

Or print SQL for D1 / migrations:

```bash
npx e9core create-api-key --print-sql --name "partner-tasks" --scopes tasks:read,tasks:schedule
```

Keys are SHA-256 hashed at rest (`api_key` table). Rotate with `SqlApiKeyStore.rotate({ id })` (or recreate + revoke).

After creating a key, list recent **remote** runs for that account (`tasks:read`; `remote` defaults to `true`; no `flow_run_id` required):

```bash
curl -sS -H "Authorization: Bearer $ENGINE9_API_KEY" \
  -H "X-ENGINE9-ACCOUNT-ID: <account_id>" \
  -H "Content-Type: application/json" \
  -d '{"limit":20}' \
  https://data.engine9.ai/flow_runs/filter
```

Pass `"remote": false` for local runs. `POST /task_runs/filter` with `{ "flow_run_id": "…" }` lists that run's tasks.

## curl variables

```bash
export BASE_URL="https://data.engine9.ai"
export AUTH="Authorization: Bearer e9key_<your-key>"
export ACCOUNT="X-ENGINE9-ACCOUNT-ID: acme"
export CURL_TLS=""          # use "-k" for self-signed HTTPS in dev
```

Every example uses:

```bash
curl $CURL_TLS -sS -H "$AUTH" -H "$ACCOUNT" ...
```

## Examples

### List flows

```bash
curl $CURL_TLS -sS \
  -H "$AUTH" \
  -H "$ACCOUNT" \
  "$BASE_URL/flows"
```

### List recent runs

Defaults to **remote** runs (`TaskWorker.listRemoteFlowRuns`). Pass `"remote": false` for local runs.

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" \
  -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{"limit":20}' \
  "$BASE_URL/flow_runs/filter"
```

### Schedule an on-demand task (`path` + `method`)

`POST /tasks/schedule`. Echo smoke test uses `@engine9/plugins/e9workers:EchoWorker` (no plugin lookup). For a published flow, see [Schedule a predefined flow](#schedule-a-predefined-flow-flow_id-required).

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" \
  -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "path": "@engine9/plugins/e9workers:EchoWorker",
    "method": "echo",
    "options": { "message": "hello from echo", "seconds": 1 },
    "label": "partner smoke"
  }' \
  "$BASE_URL/tasks/schedule"
```

### Schedule a predefined flow (`flow_id` required)

`POST /flow_runs/`. For a single plugin method, see [Schedule an on-demand task](#schedule-an-on-demand-task-path--method).

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" \
  -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{"flow_id":"nightly-sync"}' \
  "$BASE_URL/flow_runs/"
```

### Check status

`POST /task_runs/filter` with the `flow_run_id` from the schedule response:

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" \
  -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{"flow_run_id":"<id from schedule>"}' \
  "$BASE_URL/task_runs/filter"
```

### JavaScript (fetch)

```javascript
const baseUrl = 'https://data.engine9.ai';
const account_id = 'acme';
const key = process.env.ENGINE9_API_KEY; // e9key_…

const res = await fetch(`${baseUrl}/flows`, {
  headers: {
    Authorization: `Bearer ${key}`,
    'X-ENGINE9-ACCOUNT-ID': account_id,
  },
});
const flows = await res.json();
```

### Python (requests)

```python
import os
import requests

base_url = "https://data.engine9.ai"
headers = {
    "Authorization": f"Bearer {os.environ['ENGINE9_API_KEY']}",
    "X-ENGINE9-ACCOUNT-ID": "acme",
}

flows = requests.get(f"{base_url}/flows", headers=headers, verify=True)
flows.raise_for_status()
print(flows.json())
```

## Verifying credentials

A successful `GET /flows` returns **200** with a JSON array (possibly empty). A successful `POST /flow_runs/filter` with `{"limit":20}` returns **200** with recent runs for the account in `X-ENGINE9-ACCOUNT-ID`.

| Status | Likely cause |
|--------|----------------|
| 401 | Missing/invalid key, or missing `X-ENGINE9-ACCOUNT-ID` |
| 403 | Unknown/disabled account, or missing required scope |
| 503 | `api_key` table missing or account DB unreachable — contact administrator |

See [errors.md](./errors.md) for the full status list.

## Content-Type

Send `Content-Type: application/json` on all `POST` requests with a body.

## Related

- Canonical three-layer auth map and key stores: `@engine9/core` README (`engine9 auth map`)
- Administrator Task API setup: `server/api/task/docs/admin/authentication.md`
