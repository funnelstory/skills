---
name: funnelstory
description: >
  FunnelStory MCP toolkit for Customer Success. Use this skill for ANY request involving
  FunnelStory data: account briefs, book of business, meeting prep, QBR decks, case studies,
  lead/PQL reports, expansion dashboards, upsell dashboards, churn risk reports, adoption gap
  analysis, value emails, executive sponsor tracking, or flow authoring. Requires FunnelStory
  MCP to be connected.
---

# FunnelStory — Customer Success toolkit

This folder contains sub-skills for every report, dashboard, and deliverable you can build from FunnelStory data. Follow these steps in order for every request.

## Prerequisites

- **FunnelStory MCP** must be connected (login via OTP if required).
- Use **`get_data_connections`** to locate the Semantic DB `connection_id`.
- Use **`execute_query`** to run SQLite against that connection. Set `limit` high enough to avoid truncation (e.g. 25+ rows).
- `json_each` is **not available**. Use `INSTR(col, value) > 0` for JSON array membership. Use `json_extract(col, '$.field')` for JSON field access.

## Step 0 — Identify the user

Many sub-skills need to know _whose_ accounts to query. **Before doing anything else**, determine whether the request is scoped to the user's own accounts or to a named account.

**If the request is portfolio-scoped** (book of business, churn risk across "my accounts", expansion on "my book", etc.):

1. Ask: _"What email address are you logged in with in FunnelStory? I need it to find the accounts assigned to you."_
2. Look up assigned accounts:

```sql
SELECT account_id, name, domain, amount, health_score, prediction_score, expires_at
FROM accounts
WHERE json_extract(properties, '$.csm_email') = '<user_email>'
   OR json_extract(properties, '$.account_owner') = '<user_email>'
   OR json_extract(properties, '$.csm') = '<user_email>'
```

If the property-based lookup returns nothing, fall back to `get_account` per account and match on the `assignees[].email` array (see the book-of-business README for the full fallback chain).

3. Cache the email for the session — only ask once.

**If the request names a specific account** (account brief, meeting prep, QBR, case study, value email): skip this step and resolve the account by name in the sub-skill.

## Step 1 — Route to the right sub-skill

Match the user's intent to **exactly one** sub-folder. Read the `README.md` inside that folder and follow it.

| User intent                                              | Sub-folder             | Notes                                                              |
| -------------------------------------------------------- | ---------------------- | ------------------------------------------------------------------ |
| Summarize one named account, account health deep-dive    | `account-brief/`       | Internal-facing; single account                                    |
| "My book", "my assigned accounts", portfolio snapshot    | `book-of-business/`    | Needs user email (Step 0)                                          |
| Meeting prep, pre-call brief                             | `meeting-prep/`        | Single account; user names it                                      |
| QBR deck, quarterly business review                      | `qbr-deck/`            | Single account; PowerPoint default                                 |
| Case study, publishable HTML for prospects               | `case-study/`          | Single account; HTML output                                        |
| Hot leads, PQL, trial conversion, net-new leads          | `lead-reports/`        | Also read `lead-reports/reference.md` for SQL/HTML templates       |
| Expansion: more seats, users, sites, land-and-expand     | `expansion-dashboard/` | Volume/geography growth; needs user email if portfolio-scoped      |
| Upsell: higher tier, premium modules, add-on SKUs        | `upsell-dashboard/`    | Packaging/tier upgrades; needs user email if portfolio-scoped      |
| Churn risk, at-risk accounts, retention danger           | `churn-risk-report/`   | Ranked risk report; needs user email if portfolio-scoped           |
| Adoption gap, low usage, shelfware, entitlement vs usage | `adoption-gap/`        | Under-using what they bought; needs user email if portfolio-scoped |
| Value email to the customer, ROI recap                   | `value-email/`         | Customer-facing email; user names the account                      |
| Executive sponsor coverage, sponsor gaps                 | `exec-sponsor/`        | Coverage audit; needs user email if portfolio-scoped               |
| Flow authoring, workflows, automations                   | `flow-authoring/`      | Process design                                                     |

### Disambiguation rules

When intent is ambiguous, apply these rules:

- **Expansion vs Upsell**: Expansion = more of the same (seats, users, sites, regions). Upsell = different packaging (higher tier, premium modules, add-on SKUs). If the user says "growth opportunities" without specifying, ask which they mean.
- **Churn risk vs Book of business**: Churn risk = ranked danger report with drivers and plays. Book of business = personal portfolio snapshot with health + drill-down. If they say "how are my accounts doing?", use book-of-business. If they say "which accounts might churn?", use churn-risk-report.
- **Adoption gap vs Expansion**: Adoption gap = customers under-using what they already bought (shelfware, dormant users). Expansion = customers ready to buy more. If usage is low, it's adoption gap. If usage is high and there's headroom, it's expansion.
- **Value email vs Account brief**: Value email = outbound to the customer (achievements, CTA). Account brief = internal deep-dive for stakeholders. If they say "send the customer a recap", it's value email. If they say "summarize the account for me", it's account brief.
- **Executive sponsor vs Account brief**: Exec sponsor = coverage audit across accounts (who has a sponsor, gaps). Account brief = full narrative on one account. If they ask about sponsor gaps across the book, use exec-sponsor. If they want everything about one account, use account-brief.
- **Lead reports vs Expansion/Upsell**: Lead reports = net-new, trial, or free-tier accounts that are product-qualified. Expansion/Upsell = existing paying customers with room to grow. If the accounts are already paying, don't use lead-reports.

## Audience

These skills serve two personas:

- **CSM (Customer Success Manager)**: Directly owns accounts. Requests are usually scoped to "my book" or a named account. Wants actionable next steps.
- **VP of CS**: Manages a team. Requests are scoped to a segment, team, or all accounts. Wants aggregate patterns and coverage gaps.

Both personas are supported — the sub-skill READMEs explain how to adjust scope for each.
