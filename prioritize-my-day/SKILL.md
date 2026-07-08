---
name: prioritize-my-day
description: >
  Build a FunnelStory-first daily action plan for a Customer Success Manager.
  Trigger when the user says "Prioritize My Day" or asks what they should focus
  on today across their book of business. Use FunnelStory signals like needle
  movers, recent meetings and notes, activity changes, health and prediction,
  renewals, adoption, tasks, and support context. If the current user is not
  explicitly available from the calling environment, ask the user to specify the
  CSM, segment, or accounts to scope the report.
compatibility: "Requires FunnelStory MCP to be connected"
---

# Prioritize My Day

Return a FunnelStory-native daily operating view for a CSM.

This skill should feel like a CS command center, not a generic summary. It should answer:

1. Which accounts need action today?
2. What changed since the last time the CSM looked?
3. Which motions matter now: risk, renewal, or expansion?
4. What should the CSM do next on each account?

## Before you start

- Use the semantic database schema and usage guidance exposed by the MCP so you only rely on real tables and columns.
- Do not try to guess "who am I" from workspace data.
- Scope must come from one of these sources:
  - user explicitly names the CSM
  - calling environment already provides the current user
  - user asks for a segment, region, tier, or explicit account list
- If scope is missing, ask for it before building the report.
- Prefer recent signals when deciding urgency.

## Scope

The default scope is a single CSM's book of business.

If the user asks for a different slice, support it:

- specific CSM
- named segment
- tier, lifecycle, or region
- explicit account list
- renewal-only, risk-only, or expansion-only views

If ownership data is sparse or inconsistent, say so in the output instead of inventing a portfolio boundary.

## What to gather

Gather only the sources that exist in the workspace schema. Skip missing tables gracefully.

| Source | Use it for |
| --- | --- |
| `accounts` | Portfolio, health, revenue, renewal timing, prediction, adoption, utilization, and segmentation context |
| `needle_movers` | Fresh strategic signals, risks, opportunities, blockers, and urgency |
| `meetings` | Recent customer conversations, commitments, escalations, and expansion language |
| `notes` | Internal context, follow-ups, blockers, decision history, and action items |
| `activities` | Momentum, inactivity, sudden drops, and unusual engagement spikes |
| `tasks` | Open follow-through, due dates, and ownership gaps |
| `tickets` | Support pressure, unresolved issues, severity, and sentiment |
| `topics` | Repeated themes, requests, and product areas that matter right now |

## FunnelStory prioritization logic

Build the daily view the way a strong CSM works a portfolio:

1. Start with the accounts that need a decision or action today.
2. Separate motions into `risk`, `renewal`, and `expansion`.
3. Let multiple signals reinforce each other before something rises to the top.
4. Use recent meeting and note context to decide urgency, not only static scores.
5. Tie every priority to a concrete next step.

### Signals that should push an account up

Risk signals:

- fresh negative needle movers
- recent meeting or note showing blockers, escalation, or loss of confidence
- meaningful drop in activity or clear inactivity
- worsening health or prediction direction
- open support pressure close to renewal

Renewal signals:

- renewal is approaching
- unresolved risks or blockers remain open
- weak adoption or weak value realization
- not enough recent customer touch or no clear renewal plan

Expansion signals:

- positive needle movers
- recent activity spike or adoption growth
- healthy or improving prediction and health
- notes or meetings showing new use cases, champions, or budget interest

### Ranking guidance

Within each motion, rank using business impact and urgency together:

- freshness and severity of needle movers
- urgency from recent meetings and notes
- size and direction of activity change
- prediction or health direction
- renewal timing
- revenue impact

Do not expose raw scoring math unless the user asks. Use it quietly to decide order.

## Write the final view

Return concise Markdown using these exact sections:

`[ TODAY'S PRIORITIES ]`
Top accounts to act on today across risk, renewal, and expansion. Keep this short and ranked.

`[ ALERTS & SIGNALS ]`
Real-time triggers such as new needle movers, inactivity, sudden activity changes, support pressure, or sentiment shifts.

`[ CUSTOMER HEALTH OVERVIEW ]`
Health, prediction direction, and the main portfolio drivers across the scoped accounts.

`[ REVENUE & RENEWALS ]`
Renewal pressure, ARR at risk, commercial upside, and accounts with near-term revenue consequences.

`[ SEGMENTATION FILTERS ]`
Tier, lifecycle, region, or other slices used to interpret the portfolio.

`[ ADOPTION & ENGAGEMENT ]`
Usage trends, adoption, utilization, inactivity, and momentum movers.

`[ TASKS & PLAYBOOKS ]`
What to do next today. Map each top account to a concrete motion such as `churn-risk`, `renewal`, `adoption`, `expansion`, or `escalation`.

`[ WHAT I ASSUMED ]`
State the scope used, any missing data, and any portfolio limitations.

## Output rules

- Keep the result action-first and portfolio-aware.
- Do not produce a generic narrative summary.
- Use account names in prose, never internal ids.
- Explain why each top account matters now using FunnelStory evidence.
- Prefer the top few accounts over a long noisy list.
- Keep `[ TODAY'S PRIORITIES ]` to five accounts or fewer.
- If a signal source is missing, continue with the others and say so in `[ WHAT I ASSUMED ]`.
- If the user asked for "my day" but the current user is not actually known, pause and ask who or what to scope to.
