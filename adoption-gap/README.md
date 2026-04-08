# Adoption Gap Analysis

## Audience

- **CSM:** Enablement plan per account — who to train, which teams, which features.
- **VP CS:** Patterns across the segment (which features are ignored, which regions lag).

## Step 1 — Define the gap

Ask or infer:

1. **Gap type:** seat utilization, MAU/WAU vs entitled users, feature adoption %, time-to-first-value, or specific "golden path" activities not completed.
2. **Thresholds:** e.g. adoption < 40% of entitled, or zero events for key `activity_name` in 30 days.
3. **Scope:** CSM book, segment, or workspace.

## Step 2 — Metrics (illustrative)

Use what exists:

- `accounts`: `license_utilization`, `feature_adoption`, `total_users`, `properties` JSON.
- `activities`: counts by `activity_name`, `last_seen`, distinct active days.
- `users`: active vs total per account (if modeled).

Compute simple ratios; flag **staleness** (no activity in N days).

## Step 3 — Output

Default: **HTML** "gap report" with:

- Summary: X accounts with material gaps, total entitled users under-utilized (if estimable).
- Per account: gap type, top missing behaviors, **one enablement recommendation** (training, template repo, office hours).
- Optional: heat-style labels (Severe / Moderate / Watch) based on thresholds you stated upfront.

Markdown acceptable if the user prefers copy-paste into email.

## Quality bar

- Distinguish **"not using"** from **"using heavily but could expand"** (latter → expansion sub-skill).
- If schema lacks license fields, say so and use activity-only proxies with explicit caveats.
