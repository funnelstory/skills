# Upsell Opportunities Dashboard

## Audience

- **CSM:** Prioritize accounts ready for a packaging conversation (tier, modules, support tier).
- **VP CS:** Segment-level upsell pipeline for forecasting and QBRs with leadership.

## Step 1 — Scope

Confirm:

1. **What "upsell" means** in this workspace (SKU names, plan tiers, module flags in `properties` or a `subscriptions` / `plans` model).
2. **Portfolio filter:** CSM, segment, or all customers.
3. **Output:** Default **HTML dashboard**.

## Step 2 — Signals (configure per tenant)

Typical signals (use only what exists in schema):

- **Plan vs usage mismatch:** high usage on base tier → eligible for premium.
- **Feature gates:** heavy use of activity X while `properties` shows module Y not enabled.
- **Support load:** ticket volume where premium support would apply (if modeled).
- **Security / compliance:** industry or `traits` suggesting need for enterprise add-ons.
- **Renewal window:** upsell tied to renewal quarter (optional).

Exclude accounts that are **primarily churn risks** unless the user asks to include — churn-first narrative belongs in the churn-risk-report sub-skill.

## Step 3 — Query pattern

Use `json_extract(properties, '$.…')` for plan fields when stored in JSON. Join `accounts` with `activities`, `account_metrics_latest`, or `subscriptions` if present. Rank by a documented **upsell fit score**.

## Step 4 — HTML dashboard

- **Title:** Upsell opportunities.
- **KPIs:** qualified account count, ARR under consideration, top SKU opportunity (if applicable).
- **Per account:** current tier/sku (if known), top 2 **upsell hypotheses** with evidence, suggested talk track (one line).
- **Visual distinction** from expansion dashboard: use language like "Upgrade", "Premium", "Add-on", not "Land new site".

## Quality bar

- Do not label seat growth alone as upsell if the user's motion is expansion — route mentally to the expansion sub-skill.
- No fabricated SKUs — if plan fields are missing, say "Plan data not in schema" and fall back to proxy signals with clear caveats.
