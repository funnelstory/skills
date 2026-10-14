# Prioritize my day

A short daily brief for the connected user: what to act on today. This is not a portfolio snapshot (`book-of-business/`) and not a ranked danger report (`churn-risk-report/`).

## Step 1 — Whose book

Use Step 0 in the root `SKILL.md`: `who_am_i`, then the assignee lookup. Exclude `churned` accounts. Cache the account ids for the rest of this run.

## Step 2 — Open action cards

`action_cards` is the same set as Focus Areas Actions: agent-created tasks that are still open. A NULL `status` means nobody has accepted the suggestion yet.

```sql
SELECT id, title, account_id, status, state, expires_at, details
FROM action_cards
WHERE account_id IN ('<id1>', '<id2>')
ORDER BY expires_at ASC
```

If the table is missing, say "Not available in this workspace" and continue with Step 3. Do not invent cards.

Group what you find:

- Suggestions (`status` is NULL)
- Accepted work (`todo` or `in_progress`)

Lead the brief with these. Cite the title and the account name. Read `details` for the why when it is present; older rows keep that text in `body`.

## Step 3 — Accounts with no card

From the same book, add accounts that have **no** row in Step 2 and are either:

- **Critical** or **At risk** per **Health tiers** in the root `SKILL.md`, or
- renewing within 90 days (`earliest_child_expiry`, otherwise `expires_at`)

One line each: tier or renewal date, and the top driver (an open negative needle mover with `source IN ('v2', 'relationship_map')`, reported as a tier, or the renewal date). Do not build a second scoring model.

## Step 4 — Write the brief

In chat, three short sections:

1. **Do today** — the action cards, suggestions first.
2. **Uncovered** — Critical, At risk, and near renewals with no card.
3. **Not in this pass** — anything you skipped because the table was missing.

No file unless the user asks. No raw ids in the prose. No customer names from examples; use the names the query returned.
