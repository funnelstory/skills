---
name: account-meeting-prep
description: "Produces a concise Markdown meeting brief for one or more customer accounts by gathering all the data for that account using FunnelStory mcp. Use when preparing for a sales or success call, writing a pre-meeting summary or next steps with the account."
---

# Meeting prep

## Before you start

- Use the semantic database **schema** your MCP provides so you only use real tables and columns.
- Work from the **account identifier(s)** the user gives you (the keys that tie rows to that customer in this database).
- Prefer **recent** data where it matters (for example the last few weeks of activity), unless the user wants a wider range.

## What to load from each table

Gather data for the account (or accounts) from these tables, then fold it into the brief:

| Table                                           | What it is for                                                                                                                            |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **accounts**                                    | Company name, domain, health, scores, prediction, churn, subscription-style fields: whatever the row contains for an **account overview** |
| **activities**                                  | **Recent engagement**: what happened, when, and who (if the row has user or email fields)                                                 |
| **needle_movers**                               | **Strategic movements or risks** for the account: titles, status, impact, owners, dates                                                   |
| **tasks**                                       | **Open work and follow-ups**: titles, owners, status, due dates, links                                                                    |
| **notes**                                       | **Notes** written about the account                                                                                                       |
| **meetings**                                    | **Past calls**: title, time, summary, participants, links                                                                                 |
| **topics**                                      | **Themes or product asks** tied to the account                                                                                            |
| **tickets** (and **ticket_comments** if useful) | **Support issues**: status, requester, links, recent discussion                                                                           |

Skip any table that is not in the schema or returns nothing. Do not invent rows.

## Write the brief

Turn what you collected into a short document the user can use on a call:

1. Account overview and health (only if **accounts** had useful fields).
2. Recent activity and engagement from **activities**.
3. Notes from **notes** when relevant.
4. Needle movers and tasks: movements, open issues, owners, dates, real links only.
5. Optional: past **meetings**, **topics**, **tickets**, or chat/conversation snippets if you loaded them.

**Tone and rules**

- Use **company and people names** in prose, not internal account ids.
- No generic top-level title like "Meeting prep" unless the user asks.
- No made-up facts or links; **omit** a section if you have no data.
- Bullets and short sentences work well for scanning during a live meeting.

After the first version, you may ask whether the user wants a different format like HTML / PDF / etc (for example export-friendly layout).
