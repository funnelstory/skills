# Troubleshooting

Common problems and how Claude should handle them.

---

## MCP not connected

**What happened:** The FunnelStory MCP server is not available — `get_data_connections` or `execute_query` fails or is missing from the tool list.

**What to do:**
- Tell the user: "FunnelStory MCP isn't connected. Please enable it in your tools/connectors settings and try again."
- Do NOT attempt to fabricate data or proceed without MCP.

---

## Empty query results

**What happened:** A SQL query returns zero rows — the table exists but has no matching data.

**What to do:**
- Note "No data available for [section]" and move on to the next section.
- Do NOT omit the section silently — the user needs to know what's missing.
- If ALL queries return empty, ask: "I couldn't find data for this account. Can you double-check the account name or try a different spelling?"

---

## Ambiguous account name

**What happened:** `SELECT ... WHERE name LIKE '%X%'` returns multiple matches.

**What to do:**
- List the top 5 matches with name, domain, and ARR.
- Ask the user to confirm which one.
- Do NOT guess or pick the first result.

---

## Large portfolio — slow queries

**What happened:** The user's book has 50+ accounts and batch queries are slow or truncated.

**What to do:**
- Set `limit` to at least 200 on account-list queries.
- Process in batches if `get_account` is needed per account (batch size ~20).
- Show progress: "Found 84 accounts. Processing in batches..."
- If it exceeds ~200 accounts, ask: "You have a large portfolio. Want me to focus on a specific segment (e.g. at-risk, renewing soon, top ARR)?"

---

## Missing schema fields

**What happened:** A column or table referenced in the README doesn't exist in this workspace's semantic DB.

**What to do:**
- Skip that section or metric gracefully.
- Note: "[Field] is not available in this workspace's schema — skipping."
- Use available proxy fields when reasonable (e.g., if `license_utilization` is missing, compute from `total_users` vs `properties.num_contract_devices`).
- Do NOT fabricate values.

---

## Prediction score format mismatch

**What happened:** Some workspaces use -1 to 1, others use 0 to 100 for prediction scores.

**What to do:**
- Check the range of `prediction_score` in the first query result.
- If values are between -1 and 1, convert to 0–100 via `(score * 50) + 50`.
- If values are already 0–100, display as-is.
- Document which conversion you applied in any output that shows the score.

---

## User email doesn't match any accounts

**What happened:** The user's email doesn't match any accounts via the `assignees` column or account properties.

**What to do:**
- First try the `assignees` column: `SELECT ... FROM accounts WHERE EXISTS (SELECT 1 FROM json_each(assignees) WHERE json_each.value = '<email>')`.
- If no results, try account properties: `$.csm_email`, `$.account_owner`, `$.csm`.
- If still no matches: "I couldn't find accounts assigned to your email. Can you list the account names you manage, or check if your email in FunnelStory matches what you gave me?"

---

## Notes contain HTML

**What happened:** The `content` field in `notes` contains raw HTML tags.

**What to do:**
- Strip all HTML tags before displaying in any output (markdown, slides, email).
- Preserve the text content — just remove `<p>`, `<br>`, `<strong>`, etc.
- If the content is entirely HTML with no readable text, note: "Note content not displayable — view in FunnelStory UI."

---

## Evidence attribution

When generating insights, tag the source so the reader knows where claims come from:

| Tag | Meaning |
|-----|---------|
| `[Data]` | Directly from a FunnelStory DB field or metric |
| `[Analysis]` | Claude's synthesis or interpretation of data patterns |
| `[Trend]` | Derived by comparing current vs prior-period values |
| No tag | Structural copy, formatting, or common knowledge |
