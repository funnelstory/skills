# QBR Deck — Customer-Facing (External)

Generate a **customer-ready** Quarterly Business Review deck for a named account. This deck is designed to present TO the customer — it focuses on value delivered, outcomes achieved, and forward-looking partnership goals.

It does NOT include internal health scores, churn predictions, risk flags, or internal-only concerns. For an internal QBR with those details, use the `qbr-internal/` sub-skill instead.

## When to Use This Sub-Skill

Use this when the user asks for a customer-facing QBR, an external QBR, or wants to present a quarterly review to the customer. Keywords: "customer QBR", "external QBR", "present to the customer", "value review deck."

---

## Step 1: Identify the Account

```sql
SELECT account_id, name, domain, amount, health_score, prediction_score, expires_at
FROM accounts
WHERE name LIKE '%AccountName%'
LIMIT 10
```

Confirm with the user. Note `account_id` and `domain`.

## Step 2: Pull Account Data

Same query set as `qbr-internal/` — pull accounts, activities, notes, meetings, needle_movers, metrics, tasks. The difference is in what you INCLUDE vs EXCLUDE in the output.

### Key queries

- **Account details** — `SELECT * FROM accounts WHERE account_id = '<account_id>'`
- **Activities (last 90 days)** — grouped by `activity_name`, SUM(count)
- **Account metrics** — `account_metrics_latest` and `account_metrics_history` for trend data
- **Meetings** — summaries from the last 90 days (for relationship context and wins)
- **Notes** — scan for positive outcomes, customer feedback, completed initiatives
- **Needle movers** — focus on positive-impact items (impact > 0) and completed items (state = 'done' or 'closed')
- **Tasks** — completed tasks as evidence of delivered commitments

## Step 3: Filter for customer-appropriate content

**INCLUDE:**
- Usage metrics showing growth or adoption (activity counts, active users, feature adoption %)
- Completed initiatives and delivered outcomes (from needle movers, tasks, notes)
- Positive meeting highlights — wins acknowledged by the customer
- Metric improvements over the period
- Forward-looking goals and planned initiatives

**EXCLUDE:**
- Internal health scores, prediction scores, churn probability
- Internal-only notes (CSM observations about risk, internal escalations)
- Negative needle movers or risk flags (competitor mentions, pricing concerns)
- Support ticket details (unless the user specifically asks — and then frame as "issues resolved")
- Raw sentiment scores

## Step 4: Structure the Deck

1. **Title Slide** — "{Account Name} — Quarterly Business Review", quarter, date. Branded, professional.
2. **Partnership Overview** — Relationship timeline, contract summary (without exposing internal metrics), key contacts on both sides.
3. **Value Delivered This Quarter** — 3–5 cards, each with:
   - Outcome statement (e.g., "Expanded to 3 new teams")
   - Supporting metric (e.g., "Active users grew from 42 to 78")
   - How we got there (brief narrative from meetings/notes)
4. **Product Adoption & Usage** — Charts or tables showing:
   - Key activity trends (frame positively: "Your team ran X this quarter, up Y% from last quarter")
   - Feature adoption highlights (which capabilities are actively used)
   - User growth or engagement patterns
5. **Key Milestones Achieved** — Completed needle movers and tasks — positioned as joint achievements.
6. **Looking Ahead** — 2–3 goals for next quarter, framed as partnership initiatives. Sourced from open positive needle movers, planned features, or expansion discussions in meetings.
7. **Discussion Topics** — Open items for the customer conversation (non-threatening framing — "areas to explore together", not "concerns").

## Step 5: Output Format

- **Default**: PowerPoint (.pptx) — read and follow `/mnt/skills/public/pptx/SKILL.md`
- **Alternative**: HTML presentation or Markdown
- Tone: professional, warm, partnership-oriented. No internal jargon. No hedging language.

## Quality bar

- The customer should be able to read every word on every slide without seeing anything embarrassing or internal.
- Every "value delivered" card must cite a specific metric or outcome from FunnelStory data — no generic "great quarter."
- If data doesn't support strong positive framing, be honest but constructive: "We identified X as an area for deeper adoption next quarter" rather than fabricating wins.
- Frame usage metrics as the customer's achievement ("Your team processed X"), not the vendor's.
