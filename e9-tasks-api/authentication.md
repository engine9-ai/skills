# Authentication and scopes

The Task API authenticates with **Engine9 API keys** (`e9key_…`) — the same layer-1 keys used by `@engine9/core` site APIs. Keys are stored (hashed) in the **account database** and scoped to that account.

MCP (`POST /mcp`) continues to use Firebase / Engine9 OAuth / `localdev`. Do **not** send Firebase ID tokens to the Task API.

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

Engine9 API keys require a **non-empty scopes list** at creation. A request is allowed when the key includes that route’s scope, or includes `admin`. Empty scopes deny access (they do **not** mean “all access”).

| Scope | Used by | Allows |
|-------|---------|--------|
| `people:write` | Core `POST /people` | Inbound people pipeline |
| `tables:write` | Core `POST /upsert/:table` | Allowlisted table upserts |
| `data:read` | Core `GET /read/:name` | Configured reads |
| `tasks:read` | Task API | List/read flows; check run status (`GET /flows*`, `POST /tasks/listTasks`, `GET /flow_runs/:id`, `GET /task_runs/:id`, filters) |
| `tasks:schedule` | Task API | Schedule work (`POST /tasks/schedule`, `POST /flow_runs/`, `*/set_state`) |
| `admin` | Any | All scopes (use this instead of the old `*` wildcard) |
| `public` | Inbound | Public forms / e9-inbound (`e9publickey_` prefix; same `api_key` table) |

Constants: `SCOPES` from `@engine9/core` / `@engine9/core/api` (`TASKS_READ`, `TASKS_SCHEDULE`, `ADMIN`, `PUBLIC`, …).

Prefixes: `e9key_…` for normal scopes; `e9publickey_…` when the key includes `public`.

### Task API scope matrix

| Route | Scope |
|-------|--------|
| `GET /flows`, `GET /flows/:id`, `POST /flows/filter`, `GET /flows_dir` | `tasks:read` |
| `POST /tasks/listTasks` | `tasks:read` |
| `GET /flow_runs/:id`, `POST /flow_runs/filter` | `tasks:read` |
| `GET /task_runs/:id`, `POST /task_runs/filter` | `tasks:read` |
| `POST /tasks/schedule` | `tasks:schedule` |
| `POST /flow_runs/` | `tasks:schedule` |
| `POST /flow_runs/:id/set_state`, `POST /task_runs/:id/set_state` | `tasks:schedule` |

For a partner that discovers flows and schedules jobs, issue a key with both:

```text
tasks:read,tasks:schedule
```

## Obtaining a key

Your administrator creates a key against the account database (plaintext shown once):

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

### Schedule (MCP-parity)

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" \
  -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{
    "path": "@engine9/plugins/e9workers:SQLWorker",
    "method": "query",
    "options": { "sql": "select 1 as ok" },
    "label": "partner smoke"
  }' \
  "$BASE_URL/tasks/schedule"
```

### Schedule a published flow by slug

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" \
  -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{"flow_id":"echo-flow"}' \
  "$BASE_URL/flow_runs/"
```

### Check status

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" \
  -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{"flow_run_id":"<id from schedule>"}' \
  "$BASE_URL/tasks/listTasks"
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

A successful `GET /flows` returns **200** with a JSON array (possibly empty).

| Status | Likely cause |
|--------|----------------|
| 401 | Missing/invalid key, or missing `X-ENGINE9-ACCOUNT-ID` |
| 403 | Unknown/disabled account, or missing required scope |
| 503 | `api_key` table missing or account DB unreachable — contact administrator |

See [errors.md](./errors.md) for the full status list.

## Content-Type

Send `Content-Type: application/json` on all `POST` requests with a body.

## Related

- Canonical three-layer auth map and key stores: `@engine9/core` README (`Engine9 auth map`)
- Administrator Task API setup: `server/api/task/docs/admin/authentication.md`
