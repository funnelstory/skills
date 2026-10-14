# Book of Business — CSM Portfolio Dashboard

Generate an interactive HTML dashboard of all accounts assigned to the current user. Surfaces health, risk tiers, renewals, adoption, and recommended actions. Uses the same card-based layout as other dashboards (see `templates/html-dashboard.md`).

---

## Step 1 — Find the User's Accounts

The parent SKILL.md handles user identification (Step 0). Use the resolved email and account list from there.

If accounts were not yet resolved, try these strategies in order until one returns results.

### Strategy A — Assignees column

Use the `assignees` JSON array on the `accounts` table:

```sql
SELECT account_id, name, domain, aggregate_amount, health_score,
       earliest_child_expiry, feature_adoption, product_engagement,
       activity_score, support_sentiment, conversation_sentiment,
       license_utilization, total_users, assignees, parent_account_id
FROM accounts
WHERE (churned = 0 OR churned IS NULL)
  AND EXISTS (
    SELECT 1 FROM json_each(assignees) WHERE json_each.value = '<user_email>'
  )
ORDER BY health_score ASC
```

### Strategy B — Account properties

If Strategy A returns no results (the workspace may store ownership in account properties instead):

```sql
SELECT account_id, name, domain, aggregate_amount, health_score,
       earliest_child_expiry, feature_adoption, product_engagement,
       activity_score, support_sentiment, conversation_sentiment,
       license_utilization, total_users, parent_account_id
FROM accounts
WHERE (churned = 0 OR churned IS NULL)
  AND (json_extract(properties, '$.csm_email') = '<user_email>'
   OR json_extract(properties, '$.account_owner') = '<user_email>'
   OR json_extract(properties, '$.csm') = '<user_email>')
ORDER BY health_score ASC
```

### Strategy C — Ask the user

If both strategies return no results:

> "I couldn't automatically find accounts assigned to your email. Can you list the account names you manage?"

Then resolve each name with a name search query.

---

## Step 2 — Collect Enrichment Data

For each account in the book, gather additional signals.

### Recent notes (last 14 days)

```sql
SELECT n.account_id, n.title, n.note_type, n.created_at
FROM notes n
WHERE n.account_id IN ('<id1>', '<id2>', ...)
  AND n.created_at >= datetime('now', '-14 days')
ORDER BY n.created_at DESC
LIMIT 30
```

If the `notes` table is unavailable, skip — note it in the output footer.

### Open needle movers

```sql
SELECT nm.title, nm.description, nm.state, nm.impact, nm.label, nm.account_ids,
       nm.assigned_to_email, nm.updated_at
FROM needle_movers nm
WHERE nm.state = 'open'
  AND nm.source IN ('v2', 'relationship_map')
ORDER BY ABS(nm.impact) DESC
LIMIT 50
```

Filter post-query: keep only needle movers whose `account_ids` contains any of the user's account IDs (use `INSTR(account_ids, '<account_id>') > 0`).

---

## Step 3 — Classify Each Account

Band each account using **Health tiers** in the root `SKILL.md`:

| Tier | Health score |
|------|----------------|
| **Critical** | 45 or below |
| **At risk** | 46 to 54 |
| **On track** | 55 or above |

No score is **Unscored**, not a tier. Report other signals (prediction outcome, negative needle movers, declining activity or adoption, negative sentiment, renewal within 60 days) as drivers beside the tier. Do not change the tier because of them. Value on a card is `aggregate_amount`. Renewal date is `earliest_child_expiry` when set, otherwise `expires_at`. `product_engagement` is 0–3 (inactive, monthly, weekly, daily), not a percentage.

---

## Step 4 — Build the Dashboard

Output as a **self-contained HTML file** using the layout from `templates/html-dashboard.md`.

### KPI row (4 boxes)

| Box | Value |
|-----|-------|
| Accounts | Total count in book |
| Total value | Sum of `aggregate_amount`, labelled per the workspace profile, formatted as `$X.XXM` or `$XXK` |
| Critical | Count + value subtotal (use `--danger` color) |
| At risk | Count + value subtotal (use `--warning` color) |
| On track | Count + value subtotal (use `--success` color) |

### Filter buttons

Row of toggle buttons below the KPI row: **All ({count})** · **Critical ({count})** · **At risk ({count})** · **On track ({count})** · **Unscored ({count})**

Implement as JavaScript click handlers that show/hide cards by `data-tier` (`critical` / `at_risk` / `on_track` / `unscored`).

### Account cards

Two-column grid (single column on mobile). Sorted: Critical first, then At risk, then On track, then Unscored. Within each tier, sort by renewal date ascending (soonest first).

Each card:

```
┌──────────────────────────────────────────────┐
│  {Account Name}                   [Critical] │
│  ${ARR} ARR · Renews {Mon DD}                │
│                                              │
│  Health {X}/100   Adoption {Y}%   Days left {N} │
│                                              │
│  • {Risk driver or positive signal #1}       │
│  • {Risk driver or positive signal #2}       │
│                                              │
│  ┌─ Suggested action ──────────────────────┐ │
│  │ {Concrete, account-specific action}     │ │
│  └─────────────────────────────────────────┘ │
└──────────────────────────────────────────────┘
```

**Card rules:**
- **Tier badge:** Colored pill in the top-right. Critical = `--danger`, At risk = `--warning`, On track = `--success`. Unscored has no color.
- **Metrics line:** Health score (0–100), Adoption (feature_adoption × 100, as %), Days to renewal.
- **Bullet points:** 2 data-grounded insights. For Critical and At risk: cite the specific risks. For On track: cite strengths and any areas to watch.
- **Suggested action:** One concrete next step in a highlighted box. Examples: "Schedule executive sponsor call — renewal in 12 days", "Send value recap email — health strong but sentiment declining", "Address open ticket on data export — blocking expansion."
- **Prediction score display:** Convert -1→1 to 0–100 via `(score * 50) + 50` if needed.
- **Feature adoption:** Display as percentage (value × 100 if stored as 0–1).
- **Sentiment:** Convert -1→1 scale: < -0.1 = negative, -0.1 to 0.1 = neutral, > 0.1 = positive.

### Example card

```html
<div class="card" data-tier="critical">
  <div class="card-header">
    <h3>{Account Name}</h3>
    <span class="badge critical">Critical</span>
  </div>
  <div class="card-meta">${ARR} ARR · Renews {Date}</div>
  <div class="card-metrics">
    <span>Health <strong>{X}</strong>/100</span>
    <span>Adoption <strong>{Y}</strong>%</span>
    <span>Days left <strong>{N}</strong></span>
  </div>
  <ul class="card-insights">
    <li>{Risk driver #1 — cite specific data}</li>
    <li>{Risk driver #2 — cite specific data}</li>
  </ul>
  <div class="card-action">
    <strong>Suggested action</strong><br>
    {One concrete, account-specific next step with data-backed urgency.}
  </div>
</div>
```

### Footer

`"Generated by FunnelStory · {Date} · {N} accounts"`. If any data sources were unavailable, note them here.

---

## Drill-down behavior

After viewing the dashboard, the user may ask about a specific account by name. When they do:

1. Identify the account from context.
2. Trigger a **full account deep-dive** using the `account-brief/` sub-skill pattern:
   - Account details and properties
   - Recent notes (strip HTML, show content)
   - Activities (last 90 days, grouped by type)
   - Support tickets (last 90 days)
   - Needle movers (sorted by impact)
   - Prediction history and risk factors
   - Key contacts and open tasks
3. Present as a structured markdown brief.
4. End with: "Back to your book of business, or drill into another account?"

---

## Tone and Rules

- **Audience:** CSM scanning their portfolio. Keep it tight.
- **No raw IDs** — use account names everywhere.
- **No fabricated data** — if a section has no data, say "No data available" and move on.
- **Health display** — integer 0–100, tiered per **Health tiers** in the root `SKILL.md`.
- **ARR display** — format with `$` and comma thousands (e.g. `$93,760`) or abbreviated (e.g. `$43K`).
- **Dates** — `Mon DD` on cards, full `Mon DD, YYYY` in drill-downs. Include days remaining for renewals.
- **Missing data** — if notes, needle movers, or tickets aren't queryable, skip those signals and note it in the footer. Strengthen card insights using available account-level data.
