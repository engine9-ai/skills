# Deploy a flow for an account (fast path)

Use this when an operator or agent needs to **check** and **schedule** a multi-step flow (e.g. identity rebuild) for one account. Goal: seconds, not a scavenger hunt.

## Prefer MCP `task` (production)

After `mcp_auth` → `user`:

```json
{
  "action": "schedule",
  "account_id": "<account_id>",
  "flow_path": "rebuild/identity-rebuild.flow.json5",
  "label": "Identity rebuild"
}
```

A built-in slug still works for server-shipped flows: `"flow_id": "identity-rebuild"`. An account flow that is not a file the scheduler can read is `"definition": { "id", "name", "tasks" }`.

| Field | Notes |
|-------|--------|
| `flow_path` | Path to a JSON5 flow file (absolute, or relative to the server package root). This is the usual locator. |
| `definition` | The flow object itself (`id`, `name`, `tasks`, `schedule`). No file on the server. Use this when the flow is not deployed. |
| `flow_id` | Built-in server slug only (e.g. `identity-rebuild`). `$ENGINE9_ACCOUNT_DIR/<account>/flows` is not scanned. |
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

## Local CLI (when MCP server is behind your checkout)

If MCP returns `Flow not found` / `Flow file not found` because production has not picked up a new flow file yet, schedule from a local `engine9/server` checkout (reads the flow locally, enqueues remotely):

```bash
cd /path/to/engine9/server
e9 task runFlow -a <account_id> --flow=rebuild/identity-rebuild.flow.json5
```

Same defaults: remote schedule, **first task paused**. Confirm with MCP `listTasks` on the returned `flow_run_id`.

## Publish a flow file to the scheduler

`schedule` runs a flow once. To run it on its cron, publish the flow file as a Frakture dataflow with MCP `task` `action: "publishFlow"` (admin). Publishing is on demand. Nothing publishes, disables, or deletes dataflows automatically when files change or are removed.

The file's location is the lookup. It is stored as `labels.source`. Publishing the same `source` again updates that dataflow with the file's current tasks, name, and cron. Different `source`, different dataflow.

| The flow file lives | Send |
|---------------------|------|
| In engine9-accounts, deployed on the MCP server | `flow_path: "engine9-accounts/<org>/<account>/<dir>/<name>.flow.json5"`. The server reads it from `ENGINE9_ACCOUNT_DIR`, like an export `definition_path`. Push and deploy it first. It must set `labels.account_id`. |
| Anywhere else (your GitHub repo, a URL, a local checkout) | `definition` (the parsed file) plus `source` naming where it lives, e.g. `github:<org>/<repo>@main:<dir>/<name>.flow.json5`. The server does not read `source`. |

1. Set the cron in the file: `schedule: { label: "Every 2 hours", cron: "0 */2 * * *" }`. Without `schedule`, publish leaves the stored cron alone.
2. Dry run:

   ```json
   {
     "action": "publishFlow",
     "account_id": "<account_id>",
     "flow_path": "engine9-accounts/<org>/<account>/<dir>/<name>.flow.json5",
     "dry_run": true
   }
   ```

   `operation` is `create` the first time and `update` after that. For a file outside the server, send `source` and `definition` in place of `flow_path`.
3. Publish: the same call without `dry_run`. Save `dataflow_id`.
4. Check: `{ "action": "getFlow", "account_id": "<account_id>", "flow_id": "<dataflow_id>", "remote": true }`. `labels.source`, `schedule.cron`, and each task's `assignee.bot_id` should be there.
5. Optional first run now: `{ "action": "schedule", "account_id": "<account_id>", "flow_id": "<dataflow_id>" }`.

Each task's bot comes from the account's installed plugins, the same as `schedule`; a task whose plugin is not installed fails the publish. Task options that name files, such as an export `definition_path`, are read by the worker at run time, so those files must be readable there even when the flow was sent as `definition`.

To stop it: `setDisabled` with `disabled: true`, or `setSchedule` with an empty `cron`. Full reference: [e9-mcp — Publish a flow file](../e9-mcp/SKILL.md#publish-a-flow-file-as-a-remote-dataflow).

## Do not

- Do not invent task `path`/`method` lists by hand when a flow already exists — use `flow_path` or `definition`.
- Do not change a published flow's `source` (rename, move, another branch in the name) unless you mean to create a second dataflow. Disable the old one.
- Do not use cwd-relative guesses like `api/rebuild/...` — resolution is server-root first.
- Do not set `pause_first: false` for destructive rebuilds unless the user explicitly asked to run immediately.
