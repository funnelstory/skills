# Adoption Funnel by Feature

Feature-level adoption analysis: which features are used, by how many accounts, in what order, and where adoption drops off. Aggregated from `activities` grouped by `activity_name`. For PM and VP of Product.

## Audience

- **PM:** Understand which features drive stickiness, which are ignored, and where the onboarding funnel leaks.
- **VP of Product:** Strategic view of feature portfolio health — investment vs adoption.

**How this differs from `adoption-gap/`:** Adoption gap is CSM-facing and analyzes which **accounts** are under-using the product. This skill is PM-facing and analyzes which **features** are under-adopted across the customer base.

## Step 1 — Scope

Ask or infer:

1. **Segment filter** (optional): All accounts (default), or filter by plan/tier/segment.
2. **Time window:** Default last 90 days.
3. **Feature focus** (optional): All features, or a specific set of features the PM cares about.
4. **Output:** Default **HTML dashboard** (see `templates/html-dashboard.md`).

## Step 2 — Gather feature adoption data

### Feature usage across accounts

```sql
SELECT
  activity_name,
  COUNT(DISTINCT account_id) AS accounts_using,
  SUM(count) AS total_events,
  COUNT(DISTINCT date(timestamp)) AS active_days
FROM activities
WHERE timestamp >= datetime('now', '-90 days')
  AND activity_name IS NOT NULL
GROUP BY activity_name
ORDER BY accounts_using DESC
```

### Total account base (for adoption %)

```sql
SELECT COUNT(*) AS total_accounts
FROM accounts
WHERE churned = 0 OR churned IS NULL
```

If segment-filtered, add the segment WHERE clause to both queries.

### Feature adoption by segment (optional)

```sql
SELECT
  a.activity_name,
  json_extract(acc.properties, '$.segment') AS segment,
  COUNT(DISTINCT a.account_id) AS accounts_using
FROM activities a
JOIN accounts acc ON a.account_id = acc.account_id
WHERE a.timestamp >= datetime('now', '-90 days')
  AND a.activity_name IS NOT NULL
GROUP BY a.activity_name, segment
ORDER BY a.activity_name, accounts_using DESC
```

### Feature sequence (onboarding funnel)

To show adoption order, compute median first-use date per feature:

```sql
SELECT
  activity_name,
  account_id,
  MIN(timestamp) AS first_use
FROM activities
WHERE activity_name IS NOT NULL
GROUP BY activity_name, account_id
```

Then rank features by typical first-use order within accounts.

## Step 3 — Compute adoption metrics

For each `activity_name`:

| Metric | Calculation |
|--------|-------------|
| Adoption rate | `accounts_using / total_accounts * 100` |
| Depth | `total_events / accounts_using` (avg events per adopter) |
| Breadth | `accounts_using` (raw count) |
| Engagement density | `active_days / 90` (what fraction of days the feature is used) |

### Funnel view

Order features by typical adoption sequence (first-use date rank). Show:
- Feature A → 80% of accounts
- Feature B → 65% of accounts
- Feature C → 30% of accounts
- Feature D → 8% of accounts

The drop-off between stages is the funnel leak.

## Step 4 — Output

### KPI row
- Total features tracked
- Avg adoption rate
- Most adopted feature (name + %)
- Least adopted feature (name + %)

### Feature adoption table

| Feature | Adoption % | Accounts | Avg events/account | Engagement density | Trend |
|---------|-----------|----------|--------------------|--------------------|-------|
| {name} | {%} | {N} | {avg} | {density} | ↑/→/↓ |

### Funnel visualization (if adoption sequence data is available)

Show the sequential adoption path with drop-off percentages between stages. Use a simple horizontal bar chart in HTML or a markdown funnel.

### Segment comparison (if segment data available)

| Feature | Enterprise | Mid-market | SMB |
|---------|-----------|------------|-----|
| {name} | {adoption %} | {%} | {%} |

### Recommendations

For the bottom 3 features by adoption:
- Why it might be low (cite data: is it new? niche? poorly documented?)
- Suggested PM action (improve onboarding, sunset, bundle differently)

## Quality bar

- Adoption rate denominator must be clearly defined (all active accounts? all accounts? only accounts on plans with access to the feature?). State the denominator.
- Do NOT conflate "low usage" with "bad feature" — some features are used rarely but are critical (e.g., disaster recovery). Note this caveat.
- If `activity_name` values are opaque (e.g., internal event names), ask the user for a glossary or map names to human-readable labels.
