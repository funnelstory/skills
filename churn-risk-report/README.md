# Churn Risk Report

## Audience

- **CSM:** Daily / weekly triage of accounts to protect.
- **VP CS:** Team-level risk concentration, ARR at risk, coverage gaps.

## Step 1 — Scope

1. **Book:** All accounts, CSM filter, or named list?
2. **Horizon:** e.g. renewals in next 6–12 months weighted higher.
3. **Output:** Default **HTML report** (ranked sections) or **Markdown** if user requests.

## Step 2 — Risk signals (combine as available)

- `prediction` / `prediction_score` / health model outputs.
- `health_score` tier per **Health tiers** in the root `SKILL.md` (Critical ≤ 45, At risk 46–54, On track ≥ 55). No score is Unscored.
- **Negative** needle movers (`source IN ('v2', 'relationship_map')`, labels such as competitor, pricing, personnel_change, relationship_drop). Report the impact tier, not the raw integer.
- **Sentiment:** conversation or support sentiment sustained negative.
- **Engagement collapse:** activity drop vs prior period.
- **Open critical tickets** or escalations — via `references/linking-conversations-to-accounts.md`.
- **Notes** mentioning churn, competitor, or executive dissatisfaction (if `notes` queryable).

Convert prediction scores to 0–100 for display if the workspace uses -1..1: `(score * 50) + 50` (document if used).

## Step 3 — Query pattern

- Pull accounts with renewal dates and scores. Exclude `churned` unless the user asks for them. Value is `aggregate_amount`.
- Join or post-filter `needle_movers` with `INSTR(account_ids, account_id) > 0` (the `account_ids` field is text-based, not a JSON array) and `source IN ('v2', 'relationship_map')`.

## Step 4 — Report structure

**Executive summary (3–5 bullets):** value at risk (sum of `aggregate_amount`), count of Critical accounts, themes.

**Ranked table or cards:** Account, health tier (Critical / At risk / On track / Unscored), top 3 drivers (data-cited), renewal date, **recommended next action** + owner suggestion.

**Optional:** "New this week" section if comparing snapshots (user must provide or you note "single snapshot").

## Quality bar

- Every risk claim cites a field or observation (no vague "seems unhappy").
- Do not use this skill for **opportunity** language only — if the user wants expansion pipeline, switch mental model to the expansion or upsell sub-skills.
