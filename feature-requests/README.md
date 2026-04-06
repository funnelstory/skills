# Feature Requests Dashboard

Aggregate feature requests across accounts from `needle_movers`, `notes`, `topics`, and `tickets`. Rank by frequency, ARR weight, and segment. For PM and VP of Product to prioritize the product roadmap based on customer signal.

## Audience

- **PM:** Understand which features customers are asking for most, weighted by account value and frequency.
- **VP of Product:** Strategic view of demand patterns — which themes matter to which segments.

## Step 1 — Scope

Ask or infer:

1. **Segment filter** (optional): All accounts (default), or filter by segment/tier/region via `json_extract(properties, '$.plan')`, `json_extract(properties, '$.segment')`, or similar.
2. **Time window:** Default last 90 days.
3. **Output:** Default **HTML dashboard** (see `templates/html-dashboard.md`).

## Step 2 — Gather feature request signals

### From needle movers (primary source)

```sql
SELECT title, description, account_ids, impact, created_at, updated_at, state
FROM needle_movers
WHERE label = 'feature_request'
  AND created_at >= datetime('now', '-90 days')
ORDER BY created_at DESC
LIMIT 100
```

Each needle mover with `label = 'feature_request'` is an explicit feature request. The `account_ids` field links it to one or more accounts. Use `INSTR` to match.

### From notes (secondary — keyword scan)

```sql
SELECT n.account_id, n.title, n.content, n.created_at
FROM notes n
WHERE n.created_at >= datetime('now', '-90 days')
  AND (n.content LIKE '%feature request%'
    OR n.content LIKE '%would like%'
    OR n.content LIKE '%asked for%'
    OR n.content LIKE '%roadmap%'
    OR n.content LIKE '%wish list%')
ORDER BY n.created_at DESC
LIMIT 50
```

Strip HTML from content. Extract the specific feature mentioned.

### From topics (if available)

```sql
SELECT topic_name, account_id, sentiment, timestamp
FROM topics
WHERE topic_name LIKE '%feature%' OR topic_name LIKE '%request%'
ORDER BY timestamp DESC
LIMIT 50
```

### Enrich with account context

For each unique account referenced, pull ARR and segment info:

```sql
SELECT account_id, name, amount, json_extract(properties, '$.segment') AS segment,
       json_extract(properties, '$.plan') AS plan
FROM accounts
WHERE account_id IN ('<id1>', '<id2>', ...)
```

## Step 3 — Aggregate and rank

Group feature requests by theme/title. For each theme:
- **Frequency:** Number of accounts requesting it
- **ARR weight:** Sum of `amount` for requesting accounts
- **Segment distribution:** Which segments are asking (Enterprise vs SMB, etc.)
- **Recency:** Most recent request date
- **Sentiment:** Average sentiment from related topics/notes (if available)

Rank by a composite of frequency + ARR weight. Document the ranking formula.

## Step 4 — Output

### KPI row
- Total unique feature requests
- Accounts involved
- Total ARR represented
- Top segment requesting

### Ranked list (cards or table)

| Rank | Feature request | Accounts | ARR | Segments | Last requested | Status |
|------|----------------|----------|-----|----------|---------------|--------|
| 1 | {Title} | {N} | ${sum} | {list} | {date} | Open/Closed |

For the top 5, include a detail section:
- Requesting accounts (names)
- Representative quotes from notes (if available)
- Related needle mover states (open/closed/done)

### Themes section (optional)
If requests cluster into themes (e.g., "integrations", "reporting", "API"), show theme-level aggregates.

## Quality bar

- Do NOT conflate feature requests with bug reports — `label = 'feature_request'` is distinct from `label = 'task_issue_bug'`.
- ARR weighting should be transparent: show both raw frequency and ARR-weighted rank.
- If segment filter is applied, note it prominently on the dashboard header.
- Feature names should be normalized where possible (merge duplicates with similar titles).
