# Attribution and conversion tracking

Read [SKILL.md](SKILL.md) first for last-click vs origin fields, link parameters, and warehouse tables.

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

## Attribution models

Online-ad attribution (Google, Meta) sees opens/clicks **inside one platform**. engine9 spans platforms and offline, using conversion source codes. Both views can be useful.

Credit assignment is not unique; there is no single right model. Most models try to **span all transactions once**. Crediting a $100 transaction 100% to last click **and** 100% to origin double-counts $200. Fractional models split the amount so totals still equal real revenue.

engine9 ships **100% last click** and **100% origin**. Use extra models only when they still span each transaction exactly once. Pick the model that matches the question (immediate impact, lifetime value, recurring activation, etc.).

### Worked example

One person, $150 total (a $50 recurring series of three). Three models: 100% last click, 100% origin, 50% last-click / 50% origin.

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

Last click gives Facebook all credit. Origin gives the acquisition email all credit. 50% split shares those two and still ignores the middle clicks. engine9 does not ship a path-based or time-decay model out of the box.

### Last-click slices

- **Initial transactions** — one-time, or the first transaction of a recurring series. Immediate message impact.
- **Subsequent transactions** — recurring after the first. Use for upsell / recurring-activation over months.

### Origin vs last click on the same code

A code can be origin-only (list buy), last-click-only (an appeal email), or both (a message that converts *and* recruits). Origin is the right lens for where to spend on acquisition and which partnerships pay over a lifetime. Last click is the right lens for A/B tests and channel bake-offs (Facebook vs Google, subject lines, etc.).
