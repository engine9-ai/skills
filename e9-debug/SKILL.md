---
name: e9-debug
description: >-
  Interactive Engine9 debugging with an agent, a developer, and the e9 MCP
  server(s). Connect MCP first, then categorize and recreate issues from
  interfaces and data (never code first), scope across accounts, isolate
  remote/source systems, then optionally inspect code. Use when the user says
  DEBUG, /debug, or asks to debug, isolate, or recreate an Engine9 account,
  data, UI, or MCP issue.
---

# Engine9 DEBUG

Interactive debugging across **agent**, **developer**, and **e9 MCP server(s)**. Stages can overlap or repeat. Prefer evidence from MCP (SQL, analyze, search, files) over speculation.

**MCP + tools:** Follow [e9-mcp](../e9-mcp/SKILL.md) for login (`mcp_auth` → `user`), account scope, tool selection, and hard-stop on MCP errors. Use `/e9a` when account scope is unclear ([e9-cli](../e9-cli/SKILL.md)).

## Hard rules

1. **Step 0 first — MCP must be available.** You may collect issue context from the user in parallel, but do not run account-scoped DEBUG work (SQL, search, analyze, account, task, etc.) until an e9 MCP server is connected and authenticated.
2. **Step 1 never looks at code.** Recreate from interfaces and data only (MCP tools, URLs, SQL, describe/show tables).
3. **Do not open local workspace source** until Step 4 is explicitly started (and the user confirms they want code inspection).
4. **Be wary of scale.** Avoid full-table `COUNT(*)`, unbounded scans, and expensive aggregates unless scoped (filters, limits, sampled windows). Prefer `analyze`, `DESCRIBE` / `SHOW TABLES`, indexed filters, and small LIMIT probes.
5. **Account resolution uses MCP `account`, never SQL fan-out.** Installed plugins come from the `account` tool — not from querying each account database yourself. Do not loop `sql` across accounts to discover installs, plugin tables, or which accounts have a plugin.
6. **One primary account** must be selected before deep debugging, even if many are affected.
7. Stages may loop — if recreation fails or scope changes, return to the relevant earlier step.

## Progress checklist

Track and update as you go:

```
DEBUG Progress:
- [ ] Step 0: e9 MCP connected and authenticated
- [ ] Step 1: Categorize and recreate (no code)
- [ ] Step 1 output: Bug summary handed off / confirmed
- [ ] Step 2: Multi-account scoping
- [ ] Step 3: Remote / source system isolation
- [ ] Step 4: Code debugging (user-approved)
```

---

## Step 0 — Connect e9 MCP (gate for everything further)

Connecting to an e9 MCP server is the **first stage** of DEBUG. Other information can be gathered from the user while doing this, but the MCP server needs to be available to do anything further (recreation via SQL/search/analyze, account discovery, etc.).

### Find the right installed server

1. Inspect which MCP servers are available in this session (client catalog / tool descriptors).
2. Identify engine9 MCP entries — names often look like `engine9.ai`, `engine9.local`, `user-engine9.local`, `user-engine9.ai`, or similar.
3. **Vast majority of cases: assume a single e9 MCP server.** If exactly one engine9-looking server is present, use it.
4. **If multiple e9 MCP servers are installed**, prompt the user to pick which one (e.g. local vs production). Do not guess between environments when more than one is clearly available.
5. **If none are available or the right one is missing**, prompt the user:
   - Which e9 MCP server should we use?
   - Is it installed in Cursor Settings → MCP?
   - Do they need to add/reload a server (see [e9-cli](../e9-cli/SKILL.md) for `mcp.json` / endpoint setup)?
   - Stop account-scoped DEBUG work until a server is ready.

### Authenticate and verify

On the chosen server, follow [e9-mcp Step 0 — Log in](../e9-mcp/SKILL.md#step-0--log-in-always-first):

1. If the server reports `needsAuth` or tools fail with auth errors → call **`mcp_auth`** with `{}` and wait for the user to complete the sign-in prompt.
2. Call **`ok`**, then **`user`**, to confirm the server is reachable and signed in.
3. Record the server id/name used (for the bug summary).

If `user` already succeeds this session on the intended server, Step 0 is done.

**Do not proceed to Step 1 SQL/account tools until Step 0 is complete.** You may still ask clarifying questions about the issue while waiting on auth or server setup.

---

## Step 1 — Categorize and Recreate (NEVER look at code)

This stage looks at **interfaces and data** only to recreate the issue.

### A) Identify the affected account (MCP `account` only — no SQL for discovery)

Multiple accounts may be affected, but **one account must be selected** to continue deep SQL/analyze.

**The `account` MCP tool returns installed plugins.** Use it for all account/plugin discovery during resolution — SQL is for recreating data issues *after* scope is set, not for finding accounts or plugins.

| Situation | MCP call | DB hits |
|-----------|----------|---------|
| `account_id` known | `{ "account_id": "<id>" }` (plugins command) | One account |
| Find accounts by prefix/parent/tags/name | `{ "command": "search", "prefixes": ["…"], … }` | None (config only) |
| Find accounts that have a plugin | `{ "command": "search", "prefixes": ["…"], "plugins": ["…"] }` | Server probes filtered candidates only |

**When the id is unknown**, resolve candidates with **one** `account` search call — combine filters freely:

```json
{
  "command": "search",
  "prefixes": ["authentic"],
  "plugins": ["acoustic"]
}
```

Other useful search filters: `parents`, `recursive`, `ids`, `name`, `type`, `tags`, `include_plugins`. Narrow with `prefixes` / `parents` first when using `plugins` (server caps candidate probes via `max_scan`).

**When the id is known**, one call loads plugins for that account:

```json
{ "account_id": "authentic_foo" }
```

→ `{ ok: true, command: "plugins", plugins: [...] }` — cache this list; do not re-query on every subsequent tool call.

**Do not during account resolution:**

- Run `sql` (or `SHOW TABLES`, plugin-table SELECTs, etc.) across many accounts to find installs
- Call `user` and then fan out many per-account `account` plugin loads
- Probe every accessible account when config filters (`prefixes`, `parents`, `tags`, …) would narrow the set

Confirm access via MCP `user` or ask the developer if needed. Pick **one** matching account as the primary recreation target; set scope (`account_id`) before SQL/search/analyze. Keep the full candidate list for Step 2.

### B) Isolate the broad category

Examples: People, Transactions, Messaging, Timeline, Segments, Auth, Plugins/Inputs, UI/Console, Other.

Ask or infer from the report; state the category explicitly in the bug summary.

### C) UI bug vs data issue

Determine whether the bug is:

| Kind | Examples | Primary next action |
|------|----------|---------------------|
| **User interface** | Can't click this page, can't see this button, page blank/wrong | Obtain a relevant URL (unless pure DB) |
| **Data** | Number looks wrong, count seems off, missing/duplicate rows | SQL / analyze via e9 MCP |

It can be useful at this point to obtain a relevant URL of the issue unless it is a pure database issue.

#### C-1) User interface / "seeing an issue"

For user interface bugs, or bugs that people say they're 'seeing an issue', prompt the user for a URL if it's not considered a pure database issue.

Record the URL in the bug summary. Do not dive into frontend source in Step 1.

#### C-2) Data issues

For data issues, **SQL is the primary way** to analyze the problem — but **only after** a primary `account_id` is set in Step 1A. Do not use SQL for account or plugin discovery.

Use the information provided and the e9 `sql` endpoint to compose queries to recreate the issue.

- **Do not** immediately look at code.
- **Do** use describe/show tables (and related schema tools via MCP — e.g. `sql` with `command: "describe"` / `"tables"` / `"indexes"`, or `analyze`) to understand the SQL layout.
- Compose and execute SQL to recreate the issue, or iterate with the user to refine queries.
- Be wary of scale — full table counts, etc., can be quite expensive.

Prefer: filtered SELECTs with LIMIT, time-bounded windows, `analyze` for profiles, then targeted aggregates.

### Step 1 output — Bug summary

Produce a handoff-ready summary another agent or human can use without redoing discovery:

~~~~
# Bug summary (DEBUG Step 1)

## Issue
[Clear description of the reported problem]

## Recreated?
- [ ] Yes — evidence below
- [ ] Partially — what matched / what did not
- [ ] No — blockers and next questions

## MCP server
- server: … (from Step 0)

## Target account
- account_id: …
- Other mentioned accounts: …
- Account resolution: MCP `account` search filters + candidate ids (not SQL fan-out) …
- Plugins loaded: from `account` plugins command (count / key paths) …

## Category
[People | Transactions | Messaging | Timeline | Segments | …]

## Kind
[UI | Data | Both / unclear]

## Relevant URLs
- …

## Recreation evidence
### SQL / schema
-- queries run (and why)

### Key results
[Paste or summarize MCP sql/analyze/search outputs — keep focused]

### Notes
[Assumptions, scale caveats, open questions]
~~~~

Do **not** proceed to code until Steps 2–3 are addressed (or explicitly skipped with user agreement) and Step 4 is approved.

---

## Step 2 — Scoping (other accounts)

Once the issue is seen on one account, figure out if it affects other accounts.

- Reuse the Step 1A `account` **search** candidate list when the issue was framed as multi-account (prefix/parent/plugin). Do not rediscover by scanning every accessible account or by SQL/plugin-table probes across databases.
- **Ask the user** whether other accounts are affected when the candidate set is unclear.
- They may often not know — offer light checks only when feasible:
  - Same query shape on a second account from the search results, or
  - Compare against a known-good account if they provide one.
- Do not blast expensive data queries (SQL aggregates, full scans) across all accessible accounts — only across the filtered candidate set, one account (or a small named batch) at a time.
- Record: single-account / multi-account / unknown (and which accounts were checked).

---

## Step 3 — Isolate remote / source systems

Determine whether a **remote system** (e.g. a source or input stream) may be the primary issue.

While remote systems are often involved, the **first and easiest evaluation** is data available to the e9 MCP server:

1. SQL tables related to the plugin/input/timeline/transaction path
2. Files or artifacts reachable via MCP/task (when applicable)
3. Installed plugins and paths from the Step 1A `account` result (or one `{ "account_id": "…" }` call if not yet loaded) — still not application source code; do not re-discover via SQL

Ask: does bad/missing data already exist in-account, or does the break appear only after an external sync/input?

Record: in-account data issue / likely remote-source / inconclusive — with evidence.

---

## Step 4 — Debugging (code — prompt first)

Only after recreation (or a clear failed-recreation summary) and scoping notes:

1. **Prompt the user** before inspecting code. Many users will not have code access.
2. If they decline or lack access: stop with the Step 1–3 package; suggest handing off to someone with repo access.
3. If they approve: use the bug summary (account, category, URLs, SQL evidence) to navigate relevant repos — still prefer evidence-driven reads over broad greps.
4. Propose fixes only with user direction; do not expand into unrequested refactors.

---

## Further stages

Further stages to be done ….
