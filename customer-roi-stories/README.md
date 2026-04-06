# Customer ROI Stories

Batch-generate compact ROI narratives across multiple accounts. Extract quantifiable outcomes, usage deltas, and value metrics to produce marketing-ready snippets. For PMM and VP of Marketing.

## Audience

- **PMM:** Generate ROI proof points for landing pages, sales decks, email campaigns, and social content.
- **VP of Marketing:** Identify which accounts have the strongest ROI stories worth investing in for deeper case studies or customer references.

**How this differs from `case-study/`:** Case study is a deep, single-account editorial narrative rendered as polished HTML. This skill is a batch operation across many accounts, producing compact ROI snippets — each 3–5 sentences with specific metrics.

## Step 1 — Scope

Ask or infer:

1. **Account selection:** Default = top 10 accounts by signal strength (health + activity + metrics). Or user can specify: "enterprise accounts", "accounts with the best outcomes", "accounts in [segment]."
2. **Time window:** Default last 12 months (ROI stories need longer horizons than operational reports).
3. **Output:** Default **Markdown** (ranked list of ROI snippets). HTML if user requests.

## Step 2 — Identify ROI candidates

### Find accounts with strong signals

```sql
SELECT
  account_id, name, domain, amount, health_score,
  feature_adoption, activity_score, total_users,
  created_at, expires_at
FROM accounts
WHERE (churned = 0 OR churned IS NULL)
  AND health_score IS NOT NULL
ORDER BY health_score DESC, amount DESC
LIMIT 30
```

### Score ROI signal strength

For each candidate, compute a ROI signal score from:
- **Usage growth:** activity count current 6 months vs prior 6 months
- **User growth:** `total_users` increase (if historical data available via `account_metrics_history`)
- **Feature breadth:** number of distinct `activity_name` values
- **Health trajectory:** improving prediction score over time
- **Metric improvements:** positive deltas in `account_metrics_history`

Rank by composite signal. Take the top N (default 10).

### Gather story evidence per account

For each selected account:

```sql
SELECT activity_name, SUM(count) AS total
FROM activities
WHERE account_id = '<account_id>'
  AND timestamp >= datetime('now', '-12 months')
GROUP BY activity_name
ORDER BY total DESC
LIMIT 10
```

```sql
SELECT metric_id, value, timestamp
FROM account_metrics_history
WHERE account_id = '<account_id>'
ORDER BY timestamp DESC
LIMIT 20
```

Also scan `notes` and `meetings` for outcome language: "saved", "reduced", "improved", "achieved", "launched", "expanded."

## Step 3 — Generate ROI snippets

For each account, produce:

```markdown
### {Account Name}

**Industry:** {industry from traits} | **Size:** {company size} | **Customer since:** {created_at}

**ROI snapshot:** {One sentence summarizing the top outcome — must include a number.}

**Key metrics:**
- {Metric 1}: {value or delta} (e.g., "Active users grew from 12 to 89 in 9 months")
- {Metric 2}: {value or delta}
- {Metric 3}: {value or delta}

**Quotable line:** "{One sentence that a PMM could use in a landing page or email — framed as the customer's achievement, grounded in the data.}"

**Signal strength:** {Strong / Moderate} | **Good candidate for full case study:** {Yes / No}
```

### Rules for the quotable line
- Must be factual and grounded in FunnelStory data — not aspirational.
- Frame as the customer's achievement, not the vendor's.
- Include at least one specific number.
- Example: "Acme Corp expanded from a single team to company-wide deployment, growing from 12 to 89 active users while maintaining 94% feature adoption."

## Step 4 — Summary output

After all snippets:

```markdown
## ROI Story Summary

| # | Account | Industry | Top metric | Signal | Case study candidate? |
|---|---------|----------|------------|--------|----------------------|
| 1 | {Name} | {Industry} | {Best metric} | Strong | Yes |
| 2 | ... | ... | ... | ... | ... |

**Themes across accounts:**
- {Common outcome 1} — {N} accounts
- {Common outcome 2} — {N} accounts

**Recommended next steps:**
- Reach out to {Account} for a customer reference / testimonial
- Develop full case study for {Account} (strongest signal)
```

## Quality bar

- Every ROI metric must come from FunnelStory data — no fabricated numbers.
- If an account has strong health but thin metric data, note "Limited quantifiable metrics — recommend customer interview for deeper story."
- Do NOT include accounts with declining health or churn signals in ROI stories.
- The "quotable line" must be something a real marketing team could publish without fact-checking beyond what's in FunnelStory.
