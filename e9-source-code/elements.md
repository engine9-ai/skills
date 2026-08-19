# Source code elements

Open-ended text except `source_code_date`. Use whatever taxonomy the account uses; **consistency across codes** matters more than the words. Reports group and sum along these axes.

Suggested values below are suggestions only — no required menu unless noted.

## Identity and delivery

| Element | Field | Purpose |
|---------|-------|---------|
| Account | `account_prefix` | Client id for agencies serving many clients |
| Agency | `agency` | Marketing vendor; inverse of Account (ads shop vs email shop) |
| Channel | `source_code_channel` | Delivery channel: SMS, Bing Ads, Direct Mail, etc. |
| Source code date | `source_code_date` | `YYYYMMDD`. Often parsed from the code; may differ from message publish date |
| Channel subtype | `subchannel` | Finer channel detail |
| Media type | `media` | Video, text, image, etc. (MIME-like) |
| Organic/paid | `organic` | Organic vs paid |

## Primary message hierarchy

Prefer **Campaign → Message set → Message** for roll-up reporting.

| Element | Field | Purpose |
|---------|-------|---------|
| Campaign | `campaign` | Top-level cluster of related codes, often across channels |
| Message set | `message_set` | Ad set, email test set, etc. |
| Variant | `variant` | Specific message / individual ad in a set |

## Content and purpose

| Element | Field | Purpose |
|---------|-------|---------|
| Appeal | `appeal` | Content grouping; common in direct mail |
| Fund | `fund` | Operating fund, project, 501(c)3 vs 501(c)4, etc. |
| Issue | `issue` | Broad issue area |
| Theme | `theme` | Broad message theme |
| Policy | `policy` | e.g. Federal / State / International |
| Goal | `goal` | Descriptive goal: Advocacy, Fundraising, Retention, Acquisition, Ticket sales. Common for ad conversion splits |
| Secondary goal | `goal_2` | e.g. signups |

## Audience and targeting

| Element | Field | Purpose |
|---------|-------|---------|
| Audience | `audience` | Target: Lapsed Donors, Prospects, etc. |
| Targeting details | `targeting` | Extra targeting |
| Geography | `geo` | Locale: Richmond, VA, East Coast, etc. |

## People and org

| Element | Field | Purpose |
|---------|-------|---------|
| Author | `author` | Credited sender (email from line) |
| Signer | `signer` | Signature voice: staff, celebrity, constituent story, etc. |
| Partner | `partner` | Sending org for list rentals / partnerships |
| Organization/department | `department` | Internal department |
| Additional | `additional_info` | Catch-all |

Labels for any element appear as `{field}_label` (see SKILL.md).
