# Account support history

Recent support activity for one named account: tickets, escalations, open items, and
customer conversations.

Triggers: *"latest tickets for X"*, *"what's open with X"*, *"recent support history for X"*,
*"any escalations at X"*, *"what has X been complaining about"*.

> **On the word "ticket."** Some workspaces have no ticket-shaped data at all — no
> status, no priority, no assignee, no resolution. Support may exist only as chat
> threads or messages. Answer with what the workspace actually holds and say plainly
> what it does not have. Do not synthesise a status column.

## Step 0 — Read the workspace profile

Call `read_resource` on `file://workspace/profile.json` before querying.

Honor `instructions` overrides verbatim. They may rename metrics (e.g. report `amount`
as TCV rather than ARR), redirect health to a workspace-specific property, or forbid
querying certain fields entirely. They may also already record support-linkage findings —
if so, skip discovery and use them.

Never hardcode `health_score`, `amount`, or `prediction_score` semantics across workspaces.

## Step 1 — Resolve the account

Match on name **and** domain:

```sql
SELECT account_id, name, domain, amount, assignees, properties
FROM accounts
WHERE name LIKE '%<term>%' OR domain LIKE '%<term>%'
LIMIT 25;
```

Several matches is normal. Enterprise workspaces carry stale duplicates, and some carry
one account **per site** — a parent plus nine site records is a real shape, not an error.

Rank candidates by non-zero `amount`, presence of `assignees`, and recent activity
volume. Pick the top one, **state which you picked and why in one line**, and offer the
alternatives at the end. Do not open with a disambiguation question — a user who typed
five words should get an answer, not a quiz.

The one case worth asking about is **parent vs sites**, because rollup and single-site
answers differ completely and no query can rank them. See "Asking" below.

## Step 2 — Run discovery

Follow `references/linking-conversations-to-accounts.md`. It returns, per source: the
table, the linkage mechanism, coverage, and the noise exclusions.

Carry those four facts forward — the output contract requires them.

## Step 3 — Assemble bands

Sort rows into at most three bands. **Never merge bands into one list.** Ticket rows
carry state and an owner; chat rows carry neither, and a merged table where half the
rows have a status column and half are blank misrepresents both.

### Band 1 — Ticket-shaped (only if `status`/`priority` exist and are populated)

Most recent 10–20. Columns: date, title (linked via `link` if present), status,
priority, contact, assignee.

Lead the prose with **what is still open**, not the newest row. Open-and-aging with an
escalation tag is the finding; a list sorted by date is just data. Call out items that
are unresolved, escalated, or have been sitting past the workspace's usual resolution
time.

### Band 2 — Conversation-shaped (chat threads, messages)

Same recency window. Columns: date, channel or thread, visibility (**external** = the
customer is present; **internal** = colleagues discussing them), contact, opening line.

Label visibility on every row. When the deliverable is customer-facing, exclude
internal rows entirely rather than labelling them.

State explicitly when no open/closed state exists in this workspace, so the absence
reads as a property of the data and not an oversight.

### Band 3 — Alert posture (only when machine alerts are material)

An aggregate, never a list: alert count and severity mix for the account over the
window, with a comparison point. Severity **mix** discriminates better than volume.

## Step 4 — Provenance footer

Close with one short block:

```
Sources: <table>/<source> via <mechanism> (<n> rows, <coverage>% linked)
         <table>/<source> via <mechanism> (<n> rows, <coverage>% linked)
Excluded: <n> machine-generated alerts; <n> unattributable rows
Not available: <e.g. open/closed state — this workspace's support data carries no status>
```

Where coverage is under roughly 50%, say the list is partial in the prose too, not only
in the footer.

## Asking vs stating

Default to **stating the assumption inline** and offering the alternative at the end.
Ask only when the choice would change the whole answer and no query can settle it:

- **Parent vs sites**, when both exist for the named customer — roll up across sites, or
  one site?
- **Internal conversations in a customer-facing deliverable** — include or exclude?
- **Coverage under ~50%** — accept the partial list, or widen with a fuzzier mechanism
  and lower confidence?

At most one question, and never before producing something. Write it as plain prose:
this skill runs in Claude Desktop, Claude Code, and Cursor, and cannot rely on any
particular picker UI being available.

## Step 5 — Offer to record findings

When discovery ran (i.e. the profile had no recorded findings), close by offering a
paste-ready block:

```
Support data: <table>, sources <a> and <b>.
<source-a>: link via <mechanism>, ~<n>% coverage.
<source-b>: link via <mechanism>, ~<n>% coverage.
Exclude: <noise channels / process tags / vendor-internal org>.
Never merge internal-visibility conversations into customer-facing output.
Known gaps: <e.g. no status field; sources overlap <dates>, possible duplicates>.
```

Frame it as something a workspace admin can paste into the profile `instructions`.
**Do not write it yourself** — workspace configuration is a shared, permissioned
resource with side effects beyond this user. Offer the text; let them apply it.

## Related sub-skills

- `account-brief/` — full account narrative; support history is one section of it.
  Use this sub-skill when the request is specifically about tickets or support.
- `feature-requests/` — what customers asked for, aggregated across accounts.
- `churn-risk-report/` — portfolio-wide risk ranking, not single-account history.
