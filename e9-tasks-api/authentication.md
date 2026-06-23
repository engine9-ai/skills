# Authentication (API consumer)

Every Task API request must be authenticated. Unauthenticated requests receive **401 Unauthorized**.

## Required header

```http
Authorization: Bearer <your-token>
```

For most integrations, `<your-token>` is a **Firebase ID token** obtained through the Engine9 OAuth login flow your administrator configures. (Engine9 is its own OAuth authorization server backed by Firebase; the OAuth `access_token` it issues is the Firebase ID token.)

## Account header

Send your account id on **every** request:

```http
X-ENGINE9-ACCOUNT-ID: <account_id>
```

This scopes flows, runs, and results to your account. If omitted, the API may use an account embedded in your authenticated identity when one is available.

**Example — both headers:**

```http
GET /flows HTTP/1.1
Host: api.example.com
Authorization: Bearer ya29.a0AfH6...
X-ENGINE9-ACCOUNT-ID: acme
```

## Firebase ID token (via Engine9 OAuth)

This is the standard authentication path for the Task API.

### When to use it

Use a Firebase ID token when your client completes the **Engine9 OAuth 2.0 authorization code (PKCE) flow** and receives an **access token**. This is the same path used by Cursor, MCP clients, and other OAuth-based integrations against Engine9.

### How it works

1. Your client opens `/oauth/authorize`, which redirects you to **Google** sign-in (backed by Firebase).
2. After authorization, the client receives a `code` and exchanges it at `/oauth/token` for an **access token** (the Firebase ID token) and a `refresh_token`.
3. Send `Authorization: Bearer <access_token>` on every Task API request.
4. The API verifies the Firebase ID token (`verifyIdToken`) and resolves your Firebase `uid` to your Engine9 user account.
5. Combine with `X-ENGINE9-ACCOUNT-ID` to scope requests to the correct account.

The same access token works for other Engine9 API routes on the same host (including MCP at `POST /mcp`), so you authorize once per session.

### Typical clients

| Client | How you get the token |
|--------|----------------------|
| **Cursor / MCP** | Connect via the configured MCP server; Cursor completes OAuth and attaches the access token automatically |
| **Custom scripts** | Use `e9 oauth token` (loopback OAuth flow), or your administrator provides a pre-issued token for testing |
| **curl / HTTP clients** | Obtain a token through the OAuth flow, then export it as a shell variable |

### Token lifetime

Firebase ID tokens expire (about 1 hour). Your client should refresh via `/oauth/token` (`grant_type=refresh_token`) or re-authenticate before expiry. If requests suddenly return **401**, obtain a new token and retry.

## Local development token

Your administrator may provide a fixed development token for non-production environments (for example `Bearer localdev` in local setups). Use only what they document; do not use dev tokens in production.

## curl variables

Set once per shell session:

```bash
export BASE_URL="https://api.example.com"
export AUTH="Authorization: Bearer <firebase-id-token>"
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
const account_id = 'acme';
const token = process.env.TASK_API_TOKEN; // Firebase ID token (Engine9 OAuth access_token)

const res = await fetch(`${baseUrl}/flows`, {
  headers: {
    Authorization: `Bearer ${token}`,
    'X-ENGINE9-ACCOUNT-ID': account_id,
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
    "Authorization": f"Bearer {os.environ['TASK_API_TOKEN']}",
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
| 401 | Missing, expired, or invalid Firebase ID token |
| 503 | API not fully configured — contact your administrator |

See [errors.md](./errors.md) for full status code reference.

## Content-Type

Send `Content-Type: application/json` on all `POST` requests with a body.
