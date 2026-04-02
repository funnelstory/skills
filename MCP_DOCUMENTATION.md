# FunnelStory MCP Server

**Tagline:** Your customer intelligence analyst in Claude

## Description

Ask Claude questions about your customers and get answers grounded in real data. FunnelStory aggregates data from your CRM, support tools, meeting platforms, product analytics, and communication channels into a single customer intelligence layer. Query account health, churn predictions, support trends, engagement patterns, and more — all through natural conversation. Use it to prepare for customer meetings, monitor portfolio risk, investigate support issues, or identify expansion opportunities. Turn Claude into your customer intelligence analyst.

## Features

**Query your workspace data with SQL:** Run SQL against FunnelStory's semantic database — a structured view of your workspace including accounts, metrics, predictions, activities, meetings, notes, tasks, tickets, and needle movers. Claude reads the schema first, then writes and executes queries to answer questions across your entire book of business.

**Work with automated flows:** List flows in your workspace, inspect their configuration, create new flows, and execute them — all from Claude. Build a scheduled flow that checks health scores and posts a Slack alert, or run an existing workflow on demand without opening the FunnelStory UI.

**Deep account intelligence:** Pull everything Claude needs about a specific account in one pass — health score trend, prediction outcome, open renewal date, flagged needle movers, recent meetings, notes, and CSM activity. Useful for call prep, QBR preparation, and portfolio reviews.

## Setup

1. Visit the [Anthropic MCP Directory](https://www.anthropic.com/mcp)
2. Find and connect to **FunnelStory**
3. Complete OAuth authentication — log in to FunnelStory and click **Authorize**
4. Select the workspace you want Claude to access

**Direct server URL** (for manual configuration in Claude Desktop, Cursor, etc.):

```
https://app.funnelstory.ai/api/mcp
```

For Claude Desktop, add this to your config under `Settings → Developer → Edit Config`. For Cursor, go to `Settings → MCP` and add a new server entry.

## Authentication

This server uses OAuth 2.0 with PKCE. You'll need:

- A valid FunnelStory account with access to at least one workspace
- No special permissions beyond standard workspace access — the MCP client connects under your existing role

FunnelStory supports Dynamic Client Registration (DCR), so your client registers itself automatically — no manual credential setup required. After the initial authorization, tokens refresh automatically. To revoke access, go to **Settings → MCP Clients** in FunnelStory and delete the client.

## Examples

### Example 1: Prepare for a customer meeting

**User prompt:** "I have a QBR with Acme Corp next week. Give me a briefing — how is their account health trending, what support issues have come up recently, who are the key contacts we've been meeting with, and are there any risks I should know about?"

**What happens:**

- Queries Acme Corp's health score history and current prediction outcome
- Fetches recent support tickets and sentiment trends
- Pulls meeting history and contact activity
- Returns a consolidated briefing covering health, risk signals, and relationship context

---

### Example 2: Monitor churn risk across your portfolio

**User prompt:** "Which of my accounts are showing signs of churn risk? Look at declining health scores, negative support sentiment, and contracts expiring in the next 90 days."

**What happens:**

- Runs SQL across accounts joining health score trends, support sentiment, and renewal dates
- Filters to accounts matching one or more risk signals
- Returns a prioritized list with account names, CSM assignments, and the specific risk factors flagged

---

### Example 3: Investigate a customer's support experience

**User prompt:** "Summarize all support tickets for GlobalTech over the past 6 months. What are the recurring themes, how is sentiment trending, and are there any unresolved escalations?"

**What happens:**

- Queries all support tickets linked to GlobalTech within the date range
- Groups and summarizes by theme and sentiment over time
- Flags any open escalations or unresolved issues

---

### Example 4: Find expansion opportunities

**User prompt:** "Which accounts have growing product usage and increasing user counts but haven't had a recent meeting with their CSM? These could be good candidates for upsell conversations."

**What happens:**

- Queries product usage metrics and user count trends across accounts
- Joins with meeting activity to filter out recently contacted accounts
- Returns a ranked list of expansion candidates with supporting metrics

---

### Example 5: Build an executive portfolio overview

**User prompt:** "Give me a snapshot of my entire book of business — break it down by health status, show me the top risks and top performers, and flag any accounts with expiring contracts this quarter."

**What happens:**

- Queries all accounts with health scores, bucketed by status
- Identifies top-risk and top-performing accounts by score and prediction
- Filters for contracts expiring within the current quarter
- Returns a structured executive summary ready to share

## Privacy Policy

See our privacy policy: https://funnelstory.ai/privacy

## Support

- Email: support@funnelstory.ai
