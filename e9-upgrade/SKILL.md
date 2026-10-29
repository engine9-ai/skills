---
name: e9-upgrade
description: >-
  Upgrade engine9 databases after a new @engine9/core or @engine9/schemas
  release: Cloudflare sites on D1, Node sites on SQLite or MySQL, and accounts
  on an engine9 server. One mechanism everywhere: install, deploy, then
  `e9core upgrade` (PluginWorker.upgrade) with a dry run first. Covers
  POST /api/admin/upgrade, MCP `plugin` `upgrade`, plugin-declared upgrades,
  upgrade windows, blocked and opt-in upgrades (--drop-name-columns,
  --discard-names, --drop-plugin-history), the audit record of each run, release guides
  such as 1.10.1 person names moving to person_name, rollback, and why
  wrangler d1 migrations are not used for engine9 tables. Use when upgrading,
  migrating, or rolling out a core or schemas version, or when columns,
  tables, or inbound snapshots are out of date after a deploy.
---

# engine9 upgrades

A deploy changes code, never the database. After a new `@engine9/core` or `@engine9/schemas` reaches a host, every engine9 database needs `e9core upgrade` (`PluginWorker.upgrade`). It runs the data upgrades the `@engine9/schemas` plugins declare, then reinstalls the installed schema plugins so new tables, columns, and inbound snapshots are deployed. Use this skill for any version rollout, on core sites or on engine9 server accounts.

## Quick reference

| Host | Dry run | Upgrade |
| --- | --- | --- |
| Cloudflare site (D1) | `npx e9core upgrade --url https://<site> --dry-run` | `npx e9core upgrade --url https://<site>` |
| Local `wrangler dev` | `npx e9core upgrade --url http://localhost:8787 --dry-run` | same without `--dry-run` |
| Node site, local file | `npx e9core upgrade --db <connection> --dry-run` | same without `--dry-run` |
| engine9 server account (MCP) | `plugin` `{ "command": "upgrade", "account_id": "<account_id>", "dry_run": true }` | `plugin` `{ "command": "upgrade", "account_id": "<account_id>" }` |

| Release | Guide |
| --- | --- |
| 1.17.0 | Upgrades move out of core into the plugins (`@engine9/schemas` 1.13.0); upgrade windows; `plugin_history` retired with an opt-in drop (`--drop-plugin-history`) |
| 1.11.2 | `setting.value` widens from `varchar(255)` to `text` so JSON plugin settings are not truncated |
| 1.10.1 | Person names are copied from `person` to `person_name`; the `person` columns stay until the opt-in `--drop-name-columns` |
| 1.9.1 | Package rename `@engine9/interfaces` → `@engine9/schemas` (manual runbook, predates `e9core upgrade`) |

Release guides live in `@engine9/core` under `docs/upgrade-<version>.md`. The process is `docs/upgrading.md`.

**Rule:** Deploy the new code first, then upgrade. Old code expects old tables. New code keeps working on a database that is not upgraded yet.

**Rule:** Always dry run first. Never pass `--discard-names` (or `discard_names`) unless the database owner chose it.

**Rule:** Never pass `--drop-name-columns` (or `drop_name_columns`) as part of a routine upgrade. It removes columns site code may still read; run it only when the owner confirms nothing reads them.

## Concepts

### What one upgrade run does

| Step | What happens |
| --- | --- |
| Window | A database last upgraded with a core older than the release supports is refused before anything runs (`blocked`, with the release to upgrade through first) |
| Plugin upgrades | Core carries none. Each `@engine9/schemas` plugin declares its own; `shouldUpgrade` checks the schema, and only an upgrade that applies reads the data, does nothing if the account is current, otherwise moves it (for example copies names into `person_name` and verifies). Safe to re-run; a re-run finishes a partial run. Other plugins' upgrades run only with `--plugin <path>` |
| Opt-in upgrades | A removal (for example dropping `person.given_name`) runs only with its switch (`drop_name_columns`). Without it the run reports `opt_in` and changes nothing |
| Refresh | Reinstalls every installed `@engine9/schemas/*` plugin from the deployed package. Adds tables and columns (never drops) and refreshes inbound snapshots. Stacks first, so a stack that gained a plugin installs it |
| Record | Nothing is kept in the database: every run asks every upgrade again. A real run writes one `upgrade` record to the audit log (what it applied, as `{ plugin, id }`, what it reinstalled, who ran it). `upgrade.<id>` rows in `setting` are left over from core 1.10.1–1.16; nothing reads them |

### Reading the result

| Field | Meaning |
| --- | --- |
| `ok` | `false` when an upgrade is blocked or a plugin failed to reinstall. The CLI exits 1 |
| `schemas_version` | The `@engine9/schemas` version the host is running |
| `upgrades[].status` | `pending` (dry run, work to do), `applied`, `current`, `blocked`, or `opt_in` (a removal that runs only with the switch named in `opt_in`) |
| `upgrades[].plugin` | The plugin that declares it. Upgrades are listed by plugin (`@engine9/schemas/plugin` first), so find one by `id`, not by position |
| `upgrades[].plan` | What the upgrade does or did, in order |
| `window` | The core upgrade window: `ok`, `unrecorded` (no install records yet; the run records them), or `too_old` |
| `unused_options` | A flag no upgrade reads, usually a misspelling |
| `refresh.plugins` | Paths reinstalled, or that would be reinstalled on a dry run |
| `refresh.failed` | `{ path, error }` for each plugin that did not reinstall, usually a path no longer in the package |

### Why not `wrangler d1 migrations`

engine9 tables carry a `modified_at` trigger that `wrangler d1 migrations apply` cannot submit (`SQLITE_ERROR [code: 7500]`). The same upgrade must also run on SQLite and MySQL. Upgrades decide per account (PII-free or not, what is installed). Inbound snapshots come from plugin code, not SQL. Project-owned tables may still use D1 migrations.

**Rule:** Do not write `migrations/` files for engine9 tables, and do not run `wrangler d1 migrations apply` on engine9 schema SQL.

## Workflow

### Core sites (Cloudflare or Node)

Copy this checklist per database:

```
- [ ] Inventory: project dir, host, site URL, D1 database / connection, wrangler env, admin key present
- [ ] Read the release guide; search site code for what it moves (1.10.1: given_name / family_name on person)
- [ ] Back up (wrangler d1 time-travel info --remote; keep the bookmark) or dump MySQL
- [ ] npm install @engine9/core@^<v> @engine9/schemas@^<v>; build; deploy
- [ ] e9core upgrade --dry-run; resolve anything blocked with the owner
- [ ] e9core upgrade; expect ok: true
- [ ] Verify with the release guide's queries; re-run shows status: current
- [ ] Repeat for the local dev database
```

`--url` uses `E9_ADMIN_API_KEY` from `--key`, the environment, or `./.env`. The route is `POST /api/admin/upgrade` and needs the `admin` scope. It runs inside the Worker against the D1 binding, which no outside process can open.

In CI, chain it to the deploy:

```bash
wrangler deploy && npx e9core upgrade --url "$E9_SITE_URL"
```

### engine9 server accounts (MCP)

```
- [ ] mcp_auth → user
- [ ] Confirm the server already runs the new release (staff deploy it)
- [ ] Resolve account ids (one id, or account search by parent)
- [ ] plugin upgrade dry_run: true per account; collect status and blocked
- [ ] Ask about every blocked account; do not discard data by default
- [ ] plugin upgrade per account (small parallel batches)
- [ ] Re-run dry_run; every upgrade reports current (or opt_in) and ok is true
```

**Rule:** `plugin` `upgrade` needs account admin. It is one call per `account_id`; there is no bulk upgrade RPC. Do not schedule `PluginWorker.upgrade` through MCP `task`.

## Rules

**Rule:** Upgrade every database the release touches: production, every wrangler environment, and local development.

**Rule:** A `blocked` upgrade needs a decision before it can run. For 1.10.1:

- `person_is_view: true`: `person` is a view over an older table. Replace the view with a `person` table first, then re-run.
- `person_plugin_installed: false`: the `person` table exists without its `@engine9/schemas/person` plugin row. Install the account's stack, then re-run.
- With `--drop-name-columns` on a PII-free account that still has names on `person`: ask the data owner. Either the account should not be PII-free (install the standard stack, then upgrade), or the names are discarded (`--discard-names`).

**Rule:** After an opt-in drop, roll back the database and the code together (`wrangler d1 time-travel restore` plus `wrangler rollback`). Rolling back only the Worker leaves old code on new tables.

**Rule:** Skills and reports name no specific clients. Use `<account_id>`, `<parent_account_id>`, and `<site>`.

## Examples

```bash
npx e9core upgrade --url https://www.example.com --dry-run
```

```json
{
  "ok": true,
  "dry_run": true,
  "schemas_version": "1.13.0",
  "window": { "status": "ok", "from": "1.9.0", "last": "1.17.0" },
  "upgrades": [
    {
      "id": "person_name",
      "plugin": "@engine9/schemas/person",
      "since": "1.10.1",
      "status": "pending",
      "pii_free": false,
      "people_with_names": 1834,
      "plan": [
        "install @engine9/schemas/person_name",
        "copy names for 1834 people into person_name",
        "refresh the @engine9/schemas/person inbound snapshot"
      ]
    },
    {
      "id": "person_name_columns",
      "plugin": "@engine9/schemas/person",
      "since": "1.10.1",
      "opt_in": "drop_name_columns",
      "status": "opt_in",
      "plan": [
        "refresh the @engine9/schemas/person inbound snapshot",
        "drop person.given_name",
        "drop person.family_name"
      ]
    }
  ],
  "refresh": { "dry_run": true, "plugins": ["@engine9/schemas/stacks/standard", "@engine9/schemas/person"], "failed": [] }
}
```

```json
{ "command": "upgrade", "account_id": "<account_id>", "dry_run": true }
```

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `answered 403` from `--url` | Key lacks `admin` | Use `E9_ADMIN_API_KEY`, not the public key |
| `answered 404` from `--url` | Site still runs an older core | Deploy the new build first |
| `status: blocked` | A choice that loses data | Ask the owner (see Rules) |
| `refresh.failed` with `PLUGIN_NOT_FOUND` | A plugin row names a path the package no longer has | Report it; do not delete plugin rows yourself |
| Site shows old names after 1.10.1 | Site code still reads `person.given_name`, which stopped changing at the upgrade | Join `person_name` on `person_id` |
| Site errors on `given_name` after `--drop-name-columns` | Site code still reads `person.given_name` | Join `person_name` on `person_id` |
| New columns missing after a deploy | Upgrade not run | Run `e9core upgrade` |
| `blocked`: last upgraded with an older core, or a plugin outside its upgrade window | The database is older than this release upgrades | Upgrade through the release the message names first, then this one |
| No upgrades run after installing core 1.17.0 | `@engine9/schemas` is below 1.13.0, which declares them | Install `@engine9/schemas@^1.13.0` |

## Related documentation

- [Plugin install](../e9-plugin/SKILL.md)
- [MCP](../e9-mcp/SKILL.md)
- [Person identity](../e9-person-id/SKILL.md)
- `@engine9/core` `docs/upgrading.md`, `docs/plugin-upgrades.md`, and `docs/upgrade-<version>.md`
