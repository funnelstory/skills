# Success Plan Generator

Identify gaps in a named account — both usage-based and qualitative (from notes, meetings, tickets) — and produce a structured short-term (30 days) and medium-term (90 days) action plan with owners and milestones.

## Audience

- **CSM:** Build a concrete, data-grounded plan to improve account health or prepare for renewal.
- **VP CS:** Review and approve success plans for strategic accounts; ensure consistency across the team.

## Step 1 — Identify the account

If the user hasn't named an account, ask for it. Resolve via name search.

## Step 2 — Gather gap signals

### Usage gaps

```sql
SELECT activity_name, SUM(count) AS total, MAX(timestamp) AS last_seen
FROM activities
WHERE account_id = '<account_id>'
  AND timestamp >= datetime('now', '-90 days')
GROUP BY activity_name
ORDER BY total DESC
```

Compare against expected activities for a healthy account in this workspace. Look for:
- Activities with zero count (features never used)
- Activities with declining counts vs prior 90 days
- Low `feature_adoption` or `license_utilization` from the `accounts` table

### Qualitative gaps (from written context)

```sql
SELECT title, content, note_type, created_at
FROM notes
WHERE account_id = '<account_id>'
ORDER BY created_at DESC
LIMIT 15
```

```sql
SELECT m.title, m.summary, m.timestamp, m.sentiment
FROM meetings m
JOIN meeting_contacts mc ON mc.meeting_id = m.id
JOIN contacts c ON c.id = mc.contact_id
WHERE c.domain = '<account_domain>'
ORDER BY m.timestamp DESC
LIMIT 10
```

Scan notes and meeting summaries for:
- Unresolved customer concerns or complaints
- Feature requests or blocked workflows
- Personnel changes (champion left, new stakeholder)
- Competitor mentions
- Stalled initiatives

### Risk indicators

```sql
SELECT title, description, state, impact, label
FROM needle_movers
WHERE INSTR(account_ids, '<account_id>') > 0
  AND state IN ('open', 'acknowledged')
ORDER BY impact ASC
```

Open negative needle movers are direct inputs to the plan.

### Current health context

Pull `health_score`, `prediction_score`, `expires_at`, `amount` from `accounts` to frame urgency.

## Step 3 — Synthesize gaps into themes

Group the identified gaps into 3–5 themes. Example themes:
- **Adoption** — low feature usage, dormant users
- **Stakeholder alignment** — champion left, no exec sponsor
- **Product friction** — open tickets, feature gaps
- **Value perception** — no recent wins communicated, renewal approaching
- **Competitive threat** — competitor mentions in notes

Each theme should cite the specific data that supports it.

## Step 4 — Build the plan

### Structure

```
# {Account Name} — Success Plan

**Health:** {score} | **ARR:** ${amount} | **Renewal:** {date} ({days} days)
**Plan created:** {today's date}

## Gap Summary

| # | Theme | Severity | Key evidence |
|---|-------|----------|--------------|
| 1 | {Theme} | Critical / High / Medium | {One-line data citation} |
| 2 | {Theme} | ... | ... |

## Short-Term Actions (next 30 days)

| # | Action | Owner | Due | Success metric | Gap addressed |
|---|--------|-------|-----|----------------|---------------|
| 1 | {Specific action} | CSM / AE / TAM | {Date} | {Measurable outcome} | Gap #1 |
| 2 | ... | ... | ... | ... | ... |

## Medium-Term Actions (30–90 days)

| # | Action | Owner | Due | Success metric | Gap addressed |
|---|--------|-------|-----|----------------|---------------|
| 1 | {Specific action} | ... | ... | ... | ... |

## Milestones

- **Day 30:** {What should be true}
- **Day 60:** {What should be true}
- **Day 90:** {What should be true — e.g., health score above X, renewal confirmed}

## Risks to the plan

- {Risk 1 — what could derail this plan and mitigation}
- {Risk 2}
```

## Quality bar

- Actions must be **specific** (not "improve engagement" — instead "Schedule enablement session for the ops team on feature X, targeting 5 new active users by Day 30").
- Every action maps back to a numbered gap.
- Owner should be a role (CSM, AE, TAM, Support) — not a specific person unless the data provides one.
- Success metrics must be measurable from FunnelStory data (activity counts, health score, ticket resolution).
- If data is thin, produce a shorter plan and note: "Limited data — recommend a discovery call to validate these gaps with the customer."
