# Health Score Breakdown

Explain why a named account's health score is what it is — the contributing factors, their weights, trends over time, and what's dragging the score down or lifting it up.

## Audience

- **CSM:** Understand the levers to improve a specific account's health before renewal or escalation.
- **VP CS:** Validate that the health model is working as expected; understand why a flagged account scores the way it does.

## Step 1 — Identify the account

If the user hasn't named an account, ask for it. Resolve via:

```sql
SELECT account_id, name, domain, health_score, prediction_score
FROM accounts
WHERE name LIKE '%AccountName%'
LIMIT 10
```

Confirm with the user if multiple matches.

## Step 2 — Gather health components

### Current scores

```sql
SELECT health_score, prediction_score, activity_score, feature_adoption,
       license_utilization, support_sentiment, conversation_sentiment,
       product_engagement, total_users
FROM accounts
WHERE account_id = '<account_id>'
```

Each field is a component that contributes to the composite `health_score`. Not all workspaces populate every field — skip any that return NULL and note it.

### Prediction factors

```sql
SELECT timestamp, score, point
FROM prediction_account_history
WHERE account_id = '<account_id>'
ORDER BY timestamp DESC
LIMIT 6
```

The `point` JSON column contains:
- `factors` — top features contributing to the prediction (positive and negative)
- `recommendations` — suggested actions to improve the score

Parse `factors` to identify the strongest positive and negative drivers.

### Score trend over time

Use the last 6 data points from `prediction_account_history` to show the trajectory. Convert -1..1 to 0–100 if needed (`(score * 50) + 50`).

### Supporting signals

Pull recent data that explains the score components:

- **Activity trend** (last 90 vs prior 90 days):

```sql
SELECT
  SUM(CASE WHEN timestamp >= datetime('now', '-90 days') THEN count ELSE 0 END) AS recent,
  SUM(CASE WHEN timestamp < datetime('now', '-90 days') AND timestamp >= datetime('now', '-180 days') THEN count ELSE 0 END) AS prior
FROM activities
WHERE account_id = '<account_id>'
```

- **Support sentiment** — recent tickets and their sentiment.
- **Notes** — qualitative context mentioning health, risk, or satisfaction.
- **Needle movers** — open items with negative impact.

## Step 3 — Output

Default: **Markdown** in chat (one-pager style — see `templates/one-pager.md`).

### Structure

```
# {Account Name} — Health Score Breakdown

**Current health:** {score} / 100 | **Trend:** {↑ improving / → stable / ↓ declining} | **As of:** {date}

## Score Components

| Factor | Value | Weight indication | Impact |
|--------|-------|-------------------|--------|
| Activity score | {value} | {High/Med/Low} | {Positive/Neutral/Negative} |
| Feature adoption | {value}% | {High/Med/Low} | {Positive/Neutral/Negative} |
| License utilization | {value}% | {High/Med/Low} | {Positive/Neutral/Negative} |
| Support sentiment | {value} | {High/Med/Low} | {Positive/Neutral/Negative} |
| Conversation sentiment | {value} | {High/Med/Low} | {Positive/Neutral/Negative} |
| Product engagement | {category} | {High/Med/Low} | {Positive/Neutral/Negative} |

## Top Drivers (from prediction model)

**Lifting the score:**
- {Factor 1} — {explanation}
- {Factor 2} — {explanation}

**Dragging the score down:**
- {Factor 1} — {explanation with specific data}
- {Factor 2} — {explanation with specific data}

## Trend (last 6 periods)

{Show score values over time — markdown table or inline list}

## What to do about it

1. {Concrete action targeting the #1 negative driver}
2. {Concrete action targeting the #2 negative driver}
3. {Action to protect a positive driver from declining}
```

## Quality bar

- Every claim about what's hurting or helping the score must cite a specific metric value or data point.
- Distinguish between what the **prediction model** says (from `point.factors`) and what the **component scores** show — they may tell different stories.
- If weight information is not available in the schema, say "Weights are not exposed in this workspace's data model" and rank by observed impact instead.
- Do NOT fabricate factor weights — only state them if the data explicitly provides them.
