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
  account_id, name, domain, aggregate_amount, health_score,
  earliest_child_expiry, expires_at, feature_adoption, product_engagement,
  activity_score, support_sentiment, conversation_sentiment, license_utilization
FROM accounts
WHERE (churned = 0 OR churned IS NULL)
  AND COALESCE(earliest_child_expiry, expires_at) IS NOT NULL
  AND COALESCE(earliest_child_expiry, expires_at) >= datetime('now')
  AND COALESCE(earliest_child_expiry, expires_at) <= datetime('now', '+90 days')
ORDER BY COALESCE(earliest_child_expiry, expires_at) ASC
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

- **Open negative needle movers** — `state = 'open'`, `source IN ('v2', 'relationship_map')`, impact below 0. Report the product impact tier from the root `SKILL.md`, not the raw integer.
- **Recent support sentiment and open escalations** — via `references/linking-conversations-to-accounts.md` (no fixed ticket join)
- **Activity trend** (current 90 days vs prior 90 days — is engagement rising or falling?)
- **Notes** mentioning "renewal", "churn", "competitor", or "pricing" in the last 90 days

## Step 3 — Classify renewal risk

Band each account using **Health tiers** in the root `SKILL.md`:

| Tier | Health score |
|------|----------------|
| **Critical** | 45 or below |
| **At risk** | 46 to 54 |
| **On track** | 55 or above |

No score is **Unscored**. List the other signals (negative needle movers, prediction, activity trend, sentiment) as drivers beside the tier. Do not change the tier because of them. Tiers are product-defined and match the FunnelStory UI; do not apply alternative thresholds from the workspace profile. State the tier rule in a "Method" footnote.

## Step 4 — Output

### KPI row
- Total accounts renewing
- Total contract value renewing (sum of `aggregate_amount`, labelled per the workspace profile)
- Critical count + value
- At risk count + value
- On track count + value

### Per-account cards (sorted: Critical first, then At risk, then On track, then Unscored)

Each card:
- Account name (linked to FunnelStory if possible)
- Contract value, renewal date, days remaining
- Tier badge (Critical / At risk / On track / Unscored)
- Top 2 risk drivers (cited from data)
- **Suggested action** — one concrete pre-renewal step (e.g., "Schedule exec sponsor call", "Run value recap email", "Address the open escalation")

### Summary section

- Accounts needing immediate attention (Critical)
- Themes across At risk renewals (common drivers)
- Coverage check: any Critical renewals without a recent CSM touch?

## Quality bar

- Every risk driver cites a specific metric or observation — no "account seems unhappy."
- Suggested actions must be concrete and account-specific.
- Distinguish this from `churn-risk-report` — renewal dashboard is TIME-BOUNDED (next N days) and focused on renewal readiness. Churn risk report covers all at-risk accounts regardless of renewal date.
