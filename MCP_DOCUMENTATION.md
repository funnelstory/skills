# FunnelStory MCP Server

## Description

FunnelStory's MCP server gives AI assistants — Claude, Cursor, or any MCP-compatible client — live access to your workspace data.

## Features

**Query your workspace data with SQL:** Run SQL against FunnelStory's semantic database — a structured view of your workspace including accounts, metrics, predictions, activities, meetings, notes, tasks, tickets, and needle movers

**Work with automated flows:** List flows in your workspace, inspect their configuration, create new flows, and execute them — all from your AI assistant. Build a scheduled flow that checks health scores and posts a Slack alert, or run an existing renewal risk summary on demand without opening the FunnelStory UI.

**Deep account intelligence:** Pull everything your AI assistant needs about a specific account in one pass — health score trend, prediction outcome, open renewal date, flagged needle movers, recent meetings, notes, and CSM activity. Useful for call prep, QBR preparation, and portfolio reviews.

## Setup

1. Visit https://claude.ai/settings/connectors
2. Click on Browse Connectors
3. Find and connect to **FunnelStory**
4. Complete OAuth authentication — log in to FunnelStory and click **Authorize**
5. Select the workspace you want the client to access

## Authentication

This server uses OAuth 2.0. You'll need:

- A valid FunnelStory account with access to at least one workspace
- No special permissions beyond standard workspace access — the MCP client connects under your existing role

FunnelStory supports Dynamic Client Registration (DCR), so your client registers itself automatically — no manual credential setup required. After the initial authorization, tokens refresh automatically. To revoke access, go to **Settings → MCP Clients** in FunnelStory and delete the client.

## Examples

### Example 1: Account preparation

**User prompt:** "Pull up everything I need to know about Acme Corp before my call tomorrow — health score, prediction, open renewal date, recent needle movers, and any notes from the last 90 days."

**What happens:**
- The server queries the semantic database for Acme Corp's account record, health score, and prediction outcome
- Fetches recent needle mover flags and their history
- Returns notes and activity from the last 90 days
- Delivers a consolidated briefing without opening multiple tabs

---

### Example 2: Portfolio analysis

**User prompt:** "Which accounts are predicted to churn, have a renewal date in the next 60 days, and haven't had a meeting logged in the last 30 days?"

**What happens:**
- The server runs a SQL query joining predictions, account properties, and activity logs
- Filters to accounts matching all three conditions
- Returns a prioritized list with account names, CSM assignments, and renewal dates

---

### Example 3: Run a flow

**User prompt:** "Run the renewal risk summary flow for Q2 renewals and show me the output."

**What happens:**
- The server calls `get_flows` to locate the renewal risk summary flow
- Calls `run_flow` with the flow ID to execute it
- Returns the flow's output directly in the conversation — no need to navigate to the Flows section in the UI

## Privacy Policy

See our privacy policy: https://funnelstory.ai/privacy

## Support

- Email: support@funnelstory.ai