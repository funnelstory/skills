# Renewal Dashboard

Accounts renewing within the next 90 days (configurable), ranked by churn likelihood, with drivers and suggested pre-renewal actions. Portfolio-scoped — needs user email (Step 0 in SKILL.md) unless the user specifies a segment or "all accounts."

## Audience

- **CSM:** Prioritize renewal prep for accounts on their book.
- **VP CS:** Pipeline view across the team — total ARR renewing, risk concentration, coverage gaps.

## Step 1 — Scope

Ask or infer:

1. **Renewal window:** Default 90 days. User may say "next quarter", "next 60 days", etc.
2. **Portfolio filter:** My accounts (needs user email), a CSM's accounts, segment, or all.
3. **Output:** Default **HTML dashboard** (see `templates/html-dashboard.md`). Markdown table if user requests.

## Step 2 — Query renewals

```sql
SELECT
  account_id, name, domain, amount, health_score, prediction_score,
  expires_at, feature_adoption, activity_score, support_sentiment,
  conversation_sentiment, license_utilization
FROM accounts
WHERE expires_at IS NOT NULL
  AND expires_at >= datetime('now')
  AND expires_at <= datetime('now', '+90 days')
ORDER BY expires_at ASC
```

Adjust the `+90 days` window based on user input.

If scoped to the user's book, add a WHERE clause to filter by assignment. Try the `assignees` column first:
```sql
AND EXISTS (SELECT 1 FROM json_each(assignees) WHERE json_each.value = '<user_email>')
```

If that returns no results, fall back to account properties:
```sql
AND (json_extract(properties, '$.csm_email') = '<user_email>'
  OR json_extract(properties, '$.account_owner') = '<user_email>'
  OR json_extract(properties, '$.csm') = '<user_email>')
```

### Enrich with risk signals

For each renewing account, also pull:

- **Open negative needle movers** (impact < 0, state = 'open')
- **Recent support sentiment** from tickets
- **Activity trend** (current 90 days vs prior 90 days — is engagement rising or falling?)
- **Notes** mentioning "renewal", "churn", "competitor", or "pricing" in the last 90 days

## Step 3 — Classify renewal risk

For each account, assign a tier:

| Tier | Criteria |
|------|----------|
| **Critical** | Health < 50, OR prediction = churn, OR multiple negative needle movers |
| **At risk** | Health 50–69, OR declining activity trend, OR negative sentiment |
| **On track** | Health 70+, stable/growing engagement, no open risks |

Document the logic used in a "Method" footnote on the dashboard.

## Step 4 — Output

### KPI row
- Total accounts renewing
- Total ARR renewing (sum of `amount`)
- Critical count + ARR
- At-risk count + ARR

### Per-account cards (sorted: Critical first, then At risk, then On track)

Each card:
- Account name (linked to FunnelStory if possible)
- ARR, renewal date, days remaining
- Risk tier badge (Critical / At risk / On track)
- Top 2 risk drivers (cited from data)
- **Suggested action** — one concrete pre-renewal step (e.g., "Schedule exec sponsor call", "Run value recap email", "Address open ticket #X")

### Summary section

- Accounts needing immediate attention (Critical tier)
- Themes across at-risk renewals (common drivers)
- Coverage check: any critical renewals without a recent CSM touch?

## Quality bar

- Every risk driver cites a specific metric or observation — no "account seems unhappy."
- Suggested actions must be concrete and account-specific.
- Distinguish this from `churn-risk-report` — renewal dashboard is TIME-BOUNDED (next N days) and focused on renewal readiness. Churn risk report covers all at-risk accounts regardless of renewal date.
