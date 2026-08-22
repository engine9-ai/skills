---
name: e9-troubleshoot
description: >-
  Support-ticket diagnosis for people working WITH the engine9 product
  (support staff and technical clients), not for developers BUILDING it.
  Ingests a ticket, diagnoses from product surfaces and MCP, and recommends
  resolutions. Does not assume repository access. SQL is secondary. Use ONLY
  when the user explicitly writes troubleshoot or /troubleshoot. Do not use
  for implementation, tests, refactors, or ordinary development.
---

# engine9 Troubleshoot

This skill is the line between **working with the product** and **building the product**.

| Audience | This skill |
|----------|------------|
| Support staff and clients (may be technical) filing or working a ticket | **Yes** — ingest, diagnose, recommend a resolution |
| Backend developers implementing, fixing, testing, or refactoring code | **No** — leave this skill unread |

**Load this skill only when the user wrote `troubleshoot` or `/troubleshoot`.** If it was pulled in for a coding task, a failing test, or a generic “debug this,” stop following it.

Do not open workspace source. Many users have no repo. A confirmed product defect becomes an **engineering handoff**, not a code dive.

**MCP:** Follow [e9-mcp](../e9-mcp/SKILL.md) for login, account scope, tool choice, and hard-stop on MCP errors. Use `/e9a` when account scope is unclear ([e9-cli](../e9-cli/SKILL.md)).

## Hard rules

1. **Trigger is the word `troubleshoot`.** Nothing else starts this workflow.
2. **No code.** Do not read local workers, plugins, interfaces, or tests. The connected product (console + MCP) is the system of record.
3. **Ingest the ticket before querying.** A missing example person, message, source code, URL, or time window is a ticket-intake gap — not a reason to scan tables.
4. **Product surfaces first, SQL last.** Prefer `account`, `search`, `analyze`, `segment`, `plugin_id` / `input_id`, and `task` (loads / runs). SQL and EQL confirm a *specific* hypothesis. They are a poor first move: schema fishing becomes the investigation and hides config, auth, load, and remote-system causes.
5. **Account resolution uses MCP `account`, never SQL fan-out.** Installed plugins come from `account`. Do not loop `sql` across accounts to find installs.
6. **One primary account** before deep diagnosis, even if several are mentioned.
7. **Be wary of scale.** No full-table `COUNT(*)`, unbounded scans, or expensive aggregates unless filtered and limited.
8. **Recommend a resolution the ticket owner can act on** (reconnect, rerun a load, recode a link, check the vendor, wait, or escalate). Do not end on “more SQL.”

## Progress checklist

```
Troubleshoot:
- [ ] Ticket ingested (example + expected vs actual + URL if seen in UI)
- [ ] e9 MCP connected (if account data is needed)
- [ ] Primary account + installed plugins (MCP `account`)
- [ ] Category
- [ ] Diagnosed from product surfaces (stop at first gap)
- [ ] Other accounts? Remote vs in-product vs expected behavior?
- [ ] Recommended resolution (or engineering handoff)
```

---

## 1 — Ingest the ticket

Capture enough to work **one example**. Ask for what is missing; do not invent ids.

| Field | Why |
|-------|-----|
| Reporter / client vs internal | Tone and what they can change |
| `account_id` (or org / name) | Scope |
| What they see vs what they expect | Symptom |
| When it started / last time it worked | Load vs regression vs never-configured |
| Console URL if they “see” it | UI vs data |
| One example | Person (email/phone/id), message, source code, transaction, segment, or input — not “all of them” |

State the category as soon as the report allows it:

| Category | Typical ticket |
|----------|----------------|
| **Plugins / inputs / ingestion** | Data not arriving, load failed, plugin disconnected, “we sent it but engine9 doesn’t have it” |
| **People** | Duplicate or missing person, wrong match on email/phone/remote id |
| **Timeline / engagement** | Blank Timeline tab, missing open/click/send, opener segment empty |
| **Source coding / attribution** | Revenue on the wrong message, coded link not credited |
| **Transactions** | Missing or wrong payment rows (not attribution) |
| **Messaging** | Message list / content / send stats, not timeline events |
| **Segments** | Audience wrong after events exist |
| **UI / console** | Button, blank page, layout — data underneath looks fine |
| **Auth / access** | Cannot sign in, cannot see an account |
| **Other** | Say so |

Wrong `person_id` is **People**, not Timeline. Revenue on the wrong message is **Source coding / attribution**, not Transactions.

---

## 2 — Connect MCP (when the ticket needs account data)

Needed for plugins, people search, load status, or in-account evidence. Not required to ask intake questions or to conclude “check the live email link / payment form” from the ticket text alone.

1. Use the engine9 MCP server in this session. If several e9 servers are installed, ask which environment. If none, ask them to add/reload one ([e9-cli](../e9-cli/SKILL.md)) and continue intake while they do.
2. `mcp_auth` → user completes sign-in → `ok` then `user` ([e9-mcp Step 0](../e9-mcp/SKILL.md#step-0--log-in-always-first)).
3. Record the server name on the ticket.

Do not run account-scoped tools until this succeeds.

---

## 3 — Identify the account (MCP `account` only)

Pick **one** primary `account_id`. Keep other mentioned accounts for scoping.

| Situation | Call |
|-----------|------|
| Id known | `{ "account_id": "<id>" }` → installed plugins |
| Id unknown | **One** `account` `command: "search"` — combine `prefixes`, `parents`, `name`, `tags`, `plugins` |

```json
{ "command": "search", "prefixes": ["authentic"], "plugins": ["acoustic"] }
```

Cache the plugins list. Do not re-query on every later call. Do not `sql` / `SHOW TABLES` across accounts to find installs. Do not probe every accessible account when a prefix/parent/name would narrow it.

---

## 4 — Diagnose from the product, not from the schema

Work **the example from the ticket**. Stop at the first gap. Domain pipelines (what “should” have happened) live in the linked skills — use them as checklists, not as permission to dump tables.

### Tool order

| First | Then | Only if a named hypothesis remains |
|-------|------|------------------------------------|
| Ticket URL + one example | `account` (is the plugin installed? auth fields present?) | `eql` / `sql` for **that** person, code, message, or input |
| `search` (email / phone / person_id) | `analyze` on a **named** table (profile, not a fishing trip) | |
| `segment` `list` if the complaint is an audience | `task` for load / flow-run status (`list` / `listTasks`) | |
| `input_id` / `plugin_id` when the remote ids are known | | |

**Do not** start with `sql` `tables` / `describe` / `indexes` to “see what’s there.” **Do not** write exploratory `SELECT`s to discover the category — the ticket already has one.

### Plugins / inputs / ingestion

Most tickets die here.

1. Is the integration **installed** on this account? (`account` plugins)
2. Does it look **authenticated** (auth fields present, not a failed reconnect)?
3. Did a **load run**, and what state is it in? (`task` list / listTasks — failed, running, never scheduled)
4. Does the **remote system** actually have the file/event/row? If the vendor never recorded it, engine9 will not invent it.

**Resolutions:** install or reconnect the plugin; schedule or retry the load; wait for a run in progress; send the client back to the ESP/CRM/payment vendor; escalate only if a run failed in-product with no config/auth explanation.

### People

Use MCP `search` on the example email / phone / id. Conceptual model: [e9-person-id](../e9-person-id/SKILL.md) (lookup vs attributes, first-wins). Do not start in `person_identifier` SQL.

**Resolutions:** load/reload the people extract; confirm the identifier exists on the remote record; explain first-wins / one email on two people; escalate only if search shows a match the product UI contradicts.

### Timeline / engagement

One person + one expected event. Checklist: [e9-timeline missing-event A–F](../e9-timeline/SKILL.md#missing-event-debug-af) — but **A/B via `search` + plugins/inputs/loads**, not a `timeline` dump.

Person missing → People. Input never loaded → Plugins / inputs. Vendor also lacks the event → remote. Sends without opens is often real behavior.

**Resolutions:** load the engagement stream; check segment window (e.g. 90-day opener defs vs an old `publish_date`); tell the client the vendor has no event; escalate only if the event exists in-product and the Timeline tab / segment still hides it.

### Source coding / attribution

One source code (and one transaction / message if known). Checklist: [e9-source-code last-click A–F](../e9-source-code/SKILL.md#last-click-pipeline-debug-af). **A and B are live links and the payment platform — not engine9.** C–F are “did we load the message, load the transaction, attribute, roll up stats?”

Two remotes: messaging platform (link coding) and payment platform (code stored on the transaction).

**Resolutions:** recode the outbound link (recognized param, exact string); fix the form/UTM drop on the payment side; load messages or transactions; run/wait for attribution stats; escalate only if the same code is on the message **and** the transaction in-product and the report still disagrees.

### Segments / UI / auth

- **Segment:** `segment` list; confirm universe and date window after the underlying events exist.
- **UI:** keep the URL. If the same example looks correct via `search` / `analyze`, the resolution is a UI handoff — not more warehouse queries.
- **Auth:** `user` access map; they may not have the account.

---

## 5 — SQL is secondary

Use `eql` or `sql` only after a product-surface gap needs a **yes/no** on one already-named row (this `person_id`, this `source_code`, this `input`). Prefer `analyze` for “does this table have recent data?” Prefer the A–F probes in the domain skills over invented joins.

Never use SQL to:

- Discover accounts or installed plugins
- Decide the ticket category
- Browse the warehouse because the example is incomplete

If you do query: filter, `LIMIT`, time-bound. No full-table counts.

---

## 6 — Scope and remote vs in-product

**Other accounts.** Reuse the `account` search candidate list. Ask whether others are affected. Light-check a second named account only when useful — never blast aggregates across a parent tree.

Record: single-account / multi-account / unknown.

**Where it breaks.**

| Finding | Means |
|---------|--------|
| Plugin missing, auth dead, load never ran / failed | In-product, usually support-fixable |
| Vendor / live link / payment form lacks the fact | Remote — client or vendor action |
| In-product data is correct; console disagrees | UI — engineering handoff |
| In-product data is wrong after a successful load of good remote data | Engineering handoff |
| Matches documented window, first-wins, or “vendor has no event” | Expected behavior — explain, do not escalate |

For attribution, A/B gaps are remote; C–F are in-product. For timeline, vendor-missing is remote; no input is ingestion; no person is People.

---

## 7 — Recommended resolution (the deliverable)

Write so another support person can act without redoing discovery.

~~~~
# Troubleshoot

## Ticket
[Reporter] — [one-sentence symptom]

## Example
[person / message / source_code / transaction / segment / URL]

## Account
- account_id: …
- MCP server: …
- Relevant plugins: … (from `account`, not SQL)
- Other accounts: single / multi / unknown

## Category
[Plugins/inputs | People | Timeline | Source coding / attribution | …]

## Diagnosis
- Recreated? yes / partial / no
- First gap: [plugin | auth | load | remote | identity | coding | attribution stats | segment window | UI | expected behavior]
- Evidence: [search / account / task / analyze — SQL only if used, and why]

## Recommended resolution
1. [Action the ticket owner can take now]
2. …

## Escalate to engineering?
- [ ] No — support/client/vendor action above
- [ ] Yes — in-product defect after a good load / UI disagrees with confirmed data
  Need: account_id, example, first gap, evidence, URLs
~~~~

**Escalate** means this package, not opening the repo. If the user later wants a developer to change code, that is a different session — not this skill.
