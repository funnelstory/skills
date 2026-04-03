# QBR Deck Generator

Generate professional Quarterly Business Review decks for individual accounts using FunnelStory data.

## When to Use This Sub-Skill

Use this whenever a user requests a QBR deck or quarterly business review for an account. The skill:

- Queries the FunnelStory semantic database for account-specific data
- Extracts relevant business metrics, notes, activities, tickets, meetings, and insights
- Generates a structured, presentation-ready deck (PowerPoint by default)

---

## Step 1: Identify the Account

Ask the user which account they want to generate a QBR deck for, then find it:

```sql
SELECT account_id, name, domain, amount, health_score, prediction_score, expires_at
FROM accounts
WHERE name LIKE '%AccountName%'
LIMIT 10
```

Confirm the correct account with the user before proceeding. Note the `account_id` and `domain` — you'll need both for joins.

---

## Step 2: Pull Account Data

Run these queries in parallel. All timestamps use `datetime('now', '-90 days')` style. The DB is SQLite with ISO 8601 timestamp strings.

### Account Details

```sql
SELECT *
FROM accounts
WHERE account_id = '<account_id>'
```

Key fields to extract:

- `amount` — ARR
- `expires_at` — renewal date
- `health_score` — overall health (1–10)
- `prediction_score` — churn/retention score (-1 to 1). Convert to 0–100 via `(prediction_score * 50) + 50`. Lower = higher churn risk.
- `license_utilization`, `feature_adoption`, `activity_score`, `support_sentiment`, `conversation_sentiment`
- `properties` JSON — contains CSM, account owner, TAM, renewal specialist, `num_contract_devices`, `num_active_devices`, `num_inactive_devices`, region, tenant_settings

### Notes (last 90 days)

```sql
SELECT id, title, note_type, created_at, created_by_email, content, link
FROM notes
WHERE account_id = '<account_id>'
  AND created_at >= datetime('now', '-90 days')
ORDER BY created_at DESC
LIMIT 20
```

Notes often contain the richest qualitative context — action items, customer feedback, internal observations. Always read them. Note: `content` is HTML; strip tags before displaying.

### Activities (last 90 days)

```sql
SELECT activity_name, SUM(count) AS total, MAX(timestamp) AS last_seen
FROM activities
WHERE account_id = '<account_id>'
  AND timestamp >= datetime('now', '-90 days')
GROUP BY activity_name
ORDER BY total DESC
```

### Support Tickets (last 90 days)

Tickets link to accounts via contacts → domain:

```sql
SELECT t.title, t.status, t.priority, t.sentiment, t.timestamp, t.resolved_at,
       t.contact_name, t.assignee_email, t.link,
       json_extract(t.key, '$.ticket_id') AS ticket_number
FROM tickets t
JOIN ticket_contacts tc ON tc.ticket_id = t.id
JOIN contacts c ON c.id = tc.contact_id
WHERE c.domain = '<account_domain>'
  AND t.timestamp >= datetime('now', '-90 days')
ORDER BY t.timestamp DESC
LIMIT 20
```

### Meetings (last 90 days)

Meetings link to accounts via meeting_contacts → contacts → domain:

```sql
SELECT m.title, m.summary, m.timestamp, m.duration_seconds, m.sentiment,
       m.participants, m.link
FROM meetings m
JOIN meeting_contacts mc ON mc.meeting_id = m.id
JOIN contacts c ON c.id = mc.contact_id
WHERE c.domain = '<account_domain>'
  AND m.timestamp >= datetime('now', '-90 days')
ORDER BY m.timestamp DESC
LIMIT 15
```

For high-stakes accounts, read the full `summary` text rather than relying only on needle movers — raw summaries contain specific initiatives, competitor names, and expansion signals that pre-computed insights may miss.

### Needle Movers

```sql
SELECT id, title, description, state, impact, label, created_at, updated_at, assigned_to_email
FROM needle_movers
WHERE INSTR(account_ids, '<account_id>') > 0
ORDER BY ABS(impact) DESC
```

**Important**: `json_each` is not available on these virtual tables. Always use `INSTR(account_ids, account_id) > 0` to filter.

Labels: `competitor`, `feature_request`, `task_issue_bug`, `pricing`, `personnel_change`, `action_item`
Impact: -10 (critical churn risk) to +10 (critical expansion). Negative = risk, positive = growth.
States: `open`, `closed`, `acknowledged`, `done`

### Prediction History (trend)

```sql
SELECT timestamp, score
FROM prediction_account_history
WHERE account_id = '<account_id>'
ORDER BY timestamp DESC
LIMIT 6
```

Score is -1 to 1; convert to 0–100 for display. The `point` JSON column contains `factors` (top contributing features) and `recommendations` (features to increase/decrease) — parse if including AI-driven recommendations.

### Metric History

```sql
SELECT metric_id, value, timestamp
FROM account_metrics_history
WHERE account_id = '<account_id>'
ORDER BY timestamp DESC
LIMIT 20
```

### Open Tasks

```sql
SELECT title, body, status, expires_at, assigned_to_email, created_at, link
FROM tasks
WHERE account_id = '<account_id>'
  AND status != 'done'
ORDER BY expires_at ASC
LIMIT 10
```

---

## Step 3: Structure the QBR Deck

Read `/mnt/skills/public/pptx/SKILL.md` before creating the PowerPoint.

Build the deck with these sections:

1. **Title Slide** — Account name, quarter (e.g. Q1 2026), date, CSM name (from `properties.CSM`)
2. **Executive Summary** — Health score, prediction score (0–100), ARR, renewal date, 2–3 key wins, top risks
3. **Account Overview** — Contract value, devices (contracted / active / inactive from `properties`), total users, region, account owner, renewal specialist
4. **Health & Prediction Trend** — Health score, prediction score trend from `prediction_account_history`, top risk factors from `point.factors`
5. **Product Usage & Engagement** — Device usage from `account_metrics_history`, activity breakdown from `activities` (device registrations, threats found/blocked, decision overrides, logins, policy changes)
6. **Meetings & Notes** — Key takeaways from `meetings.summary` and `notes.content` (strip HTML), action items, participants
7. **Support** — Open ticket count, recently resolved tickets, avg sentiment, notable issues from `tickets`
8. **Needle Movers** — Open needle movers ranked by `ABS(impact)`, grouped by label; flag critical negatives (impact ≤ -5)
9. **Next Quarter Priorities** — Recommended actions based on open needle movers, prediction `recommendations`, and renewal timeline
10. **Action Items** — Owners, due dates, links

---

## Step 4: Output Format

- **Default**: PowerPoint (.pptx) — read and follow `/mnt/skills/public/pptx/SKILL.md`
- **Alternative**: Word (.docx) — read and follow `/mnt/skills/public/docx/SKILL.md`
- Save final file to `/mnt/user-data/outputs/`

---

## Implementation Notes

- **JSON arrays**: `json_each` is not available. Use `INSTR(col, value) > 0` for array membership (e.g. `needle_movers.account_ids`).
- **JSON field extraction**: Use `json_extract(col, '$.field')` — e.g. `json_extract(key, '$.ticket_id')` for user-facing ticket numbers.
- **Timestamps**: Use `datetime('now', '-90 days')` for relative time. All timestamps are ISO 8601 strings.
- **Prediction score display**: Convert -1→1 scale to 0–100 via `(score * 50) + 50`.
- **Notes content**: Strip HTML tags before including in slides.
- **Missing data**: Skip sections gracefully if data is unavailable; note it on the slide.
- **Sparse needle movers**: If empty, lean more heavily on meeting summaries and notes for qualitative context — they often contain signals needle movers didn't capture.
