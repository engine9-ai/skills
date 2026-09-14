# engine9 skills

This directory contains public skills for working with engine9 deployments, data, APIs, and extension packages. Use this index to select the narrowest skill for the task before reading its detailed rules and examples. Developer-only references are labeled explicitly.

## Quick reference

| Task | Skill |
| --- | --- |
| Build an interface or plugin | [Create plugin or interface](create-engine9-plugin/SKILL.md) |
| Connect Cursor and choose account scope | [CLI](e9-cli/SKILL.md) |
| Select and call MCP tools | [MCP](e9-mcp/SKILL.md) |
| Query data with EQL | [EQL](e9-eql/SKILL.md) |
| Schedule work through REST | [Task API](e9-tasks-api/SKILL.md) |
| Author task flows | [Dev tasks](e9-dev-tasks/SKILL.md) |
| Understand person activity | [Timeline](e9-timeline/SKILL.md) |
| Build timeline input files | [Timeline files](inputs/timeline/SKILL.md) |
| Map payment records | [Transaction mapping](inputs/transaction-mapping/SKILL.md) |
| Build or consume exports | [Exports](e9-export/SKILL.md) |

## Concepts

Skills are task-focused product documentation. Some explain public workflows, while explicitly marked developer skills cover internal implementation and authoring.

When using an engine9 MCP server, discover plugins, methods, and tool parameters from MCP responses only. Do not infer MCP capabilities from local workspace code.

## Rules

**Rule:** Skills must not name specific clients or accounts. Use placeholders such as `<account_id>`, `<parent_account_id>`, `<prefix>`, and `engine9-accounts/<org>/<account>/export`.

**Rule:** Product terms, plugin and vendor integrations, `plugin_id`, table names, and column identifiers may be named when technically relevant.

**Rule:** Use MCP-only discovery for MCP plugins, methods, and tool parameters.

## Skill catalog

| Skill | Path | Use when |
| --- | --- | --- |
| Create plugin or interface | [create-engine9-plugin/SKILL.md](create-engine9-plugin/SKILL.md) | Implement `@engine9/interfaces/*` packages and `@engine9/plugins/*` native plugins |
| CLI and account scope | [e9-cli/SKILL.md](e9-cli/SKILL.md) | Connect Cursor to engine9 MCP, choose account scope, search, or schedule tasks |
| MCP tools | [e9-mcp/SKILL.md](e9-mcp/SKILL.md) | Select native MCP tools, discover accounts, or use task fallback |
| API keys | [e9-api-key/SKILL.md](e9-api-key/SKILL.md) | Manage generic non-session `e9key_` authentication and scopes for forms, inbound APIs, and Task API |
| EQL | [e9-eql/SKILL.md](e9-eql/SKILL.md) | Write engine9 Query Language expressions and query objects or call MCP `eql` |
| Task API | [e9-tasks-api/SKILL.md](e9-tasks-api/SKILL.md) | Schedule predefined flows by `flow_id` or on-demand tasks by `path` and `method` over REST |
| Dev tasks | [e9-dev-tasks/SKILL.md](e9-dev-tasks/SKILL.md) | Design JSON5 flows or develop TaskWorker and SQLTaskManager; developers only |
| Timeline | [e9-timeline/SKILL.md](e9-timeline/SKILL.md) | Understand person activity entries, entry types, queries, and missing-entry diagnosis |
| Timeline loading | [e9-timeline/loading.md](e9-timeline/loading.md) | Load InputWorker ID files into timeline and detail tables; developers only |
| Timeline files | [inputs/timeline/SKILL.md](inputs/timeline/SKILL.md) | Choose Timeline Raw or Timeline ID shapes and use `@engine9/input-tools` |
| Transaction mapping | [inputs/transaction-mapping/SKILL.md](inputs/transaction-mapping/SKILL.md) | Map third-party payment data into the Transaction schema |
| Source codes | [e9-source-code/SKILL.md](e9-source-code/SKILL.md) | Work with dictionaries, parsing, last-click attribution, and overrides |
| Global message tables | [e9-global-message/SKILL.md](e9-global-message/SKILL.md) | Read `global_message_summary` and `global_message_summary_by_date` engagement and attribution data |
| Models | [e9-model/SKILL.md](e9-model/SKILL.md) | Analyze timeline lifetime value in `{prefix}_*` model tables |
| Person identity | [e9-person-id/SKILL.md](e9-person-id/SKILL.md) | Understand selection of `person_id`, compact identifier tables, and legacy identity models |
| Person remotes | [e9-person-remote/SKILL.md](e9-person-remote/SKILL.md) | Work with plugin-scoped `person_remote` rows, identity loading, and export joins |
| Exports | [e9-export/SKILL.md](e9-export/SKILL.md) | Understand tables, ID files, metadata, and entry types in an export |
| Export building | [e9-export/building.md](e9-export/building.md) | Create, run, and debug exports with `e9 exportworker` |
| Inventory | [e9-inventory/SKILL.md](e9-inventory/SKILL.md) | Run warehouse inventory and monthly statistics with `e9 inventoryworker` |
| Reports | [e9-reports/SKILL.md](e9-reports/SKILL.md) | Install JSON dashboards, list filters, and execute reports through MCP or HTTP |

## Related documentation

- [MCP-only discovery](e9-mcp/SKILL.md#mcp-only-discovery--do-not-use-local-code)
- [API key management UI](e9-api-key/ui.md)
- [Export building](e9-export/building.md)
- [Timeline loading](e9-timeline/loading.md)
