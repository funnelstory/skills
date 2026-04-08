# Executive Sponsor Tracker

## Audience

- **CSM:** Know who to escalate for exec alignment before renewal.
- **VP CS:** Audit coverage across the team; plan exec time for QBRs / EBRs.

## Step 1 — Schema discovery

Executive sponsor data **varies by workspace**. Check:

- `json_extract(properties, '$.executive_sponsor')` or similar keys (`sponsor_email`, `exec_sponsor`, `vp_sponsor`).
- `meetings` joined via `meeting_contacts` → `contacts` → account domain: filter by title containing "VP", "Director", "Chief", or participant email domain = customer exec (tune per tenant).
- `tasks` or `needle_movers` labeled as executive action items (if used).

If no sponsor field exists, **derive** "last exec-level meeting" from meeting participants/titles and flag accounts with **no qualifying meeting in N days**.

State assumptions explicitly in the output.

## Step 2 — Scope

1. Accounts: all, CSM filter, or renewal window.
2. **Definition** of "executive" for this run (title keywords or seniority list).
3. Lookback: e.g. last 180 days for exec touch.

## Step 3 — Output

Default **HTML** table + summary:

| Account | Sponsor (named) | Last exec touch | Days since | Gap? | Suggested action |
|---------|-----------------|-----------------|------------|------|------------------|

Add sections:

- **Missing sponsor record** (property empty).
- **Stale exec engagement** (no meeting > threshold).
- **Healthy coverage** (optional compact list).

## Quality bar

- Do not invent sponsor names — use CRM/properties/meeting data only.
- If inference is used, label rows "Inferred from meetings" vs "From CRM field".
