# Meeting Prep

## Before you start

- Use the semantic database **schema** your MCP provides so you only use real tables and columns.
- Resolve the account the user names using **Resolving a named account** in the root `SKILL.md`.
- Prefer **recent** data where it matters (for example the last few weeks of activity), unless the user wants a wider range.

## What to load from each table

Gather data for the account (or accounts) from these tables, then fold it into the brief:

| Table                                           | What it is for                                                                                                                            |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **accounts**                                    | Company name, domain, health tier, value (`aggregate_amount`), renewal (`earliest_child_expiry` or `expires_at`) |
| **account_profiles**                            | Narrative and stakeholders. Read `file://semantic/account_profiles.md` first.                                    |
| **activities**                                  | **Recent engagement**: what happened, when, and who (if the row has user or email fields)                        |
| **needle_movers**                               | Open stories. `source IN ('v2', 'relationship_map')`. Report the impact tier, not the raw integer.               |
| **action_cards**                                | Open suggested actions for this account. A NULL `status` is still a suggestion.                                  |
| **tasks**                                       | **Open work and follow-ups**: titles, owners, status, due dates, links                                           |
| **notes**                                       | **Notes** written about the account                                                                              |
| **calendar_events**                             | **Upcoming invites**, via `calendar_event_accounts`. Not the same as recorded calls.                             |
| **meetings**                                    | **Past recorded calls**: title, time, summary, participants, links                                               |
| **content_labels**                              | **Themes** on conversations and notes. Do not use `topics`.                                                      |
| **tickets** / **conversations**                 | **Support issues and customer conversations** — via `references/linking-conversations-to-accounts.md`            |

If a table is not in the schema, note "Not available in this workspace"; if it exists but returns nothing for this account, note "No data available". Do not invent rows.

## Write the brief

Turn what you collected into a short document the user can use on a call:

1. Account overview and health tier (only if **accounts** had useful fields).
2. Who you are talking to, from **account_profiles**.
3. What is on the calendar next, from **calendar_events**, then recent recorded calls from **meetings**.
4. Recent activity from **activities**, and themes from **content_labels**.
5. Notes, needle movers, action cards, and open tasks: owners, dates, real links only.
6. Support, via the linking reference.

**Tone and rules**

- Use **company and people names** in prose, not internal account ids.
- No generic top-level title like "Meeting prep" unless the user asks.
- No made-up facts or links. For empty sections, use one short line ("No data available" / "Not available in this workspace") rather than dropping them.
- Bullets and short sentences work well for scanning during a live meeting.

After the first version, you may ask whether the user wants a different format like HTML / PDF / etc (for example export-friendly layout).
