# Authentication (API consumer)

Every Task API request must be authenticated. Unauthenticated requests receive **401 Unauthorized**.

## Required header

```http
Authorization: Bearer <your-token>
```

For most integrations, `<your-token>` is a **Google OAuth access token** obtained through the login flow your administrator configures.

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

## Google OAuth access token

This is the standard authentication path for the Task API.

### When to use it

Use a Google OAuth access token when your client completes a **Google OAuth 2.0 authorization code flow** and receives an **access token**. This is the same path used by Cursor, MCP clients, and other OAuth-based integrations against Engine9.

### How it works

1. Your client redirects you through Google sign-in (OAuth).
2. After authorization, the client receives a Google **access token**.
3. Send `Authorization: Bearer <google_access_token>` on every Task API request.
4. The API validates the token with Google's OpenID userinfo endpoint, reads your email, and matches it to your Engine9 user account.
5. Combine with `X-ENGINE9-ACCOUNT-ID` to scope requests to the correct account.

The same access token typically works for other Engine9 API routes on the same host (including MCP at `POST /mcp`), so you authorize once per session.

### Typical clients

| Client | How you get the token |
|--------|----------------------|
| **Cursor / MCP** | Connect via the configured MCP server; Cursor completes OAuth and attaches the access token automatically |
| **Custom scripts** | Your administrator documents the OAuth flow, or provides a pre-issued token for testing |
| **curl / HTTP clients** | Obtain a token through your organization's OAuth flow, then export it as a shell variable |

### Token lifetime

Google access tokens expire. Your client should refresh or re-authenticate before expiry. If requests suddenly return **401**, obtain a new token and retry.

## Local development token

Your administrator may provide a fixed development token for non-production environments (for example `Bearer localdev` in local setups). Use only what they document; do not use dev tokens in production.

## curl variables

Set once per shell session:

```bash
export BASE_URL="https://api.example.com"
export AUTH="Authorization: Bearer <google-access-token>"
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
const token = process.env.TASK_API_TOKEN; // Google OAuth access token

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
| 401 | Missing, expired, or invalid Google OAuth access token |
| 503 | API not fully configured — contact your administrator |

See [errors.md](./errors.md) for full status code reference.

## Content-Type

Send `Content-Type: application/json` on all `POST` requests with a body.
