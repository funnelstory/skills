# Value Summary Email (Customer-Facing)

## Audience

- **CSM:** Sends or pastes into email client / sequences.
- **VP CS:** Reviews tone for a strategic account; rarely bulk-generates without naming accounts.

## Step 1 — Confirm

1. **Account name(s)** — one primary account per email unless user explicitly wants a joint customer email.
2. **Recipient persona** (optional): champion vs executive — adjusts tone and depth.
3. **Format:** Markdown + **HTML body** (email-safe) default; plain Markdown only on request.

## Step 2 — Gather evidence

Same data family as account brief (adapt SQL to schema):

- `accounts` — ARR, renewal, health (for internal sanity; do not lead with internal health in customer copy unless framed carefully).
- `activities` — concrete usage wins (volumes, trends last 90 days).
- `account_metrics_history` / `account_metrics_latest` — KPIs the customer cares about.
- `notes` / `meetings` — **short** paraphrased wins or quotes (strip HTML; no internal-only content).
- `needle_movers` — positive outcomes only for customer-facing narrative.

**Never** paste internal ticket IDs, internal emails, or competitor attack lines unless user approves.

## Step 3 — Email structure

1. **Subject line** — specific (e.g. "How your teams used [Product] last quarter").
2. **Opening** — warm, 1–2 sentences, thank you.
3. **Impact** — 3 bullets with **numbers** (adoption %, incidents resolved, time saved if credible).
4. **Looking ahead** — partnership tone; optional CSM-scheduled call.
5. **Sign-off** — CSM name placeholder.

## Step 4 — HTML rules (when generating HTML)

- Inline CSS, email-safe, no external scripts.
- Single column, max-width ~680px.
- Do not fabricate quotes — attribute only what exists in data.

## Quality bar

- Sounds like a human who knows the account, not a mail-merge.
- If data is thin, shorten the email and say what you'll track next quarter rather than padding.
