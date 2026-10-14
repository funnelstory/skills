---
name: funnelstory
version: 1.3.0
description: >
  FunnelStory MCP toolkit for Customer Success, Product, and Marketing teams. Use this skill
  whenever a request needs customer or account data, even if FunnelStory is not named:
  "prep me for my call with <account>", "tell me about <account>", "how's my book",
  "what should I do today", "who's renewing next quarter", "which accounts might churn",
  "what's open with <account>", "build a QBR for <account>", "draft a value email",
  "create a flow", "add a task", "set up our workspace profile". Covers account briefs, book of business, a daily priority
  list, meeting prep, support history, QBR decks (internal or customer-facing), case studies,
  lead/PQL reports, expansion and upsell dashboards, churn risk reports, renewal dashboards,
  adoption gap analysis, adoption funnels, health score breakdowns, success plans, value
  emails, executive sponsor tracking, feature request dashboards, customer ROI stories,
  projects and tasks, flow authoring, workspace profile setup, data model configuration, and
  connection query authoring.
  Requires FunnelStory MCP to be connected.
---

# FunnelStory — Customer Success toolkit

This folder contains sub-skills for every report, dashboard, and deliverable you can build from FunnelStory data. Follow these steps in order for every request.

For troubleshooting common issues, see `references/troubleshooting.md`.
For persona-specific trigger examples, see `references/use-cases.md`.

## Prerequisites
- **Read context resources first (always).** Before doing any work on a request — querying data, routing to a sub-skill, or producing a deliverable — call the **`read_resource`** tool with the relevant resource URI(s) to load the current context for the task. The MCP server exposes reference context (schema, field definitions, workspace configuration, and similar) addressable by URI. Identify which resource(s) the request depends on, read them with `read_resource`, and only then proceed. Do not rely on assumed schema, field names, or prior-session memory when a resource can confirm the current state — read first, then act.
- **FunnelStory MCP** must be connected. Authentication is OAuth 2.0 with PKCE; see `MCP_DOCUMENTATION.md`. Do not start an OTP login unless `who_am_i` fails and the connected server exposes a `login` tool.
- **Read `file://workspace/profile.json` before querying.** Its `instructions` field may override terminology, value labels, ownership fields, and which fields may be surfaced. Workspace instructions take precedence over the defaults in any sub-skill README — **except health**, which is product-defined (see **Health tiers**).
- Use **`query_semantic_db`** to run SQLite against the semantic database. It takes raw SQL directly; no connection lookup is required. Set limits high enough to avoid truncation (e.g. 25+ rows).
- Before writing queries, read `file://semantic/usage.md` and `file://semantic/schema.sql` via **`read_resource`**. Table and column availability differ between workspaces; confirm with `SELECT name FROM pragma_table_info('<table>')` when in doubt.
- **JSON arrays:** Use `json_each(col)` for proper JSON array columns (e.g. `assignees`). For text-based array fields (e.g. `needle_movers.account_ids`), use `INSTR(col, value) > 0` instead.
- **JSON fields:** Use `json_extract(col, '$.field')` for nested property access.
- **Never `SELECT *`** on `tickets` or `conversations` — wide JSON columns such as `custom_fields` can carry dozens of fields per row. Name the columns you need.

## Step 0 — Identify the user

Many sub-skills need to know *whose* accounts to query. **Before doing anything else**, determine whether the request is scoped to the user's own accounts or to a named account.

**If the request is portfolio-scoped** (book of business, churn risk across "my accounts", expansion on "my book", renewal pipeline, etc.):

1. Call **`who_am_i`** (FunnelStory MCP). It returns the connected user's id, name, email, and role — use that email. Only if `who_am_i` is unavailable or returns no email, ask: *"What email address are you logged in with in FunnelStory? I need it to find the accounts assigned to you."*
2. Look up assigned accounts. Try these approaches in order until one returns results:

**Primary — assignees column:**
```sql
SELECT account_id, name, domain, aggregate_amount, health_score, earliest_child_expiry, parent_account_id
FROM accounts
WHERE (churned = 0 OR churned IS NULL)
  AND EXISTS (
    SELECT 1 FROM json_each(assignees) WHERE json_each.value = '<user_email>'
  )
```

**Fallback — account properties:**
```sql
SELECT account_id, name, domain, aggregate_amount, health_score, earliest_child_expiry, parent_account_id
FROM accounts
WHERE (churned = 0 OR churned IS NULL)
  AND (json_extract(properties, '$.csm_email') = '<user_email>'
   OR json_extract(properties, '$.account_owner') = '<user_email>'
   OR json_extract(properties, '$.csm') = '<user_email>')
```

If both return no results, ask the user to list account names manually (see book-of-business README for the full fallback chain). If the workspace profile `instructions` name a different ownership field, use it first.

3. Cache the identity for the session — resolve it only once.

**If the request names a specific account** (account brief, meeting prep, QBR, case study, value email, health score breakdown, success plan, support history): skip this step and resolve the account using the shared rule below.

**If the request is workspace-wide** (feature requests dashboard, adoption funnel — typically PM/PMM personas): skip this step; query all accounts.

## Shared conventions (all sub-skills)

These apply to every sub-skill. A sub-skill README may add detail but should not contradict them; workspace profile `instructions` override both.

### Resolving a named account

Match on name **and** domain (`name LIKE '%X%' OR domain LIKE '%X%'`, limit 25). Several matches is normal — workspaces carry stale duplicates and sometimes one record per site, and sites often share a domain.

- Rank candidates by non-zero `aggregate_amount` (fall back to `amount`), presence of `assignees`, and recent activity.
- Proceed with the top candidate, state in one line which one you chose and why, and list the alternatives at the end so the user can redirect.
- **Ask first only** when a parent and its children both match, or when the deliverable is customer-facing and more than one plausible match exists.
- A parent has `parent_account_id` NULL and children pointing at it. For a parent, report `aggregate_amount` and `earliest_child_expiry`, and say so. `amount` and `expires_at` are the account's own values and are often empty on a parent.

### Health tiers

`health_score` is the score Focus Areas shows. It is already 0–100: `50 + trunc(prediction_score * 100 / 2)`. Band it the same way:

| Tier | Health score |
|------|----------------|
| **Critical** | 45 or below |
| **At risk** | 46 to 54 |
| **On track** | 55 or above |

- No score means no tier. Say **Unscored**.
- The tier comes from the score alone. Report other signals (negative needle movers, an overdue renewal, prediction outcome `churn`, a weak relationship) as **drivers** next to the tier. Do not change the tier because of them.
- **Health is product-defined and not overridable.** Always use `health_score` and these tiers so Claude and the FunnelStory UI never disagree. If the workspace profile names a different health index or different thresholds, keep `health_score` and these tiers, and tell the user once that the profile asked for something else and the product definition was used. The profile may still restrict *where* health appears (for example, never in customer-facing output) — that is a visibility rule, not a redefinition.

### Value and renewal

- Portfolio totals use `aggregate_amount`. Label it the way the workspace profile does (often ARR).
- Renewal timing uses `earliest_child_expiry` when it is set, otherwise `expires_at`.
- Portfolio skills exclude `churned` accounts unless the user asks for them.

### Needle movers

Match the product list: `source IN ('v2', 'relationship_map')` and `state = 'open'` unless the user asks for closed items. Include `content_theme` only when the user asks about themes. Report impact as a tier, never the raw integer:

| Tier | Integer `impact` |
|------|------------------|
| **Negative high** | −7 or below |
| **Negative medium** | below −2 and above −7 |
| **Negative low** | −2 through just under 0 |
| **Positive low** | above 0 through 2 |
| **Positive medium** | above 2 and below 7 |
| **Positive high** | 7 or above |

### Where to read first

Column rules live in `file://semantic/usage.md`. Also read, when the question needs them:

- `file://semantic/account_profiles.md` — account narrative and stakeholders
- `file://semantic/relationship_map_history.md` — who talked to whom
- `file://data_model/guide.md` — data model configuration
- `file://flow/guide.md` — flow authoring
- `file://projects/guide.md` — projects and tasks

Themes are `content_labels`, not `topics`. Upcoming invites are `calendar_events` joined through `calendar_event_accounts`. `meetings` are recorded calls. `product_engagement` is 0 (inactive), 1 (monthly), 2 (weekly), or 3 (daily). "What should I do" starts from `action_cards`: a NULL `status` is still a suggestion.

### Support data

Any per-account ticket, escalation, or conversation data goes through `references/linking-conversations-to-accounts.md`. Do not use a fixed `tickets` ↔ `ticket_contacts` ↔ `contacts.domain` join without its preconditions. Internal conversations (colleagues discussing a customer) never appear in customer-facing output.

### Missing data

Distinguish **"No data available"** (the source exists, nothing for this account) from **"Not available in this workspace"** (the workspace does not carry it). Never omit a section silently.

### Output files

Use the client's own document capabilities rather than hard-coded paths. In Claude, use the built-in pptx / docx / pdf skills and present the file to the user; in other clients, write the file to the workspace and report its path. Name files `<account-name>-<deliverable>.<ext>`.

## Step 1 — Route to the right sub-skill

Match the user's intent to **exactly one** sub-folder. Read the `README.md` inside that folder and follow it.

### CSM / VP of CS — Account-level

| User intent | Sub-folder | Notes |
|-------------|------------|-------|
| Summarize one named account, account health deep-dive | `account-brief/` | Internal-facing; single account |
| Latest tickets, open escalations, support history for one account | `account-support-history/` | Single account; run linkage discovery first |
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
| Churn risk, at-risk accounts, retention danger | `churn-risk-report/` | Ranked risk report; all at-risk accounts regardless of renewal |
| Renewals coming up, renewal pipeline, renewal readiness | `renewal-dashboard/` | Time-bounded (next 90 days); action-plan focused |
| Expansion: more seats, users, sites, land-and-expand | `expansion-dashboard/` | Volume/geography growth |
| Upsell: higher tier, premium modules, add-on SKUs | `upsell-dashboard/` | Packaging/tier upgrades |
| Adoption gap, low usage, shelfware, entitlement vs usage | `adoption-gap/` | Which accounts under-use (CSM-facing) |
| Executive sponsor coverage, sponsor gaps | `exec-sponsor/` | Coverage audit |
| Hot leads, PQL, trial conversion, net-new leads | `lead-reports/` | Also read `lead-reports/reference.md` |
| What should I do today, prioritize my book | `prioritize-my-day/` | Open action cards first, then uncovered Critical / At risk and near renewals |

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

### Workspace setup

| User intent | Sub-folder | Notes |
|-------------|------------|-------|
| Set up, review, or update the workspace profile and its `instructions`; onboard a new workspace | `workspace-profile-setup/` | `admin` or `super_admin` only. Otherwise stop and tell them to ask a workspace admin. Drafts and tests; the admin applies it on Workspace Grounding |

### Data model setup

| User intent | Sub-folder | Notes |
|-------------|------------|-------|
| Write or validate a connection query (SQL, SOQL, HubSpot, Attio blocks) | `connection-queries/` | Use before configuring a model |
| Create or update a data model, map columns, add joins, preview, save | `configure-data-model/` | Use after the query works |

### Automation

| User intent | Sub-folder | Notes |
|-------------|------------|-------|
| Create or edit a flow, agent, or workflow | `flow-authoring/` | Read the flow resources first; do not paste a flow from memory |
| Create a project, add a task, turn a plan into tasks | `projects-and-tasks/` | Read `file://projects/guide.md` first |

### Disambiguation rules

When intent is ambiguous, apply these rules:

- **Health score breakdown vs Account brief**: Health score breakdown = laser-focused on the score mechanics (factors, weights, trends, what to fix). Account brief = full narrative across all dimensions (contacts, tickets, notes, usage, needle movers). If they ask "why is the health score low?", use health-score-breakdown. If they ask "tell me about this account", use account-brief.
- **Account support history vs Account brief**: Support history = tickets, escalations, and customer conversations for one account, with open items called out. Account brief = full narrative across all dimensions, of which support is one section. If they ask "what's open with this account?" or "latest tickets", use account-support-history. If they ask "tell me about this account", use account-brief.
- **Account support history vs Feature requests**: Support history = what one account raised, chronological. Feature requests = what customers asked for, aggregated across accounts and ranked. If they name a single account and want recency, use account-support-history.
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
- **Prioritize my day vs Book of business vs Churn risk**: Prioritize my day = today's open action cards, plus Critical or At risk accounts and near renewals that have no card. Book of business = a portfolio snapshot. Churn risk = a ranked danger report, not a daily plan.
- **Success plan vs Projects and tasks**: Success plan = the written 30/60/90 plan. Projects and tasks = creating those items in FunnelStory. If they want the plan written, use success-plan. If they want the tasks created, use projects-and-tasks.
- **Flow authoring vs Data model setup**: Flow authoring = an agent or workflow. Data model setup = mapping source data into FunnelStory. A flow that queries a model still starts in flow-authoring.

## Audience

These skills serve five personas:

- **CSM (Customer Success Manager)**: Directly owns accounts. Requests are usually scoped to "my book" or a named account. Wants actionable next steps.
- **VP of CS**: Manages a team. Requests are scoped to a segment, team, or all accounts. Wants aggregate patterns and coverage gaps.
- **PM / VP of Product**: Cares about feature-level signals across the customer base. Requests are typically workspace-wide with optional segment filters.
- **PMM / VP of Marketing**: Needs customer proof points and ROI stories for marketing collateral. Requests span multiple accounts.
- **Workspace admin**: Sets up and maintains the workspace profile that every other skill reads first. Only `admin` and `super_admin` run this. Anyone else is told to ask a workspace admin.

## Shared resources

- **`templates/html-dashboard.md`** — Shared HTML dashboard layout (KPI row, card grid, styling). Referenced by dashboard-style sub-skills.
- **`templates/one-pager.md`** — Compact one-page summary template. Referenced when a quick format is requested.
- **`references/linking-conversations-to-accounts.md`** — How to locate support data in a workspace and link it to accounts. Read before any per-account ticket or conversation query. Used by `account-support-history/` and `account-brief/`; also applies to `meeting-prep/` and `churn-risk-report/`.
- **`references/troubleshooting.md`** — Common problems and how to handle them (MCP not connected, empty results, ambiguous names, etc.).
- **`references/use-cases.md`** — Persona-specific trigger examples mapped to sub-skills.
- **`file://semantic/usage.md`** — Column rules for the semantic database. Shared conventions above defer to it.
