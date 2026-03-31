---
name: funnelstory-prioritize-my-day
description: >
  FunnelStory-first daily prioritization for CSMs. Trigger when the user says
  "Prioritize My Day", "what should I prioritize today", or "today's CSM priorities".
  The skill resolves the target CSM in the current workspace, then builds a Customer
  Success operating view across top priorities, alerts, health, renewals, adoption,
  segmentation, and next-playbook actions.
compatibility: "Requires FunnelStory MCP to be connected (SQLite semantic database)"
---

# Prioritize My Day

Return a FunnelStory-native daily operating view for a CSM.

This skill should feel like a CS command center, not a generic summary. It should answer:

1. Which 5 accounts need action today?
2. What alerts or signal changes just happened?
3. Which renewals, risks, and expansion motions matter now?
4. What should I do next on each account?

## CS Design Principles

Build the output the way a strong CSM thinks about their book:

1. Portfolio first, then account detail.
2. Separate motions: `risk`, `renewal`, and `expansion`.
3. Pair every alert with a recommended playbook or next step.
4. Combine quantitative signals with qualitative context.
5. Segment the book by tier, lifecycle, region, or other meaningful account properties when available.

## MCP Tools Used Under the Hood

Use these MCP tools directly:

1. `read_resource`
2. `query_semantic_db`

Always start with:

- `read_resource` -> `file://semantic/usage.md`
- `read_resource` -> `file://semantic/schema.sql`

Then run all retrieval using `query_semantic_db`.

Do not assume tables or columns without checking the schema resource first.

## Inputs

- Optional user identifier: email or name
- Optional account subset
- Optional focus area: `risk`, `renewal`, `expansion`
- Optional segmentation filters such as tier, lifecycle, or region

Default invocation:
`Prioritize My Day`

If no user is specified, infer the best matching active CSM and clearly state the assumption.

## Output Format

Return concise Markdown using these exact sections:

`[ TODAY'S PRIORITIES ]`
Top 5 accounts to act on today across risk, renewal, and expansion.

`[ ALERTS & SIGNALS ]`
Real-time triggers such as usage drops, inactivity, fresh needle movers, support spikes, and sentiment shifts.

`[ CUSTOMER HEALTH OVERVIEW ]`
Health score, prediction direction, and the main health drivers across only the CSM's owned accounts.

`[ REVENUE & RENEWALS ]`
Renewal pipeline, ARR at risk, ARR with upside, and near-term commercial pressure.

`[ SEGMENTATION FILTERS ]`
Tier, lifecycle, region, or other useful slices used to interpret the book.

`[ ADOPTION & ENGAGEMENT ]`
Usage trends, feature adoption, license utilization, inactivity, and momentum movers.

`[ TASKS & PLAYBOOKS ]`
What to do next, using a recommended motion such as `churn-risk`, `renewal`, `adoption`, `expansion`, or `escalation`.

`[ WHAT I ASSUMED ]`
Resolved user, filters applied, and any data gaps.

## Workflow

## Step 1 - Resolve Target CSM

If the user gives an email or name, use that.

Otherwise infer from active workspace users, preferring Customer Success roles/designations and recent activity.

```sql
SELECT user_id, name, email, user_role, user_designation, last_activity
FROM workspace_users
WHERE deactivated_at IS NULL
ORDER BY last_activity DESC
LIMIT 50
```

If there is ambiguity, choose the best match and explicitly state that assumption.

## Step 2 - Build the Owned Book

Use `accounts.properties` ownership fields first. Pull enough account metadata to support health, adoption, revenue, renewal, and segmentation views.

```sql
SELECT
  account_id,
  name,
  domain,
  amount,
  health_score,
  prediction,
  prediction_score,
  feature_adoption,
  license_utilization,
  activity_score,
  product_engagement,
  expires_at,
  json_extract(properties, '$.csm_email') AS csm_email,
  json_extract(properties, '$.csm_name') AS csm_name,
  json_extract(properties, '$.tier') AS tier,
  json_extract(properties, '$.lifecycle_stage') AS lifecycle_stage,
  json_extract(properties, '$.region') AS region
FROM accounts
WHERE LOWER(COALESCE(json_extract(properties, '$.csm_email'), '')) = LOWER('<user_email>')
LIMIT 250
```

If the owned book is sparse, extend scope using assigned tasks and assigned/open needle movers.

## Step 3 - Pull FunnelStory Signals in Parallel

### 3A) Needle movers

Needle movers are the strongest product-native strategic signal. Focus on open and recent items.

```sql
SELECT
  nm.id,
  nm.title,
  nm.description,
  nm.label,
  nm.state,
  nm.impact,
  nm.created_at,
  nm.updated_at,
  nm.assigned_to_email,
  a.account_id,
  a.name AS account_name,
  a.amount,
  a.expires_at,
  a.health_score,
  a.prediction,
  a.prediction_score
FROM needle_movers nm
JOIN accounts a ON INSTR(nm.account_ids, a.account_id) > 0
WHERE nm.state = 'open'
  AND (
    LOWER(COALESCE(nm.assigned_to_email, '')) = LOWER('<user_email>')
    OR LOWER(COALESCE(json_extract(a.properties, '$.csm_email'), '')) = LOWER('<user_email>')
  )
  AND nm.updated_at >= datetime('now', '-30 days')
ORDER BY nm.updated_at DESC
LIMIT 200
```

Use impact tiers from `semantic/usage.md`. Do not present raw impact numbers in the final write-up.

### 3B) Recent meetings

Meetings explain what changed and whether the issue is active now.

```sql
SELECT
  m.id,
  m.title,
  m.timestamp,
  m.sentiment,
  m.summary,
  m.link,
  a.account_id,
  a.name AS account_name,
  a.amount
FROM meetings m
JOIN meeting_contacts mc ON mc.meeting_id = m.id
JOIN contacts c ON c.id = mc.contact_id
JOIN accounts a ON a.domain = c.domain
WHERE LOWER(COALESCE(json_extract(a.properties, '$.csm_email'), '')) = LOWER('<user_email>')
  AND m.timestamp >= datetime('now', '-14 days')
ORDER BY m.timestamp DESC
LIMIT 200
```

### 3C) Recent notes

Notes capture internal context, blockers, escalations, and action items.

```sql
SELECT
  n.id,
  n.account_id,
  a.name AS account_name,
  a.amount,
  n.title,
  n.note_type,
  n.created_at,
  n.created_by_email,
  n.content,
  n.link
FROM notes n
JOIN accounts a ON a.account_id = n.account_id
WHERE LOWER(COALESCE(json_extract(a.properties, '$.csm_email'), '')) = LOWER('<user_email>')
  AND n.created_at >= datetime('now', '-14 days')
ORDER BY n.created_at DESC
LIMIT 250
```

Strip HTML from `content` before interpreting it.

### 3D) Activity momentum and inactivity

Compare recent engagement to baseline so you can spot drop-offs and surges, not just raw volume.

```sql
WITH scoped_accounts AS (
  SELECT account_id, name, amount
  FROM accounts
  WHERE LOWER(COALESCE(json_extract(properties, '$.csm_email'), '')) = LOWER('<user_email>')
),
recent AS (
  SELECT account_id, SUM(count) AS recent_7d
  FROM activities
  WHERE timestamp >= datetime('now', '-7 days')
  GROUP BY account_id
),
baseline AS (
  SELECT account_id, SUM(count) AS baseline_28d
  FROM activities
  WHERE timestamp >= datetime('now', '-35 days')
    AND timestamp < datetime('now', '-7 days')
  GROUP BY account_id
)
SELECT
  sa.account_id,
  sa.name AS account_name,
  sa.amount,
  COALESCE(r.recent_7d, 0) AS recent_7d,
  COALESCE(b.baseline_28d, 0) AS baseline_28d,
  CASE
    WHEN COALESCE(b.baseline_28d, 0) = 0 THEN NULL
    ELSE ((COALESCE(r.recent_7d, 0) * 1.0 / 7.0) - (b.baseline_28d * 1.0 / 28.0))
      / NULLIF((b.baseline_28d * 1.0 / 28.0), 0)
  END AS momentum_delta
FROM scoped_accounts sa
LEFT JOIN recent r ON r.account_id = sa.account_id
LEFT JOIN baseline b ON b.account_id = sa.account_id
LIMIT 250
```

Interpretation:

- `momentum_delta <= -0.5` -> real drop, likely risk
- `momentum_delta >= 1.0` -> real surge, possible expansion or milestone
- `recent_7d = 0` with prior baseline activity -> inactivity alert

### 3E) Prediction, health, adoption, and renewal pressure

```sql
SELECT
  account_id,
  name,
  amount,
  health_score,
  prediction,
  prediction_score,
  feature_adoption,
  license_utilization,
  activity_score,
  product_engagement,
  expires_at
FROM accounts
WHERE LOWER(COALESCE(json_extract(properties, '$.csm_email'), '')) = LOWER('<user_email>')
LIMIT 250
```

```sql
SELECT
  pah.account_id,
  a.name AS account_name,
  pah.timestamp,
  pah.score
FROM prediction_account_history pah
JOIN accounts a ON a.account_id = pah.account_id
WHERE LOWER(COALESCE(json_extract(a.properties, '$.csm_email'), '')) = LOWER('<user_email>')
  AND pah.timestamp >= datetime('now', '-30 days')
ORDER BY pah.account_id, pah.timestamp DESC
LIMIT 250
```

### 3F) Support signal pressure

```sql
SELECT
  t.id,
  t.title,
  t.status,
  t.priority,
  t.sentiment,
  t.timestamp,
  t.resolved_at,
  t.assignee_email,
  t.link,
  a.account_id,
  a.name AS account_name,
  a.amount
FROM tickets t
JOIN ticket_contacts tc ON tc.ticket_id = t.id
JOIN contacts c ON c.id = tc.contact_id
JOIN accounts a ON a.domain = c.domain
WHERE LOWER(COALESCE(json_extract(a.properties, '$.csm_email'), '')) = LOWER('<user_email>')
  AND t.timestamp >= datetime('now', '-30 days')
ORDER BY t.timestamp DESC
LIMIT 200
```

Treat sharp increases in open, high-priority, or negative-sentiment tickets as alerts.

### 3G) Open tasks

```sql
SELECT
  t.id,
  t.title,
  t.body,
  t.status,
  t.expires_at,
  t.assigned_to_email,
  t.created_at,
  t.link,
  a.account_id,
  a.name AS account_name,
  a.amount,
  a.expires_at AS renewal_at
FROM tasks t
LEFT JOIN accounts a ON a.account_id = t.account_id
WHERE LOWER(COALESCE(t.assigned_to_email, '')) = LOWER('<user_email>')
  AND LOWER(COALESCE(t.status, '')) NOT IN ('done', 'closed')
ORDER BY
  CASE WHEN t.expires_at IS NULL THEN 1 ELSE 0 END,
  t.expires_at ASC,
  t.created_at DESC
LIMIT 150
```

## Step 4 - Build Three Priority Queues

Do not think in one generic blended list first. Build separate CS motions:

1. `Risk Queue`
2. `Renewal Queue`
3. `Expansion Queue`

### Risk Queue

An account belongs here if any of the following are true:

- open negative needle mover
- activity drop or inactivity alert
- worsening prediction trend
- negative meeting or note context
- support spike or escalation signal

### Renewal Queue

An account belongs here if:

- `expires_at` is within the next 120 days
- and there is either risk, weak adoption, missing value proof, or unresolved blockers

### Expansion Queue

An account belongs here if:

- strong positive momentum
- healthy or improving prediction/health direction
- meaningful adoption or utilization
- positive needle movers, new use cases, or growth language in notes/meetings

## Step 5 - Rank Within Each Queue

Use FunnelStory signals to rank inside each queue.

Internal ranking should consider:

- needle mover freshness and severity
- qualitative urgency from meetings/notes
- size of activity shift
- prediction direction over the last 30 days
- renewal timing
- account revenue (`amount`)

Do not expose raw math unless asked. Use ranking quietly to decide order.

## Step 6 - Map to Best-Practice Playbooks

Every top account should recommend one motion:

- `churn-risk playbook`
- `renewal playbook`
- `adoption playbook`
- `expansion playbook`
- `escalation playbook`

For each account, recommend exactly one concrete next step for today.

Examples:

- schedule risk review with current blockers and owners
- prepare value review or QBR for a renewal account
- launch targeted adoption outreach for low-usage features
- coordinate with AM on an expansion-ready account
- create or advance an escalation owner/update cadence

## Step 7 - Write the Final Daily View

Write the output in the exact bracketed sections from `Output Format`.

Rules:

- `[ TODAY'S PRIORITIES ]` should contain 5 accounts max
- mix motions when relevant: risk first, then renewal, then expansion
- do not let a noisy low-value account outrank a high-value account with clear risk unless the smaller account has a severe live escalation
- use account names in prose, never internal IDs
- explain `why now` with actual FunnelStory evidence
- make `TASKS & PLAYBOOKS` operational, not conceptual

## Guardrails

- Do not output a generic narrative summary.
- Keep the result action-first and portfolio-aware.
- Always scope to the current workspace and the resolved CSM.
- Use SQLite-safe SQL.
- Use `INSTR(...) > 0` for needle mover account linkage.
- Respect semantic guidance:
  - `health_score` is the main health indicator
  - `prediction_score` is model output, not the same thing as health
  - final prose should use needle mover impact tiers, not raw impact integers
- If segmentation fields are missing, say so and continue without inventing them.

## Example Prompts

- `Prioritize My Day`
- `Prioritize my day for jane@acme.com`
- `Show me my top renewal and risk accounts for today`
- `Which accounts need attention today because of usage drops or new needle movers?`
