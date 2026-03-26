---
name: funnelstory-account-brief
description: >
  Use this skill to generate an account summary brief using the FunnelStory MCP.
  Trigger this skill whenever the user mentions "account summary", "account brief",
  "customer brief", or asks to summarize, review, or report on a specific customer
  or account — even if they don't say "FunnelStory" explicitly. Also trigger when
  the user asks about account health, usage, contacts, support tickets, or purchased
  products for a named account.
compatibility: "Requires FunnelStory MCP to be connected (SQLite semantic database)"
---

# FunnelStory Account Brief Skill

Generate a structured, professional account summary brief for a named customer using FunnelStory's semantic database.

---

## Step 1 — Identify the Account

If the user hasn't specified an account name, ask for it. Then look it up:

```sql
SELECT account_id, name, domain, amount, health_score, prediction_score, expires_at
FROM accounts
WHERE name LIKE '%AccountName%'
LIMIT 10
```

Confirm the correct account with the user before proceeding. Note the `account_id` and `domain` — you'll need both for subsequent queries.

If no match is found, ask the user to verify the account name or try a partial/alternate spelling.

---

## Step 2 — Gather Data

Run all queries using the confirmed `account_id` and `domain`. All timestamps use SQLite ISO 8601 strings. Use `datetime('now', '-90 days')` for the 90-day window.

### Account Details

```sql
SELECT *
FROM accounts
WHERE account_id = '<account_id>'
```

Key fields:

- `amount` — ARR
- `expires_at` — renewal date
- `health_score` — overall health (1–10)
- `prediction_score` — churn/retention score (-1 to 1); convert to 0–100 via `(score * 50) + 50`. Lower = higher churn risk.
- `license_utilization`, `feature_adoption`, `activity_score`, `support_sentiment`
- `properties` JSON — CSM, account owner, TAM, renewal specialist, `num_contract_devices`, `num_active_devices`, `num_inactive_devices`, region, tenant settings

### Product Usage & Activities (last 90 days)

```sql
SELECT activity_name, SUM(count) AS total, MAX(timestamp) AS last_seen
FROM activities
WHERE account_id = '<account_id>'
  AND timestamp >= datetime('now', '-90 days')
GROUP BY activity_name
ORDER BY total DESC
```

### Metric History

```sql
SELECT metric_id, value, timestamp
FROM account_metrics_history
WHERE account_id = '<account_id>'
ORDER BY timestamp DESC
LIMIT 20
```

### Notes (last 90 days)

```sql
SELECT id, title, note_type, created_at, created_by_email, content, link
FROM notes
WHERE account_id = '<account_id>'
  AND created_at >= datetime('now', '-90 days')
ORDER BY created_at DESC
LIMIT 20
```

Notes often contain the richest qualitative context — action items, customer feedback, internal observations. Always read them. Strip HTML tags from `content` before displaying.

### Key Contacts

```sql
SELECT name, email, title, role
FROM contacts
WHERE domain = '<account_domain>'
ORDER BY role ASC
LIMIT 20
```

### Support Tickets (last 90 days)

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

### Needle Movers

```sql
SELECT id, title, description, state, impact, label, created_at, updated_at, assigned_to_email
FROM needle_movers
WHERE INSTR(account_ids, '<account_id>') > 0
ORDER BY ABS(impact) DESC
```

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

Convert score to 0–100 for display. The `point` JSON column contains `factors` (top contributing features) and `recommendations` — parse these for the Signals section.

If any section returns no data, note "No data available" in the brief rather than omitting the section.

---

## Step 3 — Confirm Output Format

Default to **PDF**. Before generating, ask the user:

> "I'll generate this as a PDF. Would you prefer a different format — Markdown in chat or a Word document?"

If they confirm PDF (or don't respond with a preference), proceed. Adjust if they specify otherwise.

---

## Step 4 — Write the Brief

Structure:

```
[Account Name] — Account Summary Brief
Generated: [Today's date]
Prepared by: Claude + FunnelStory

─────────────────────────────────────────
1. ACCOUNT OVERVIEW
   - created at, domain
   - Contracted Agents / active Agents / inactive Agents

2. KEY CONTACTS
   - Name | Title | Role (Champion / Decision-Maker / User / etc.)

3. PRODUCT USAGE & ENGAGEMENT
   - Device utilization (contracted vs. active vs. inactive)
   - Top activities by volume (last 90 days)
   - Notable metric trends from account_metrics_history

4. NEEDLE MOVERS
   - Key recent and high impact Needle Movers with Categories
   - Open needle movers ranked by ABS(impact); flag anything ≤ -5 as critical


5. SUPPORT TICKET ANALYSIS
   - Open tickets | Recently resolved tickets | Avg sentiment
   - Summary of themes/categories
   - Notable high-priority issues

6. SIGNALS & RECOMMENDATIONS
   - Health score | Prediction score (0–100 scale)
   - Prediction risk factors and recommended actions
   - Suggested next steps for the account team

─────────────────────────────────────────
```

Keep tone professional and concise. Bullets over paragraphs. Surface insights — don't dump raw data. If needle movers are sparse, lean on notes and ticket summaries for qualitative signals.

---

## Step 5 — Generate the Output

### PDF (default)

Read and follow `/mnt/skills/public/pdf/SKILL.md`. Use `reportlab` to produce a clean, formatted PDF with section headers, dividers, and a cover line (account name + date).
Save to `/mnt/user-data/outputs/[account-name]-account-brief.pdf` and use `present_files`.

### Markdown (in-chat)

Render the structured brief directly using Markdown headers and tables.

### Word Document (.docx)

Read and follow `/mnt/skills/public/docx/SKILL.md`.
Save to `/mnt/user-data/outputs/[account-name]-account-brief.docx` and use `present_files`.

---

## Implementation Notes

- **JSON arrays**: `json_each` is not available. Use `INSTR(col, value) > 0` for array membership (e.g. `needle_movers.account_ids`).
- **JSON field extraction**: Use `json_extract(col, '$.field')` — e.g. `json_extract(key, '$.ticket_id')`.
- **Prediction score display**: Convert -1→1 to 0–100 via `(score * 50) + 50`.
- **Notes content**: Strip HTML tags before including in the brief.
- **Timestamps**: All ISO 8601 strings; use `datetime('now', '-90 days')` for relative ranges.
- **Missing data**: Note "No data available" per section; don't silently skip.
- **MCP not connected**: Surface a clear message — "FunnelStory MCP isn't connected. Please enable it in your tools/connectors settings."
