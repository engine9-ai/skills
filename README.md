# skills

Skills for working with engine9 deployments.

**MCP rule:** When using an engine9 MCP server, discover plugins, methods, and tool parameters from MCP responses only — not from local workspace code. See [e9-mcp — MCP-only discovery](./e9-mcp/SKILL.md#mcp-only-discovery--do-not-use-local-code).

| Skill | Path | Purpose |
|-------|------|---------|
| Create plugin / interface | [create-engine9-plugin/SKILL.md](./create-engine9-plugin/SKILL.md) | Implement `@engine9/interfaces/*` packages and `@engine9/plugins/*` native plugins |
| CLI / `/e9`, `/e9a` | [e9-cli/SKILL.md](./e9-cli/SKILL.md) | Connect Cursor to engine9 MCP, account scope, search, task scheduling |
| MCP tools | [e9-mcp/SKILL.md](./e9-mcp/SKILL.md) | Native MCP tool selection, account discovery, task fallback |
| EQL | [e9-eql/SKILL.md](./e9-eql/SKILL.md) | engine9 Query Language — expressions, query objects, MCP `eql` |
| DEBUG | [e9-debug/SKILL.md](./e9-debug/SKILL.md) | Interactive issue recreation, scoping, remote isolation, then optional code debug |
| Task API (REST) | [e9-tasks-api/SKILL.md](./e9-tasks-api/SKILL.md) | Execute flows via REST — list, create runs, poll state |
| Dev tasks (flows) | [e9-dev-tasks/SKILL.md](./e9-dev-tasks/SKILL.md) | Design/build JSON5 flows, TaskWorker, SQLTaskManager *(developers only)* |
| Timeline (server) | [e9-timeline/SKILL.md](./e9-timeline/SKILL.md) | InputWorker timeline ID/raw files and database loading |
| Timeline (input-tools) | [inputs/timeline/SKILL.md](./inputs/timeline/SKILL.md) | Timeline ID vs Raw file shapes and `@engine9/input-tools` helpers |
| Transaction mapping | [inputs/transaction-mapping/SKILL.md](./inputs/transaction-mapping/SKILL.md) | Map 3rd-party payment data into the Transaction schema |
| Source codes | [e9-source-code/SKILL.md](./e9-source-code/SKILL.md) | Dictionary, parsing, labels, last-click vs origin attribution, overrides |
| Person identity (`person_id`) | [e9-person-id/SKILL.md](./e9-person-id/SKILL.md) | How a quality `person_id` is chosen from email, phone, `remote_person_id`; `person_id_*` vs `person_identifier`; old `person_metadata` / `person_id_int` model |
