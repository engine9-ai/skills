# Attribution and conversion tracking

Read [SKILL.md](SKILL.md) first for last-click attribution vs the origin model, link parameters, and warehouse tables.

## Conversion methods

Conversion tracking is a prerequisite for attribution. Same ideas apply to non-monetary conversions (signups, petitions); this skill talks about monetary **transactions**.

### Native platform tracking

An integrated email + payment stack follows a person from click to completed transaction (cookies, URL params, or JS). Convenient on one platform; weak across platforms (e.g. emails and ads driving a third-party payment page). At best an **estimate**: thank-you refreshes, cookie blocks, JS off, and flaky networks under- or over-count.

### Tag and pixel

A pixel on the post-transaction page (Google Tag Manager, Meta Pixel, etc.). Fine when the ad platform is also the tracker (ROI next to the ad). Weaknesses:

1. Estimate — same cookie/JS failure modes as native.
2. Third-party privacy / ATT-style tracking loss.
3. Web-only — mail, OTT, wallet pay, offline are poorly covered.
4. Labor — JS on every property; consultants and locked platforms rot the install.
5. Vendor APIs and pixel names keep changing.

### Source-code tracking (preferred)

Attach a unique string to the outbound message and carry it through the conversion (UTM, origin codes, tracking codes — same idea). Example: `DM_2020_EOQ2_Water_FNEIV2`.

Pros:

- **Not an estimate.** Every transaction has a code, including blank (white mail). Bank totals can be audited against inbound codes.
- **Channel-agnostic.** Mail, web, social, and newer channels all carry a string.
- **Fails visibly.** Uncoded transactions are findable; the payment still completes.
- **No JS required.** Payment processors natively store a source code.
- **Stable vocabulary** across decades of marketers.

Cons:

- Messages and links must actually be coded.
- Codes must be **unique** and documented. Reusing one code on ten sends makes per-send performance unknowable.
- Message performance (impressions, clicks, spend) must be joined to conversions afterward — engine9 calls that join **attribution**.

engine9 automates generation, parse of legacy codes, attribution, and cross-channel reporting. Native and tag/pixel stats may still be ingested; source-code attribution is the system of record.

## Attribution vs models

**Attribution** connects a **transaction** to a **message**. In engine9 that is last-click: match `transaction_summary.transaction_source_code` to `message_source_code.source_code` when `ts` is after `publish_date`, write `recommended_message_id`, then roll `attributed_revenue` / `attributed_transactions` onto `global_message_summary`. Pipeline: [A–F](SKILL.md#last-click-pipeline-debug-af).

A **model** connects a person or transaction to an item in their [timeline](../e9-timeline/SKILL.md) history — not to a message. Current output is `{prefix}_*` tables (`model_crm_origin_person`, `model_first_touch_transaction_stats`, …) with the new identity. First touch, last acquisition, CRM origin, and other identified LTV rules are models. Say **model** only in that sense. Last-click is attribution, not a “last-click model.”

`source_code_summary.origin_people` / `origin_revenue` are the **legacy** origin implementation (old identity). Do not use them as the current origin model.

Ad platforms (Google, Meta) see opens/clicks inside one platform. engine9 attribution spans platforms and offline via conversion source codes.

Last-click attribution (`source_code_summary.revenue` / `attributed_*`) and a current origin model (`{prefix}_*_stats`) answer different questions; adding them double-counts. Current models: [e9-model](../e9-model/SKILL.md).

### Worked example

One person, $150 total (a $50 recurring series of three). Three assignments: 100% last-click attribution, 100% origin model (lifetime to first timeline engagement), and a 50/50 split of those two. The split is illustrative; engine9 does not ship it.

| Date | Interaction | Source code | 100% last click | 100% origin | 50% adjusted |
|------|-------------|-------------|----------------:|------------:|-------------:|
| 2019-01-10 | First acquired (origin) | `EMAIL_JAN_2019` | — | $150 | $75 |
| 2019-03-12 | Clicked an email | `EM_CAN_456` | — | — | — |
| 2019-07-09 | Clicked a banner | `BAN_456` | — | — | — |
| 2019-08-10 | FB ad click, started $50 recurring | `FB_123` | $50 | — | $25 |
| 2019-09-10 | $50 recurring | `FB_123` | $50 | — | $25 |
| 2019-10-10 | $50 recurring | `FB_123` | $50 | — | $25 |
| | **Total** | | **$150** | **$150** | **$150** |

By source code:

| Source code | 100% last click | 100% origin | 50% adjusted |
|-------------|----------------:|------------:|-------------:|
| `FB_123` | $150 | $0 | $75 |
| `EMAIL_JAN_2019` | $0 | $150 | $75 |
| `EM_CAN_456` | $0 | $0 | $0 |
| `BAN_456` | $0 | $0 | $0 |

Last-click attribution gives Facebook the transactions. The origin model gives the acquisition email the lifetime total. The 50% split shares those two and still ignores the middle clicks. Path-based or time-decay assignment is not shipped; write a [model](../e9-model/SKILL.md) if you need another timeline rule.

### Last-click slices

- **Initial transactions** — one-time, or the first transaction of a recurring series. Immediate message impact.
- **Subsequent transactions** — recurring after the first. Use for upsell / recurring-activation over months.

### Origin model vs last-click attribution on the same code

A code can be origin-only (list buy), last-click-only (an appeal email), or both (a message that converts *and* recruits). Origin is a model: the right lens for where to spend on acquisition and which partnerships pay over a lifetime — read `{prefix}_*` tables, not `source_code_summary.origin_*`. Last click is attribution: the right lens for A/B tests and channel bake-offs (Facebook vs Google, subject lines, etc.).
