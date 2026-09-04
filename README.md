# skills

Public skills for working with engine9 deployments.

**MCP rule:** When using an engine9 MCP server, discover plugins, methods, and tool parameters from MCP responses only — not from local workspace code. See [e9-mcp — MCP-only discovery](./e9-mcp/SKILL.md#mcp-only-discovery--do-not-use-local-code).

| Skill | Path | Purpose |
|-------|------|---------|
| Create plugin / interface | [create-engine9-plugin/SKILL.md](./create-engine9-plugin/SKILL.md) | Implement `@engine9/interfaces/*` packages and `@engine9/plugins/*` native plugins |
| CLI / `/e9`, `/e9a` | [e9-cli/SKILL.md](./e9-cli/SKILL.md) | Connect Cursor to engine9 MCP, account scope, search, task scheduling |
| MCP tools | [e9-mcp/SKILL.md](./e9-mcp/SKILL.md) | Native MCP tool selection, account discovery, task fallback |
| EQL | [e9-eql/SKILL.md](./e9-eql/SKILL.md) | engine9 Query Language — expressions, query objects, MCP `eql` |
| Task API (REST) | [e9-tasks-api/SKILL.md](./e9-tasks-api/SKILL.md) | Task API REST — predefined flows (`flow_id`) or an on-demand task (`path` + `method`; Echo: `@engine9/plugins/e9workers:EchoWorker`) |
| Dev tasks (flows) | [e9-dev-tasks/SKILL.md](./e9-dev-tasks/SKILL.md) | Design/build JSON5 flows, TaskWorker, SQLTaskManager *(developers only)* |
| Timeline | [e9-timeline/SKILL.md](./e9-timeline/SKILL.md) | Person activity log, entries (never events), entry types, querying, missing-entry debug |
| Timeline loading *(developers)* | [e9-timeline/loading.md](./e9-timeline/loading.md) | InputWorker ID files → `timeline` / detail tables |
| Timeline files *(plugins)* | [inputs/timeline/SKILL.md](./inputs/timeline/SKILL.md) | Timeline ID vs Raw file shapes and `@engine9/input-tools` helpers |
| Transaction mapping | [inputs/transaction-mapping/SKILL.md](./inputs/transaction-mapping/SKILL.md) | Map 3rd-party payment data into the Transaction schema |
| Source codes | [e9-source-code/SKILL.md](./e9-source-code/SKILL.md) | Dictionary, parsing, last-click attribution (transaction ↔ message), overrides |
| Models | [e9-model/SKILL.md](./e9-model/SKILL.md) | Timeline long-term value in `{prefix}_*` tables (new identity). Not `source_code_summary.origin_*` (legacy) |
| Person identity (`person_id`) | [e9-person-id/SKILL.md](./e9-person-id/SKILL.md) | How a quality `person_id` is chosen from email, phone, `remote_person_id`; `person_id_*` vs `person_identifier`; old `person_metadata` / `person_id_int` model |
| `person_remote` | [e9-person-remote/SKILL.md](./e9-person-remote/SKILL.md) | Plugin-scoped remotes table; written by `loadPeople` / `idFiles` (not timeline); export search joins `input.plugin_id` |
| Exports | [e9-export/SKILL.md](./e9-export/SKILL.md) | What an export contains (tables, idv1, `metadata.json`, entry types) for receivers |
| Exports *(building)* | [e9-export/building.md](./e9-export/building.md) | Create/run/debug export files via `e9 exportworker` (bundle dumps, person-search) |
| Inventory | [e9-inventory/SKILL.md](./e9-inventory/SKILL.md) | Warehouse inventory and monthly statistics via `e9 inventoryworker` (plan + `statistics`) |
