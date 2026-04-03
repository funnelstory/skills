# Expansion Opportunities Dashboard

## Audience

- **CSM:** Prioritize expansion plays on assigned accounts.
- **VP CS:** Portfolio view across the team or segment (filter by CSM, region, or segment when schema allows).

## Prerequisites

- Use the **semantic schema** your MCP exposes (`get_models`, `list_tables`, or tool docs). Adapt all SQL to real column names.
- Prefer `get_data_connections` → `execute_query` with a high enough `limit` for ranked lists.

## Step 1 — Scope

Ask if not clear:

1. **Portfolio scope:** Single CSM's book, whole workspace, or named segment?
2. **Time window** for activity trends (e.g. last 30 / 90 days).
3. **Output:** Default = **single HTML artifact** (dashboard). Markdown table only if the user refuses HTML.

## Step 2 — Define "expansion candidate" (tune per workspace)

Combine signals such as:

- **Headroom:** `total_users` or seat count vs license cap (from `accounts` / `properties`).
- **Growth:** rising `activities` volume, new `users`, or week-over-week active days.
- **Breadth:** multiple `activity_name` types or repos/projects above a threshold.
- **Health:** not in free trial; `health_score` or prediction not "churn" (exclude obvious churn plays — those belong in churn report).
- **Renewal timing:** optional boost for accounts 6–18 months from renewal (expansion before renewal).

Document the rule you used in a short "Method" line on the dashboard.

## Step 3 — Query pattern (illustrative)

Adapt joins and JSON fields to the tenant. Example shape:

```sql
-- Illustrative only — replace columns with your workspace schema
SELECT
  a.account_id,
  a.name,
  a.domain,
  a.amount,
  a.health_score,
  a.expires_at,
  a.total_users
FROM accounts a
WHERE a.churned = 0 OR a.churned IS NULL
ORDER BY a.health_score DESC
LIMIT 200;
```

Enrich with activity aggregates in a CTE (sum counts, distinct active days, feature breadth). Rank by a **composite expansion score** (document formula in the HTML). **Do not** expose raw internal scores in customer-facing copy; OK in a small "Method" footnote for internal demos.

## Step 4 — HTML dashboard

Output one self-contained HTML file:

- **Header:** title "Expansion opportunities", period, scope (CSM / all accounts).
- **KPI row:** count of qualified accounts, total ARR in pipeline, avg headroom signal (if defined).
- **Cards or rows:** one per account — name, domain, key metrics (users, activity trend, headroom), **why expansion** (2 bullets from data), suggested next play.
- **Styling:** Dark or light consistent with other FunnelStory demo artifacts; distinctive font (not Inter-only); CSS variables; no external JS charts unless user asks.

## Quality bar

- Every row ties to **observable data** (numbers or named activities).
- Separate clearly from **upsell** (no "upgrade tier" language unless the signal is seat/usage growth).
