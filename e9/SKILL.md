---
name: e9
description: Connect Cursor (and similar MCP clients) to an engine9 MCP server and support `/e9` command-style interactions including account-aware person search.
---

# Engine9 MCP

Use this skill when setting up or troubleshooting an MCP connection to an engine9 server from Cursor (or another MCP client), and when handling `/e9` command-style requests (especially `/e9 search ...`).

## What this skill does

1. Documents how to connect a client to engine9 MCP.
2. Shows required Cursor config (`mcp.json`, `permissions.json`).
3. Explains auth expectations (OAuth vs `localdev` bearer).
4. Defines `/e9` session behavior, including required `accountId` handling.
5. Defines `/e9 search` argument parsing and MCP `search` payload mapping.
6. Lists currently registered tools, including `user` and `search`.

## `/e9` command contract

This skill supports a command flow along the lines of:

- `/e9`
- `/e9 search foo@bar.com`

### Session and account requirements

- Every MCP call that is account-scoped (including `search`) MUST include `accountId`.
- If `accountId` is already known in session context, reuse it.
- If `accountId` is not known, request it from the user after `/e9` before running account-scoped tools.
- Do not guess `accountId`; require explicit user/session context.

### Recommended `/e9` bootstrap flow

1. `/e9` with no subcommand:
   - Verify connectivity via `ok`.
   - Verify identity/access via `user`.
   - Confirm/set active `accountId` for this session.
2. `/e9 search ...`:
   - Ensure active `accountId` exists (or request it).
   - Parse search terms into MCP `search` arguments.
   - Call `search` with mapped filters and `accountId`.

## `/e9 search` parsing rules

For tokenized arguments after `search`:

- Tokens containing `@` MUST be treated as email inputs and mapped to `emails`.
- Tokens that are straight integers (digits only, e.g. `12345`) MUST be treated as `person_ids`.
- Other terms are at the agent's discretion and may map to other supported filters (`phones`, `given_names`, `last_names`) based on context.

When multiple values exist for a filter, pass them as arrays.

## Server endpoint and startup

Engine9 MCP is exposed at `POST /mcp` (with `GET /mcp/health` for health checks).

- Preferred server startup: run the main API (`node api/index.js`) so MCP and other APIs share one port.
- Standalone MCP startup: from `engine9/server`, run:
  - `npm run mcp`
- Default standalone MCP URL:
  - `http://127.0.0.1:3334/mcp` (when TLS is off)
  - `https://127.0.0.1:3334/mcp` (when `ENGINE9_SSL_CERT_PATH` is set)

Quick health checks:

- HTTP: `curl -sS http://127.0.0.1:3334/mcp/health`
- HTTPS (self-signed local cert): `curl -k -sS https://127.0.0.1:3334/mcp/health`

## Cursor MCP configuration

Create or update `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "engine9.io": {
      "url": "https://<your-api-host>/mcp"
    }
  }
}
```

Notes:

- The server key (`engine9.io` above) should match engine9's MCP server name (`ENGINE9_MCP_SERVER_NAME`, default `engine9.io`).
- If local standalone MCP runs without TLS, use `http://127.0.0.1:3334/mcp`.
- After config changes, reload MCP servers in Cursor.

## Cursor permissions configuration

Create or update `~/.cursor/permissions.json`:

Allow all engine9 tools:

```json
{
  "mcpAllowlist": ["engine9.io:*"]
}
```

Or allow a subset:

```json
{
  "mcpAllowlist": ["engine9.io:ok", "engine9.io:user", "engine9.io:search"]
}
```

## Auth expectations

- MCP requests to `/mcp` require Bearer auth (OAuth or `localdev`).
- In Cursor, normal flow is OAuth connect on first request.
- For local/offline testing, `Authorization: Bearer localdev` can be used with server-side localdev account configuration (`ENGINE9_MCP_LOCALDEV_ACCOUNTS`).

## Available tools (current)

The engine9 MCP server currently registers these tools:

1. `ok`  
   Returns server status/time and whether request is authenticated.
2. `plugin_id`  
   Computes `plugin_id` from `account_id` and `remote_plugin_id`.
3. `input_id`  
   Computes `input_id` from `plugin_id` and `remote_input_id`.
4. `user` (often invoked as `/user` in slash-style clients)  
   Returns current authenticated user details and account access.
5. `worker_invoke`  
   Invokes allowlisted worker methods (account-scoped).
6. `eql`  
   Runs a query from an EQL object and returns SQL + rows.
7. `search` (new)  
   Searches people by metadata filters and returns person summaries/related records.
   - Supported filters: `emails`, `person_ids`, `phones`, `given_names`, `last_names`
   - Required: `accountId`
   - Optional: `limit` (max 1000, default 10)
8. `schedule_tasks`  
   Schedules tasks via flow path or direct worker task spec.

## Suggested quick validation flow

1. Call `ok` to verify connectivity and auth state.
2. Call `user` (or `/user`) to confirm signed-in identity and account access.
3. Confirm active `accountId` for the `/e9` session.
4. Call `search` with a known email/person id to validate account-scoped data access.

Example `search` payload:

```json
{
  "accountId": "test",
  "emails": "person@example.com",
  "limit": 5
}
```

Example mapping for `/e9 search foo@bar.com 12345`:

```json
{
  "accountId": "test",
  "emails": ["foo@bar.com"],
  "person_ids": [12345],
  "limit": 10
}
```
---
name: e9
description: Connect Cursor (and similar MCP clients) to an engine9 MCP server and support `/e9` command-style interactions including account-aware person search.
---

# Engine9 MCP

Use this skill when setting up or troubleshooting an MCP connection to an engine9 server from Cursor (or another MCP client), and when handling `/e9` command-style requests (especially `/e9 search ...`).

## What this skill does

1. Documents how to connect a client to engine9 MCP.
2. Shows required Cursor config (`mcp.json`, `permissions.json`).
3. Explains auth expectations (OAuth vs `localdev` bearer).
4. Defines `/e9` session behavior, including required `accountId` handling.
5. Defines `/e9 search` argument parsing and MCP `search` payload mapping.
6. Lists currently registered tools, including `user` and `search`.

## `/e9` command contract

This skill supports a command flow along the lines of:

- `/e9`
- `/e9 search foo@bar.com`

### Session and account requirements

- Every MCP call that is account-scoped (including `search`) MUST include `accountId`.
- If `accountId` is already known in session context, reuse it.
- If `accountId` is not known, request it from the user after `/e9` before running account-scoped tools.
- Do not guess `accountId`; require explicit user/session context.

### Recommended `/e9` bootstrap flow

1. `/e9` with no subcommand:
   - Verify connectivity via `ok`.
   - Verify identity/access via `user`.
   - Confirm/set active `accountId` for this session.
2. `/e9 search ...`:
   - Ensure active `accountId` exists (or request it).
   - Parse search terms into MCP `search` arguments.
   - Call `search` with mapped filters and `accountId`.

## `/e9 search` parsing rules

For tokenized arguments after `search`:

- Tokens containing `@` MUST be treated as email inputs and mapped to `emails`.
- Tokens that are straight integers (digits only, e.g. `12345`) MUST be treated as `person_ids`.
- Other terms are at the agent's discretion and may map to other supported filters (`phones`, `given_names`, `last_names`) based on context.

When multiple values exist for a filter, pass them as arrays.

## Server endpoint and startup

Engine9 MCP is exposed at `POST /mcp` (with `GET /mcp/health` for health checks).

- Preferred server startup: run the main API (`node api/index.js`) so MCP and other APIs share one port.
- Standalone MCP startup: from `engine9/server`, run:
  - `npm run mcp`
- Default standalone MCP URL:
  - `http://127.0.0.1:3334/mcp` (when TLS is off)
  - `https://127.0.0.1:3334/mcp` (when `ENGINE9_SSL_CERT_PATH` is set)

Quick health checks:

- HTTP: `curl -sS http://127.0.0.1:3334/mcp/health`
- HTTPS (self-signed local cert): `curl -k -sS https://127.0.0.1:3334/mcp/health`

## Cursor MCP configuration

Create or update `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "engine9.io": {
      "url": "https://<your-api-host>/mcp"
    }
  }
}
```

Notes:

- The server key (`engine9.io` above) should match engine9's MCP server name (`ENGINE9_MCP_SERVER_NAME`, default `engine9.io`).
- If local standalone MCP runs without TLS, use `http://127.0.0.1:3334/mcp`.
- After config changes, reload MCP servers in Cursor.

## Cursor permissions configuration

Create or update `~/.cursor/permissions.json`:

Allow all engine9 tools:

```json
{
  "mcpAllowlist": ["engine9.io:*"]
}
```

Or allow a subset:

```json
{
  "mcpAllowlist": ["engine9.io:ok", "engine9.io:user", "engine9.io:search"]
}
```

## Auth expectations

- MCP requests to `/mcp` require Bearer auth (OAuth or `localdev`).
- In Cursor, normal flow is OAuth connect on first request.
- For local/offline testing, `Authorization: Bearer localdev` can be used with server-side localdev account configuration (`ENGINE9_MCP_LOCALDEV_ACCOUNTS`).

## Available tools (current)

The engine9 MCP server currently registers these tools:

1. `ok`  
   Returns server status/time and whether request is authenticated.
2. `plugin_id`  
   Computes `plugin_id` from `account_id` and `remote_plugin_id`.
3. `input_id`  
   Computes `input_id` from `plugin_id` and `remote_input_id`.
4. `user` (often invoked as `/user` in slash-style clients)  
   Returns current authenticated user details and account access.
5. `worker_invoke`  
   Invokes allowlisted worker methods (account-scoped).
6. `eql`  
   Runs a query from an EQL object and returns SQL + rows.
7. `search` (new)  
   Searches people by metadata filters and returns person summaries/related records.
   - Supported filters: `emails`, `person_ids`, `phones`, `given_names`, `last_names`
   - Required: `accountId`
   - Optional: `limit` (max 1000, default 10)
8. `schedule_tasks`  
   Schedules tasks via flow path or direct worker task spec.

## Suggested quick validation flow

1. Call `ok` to verify connectivity and auth state.
2. Call `user` (or `/user`) to confirm signed-in identity and account access.
3. Confirm active `accountId` for the `/e9` session.
4. Call `search` with a known email/person id to validate account-scoped data access.

Example `search` payload:

```json
{
  "accountId": "test",
  "emails": "person@example.com",
  "limit": 5
}
```

Example mapping for `/e9 search foo@bar.com 12345`:

```json
{
  "accountId": "test",
  "emails": ["foo@bar.com"],
  "person_ids": [12345],
  "limit": 10
}
```
