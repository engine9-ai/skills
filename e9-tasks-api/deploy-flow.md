# Deploy a flow for an account (fast path)

Use this when an operator or agent needs to **check** and **schedule** a multi-step flow (e.g. identity rebuild) for one account. Goal: seconds, not a scavenger hunt.

## Prefer MCP `task` (production)

After `mcp_auth` → `user`:

```json
{
  "action": "schedule",
  "account_id": "<account_id>",
  "flow_id": "identity-rebuild",
  "label": "Identity rebuild"
}
```

| Field | Notes |
|-------|--------|
| `flow_id` | Built-in slug (e.g. `identity-rebuild`) or account-published flow. Prefer this. |
| `flow_path` | Fallback: `rebuild/identity-rebuild.flow.json5` (resolved from **server package root**, not process cwd). |
| `pause_first` | Default **true** — first task is PAUSED; resume it to start the chain. |

Then:

```json
{ "action": "listTasks", "account_id": "<account_id>", "flow_run_id": "<from schedule>" }
```

Resume when ready:

```json
{ "action": "resume", "account_id": "<account_id>", "task_run_id": "<paused_task_run_id>" }
```

### Built-in flows (no account publish step)

Server ships flows discovered without copying into `$ENGINE9_ACCOUNT_DIR/<account>/flows/`:

| Slug | Path |
|------|------|
| `identity-rebuild` | `rebuild/identity-rebuild.flow.json5` |

Account-published files still override the same slug.

## Local CLI (when MCP server is behind your checkout)

If MCP returns `Flow not found` / `Flow file not found` because production has not picked up a new flow file yet, schedule from a local `engine9/server` checkout (reads the flow locally, enqueues remotely):

```bash
cd /path/to/engine9/server
e9 task runFlow -a <account_id> --flow=rebuild/identity-rebuild.flow.json5
```

Same defaults: remote schedule, **first task paused**. Confirm with MCP `listTasks` on the returned `flow_run_id`.

## Do not

- Do not invent task `path`/`method` lists by hand when a flow already exists — use `flow_id` / `flow_path`.
- Do not use cwd-relative guesses like `api/rebuild/...` — resolution is server-root first.
- Do not set `pause_first: false` for destructive rebuilds unless the user explicitly asked to run immediately.
