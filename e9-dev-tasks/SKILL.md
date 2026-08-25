---
name: e9-dev-tasks
description: >-
  Design and build engine9 flow definitions (JSON5), TaskWorker lifecycle, and
  SQLTaskManager execution. For engine9 developers only — do not use when calling
  the Task API as an integrator or end user.
disable-model-invocation: true
---

# engine9 Dev Tasks (flow authoring)

**Developer skill only.** Do not load this skill unless you are in a development context — authoring JSON5 flows, working in `engine9/server`, debugging TaskWorker/SQLTaskManager, or extending the task execution pipeline.

Normal engine9 users and API integrators should use [e9-tasks-api](../e9-tasks-api/SKILL.md) to **schedule** work via REST: a predefined flow (`flow_id` on `POST /flow_runs/`) or an on-demand task (`path` + `method` on `POST /tasks/schedule`, e.g. `@engine9/plugins/e9workers:EchoWorker` + `echo`). They do not author flow definitions through the API.

## When to use this skill

- Authoring or editing `.json5` flow files
- Debugging `TaskWorker.createFlowRun`, `runFlow`, `executeTaskRun`, or `scheduleTasks`
- Configuring or troubleshooting `SQLTaskManager` polling
- Adding SQL, worker, or ETL steps to multi-step account workflows
- Working with sample flows in `engine9/server/test/task/*.json5`

## When NOT to use this skill

- Calling the Task API from curl, scripts, or external integrations → [e9-tasks-api](../e9-tasks-api/SKILL.md)
- Scheduling plugin jobs via MCP → [e9-mcp](../e9-mcp/SKILL.md) / [e9-cli](../e9-cli/SKILL.md)
- Deploying or operating the Task API → `engine9/server/api/task/docs/admin/`

## Documentation

| Doc | Use when |
|-----|----------|
| [flow-authoring.md](./flow-authoring.md) | JSON5 format, TaskWorker, SQLTaskManager, MCP `task` with `flow_path` |
