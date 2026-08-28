# FunnelStory Account Brief

Generate a structured, professional account summary brief for a named customer using FunnelStory's semantic database.

---

## Step 0 — Read the workspace profile

Call `read_resource` on `file://workspace/profile.json` before any query.

Honor its `instructions` verbatim. Workspaces override metric semantics — one may
require `amount` be reported as TCV rather than ARR, another may redirect health to a
workspace-specific property and forbid querying the default score at all. The field
names in this README are defaults, not guarantees.

If `instructions` already records support-data findings, use them and skip the discovery
in Step 2.

---

## Step 1 — Identify the Account

If the user hasn't specified an account name, ask for it. Then look it up on name **and** domain:

```sql
SELECT account_id, name, domain, amount, assignees, expires_at
FROM accounts
WHERE name LIKE '%AccountName%' OR domain LIKE '%AccountName%'
LIMIT 25
```

Several matches is normal. Workspaces carry stale duplicates, and some carry one account
**per site** — a parent plus a dozen site records is a real shape, not an error. Note that
sites frequently **share a domain**, so `domain` alone does not identify an account.

Rank candidates by non-zero `amount`, presence of `assignees`, and recent activity volume.
Proceed with the top candidate, state in one line which you chose and why, and list the
alternatives at the end so the user can redirect.

Ask first only when a parent account and its sites both match — rollup and single-site
briefs differ completely, and no query can rank that choice.

If no match is found, ask the user to verify the account name or try a partial spelling.

---

## Step 2 — Gather Data

Run all queries using the confirmed `account_id` and `domain`. All timestamps use SQLite ISO 8601 strings. Use `datetime('now', '-90 days')` for the 90-day window.

Column names vary between workspaces. Before a query that names columns beyond
`account_id`, confirm they exist:

```sql
SELECT name FROM pragma_table_info('<table>')
```

### Account Details

```sql
SELECT *
FROM accounts
WHERE account_id = '<account_id>'
```

Key fields (subject to workspace overrides from Step 0):

- `amount` — contract value; **report it under the name the workspace profile specifies**
- `expires_at` — renewal date
- `health_score` — overall health (1–10); a workspace may direct you to a property such as a custom health index instead
- `prediction_score` — churn/retention score (-1 to 1); convert to 0–100 via `(score * 50) + 50`. Lower = higher churn risk. Some workspaces forbid surfacing this — check `instructions`.
- `license_utilization`, `feature_adoption`, `activity_score`, `support_sentiment`
- `properties` JSON — CSM, account owner, TAM, renewal specialist, region, tenant settings, and any workspace-specific health factors

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

`title` and `role` are absent in some workspaces — run the `pragma_table_info` check above
and select only the columns that exist. Where several accounts share a domain, this query
returns contacts for all of them; say so rather than presenting them as one account's.

### Support History (last 90 days)

**Do not use a fixed ticket query here.** Where support data lives, and how it links to an
account, varies by workspace: the table differs, the linkage mechanism differs, and some
workspaces have no ticket-shaped data at all.

Read **`references/linking-conversations-to-accounts.md`** and follow it. In short:
confirm which table actually holds support, split by `source`, measure each candidate
linkage mechanism and adopt the highest-coverage one per source, then exclude
machine-generated noise.

Two failure modes this replaces, both of which produce a confident but wrong brief:

- `tickets` can be **empty** while `ticket_contacts` is fully populated — an inner join
  through it returns zero rows and looks like a quiet account rather than a wrong query.
- Matching contacts by `domain` **collapses every site sharing that domain into one
  account**, and where the contact field holds the internal filer rather than the
  customer, attributes the whole history to the vendor.

The former hardcoded join is retained below only as one candidate mechanism. Use it only
after confirming all three preconditions:

```sql
-- Valid only when: (1) tickets is populated, (2) contact domains are the customer's
-- rather than the vendor's, (3) no other account shares this domain.
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

Never `SELECT *` from `tickets` — `custom_fields` can carry dozens of fields per row.

For a request specifically about tickets or open escalations, route to
`account-support-history/` instead of generating a full brief.

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

If a section returns no data, distinguish two cases: **"No data available"** when the
source exists and is genuinely empty for this account, and **"Not available in this
workspace"** when the workspace does not carry that data at all. Never let the second
read as the first.

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
   - Contract value, renewal date, assigned team

2. KEY CONTACTS
   - Name | Title | Role (Champion / Decision-Maker / User / etc.)

3. PRODUCT USAGE & ENGAGEMENT
   - Entitlement vs active utilization
   - Top activities by volume (last 90 days)
   - Notable metric trends from account_metrics_history

4. NEEDLE MOVERS
   - Key recent and high impact Needle Movers with Categories
   - Open needle movers ranked by ABS(impact); flag anything ≤ -5 as critical

5. SUPPORT HISTORY
   - Open items first, then recently resolved; avg sentiment
   - Themes and notable high-priority issues
   - Customer conversations (chat/Slack) as a SEPARATE band from tickets — they
     carry no status or owner, and merging them misrepresents both. Label each as
     external (customer present) or internal (colleagues discussing them), and
     exclude internal from anything customer-facing.
   - Machine-generated alerts as an aggregate (count + severity mix), never as a list
   - One line of provenance: table, source, linkage mechanism, coverage %, what was excluded

6. SIGNALS & RECOMMENDATIONS
   - Health and prediction indicators, per workspace conventions
   - Risk factors and recommended actions
   - Suggested next steps for the account team

─────────────────────────────────────────
```

Keep tone professional and concise. Bullets over paragraphs. Surface insights — don't dump raw data. If needle movers are sparse, lean on notes and support history for qualitative signals.

Where support linkage coverage is under roughly 50%, say the support section is partial
in the prose, not only in the provenance line.

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

- **Workspace overrides win.** Metric names and health conventions come from the profile `instructions`, not from this file.
- **Verify columns before selecting them** with `pragma_table_info`; schemas differ between workspaces.
- **Support data**: always via `references/linking-conversations-to-accounts.md`. Never assume `tickets` is populated, and never assume a populated `ticket_contacts` implies populated `tickets`.
- **Never `SELECT *`** on `tickets` or `conversations`.
- **Shared domains**: several accounts may share one domain; domain-based filters can silently span them.
- **JSON arrays**: Use `json_each(col)` for proper JSON array columns (e.g. `accounts.assignees`). For text-based array fields like `needle_movers.account_ids`, use `INSTR(col, value) > 0` instead.
- **JSON field extraction**: Use `json_extract(col, '$.field')` — e.g. `json_extract(key, '$.ticket_id')`.
- **Prediction score display**: Convert -1→1 to 0–100 via `(score * 50) + 50`.
- **Notes content**: Strip HTML tags before including in the brief.
- **Timestamps**: All ISO 8601 strings; use `datetime('now', '-90 days')` for relative ranges.
- **Missing data**: Distinguish "no data for this account" from "not available in this workspace".
- **Large tables**: correlated subqueries over 100k+ rows time out; split per source.
- **MCP not connected**: Surface a clear message — "FunnelStory MCP isn't connected. Please enable it in your tools/connectors settings."
