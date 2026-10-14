# QBR Deck — Internal

Generate an **internal-use** Quarterly Business Review deck for a named account. This deck is for the CS team, leadership, and cross-functional stakeholders — **NOT for sharing directly with the customer.**

It focuses on risks, concerns, internal metrics, and honest assessment of account health. For a customer-facing QBR, use the `qbr-external/` sub-skill instead.

## When to Use This Sub-Skill

Use this when the user asks for an internal QBR, a QBR prep deck, or a quarterly review for internal stakeholders. The skill:

- Queries the FunnelStory semantic database for account-specific data
- Extracts relevant business metrics, notes, activities, tickets, meetings, and insights
- Generates a structured, presentation-ready deck (PowerPoint by default)

---

## Step 1: Identify the Account

If the user hasn't named an account, ask for it. Then resolve it:

```sql
SELECT account_id, name, domain, amount, health_score, prediction_score, expires_at
FROM accounts
WHERE name LIKE '%AccountName%' OR domain LIKE '%AccountName%'
LIMIT 25
```

Follow **Resolving a named account** in the root `SKILL.md`: rank, proceed with the top candidate, state the choice, and list alternatives. Ask first only for a parent-vs-site ambiguity. Note the `account_id` and `domain`.

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

Field names below are defaults; the workspace profile `instructions` may rename or redirect them.

- `amount` — contract value; report it under the name the workspace profile specifies (e.g. ARR)
- `expires_at` — renewal date
- `health_score` — displayed health, already 0–100. Band it per **Health tiers** in the root `SKILL.md`
- `prediction_score` — churn/retention score (-1 to 1). Convert to 0–100 via `(prediction_score * 50) + 50`. Lower = higher churn risk.
- `license_utilization`, `feature_adoption`, `activity_score`, `support_sentiment`, `conversation_sentiment`
- `properties` JSON — typically CSM, account owner, renewal owner, region, and any workspace-specific entitlement fields (e.g. contracted vs. active seats or units). Inspect the keys present rather than assuming names.

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

### Support History (last 90 days)

Do not use a fixed ticket query. Read **`references/linking-conversations-to-accounts.md`** and follow it: confirm which table holds support data in this workspace, choose the highest-coverage linkage per source, and exclude machine-generated noise. Keep tickets and chat conversations as separate bands, and record a one-line provenance note (table, source, linkage, coverage) for the Support slide.

Never `SELECT *` from `tickets` or `conversations`.

### Meetings (last 90 days)

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

This join matches contacts by domain. If other accounts share this domain (check with `SELECT COUNT(*) FROM accounts WHERE domain = '<account_domain>'`), the results can include their meetings — say so on the slide rather than presenting them as this account's.

For high-stakes accounts, read the full `summary` text rather than relying only on needle movers — raw summaries contain specific initiatives, competitor names, and expansion signals that pre-computed insights may miss.

### Needle Movers

```sql
SELECT id, title, description, state, impact, label, created_at, updated_at, assigned_to_email
FROM needle_movers
WHERE INSTR(account_ids, '<account_id>') > 0
ORDER BY ABS(impact) DESC
```

**Important**: The `account_ids` field is a text column, not a JSON array. Use `INSTR(account_ids, '<account_id>') > 0` to filter.

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

See **Output files** in the root `SKILL.md` for how to produce the deck.

Build the deck with these sections:

1. **Title Slide** — Account name, quarter (e.g. Q1 2026), date, CSM name (from `properties.CSM`). Label: "Internal QBR — Not for customer distribution."
2. **Executive Summary** — Health score and risk band, prediction score (0–100, if the workspace allows surfacing it), contract value, renewal date, 2–3 key wins, top risks
3. **Account Overview** — Contract value, entitlement vs. active usage (from workspace-specific `properties` fields, if present), total users, region, account owner, renewal owner
4. **Health & Prediction Trend** — Health score, prediction score trend from `prediction_account_history`, top risk factors from `point.factors`
5. **Product Usage & Engagement** — Usage trends from `account_metrics_history`, activity breakdown from `activities`
6. **Meetings & Notes** — Key takeaways from `meetings.summary` and `notes.content` (strip HTML), action items, participants
7. **Support** — Open items first, recently resolved, sentiment, notable issues; tickets and conversations as separate bands; provenance line
8. **Needle Movers** — Open needle movers with `source IN ('v2', 'relationship_map')`, grouped by label; flag **negative high** (impact −7 or below). Report the tier, not the raw integer.
9. **Next Quarter Priorities** — Recommended actions based on open needle movers, prediction `recommendations`, and renewal timeline
10. **Action Items** — Owners, due dates, links

---

## Step 4: Output Format

- **Default**: PowerPoint (.pptx)
- **Alternative**: Word (.docx)
- Follow **Output files** in the root `SKILL.md`.

---

## Implementation Notes

- **JSON arrays**: Use `json_each(col)` for proper JSON array columns (e.g. `accounts.assignees`). For text-based array fields like `needle_movers.account_ids`, use `INSTR(col, value) > 0` instead.
- **JSON field extraction**: Use `json_extract(col, '$.field')` — e.g. `json_extract(key, '$.ticket_id')` for user-facing ticket numbers.
- **Timestamps**: Use `datetime('now', '-90 days')` for relative time. All timestamps are ISO 8601 strings.
- **Prediction score display**: Convert -1→1 scale to 0–100 via `(score * 50) + 50`.
- **Notes content**: Strip HTML tags before including in slides.
- **Missing data**: Keep the slide and state either "No data available" or "Not available in this workspace" — never drop it silently.
- **Support data**: always via `references/linking-conversations-to-accounts.md`.
- **Sparse needle movers**: If empty, lean more heavily on meeting summaries and notes for qualitative context — they often contain signals needle movers didn't capture.
