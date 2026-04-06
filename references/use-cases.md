# Use Cases by Persona

Trigger examples mapped to sub-skills. Use this to disambiguate intent when the user's request could match multiple skills.

---

## CSM (Customer Success Manager)

### Day-to-day account work

| Trigger | Sub-skill | Output | Value |
|---------|-----------|--------|-------|
| "Summarize Acme Corp for me" | `account-brief/` | Markdown brief or PDF | Full internal snapshot before a sync |
| "Prep me for my call with Globex" | `meeting-prep/` | Markdown brief | Talking points, recent activity, open issues |
| "Why is Acme's health score so low?" | `health-score-breakdown/` | Markdown or one-pager | Understand score drivers and what to fix |
| "Create a success plan for Acme" | `success-plan/` | Action plan with milestones | Forward-looking gap-closing plan |
| "Send Acme a value recap email" | `value-email/` | Email-safe HTML | Customer-facing value summary |

### Portfolio management

| Trigger | Sub-skill | Output | Value |
|---------|-----------|--------|-------|
| "How's my book doing?" | `book-of-business/` | Markdown with drill-down | Personal portfolio snapshot |
| "Which of my accounts might churn?" | `churn-risk-report/` | HTML report | Ranked risk list with drivers |
| "Who's renewing in the next 90 days?" | `renewal-dashboard/` | HTML dashboard | Time-bounded renewal pipeline |
| "Where are expansion opportunities?" | `expansion-dashboard/` | HTML dashboard | Accounts ready for more seats/sites |
| "Who could upgrade to premium?" | `upsell-dashboard/` | HTML dashboard | Tier/module upsell candidates |
| "Which accounts aren't using what they bought?" | `adoption-gap/` | HTML report | Under-utilization by account |
| "Which accounts are missing an exec sponsor?" | `exec-sponsor/` | HTML table | Sponsor coverage audit |

### QBR and presentations

| Trigger | Sub-skill | Output | Value |
|---------|-----------|--------|-------|
| "QBR deck for Acme — internal" | `qbr-internal/` | PowerPoint | Internal prep: risks, concerns, metrics |
| "QBR deck for Acme — customer-facing" | `qbr-external/` | PowerPoint or HTML | Value-focused deck to present TO the customer |
| "Case study for Acme" | `case-study/` | HTML artifact | Publication-ready editorial narrative |

---

## VP of Customer Success

| Trigger | Sub-skill | Output | Value |
|---------|-----------|--------|-------|
| "Show me the churn risk across the team" | `churn-risk-report/` | HTML report | Team-level ARR at risk |
| "Renewal pipeline for Q3" | `renewal-dashboard/` | HTML dashboard | Time-bounded pipeline across all CSMs |
| "Expansion pipeline for the enterprise segment" | `expansion-dashboard/` | HTML dashboard | Segment-filtered growth opportunities |
| "Executive sponsor gaps across the portfolio" | `exec-sponsor/` | HTML table | Coverage audit for exec planning |
| "Adoption gaps in the mid-market segment" | `adoption-gap/` | HTML report | Segment-level under-utilization |
| "Leadership update on our top 5 accounts" | `book-of-business/` | Markdown | Portfolio snapshot (VP scopes to team/segment) |

---

## PM / VP of Product

| Trigger | Sub-skill | Output | Value |
|---------|-----------|--------|-------|
| "What features are customers requesting?" | `feature-requests/` | HTML dashboard | Ranked feature requests by frequency and ARR |
| "Feature requests from enterprise accounts" | `feature-requests/` | HTML dashboard (filtered) | Segment-specific request priorities |
| "Which features have the lowest adoption?" | `adoption-funnel/` | HTML dashboard | Feature-level adoption analysis |
| "Show me the adoption funnel for our key features" | `adoption-funnel/` | HTML dashboard | Sequential adoption and drop-off |

---

## PMM / VP of Marketing

| Trigger | Sub-skill | Output | Value |
|---------|-----------|--------|-------|
| "Generate ROI stories from our top accounts" | `customer-roi-stories/` | Markdown snippets | Batch ROI narratives for marketing collateral |
| "Which accounts have the best outcomes for a case study?" | `customer-roi-stories/` | Ranked list + snippets | Identify strongest ROI stories |
| "Full case study for Acme" | `case-study/` | HTML artifact | Deep single-account editorial narrative |

---

## Tips

- **Specify audience** when asking for a QBR: "internal" vs "customer-facing" changes the entire output.
- **Specify scope** for portfolio skills: "my accounts" vs "all accounts" vs "enterprise segment" drastically changes what gets queried.
- **Combine skills**: Run `book-of-business` first to scan the portfolio, then drill into specific accounts with `account-brief`, `health-score-breakdown`, or `success-plan`.
