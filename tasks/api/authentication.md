# Authentication (API consumer)

Every Task API request must be authenticated. Unauthenticated requests receive **401 Unauthorized**, including on `127.0.0.1`.

## Required header

```http
Authorization: Bearer <your-token>
```

Your administrator tells you how to obtain `<your-token>`:

| Token type | Typical source |
|------------|----------------|
| Google OAuth access token | OAuth login flow (same as MCP / Cursor) |
| Firebase ID token | Engine9 web UI or Firebase SDK |
| `localdev` | Local development only — administrator enables this |

## Account header

Send your account id on **every** request:

```http
X-ENGINE9-ACCOUNT-ID: <account_id>
```

This scopes flows, runs, and SQL data to your account. If omitted, the server may use an account embedded in your token (if any).

**Example — both headers:**

```http
GET /flows HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOi...
X-ENGINE9-ACCOUNT-ID: acme
```

## curl variables

Set once per shell session:

```bash
export BASE_URL="https://api.example.com"
export AUTH="Authorization: Bearer <your-token>"
export ACCOUNT="X-ENGINE9-ACCOUNT-ID: acme"
export CURL_TLS=""          # use "-k" for self-signed HTTPS in dev
```

Every example in this documentation uses:

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

### Create flow run (POST with JSON body)

```bash
curl $CURL_TLS -sS -X POST \
  -H "$AUTH" \
  -H "$ACCOUNT" \
  -H "Content-Type: application/json" \
  -d '{"flow_id":"echo-flow"}' \
  "$BASE_URL/flow_runs/"
```

### JavaScript (fetch)

```javascript
const baseUrl = 'https://api.example.com';
const accountId = 'acme';
const token = process.env.ENGINE9_TOKEN;

const res = await fetch(`${baseUrl}/flows`, {
  headers: {
    Authorization: `Bearer ${token}`,
    'X-ENGINE9-ACCOUNT-ID': accountId,
  },
});
const flows = await res.json();
```

### Python (requests)

```python
import os
import requests

base_url = "https://api.example.com"
headers = {
    "Authorization": f"Bearer {os.environ['ENGINE9_TOKEN']}",
    "X-ENGINE9-ACCOUNT-ID": "acme",
}

flows = requests.get(f"{base_url}/flows", headers=headers, verify=True)
flows.raise_for_status()
print(flows.json())
```

## Verifying your credentials

A successful `GET /flows` returns **200** with a JSON array (possibly empty).

Failures:

| Status | Likely cause |
|--------|----------------|
| 401 | Missing, expired, or invalid Bearer token |
| 503 | Server misconfiguration (contact administrator) |

See [errors.md](./errors.md) for full status code reference.

## Local development token

When your administrator enables `localdev`:

```bash
export AUTH="Authorization: Bearer localdev"
export ACCOUNT="X-ENGINE9-ACCOUNT-ID: test"
```

Only works if the server has your account id in `ENGINE9_MCP_LOCALDEV_ACCOUNTS`. Not for production.

## Content-Type

Send `Content-Type: application/json` on all `POST` requests with a body.
