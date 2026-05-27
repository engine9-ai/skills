---
name: e9a
description: Set engine9 account scope directly from an exact account id for subsequent /e9 requests.
---

# Engine9 Account Selector

Use this skill to set account scope for later engine9 requests.

Primary command form:

- `/e9a <lookup>`

Examples:

- `/e9a test`
- `/e9a engine9`
- `/e9a authentic_amy_mcgrath`

## Goal

Treat `<lookup>` as the exact `accountId` and persist it immediately for all subsequent `/e9` requests.

## Required behavior

1. Do not call `user` (or any account lookup tool) to resolve `<lookup>`.
2. Do not perform fuzzy matching, alias matching, or ambiguity checks.
3. Set the provided value as active account scope immediately.
4. Reuse this stored account for subsequent `/e9` calls unless the user overrides it.

## Input parsing for `/e9a <lookup>`

- Treat `<lookup>` as required text.
- Trim leading/trailing whitespace.
- Use the resulting string directly as `accountId`.

## Session persistence contract

Persist:

- `engine9.accountId`: exact lookup value
- `engine9.accountIds`: single-item array with that same value

Rules:

- For `/e9a <lookup>`, always replace prior account scope with the new value.
- Subsequent `/e9 ...` requests should use `engine9.accountId` by default.

## Response requirements

On success:

- Confirm the exact `accountId` that was set.
- State that future `/e9` requests will use it.

## Suggested user-facing success message

`Active accountId set to authentic_amy_mcgrath for subsequent /e9 requests.`

