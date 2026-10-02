---
name: e9-dev-tasks
description: >-
  Design, author, and debug engine9 JSON5 flow definitions, TaskWorker lifecycle
  methods, and SQLTaskManager execution. Use only in engine9 development contexts
  that build multi-step workflows or extend the task execution pipeline; API
  integrators and operators should use the Task API documentation instead.
disable-model-invocation: true
---

# Develop engine9 tasks and flows

This developer-only skill covers JSON5 flow authoring, TaskWorker lifecycle behavior, and SQLTaskManager execution. Use it while developing in `engine9/server`, adding SQL, worker, or ETL steps, or debugging the task pipeline. Do not use it merely to schedule existing work through REST or MCP.

## Quick reference

| Goal | Use |
| --- | --- |
| Author or edit a flow | A `.json5` flow definition |
| Debug flow creation or execution | `TaskWorker.createFlowRun`, `runFlow`, or `executeTaskRun` |
| Schedule tasks internally | `TaskWorker.scheduleTasks` |
| Debug polling and task claims | `SQLTaskManager` |
| Study working definitions | `engine9/server/test/task/*.json5` |
| Schedule through REST | [e9-tasks-api](../e9-tasks-api/SKILL.md) |
| Schedule through MCP | [e9-mcp](../e9-mcp/SKILL.md) or [e9-cli](../e9-cli/SKILL.md) |

## Concepts

A flow is a predefined, multi-step workflow represented as JSON5. TaskWorker creates flow runs, schedules and executes task runs, and merges remote execution state. SQLTaskManager polls SQL-backed task state and coordinates execution.

REST integrators schedule a predefined flow by sending `flow_id` to `POST /flow_runs/`, or an on-demand worker method by sending `path` and `method` to `POST /tasks/schedule`. They do not author flow definitions through the API.

## File format

Flow definitions are `.json5` files. Every task requires identifiers that remain unique within the flow and labels that remain unambiguous during remote-state merging.

| Field | Requirement |
| --- | --- |
| `task_key` | Unique for every task in the flow |
| `name` | Unique for every task in the flow |
| Worker method | May repeat only when `task_key` and `name` remain distinct |

## Workflow

1. Start from a sample in `engine9/server/test/task/*.json5`.
2. Define each SQL, worker, or ETL step.
3. Assign a unique `task_key` and unique `name` to every step.
4. For a flow bound to one account, load that account's installed plugins and put each step's `plugin_id` and `remote_plugin_id` (`assignee.bot_id`) on the task. If the plugins cannot be loaded, stop and do not write the flow.
5. Exercise flow creation through `TaskWorker.createFlowRun`.
6. Trace scheduling and execution through `runFlow`, `scheduleTasks`, and `executeTaskRun`.
7. Inspect SQLTaskManager polling when tasks are not claimed or advanced.
8. Use the Task API or MCP documentation only when testing an external scheduling boundary.

## Rules

**Rule:** Use this skill only for engine9 development: flow authoring, TaskWorker or SQLTaskManager debugging, and task-pipeline extension.

**Rule:** Every step must have a unique `task_key` and a unique `name`.

**Rule:** Do not reuse a worker method such as `echo` or `query` as the `name` of multiple tasks. Remote merge indexes by `task_key`, `context_id`, and `label` or `name`; duplicate names collide.

**Rule:** API consumers schedule existing flows and methods; they do not author JSON5 flow definitions through the Task API.

**Rule:** A flow for a specific account must carry that account's plugin identifiers on every plugin step. `plugin_id` is the installed plugin row id. `assignee.bot_id` is that row's `remote_plugin_id`. Load both from the account's installed plugins before writing the file. A package path is not a substitute. If those plugins cannot be loaded, stop and do not write the flow. A shared flow that is not bound to one account (for example `identity-rebuild`) names the package path; those ids differ per account.

## Examples

An API integrator may schedule the built-in echo worker by using:

```text
path: @engine9/plugins/e9workers:EchoWorker
method: echo
```

That request belongs to the Task API or MCP documentation. The flow definition and its unique task names belong to this developer skill.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| One task overwrites another during merge | Duplicate `task_key`, `name`, or label |
| Flow run is created but does not advance | `runFlow`, scheduled task state, and SQLTaskManager polling |
| Worker task is never executed | `path`, `method`, task scheduling, and `executeTaskRun` |
| Account flow has plugin paths and no plugin ids | Load that account's plugins and set `plugin_id` and `assignee.bot_id` before writing the file |
| Unsure whether to use this skill | Use it only when editing or debugging implementation internals |

## Related documentation

- [Flow authoring reference](flow-authoring.md)
- [Task API for REST integrators](../e9-tasks-api/SKILL.md)
- [engine9 MCP task scheduling](../e9-mcp/SKILL.md)
- [engine9 CLI task scheduling](../e9-cli/SKILL.md)
- `engine9/server/api/task/docs/admin/`
- `engine9/server/test/task/*.json5`
