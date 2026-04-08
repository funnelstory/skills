---
name: funnelstory
description: >
  FunnelStory MCP toolkit for Customer Success, Product, and Marketing teams. Use this skill
  for ANY request involving FunnelStory data: account briefs, book of business, meeting prep,
  daily prioritization for a CSM,
  QBR decks (internal or customer-facing), case studies, lead/PQL reports, expansion dashboards,
  upsell dashboards, churn risk reports, renewal dashboards, adoption gap analysis, adoption
  funnels, health score breakdowns, success plans, value emails, executive sponsor tracking,
  feature request dashboards, or customer ROI stories. Requires FunnelStory
  MCP to be connected.
---

# FunnelStory — Customer Success toolkit

This folder contains sub-skills for every report, dashboard, and deliverable you can build from FunnelStory data. Follow these steps in order for every request.

For troubleshooting common issues, see `references/troubleshooting.md`.
For persona-specific trigger examples, see `references/use-cases.md`.

## Prerequisites

- **FunnelStory MCP** must be connected (login via OTP if required).
- Use **`get_data_connections`** to locate the Semantic DB `connection_id`.
- Use **`execute_query`** to run SQLite against that connection. Set `limit` high enough to avoid truncation (e.g. 25+ rows).
- **JSON arrays:** Use `json_each(col)` for proper JSON array columns (e.g. `assignees`). For text-based array fields (e.g. `needle_movers.account_ids`), use `INSTR(col, value) > 0` instead.
- **JSON fields:** Use `json_extract(col, '$.field')` for nested property access.

## Step 0 — Identify the user

Many sub-skills need to know *whose* accounts to query. **Before doing anything else**, determine whether the request is scoped to the user's own accounts or to a named account.

**If the request is portfolio-scoped** (book of business, churn risk across "my accounts", expansion on "my book", renewal pipeline, etc.):

1. Ask: *"What email address are you logged in with in FunnelStory? I need it to find the accounts assigned to you."*
2. Look up assigned accounts. Try these approaches in order until one returns results:

**Primary — assignees column:**
```sql
SELECT account_id, name, domain, amount, health_score, prediction_score, expires_at
FROM accounts
WHERE EXISTS (
  SELECT 1 FROM json_each(assignees) WHERE json_each.value = '<user_email>'
)
```

**Fallback — account properties:**
```sql
SELECT account_id, name, domain, amount, health_score, prediction_score, expires_at
FROM accounts
WHERE json_extract(properties, '$.csm_email') = '<user_email>'
   OR json_extract(properties, '$.account_owner') = '<user_email>'
   OR json_extract(properties, '$.csm') = '<user_email>'
```

If both return no results, ask the user to list account names manually (see book-of-business README for the full fallback chain).

3. Cache the email for the session — only ask once.

**If the request names a specific account** (account brief, meeting prep, QBR, case study, value email, health score breakdown, success plan): skip this step and resolve the account by name in the sub-skill.

**If the request is workspace-wide** (feature requests dashboard, adoption funnel — typically PM/PMM personas): skip this step; query all accounts.

## Step 1 — Route to the right sub-skill

Match the user's intent to **exactly one** sub-folder. Read the `README.md` inside that folder and follow it.

### CSM / VP of CS — Account-level

| User intent | Sub-folder | Notes |
|-------------|------------|-------|
| Summarize one named account, account health deep-dive | `account-brief/` | Internal-facing; single account |
| Why is the health score what it is, score factors and weights | `health-score-breakdown/` | Single account; focused on score mechanics |
| Create a success plan, gap analysis + action plan | `success-plan/` | Single account; forward-looking |
| Meeting prep, pre-call brief | `meeting-prep/` | Single account; user names it |
| Internal QBR deck, quarterly review for internal prep | `qbr-internal/` | Single account; PowerPoint default; NOT for customer |
| Customer-facing QBR, value review to present TO the customer | `qbr-external/` | Single account; focuses on value delivered |
| Case study, publishable HTML for prospects | `case-study/` | Single account; HTML output |
| Value email to the customer, ROI recap | `value-email/` | Customer-facing email; user names the account |

### CSM / VP of CS — Portfolio-level

| User intent | Sub-folder | Notes |
|-------------|------------|-------|
| "My book", "my assigned accounts", portfolio snapshot | `book-of-business/` | Needs user email (Step 0) |
| "Prioritize my day", what should I work on today, daily CS priorities | `prioritize-my-day/` | Needs current user or explicit scope; returns a ranked action plan |
| Churn risk, at-risk accounts, retention danger | `churn-risk-report/` | Ranked risk report; all at-risk accounts regardless of renewal |
| Renewals coming up, renewal pipeline, renewal readiness | `renewal-dashboard/` | Time-bounded (next 90 days); action-plan focused |
| Expansion: more seats, users, sites, land-and-expand | `expansion-dashboard/` | Volume/geography growth |
| Upsell: higher tier, premium modules, add-on SKUs | `upsell-dashboard/` | Packaging/tier upgrades |
| Adoption gap, low usage, shelfware, entitlement vs usage | `adoption-gap/` | Which accounts under-use (CSM-facing) |
| Executive sponsor coverage, sponsor gaps | `exec-sponsor/` | Coverage audit |
| Hot leads, PQL, trial conversion, net-new leads | `lead-reports/` | Also read `lead-reports/reference.md` |

### PM / VP of Product

| User intent | Sub-folder | Notes |
|-------------|------------|-------|
| Feature requests, what are customers asking for | `feature-requests/` | Cross-account; ranked by frequency + ARR weight |
| Feature adoption, adoption funnel, which features are used | `adoption-funnel/` | Cross-account; which features are under-adopted (PM-facing) |

### PMM / VP of Marketing

| User intent | Sub-folder | Notes |
|-------------|------------|-------|
| ROI stories, customer outcomes for marketing, batch ROI | `customer-roi-stories/` | Multi-account batch; compact marketing-ready snippets |
| Full case study, publication-ready HTML | `case-study/` | Single-account deep editorial narrative |

### Disambiguation rules

When intent is ambiguous, apply these rules:

- **Health score breakdown vs Account brief**: Health score breakdown = laser-focused on the score mechanics (factors, weights, trends, what to fix). Account brief = full narrative across all dimensions (contacts, tickets, notes, usage, needle movers). If they ask "why is the health score low?", use health-score-breakdown. If they ask "tell me about this account", use account-brief.
- **Success plan vs Account brief**: Success plan = forward-looking action plan with milestones and owners. Account brief = current-state snapshot. If they say "create a plan to improve this account", use success-plan. If they say "summarize this account", use account-brief.
- **Success plan vs Adoption gap**: Success plan = single-account action plan combining usage AND qualitative gaps. Adoption gap = portfolio-wide report of accounts under-using. If they name one account and want a plan, use success-plan. If they want a list of under-using accounts, use adoption-gap.
- **Renewal dashboard vs Churn risk report**: Renewal dashboard = time-bounded (next 90 days), focused on renewal readiness. Churn risk report = all at-risk accounts ranked by danger, regardless of renewal timing. If they say "who's renewing soon?", use renewal-dashboard. If they say "who might churn?", use churn-risk-report.
- **QBR internal vs QBR external**: Internal = risks, concerns, honest internal metrics, team prep. External = value delivered, customer-ready, no internal scores or risk flags. If the user says "QBR" without specifying, ask: "Is this for internal prep or to present to the customer?"
- **Feature requests vs Adoption funnel**: Feature requests = what customers ASKED for (from needle movers, notes, tickets). Adoption funnel = how customers actually USE features (from activities). If they ask "what are customers requesting?", use feature-requests. If they ask "which features are being used?", use adoption-funnel.
- **Adoption funnel vs Adoption gap**: Adoption funnel = PM-facing, feature-level analysis (which features are adopted across the base). Adoption gap = CSM-facing, account-level analysis (which accounts under-use). If a PM asks, lean toward adoption-funnel. If a CSM asks, lean toward adoption-gap.
- **Customer ROI stories vs Case study**: ROI stories = batch across multiple accounts, compact marketing snippets. Case study = deep single-account editorial with full HTML design. If they want stories for multiple accounts, use customer-roi-stories. If they want a deep dive on one account, use case-study.
- **Expansion vs Upsell**: Expansion = more of the same (seats, users, sites, regions). Upsell = different packaging (higher tier, premium modules, add-on SKUs). If the user says "growth opportunities" without specifying, ask which they mean.
- **Churn risk vs Book of business**: Churn risk = ranked danger report with drivers and plays. Book of business = personal portfolio snapshot with health + drill-down. If they say "how are my accounts doing?", use book-of-business. If they say "which accounts might churn?", use churn-risk-report.
- **Value email vs Account brief**: Value email = outbound to the customer (achievements, CTA). Account brief = internal deep-dive for stakeholders.
- **Lead reports vs Expansion/Upsell**: Lead reports = net-new, trial, or free-tier accounts that are product-qualified. Expansion/Upsell = existing paying customers with room to grow.

## Audience

These skills serve four personas:

- **CSM (Customer Success Manager)**: Directly owns accounts. Requests are usually scoped to "my book" or a named account. Wants actionable next steps.
- **VP of CS**: Manages a team. Requests are scoped to a segment, team, or all accounts. Wants aggregate patterns and coverage gaps.
- **PM / VP of Product**: Cares about feature-level signals across the customer base. Requests are typically workspace-wide with optional segment filters.
- **PMM / VP of Marketing**: Needs customer proof points and ROI stories for marketing collateral. Requests span multiple accounts.

## Shared resources

- **`templates/html-dashboard.md`** — Shared HTML dashboard layout (KPI row, card grid, styling). Referenced by dashboard-style sub-skills.
- **`templates/one-pager.md`** — Compact one-page summary template. Referenced when a quick format is requested.
- **`references/troubleshooting.md`** — Common problems and how to handle them (MCP not connected, empty results, ambiguous names, etc.).
- **`references/use-cases.md`** — Persona-specific trigger examples mapped to sub-skills.
