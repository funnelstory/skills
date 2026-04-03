# Book of Business — CSM Portfolio View

Generate a single-pane summary of all accounts assigned to the current user, surfacing health, risk, renewals, activity, and recommended focus areas. Output as clean markdown in chat with numbered accounts for drill-down.

---

## Step 1 — Find the User's Accounts

The parent SKILL.md handles user identification (Step 0). Use the resolved email and account list from there.

If accounts were not yet resolved, try these strategies in order until one returns results.

### Strategy A — Semantic DB (workspaces with CSM in properties)

If the workspace has a semantic database with `properties` containing CSM or owner email:

```sql
SELECT account_id, name, domain, amount, health_score, prediction_score, expires_at
FROM accounts
WHERE json_extract(properties, '$.csm_email') = '<user_email>'
   OR json_extract(properties, '$.account_owner') = '<user_email>'
   OR json_extract(properties, '$.csm') = '<user_email>'
```

### Strategy B — get_account with assignees (demo / smaller workspaces)

If Strategy A fails or no semantic DB is available:

1. Query the data connection for all account identifiers:

```sql
SELECT account_id, name FROM accounts ORDER BY name
```

Use `get_data_connections` to find the connection id. Set `limit` high enough to return all rows.

2. Call `get_account` for each account_id. The response includes an `assignees` array:

```json
"assignees": [
  { "id": "...", "email": "arun+support@funnelstory.ai", "name": "Arun Balakrishnan" }
]
```

3. Keep only accounts where any `assignees[].email` matches the user's email (case-insensitive).

**Performance note:** If the account list exceeds ~50 accounts, process in batches and show progress. If it exceeds ~200, fall back to Strategy C.

### Strategy C — Ask the user

If both strategies fail or the account list is too large to scan:

> "I couldn't automatically find your assigned accounts. Can you list the account names you manage?"

Then resolve each name with `get_account` or a name search query.

---

## Step 2 — Collect Data for Each Account

For each account in the book, gather data from two sources:

### From get_account (already fetched in Strategy B, or fetch now)

Each `get_account` response contains:

- **`name`**, **`id`** — display name and account id
- **`amount`** — ARR / contract value
- **`expires_at`** — renewal date (ISO 8601)
- **`prediction`** — `{ outcome, confidence, health_score }`
- **`metrics[]`** — array of typed metrics:
  - `health_score` — overall health (0–100)
  - `activity_score` — product activity volume
  - `feature_adoption` — 0–1 scale
  - `product_engagement` — categorical (daily_active, weekly_active, etc.)
  - `conversation_sentiment` — -1 to 1
  - `support_sentiment` — -1 to 1
  - `total_support_tickets` — count
  - `total_users` — user count
- **`assignees[]`** — who manages the account
- **`traits`** — company metadata (size, country, industry)

Extract what you need from each account's response. Do not re-fetch if already retrieved during Step 1.

### From data connection queries (batch across all accounts)

Use `execute_query` against the data connection. Substitute the user's account_id list.

**Recent notes (last 14 days):**

```sql
SELECT n.account_id, n.title, n.note_type, n.created_at
FROM notes n
WHERE n.account_id IN ('<id1>', '<id2>', ...)
  AND n.created_at >= datetime('now', '-14 days')
ORDER BY n.created_at DESC
LIMIT 30
```

If `notes` table is not available on the data connection, skip this section — note it as "Notes: not available on this data connection."

**Open needle movers:**

```sql
SELECT nm.title, nm.description, nm.state, nm.impact, nm.label, nm.account_ids,
       nm.assigned_to_email, nm.updated_at
FROM needle_movers nm
WHERE nm.state IN ('open', 'acknowledged')
ORDER BY ABS(nm.impact) DESC
LIMIT 30
```

Then filter in code: keep only needle movers whose `account_ids` contains any of the user's account ids (use `INSTR` if filtering in SQL, or filter post-query).

**If the data connection does not have these tables** (common in demo workspaces with raw Postgres), fall back to the MCP tools:

- Use `list_needle_movers` if available
- Use `get_needle_mover` for specific items
- Or simply note "Needle movers / notes: use FunnelStory UI for detailed view" and proceed with what `get_account` provides

---

## Step 3 — Write the Portfolio Brief

Output as clean, scannable markdown in the chat. **Number every account** so the user can drill down by typing a number.

### Section 1: Portfolio Snapshot

```
## Portfolio Snapshot

| Metric | Value |
|--------|-------|
| Accounts | {count} |
| Total ARR | ${sum of amount, formatted} |
| Avg Health Score | {average health_score across accounts} |
| Accounts at Risk | {count where health_score < 60 or prediction.outcome == 'churn'} |
| Renewals in 90 days | {count where expires_at within 90 days} |
```

### Section 2: Account-by-Account Summary

For each account, output a **numbered block**. Sort: accounts requiring attention first (lowest health, then soonest renewal).

Format per account:

```markdown
### 1. Accenture — $93,760 ARR

**Health:** 99 · **Prediction:** Retention (High) · **Engagement:** Daily Active · **Users:** 95
**Renewal:** Aug 16, 2026 (136 days) · **Feature Adoption:** 78% · **Sentiment:** Neutral

Strong engagement with daily active usage. Renewal in 4 months with no blockers.
Feature adoption at 78% — room to grow. Conversation sentiment neutral; monitor for drift.
```

```markdown
### 2. American Financial Group — $47,230 ARR 🔴

**Health:** 45 · **Prediction:** Churn (Medium) · **Engagement:** Monthly Active · **Users:** 36
**Renewal:** Jun 30, 2026 (89 days) · **Feature Adoption:** 22% · **Sentiment:** Negative

At risk. Health score critically low at 45, engagement dropped to monthly active, and
feature adoption at 22% suggests most teams aren't using the platform. Support sentiment
negative with recurring ticket escalations. Renewal in 89 days with no executive sponsor.
```

**Rules for each account block:**

- Use the numbered heading format: `### 1. Account Name — $ARR`
- Append a colored indicator after ARR only for non-healthy accounts: 🔴 (health < 50 or churn prediction), 🟡 (health 50–69 or renewal within 60 days)
- Healthy accounts get no indicator (clean = good)
- **Metrics line:** Bold labels, dot-separated. Show all available metrics on 1–2 lines.
- **Renewal:** Show date + days remaining in parentheses
- **AI Summary:** 2–3 sentences grounded in specific data. Must reference at least two numbers. No generic "doing well." Explain _why_ the account is healthy or _what_ makes it risky.
- **Sentiment display:** convert -1→1: < -0.1 = "Negative", -0.1 to 0.1 = "Neutral", > 0.1 = "Positive"
- **Feature adoption:** display as percentage (value × 100)
- **Engagement:** display the category as-is: Daily Active, Weekly Active, Monthly Active, etc.

### Section 3: This Week's Focus

Synthesize everything into **3–5 concrete recommended actions**:

```markdown
## This Week's Focus

1. **American Financial Group** — Schedule executive rescue sync with VP Engineering.
   Health at 45, renewal in 89 days, support sentiment negative.

2. **Accenture** — Prepare renewal proposal. Renewal in 136 days but conversation
   sentiment trending neutral — get ahead of it.
```

Each action: bold account name, specific action, data-grounded reason on the next line.

### Section 4: Drill-down prompt

End the output with:

```markdown
---

**Type a number (e.g. `1`) to drill into any account** — I'll pull the full summary, recent notes, tickets, and needle movers.
```

---

## Drill-down behavior

When the user types a number or account name after seeing the book:

1. Identify which account they mean from the numbered list.
2. Trigger a **full account deep-dive** using the same data-gathering pattern as the `account-brief` sub-skill:
   - Account details and properties
   - Recent notes (strip HTML, show content)
   - Activities (last 90 days, grouped by type)
   - Support tickets (last 90 days)
   - Needle movers (sorted by impact)
   - Prediction history and risk factors
   - Key contacts
   - Open tasks
3. Present as a structured markdown brief (same style as the `account-brief` sub-skill output).
4. End with: "Back to your book of business, or drill into another account?"

This creates a **navigate → drill → navigate** loop that feels interactive without requiring clickable elements.

---

## Tone and Rules

- **Audience:** Customer Success Manager scanning before or during their day. Keep it tight.
- **No raw IDs** — use account names everywhere.
- **No fabricated data** — if a section has no data, say "No data available" and move on.
- **Health display** — use the integer from `metrics[type=health_score].value` directly (0–100 scale).
- **Prediction display** — use `prediction.outcome` (retention/churn) and `prediction.confidence` (high/medium/low).
- **ARR display** — format with `$` and comma thousands (e.g. `$93,760`).
- **Dates** — `Mon DD, YYYY` in summaries; include days remaining for renewals.
- **Missing sections** — if notes, needle movers, or tickets aren't queryable, skip those sections with a one-liner explaining why, and strengthen the account-by-account commentary using `get_account` data.
