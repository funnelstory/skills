# Semantic lead reports — templates

Fill **Configuration** first. Then run SQL (customer-specific or adapted from their FunnelStory agent). Finally run the **HTML prompt** with placeholders replaced.

---

## Configuration (copy and fill)

| Placeholder | Example | Notes |
|-------------|---------|--------|
| `WORKSPACE_ID` | UUID from FunnelStory URL or workspace settings | Required for account deep links |
| `PRODUCT_NAME` | Acme Platform | Title bar and footer |
| `REPORT_TITLE` | Hot leads · January 2026 | Full header string |
| `PRIMARY_SECTION_TITLE` | Hot leads / PQLs / Conversion opportunities | Main list section |
| `PRIMARY_QUALIFICATION_LABEL` | Why hot / Why qualified / Why prioritize | Short label above narrative |
| `KPI_1_LABEL` | Report period | First KPI box |
| `KPI_2_LABEL` | New accounts (cohort) | Second KPI |
| `KPI_3_LABEL` | Qualified leads | Third KPI; value = row count of qualified list |
| `SECONDARY_SECTION_TITLE` | High activity — reviewed & not prioritized | Optional; omit section if no third query |
| `SECONDARY_RATIONALE_LABEL` | Why not prioritized | Label for `exclusion_reason` |
| `NOISE_ACTIVITIES` | SQL list: `'Login'`, `'Created User'`, … | Activities to **exclude** from meaningful usage |
| `COHORT_TIME_FILTER` | e.g. previous calendar month on `accounts.created_at` | Must be consistent across queries |
| `QUALIFIED_THRESHOLD` | numeric | Minimum composite score for primary list |
| `QUALIFIED_LIMIT` | 20 | Max rows primary list |
| `FLAGGED_LIMIT` | 15 | Max rows secondary list |

**FunnelStory account URL pattern** (encode `account_id` for query string as needed):

`https://app.funnelstory.ai/accounts?account_id={account_id_encoded}&workspace_id={WORKSPACE_ID}`

---

## Query architecture

### Query A — Report metrics (one row)

Return a single row the HTML layer can bind to KPIs and titles, for example:

- `report_period_label` — human-readable period
- `email_subject` — optional, if the user will email the report
- Cohort counts (e.g. accounts in period, prior-period comparison)
- Any KPI needed in the dashboard row

Use SQLite date functions aligned with the customer's definition of "period." Keep **the same period boundaries** in Query B (and C).

### Query B — Qualified leads

Pattern:

1. **Cohort** — CTE of accounts in scope (`accounts` + time + optional stage/plan filters).
2. **Meaningful activity** — From `activities`, aggregate by `account_id` and `activity_name`, excluding `NOISE_ACTIVITIES`.
3. **Features** — Derive `activity_detail` (e.g. `GROUP_CONCAT('Activity: count', ' | ')`), `feature_count`, `active_days` (`COUNT(DISTINCT date(timestamp))`), etc.
4. **Contacts** — Join `users` (or your semantic contact source) for name/email.
5. **Score** — Composite expression using fields available in **this** workspace (`product_engagement`, `health_score`, `total_users`, breadth metrics, etc.). **Name the expression** in comments for maintainability; do not print raw score in HTML.
6. **Filter and sort** — `WHERE score >= QUALIFIED_THRESHOLD` → `ORDER BY score DESC` → `LIMIT QUALIFIED_LIMIT`.

Expose columns the renderer needs: at minimum `account_id`, `account_id_encoded`, display domain/name, `signup_date` (or "first seen"), `active_days`, `feature_count`, `activity_detail`, `contact_name`, `contact_email`.

### Query C — Reviewed, not prioritized (optional)

For accounts that **did not** qualify but **warrant executive visibility** (high raw volume, single-feature spikes, suspected automation):

- Reuse the same cohort and scoring CTEs as Query B.
- Select rows **below** the qualification threshold that meet **visibility rules** (thresholds on totals, specific activity names, etc.).
- Emit a stable, **SQL-composed** `exclusion_reason` string per row (auditable logic).
- `ORDER BY` sensible priority (e.g. volume) → `LIMIT FLAGGED_LIMIT`.

If the customer does not want this section, skip Query C and pass an **empty** list to the HTML step; the prompt should print a one-line empty state.

---

## SQL skeleton (illustrative only)

Replace bracketed sections. Adjust table/column names to match the workspace semantic model.

```sql
-- QUERY A: metrics (example shape)
SELECT
  '<Report period label>' AS report_period_label,
  ('«PRODUCT_NAME» — «REPORT_SHORT_TITLE» · ' || '<period>') AS email_subject,
  (SELECT COUNT(*) FROM accounts WHERE <COHORT_TIME_FILTER>) AS cohort_current,
  (SELECT COUNT(*) FROM accounts WHERE <PRIOR_PERIOD_FILTER>) AS cohort_prior;
```

```sql
-- QUERY B: qualified leads (illustrative skeleton)
WITH recent_accounts AS (
  SELECT account_id, domain, created_at, health_score, total_users
    -- , other account fields used in scoring
  FROM accounts
  WHERE <COHORT_TIME_FILTER>
),
activity_per_type AS (
  SELECT a.account_id, a.activity_name, SUM(a.count) AS total_count
  FROM activities a
  WHERE a.account_id IN (SELECT account_id FROM recent_accounts)
    AND a.activity_name IS NOT NULL
    AND a.activity_name NOT IN (<NOISE_ACTIVITIES>)
  GROUP BY a.account_id, a.activity_name
  HAVING SUM(a.count) > 0
),
activity_summary AS (
  SELECT account_id,
    GROUP_CONCAT(activity_name || ': ' || CAST(total_count AS TEXT), ' | ') AS activity_detail,
    COUNT(*) AS feature_count
  FROM activity_per_type
  GROUP BY account_id
),
active_days_cte AS (
  SELECT a.account_id, COUNT(DISTINCT date(a.timestamp)) AS active_days
  FROM activities a
  WHERE a.account_id IN (SELECT account_id FROM recent_accounts)
    AND a.activity_name IS NOT NULL
    AND a.activity_name NOT IN (<NOISE_ACTIVITIES>)
  GROUP BY a.account_id
),
contacts_cte AS (
  SELECT u.account_id, MIN(u.name) AS contact_name, MIN(u.email) AS contact_email
  FROM users u
  WHERE u.account_id IN (SELECT account_id FROM activity_summary)
  GROUP BY u.account_id
),
scored AS (
  SELECT
    ra.*,
    asumm.activity_detail,
    asumm.feature_count,
    COALESCE(ad.active_days, 0) AS active_days,
    COALESCE(ct.contact_name, '') AS contact_name,
    COALESCE(ct.contact_email, '') AS contact_email,
    (/* composite score: tailor to tenant */ 0) AS qualification_score
  FROM recent_accounts ra
  INNER JOIN activity_summary asumm ON ra.account_id = asumm.account_id
  LEFT JOIN active_days_cte ad ON ra.account_id = ad.account_id
  LEFT JOIN contacts_cte ct ON ra.account_id = ct.account_id
)
SELECT
  account_id,
  domain,
  replace(replace(replace(account_id, '@', '%40'), '+', '%2B'), ' ', '%20') AS account_id_encoded,
  /* formatted signup_date column as needed */
  active_days,
  feature_count,
  activity_detail,
  contact_name,
  contact_email
FROM scored
WHERE qualification_score >= <QUALIFIED_THRESHOLD>
ORDER BY qualification_score DESC
LIMIT <QUALIFIED_LIMIT>;
```

Customers should **replace** the `qualification_score` expression with their own model (often copied from an existing FunnelStory agent).

---

## HTML generation — system prompt pattern

Output **only raw HTML**. Replace `«…»` placeholders before sending to the model.

```
You render an executive «REPORT_STYLE» email for «PRODUCT_NAME». Output ONLY raw HTML (no markdown fences, no preamble, no explanation).

AUDIENCE: «AUDIENCE» (e.g. VP Sales, CEO, RevOps). Formal, concise.

INPUTS:
(1) report_metrics — one row: KPIs and labels
(2) qualified_leads — pre-qualified accounts; include EVERY row as its own card
(3) reviewed_not_prioritized — optional; accounts with meaningful usage that did NOT qualify, each with exclusion_reason; include EVERY row or use empty-state copy if none

Do NOT show internal numeric scores (qualification_score, activity_score, etc.) in the rendered HTML.

LAYOUT:
- Mobile-first. Do NOT use wide multi-column DATA tables for leads. Use one full-width card per qualified account, stacked vertically.
- KPI row: exactly three equal-width columns; nested tables with identical fixed min-height/height per cell so all three KPI boxes match height (e.g. min-height and height 118px on inner tables).
- For each qualified card: Account (link to FunnelStory with workspace_id=«WORKSPACE_ID» and account_id=account_id_encoded), Contact (mailto), meta (dates, active days), then «PRIMARY_QUALIFICATION_LABEL» with a specific narrative from activity_detail and active_days. Format integers with comma thousands in prose.
- If reviewed_not_prioritized is non-empty: second section titled «SECONDARY_SECTION_TITLE»; same card shell; show «SECONDARY_RATIONALE_LABEL» using exclusion_reason verbatim.
- If reviewed_not_prioritized is empty: one muted line stating no exclusions to flag.

ACTIVITY GLOSSARY (tailor per customer — map their activity_name values to business meaning):
«BULLET_LIST_OF_ACTIVITY_MAPPINGS»

STYLING: Inline CSS only; email-safe; table-based structure; outer max-width ~820px centered; dark title bar; light grey page background; footer: «FOOTER_ATTRIBUTION» (e.g. «PRODUCT_NAME» · «REPORT_TITLE» · Generated by FunnelStory).
```

## HTML generation — user message template

```
REPORT METRICS (single row):
«paste JSON rows from Query A»

QUALIFIED LEADS (include ALL as cards, order preserved):
«paste JSON rows from Query B»

REVIEWED, NOT PRIORITIZED (include ALL as cards, or empty array):
«paste JSON rows from Query C or []»

Build the complete HTML. KPI row: three equal-height boxes. Lead sections: full-width cards only. Output ONLY raw HTML.
```

---

## Report-type wording (swap in «REPORT_STYLE» / section titles)

| Goal | Example `REPORT_TITLE` | Example `PRIMARY_SECTION_TITLE` | Example `PRIMARY_QUALIFICATION_LABEL` |
|------|-------------------------|----------------------------------|----------------------------------------|
| Hot leads | Hot leads · February 2026 | Hot leads | Why hot |
| PQL | Product-qualified leads · Q1 2026 | Product-qualified leads | Why PQL |
| Conversion | Conversion opportunities · Trial cohort | Conversion opportunities | Why prioritize |

Keep **one** framing per report so the narrative stays coherent.
