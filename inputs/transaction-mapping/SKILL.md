---
name: transaction-mapping
description: >-
  Write JavaScript mapping functions that convert third-party payment,
  subscription, donation, API, or CSV records into the standard engine9
  Transaction schema. Use when normalizing source fields, classifying transaction
  entry types, preserving idempotency, handling refunds, or validating mapped rows
  before the transaction inbound pipeline.
---

# Map transactions into engine9

This skill covers pure JavaScript mappers from third-party payment records to the standard engine9 Transaction shape. Use it for payment platforms, donor databases, API responses, and CSV exports. The canonical implementation lives in `interfaces/transaction/core/schema.js`, and mapped rows are processed by `interfaces/transaction/core/transforms/inbound/upsert_tables.js`.

## Quick reference

| Need | Guidance |
| --- | --- |
| Mapper signature | `(rawRow) => transactionRow` |
| Required values | `ts`, `amount`, `remote_person_id`, and `entry_type` or `entry_type_id` |
| Stable source key | `remote_transaction_id` |
| Recurring-series key | `remote_recurring_id` |
| Refund classification | `TRANSACTION_REFUND` with `refund_amount` |
| Pipeline-derived fields | `id`, `entry_type_id`, `input_id`, `person_id`, `recurs_id` |
| Full column reference | [reference.md](reference.md) |

## Concepts

The mapper normalizes one source payment or logical payment into one Transaction-shaped object. It does not perform person resolution, timeline UUID assignment, input assignment, or database upserts. Those steps belong to the inbound transform.

`remote_person_id` is the payer or donor identifier in the source system; the pipeline resolves it to the internal `person_id`. `remote_transaction_id` is the preferred source idempotency key and participates in upsert and UUID behavior.

## File format

### Required fields

| Field | Type | Notes |
| --- | --- | --- |
| `ts` | Date-compatible | ISO string or milliseconds accepted by `new Date(ts)` |
| `amount` | number | Amount in currency units |
| `remote_person_id` | string | Source-system payer or donor identifier |
| `entry_type` | string | Use a supported transaction type, or provide `entry_type_id` |

### Optional fields

| Field | Type | Notes |
| --- | --- | --- |
| `refund_amount` | number | Refund amount, normally with `TRANSACTION_REFUND` |
| `remote_transaction_id` | string | Stable source idempotency key |
| `remote_page_name` | string | Campaign or form name used to derive `input_id` |
| `remote_input_id` | string | Source input identity used to derive `input_id` |
| `remote_recurring_id` | string | External subscription or recurring-series ID |
| `recurs` | string | `daily`, `weekly`, `monthly`, `quarterly`, `annually`, or `semi-annually` |
| `recurring_number` | number | Payment occurrence within a recurring series |
| `given_name`, `family_name`, `email` | string | Person matching or display data |
| `source_code_id` | number | Source attribution |
| `override_source_code_id` | number | Explicit source attribution override |
| `final_source_code_id` | number | Final source attribution |
| `recommended_message_id` | UUID | Recommended message attribution |
| `override_message_id` | UUID | Explicit message attribution override |
| `final_message_id` | UUID | Final message attribution |
| `extra` | object | Vendor-specific data outside the standard schema |
| `remote_entry_uuid` | UUID | Stable source UUID used as record ID |

### Transaction entry types

| `entry_type` | `entry_type_id` | Use when |
| --- | ---: | --- |
| `TRANSACTION` | 10 | Generic transaction; prefer a specific type when known |
| `TRANSACTION_ONE_TIME` | 11 | Single, non-recurring payment |
| `TRANSACTION_INITIAL` | 12 | First payment in a recurring series |
| `TRANSACTION_SUBSEQUENT` | 13 | Later payment in a recurring series |
| `TRANSACTION_RECURRING` | 14 | Recurring payment with unknown order |
| `TRANSACTION_REFUND` | 15 | Refund; set `refund_amount` and optionally link the original |

Import `TIMELINE_ENTRY_TYPES` from `@engine9/input-tools` for the authoritative map.

## Workflow

1. Inspect the source fields for amount, timestamp, source person identity, transaction ID, recurring status, refunds, and attribution.
2. Implement a pure `(rawRow) => transactionRow` mapping function.
3. Return one object for each source payment or logical payment.
4. Classify the row with `entry_type`, preferring the most specific known transaction type.
5. Preserve stable source IDs for idempotency and recurring-series identity.
6. Validate every sample for `ts`, `amount`, `remote_person_id`, and `entry_type` or `entry_type_id`.
7. Pass mapped rows to the standard transaction inbound transform.

For a separate source refund row, map it independently with `TRANSACTION_REFUND`, `refund_amount`, and its own timestamp. If the source exposes only a refund flag, one row may contain `amount` and optional `refund_amount`; use `TRANSACTION_REFUND` when product rules classify it as a refund.

## Rules

**Rule:** Return one Transaction-shaped object per source payment or logical payment.

**Rule:** Every mapped object must contain `ts`, `amount`, `remote_person_id`, and either `entry_type` or `entry_type_id`.

**Rule:** Do not set `id`, `entry_type_id`, `person_id`, `input_id`, or `recurs_id` in the mapper. Supply string `entry_type`; the inbound pipeline derives those fields.

**Rule:** Use string `entry_type` in the mapper and let the inbound transform derive `entry_type_id`.

**Rule:** Normalize timestamps to ISO strings or millisecond values parseable by `new Date(ts)`.

**Rule:** Use the source payer or donor identifier as `remote_person_id`; do not substitute the internal `person_id`.

**Rule:** Set `remote_transaction_id` whenever the source provides a stable transaction ID.

**Rule:** Do not set `input_id`; the pipeline derives it from `remote_page_name` or `remote_input_id`.

## Examples

### Mapping function

```javascript
/**
 * Map one source payment into the Transaction schema.
 *
 * Required output: ts, amount, remote_person_id, and entry_type.
 */
function mapPaymentToTransaction(row) {
  return {
    ts: row.created_at ?? row.date ?? row.timestamp,
    amount: Number.parseFloat(row.amount ?? row.total ?? 0),
    remote_person_id:
      row.donor_id ?? row.customer_id ?? row.external_id,
    entry_type: row.recurring
      ? "TRANSACTION_SUBSEQUENT"
      : "TRANSACTION_ONE_TIME",
    remote_transaction_id: row.id ?? row.transaction_id,
    remote_page_name: row.campaign ?? row.form_name ?? null,
    email: row.email ?? null,
    given_name: row.first_name ?? null,
    family_name: row.last_name ?? null,
    remote_recurring_id: row.subscription_id ?? null,
    recurs: row.interval === "month" ? "monthly" : null,
    recurring_number: row.occurrence ?? null,
    extra: row.raw ? { raw: row.raw } : undefined,
  };
}
```

### Separate refund row

```javascript
function mapRefund(row) {
  return {
    ts: row.refunded_at,
    amount: 0,
    refund_amount: Number.parseFloat(row.refund_amount),
    remote_person_id: row.customer_id,
    remote_transaction_id: row.refund_id,
    entry_type: "TRANSACTION_REFUND",
    extra: {
      original_transaction_id: row.original_transaction_id,
    },
  };
}
```

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Pipeline throws on the first mapped row | Required fields and timestamp parsing |
| Person resolution fails | `remote_person_id` must be the source-system identity |
| Duplicate payments appear | Preserve stable `remote_transaction_id` |
| Recurring payments are misclassified | Distinguish initial, subsequent, and unknown-order recurring rows |
| Input assignment is missing | Supply `remote_page_name` or `remote_input_id`; do not set `input_id` |
| Refund totals are incorrect | Confirm `TRANSACTION_REFUND`, `refund_amount`, and product-specific `amount` behavior |
| Entry type is rejected | Use `TIMELINE_ENTRY_TYPES` and a supported transaction type |

## Related documentation

- [Input files and `remote_input_id` grain](../../e9-input/SKILL.md)
- [Complete transaction column reference](reference.md)
- [Canonical Transaction schema](https://frakture.notion.site/Frakture-Transactions-Data-442349ac436a4f7db8e7d732359e7d8f)
- `interfaces/transaction/core/schema.js`
- `interfaces/transaction/core/transforms/inbound/upsert_tables.js`
- `interfaces/transaction/profile/schema.js`
- `input-tools/timelineTypes.js`
- `TIMELINE_ENTRY_TYPES`
