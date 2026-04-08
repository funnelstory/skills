# Case Study — Publication-Ready HTML

Fetches all account data from FunnelStory's semantic database and outputs a complete, self-contained
HTML artifact that Claude renders as a canvas. The result looks like a real published case study —
with a dramatic editorial hero, stat cards, initiative cards, and pull quotes — all in a single file.

---

## Step-by-step process

### 1. Resolve the account

Use the `query_semantic_db` MCP tool for all queries.

If the user gave an `account_id`, use it directly. Otherwise:

```sql
SELECT account_id, name, domain FROM accounts WHERE name LIKE '%<company>%' LIMIT 5
```

If multiple results, ask the user to confirm. Note `account_id` and `domain`. If `domain` is null,
skip queries that filter by domain.

### 2. Fetch all account data in parallel

Pull from these tables simultaneously using `query_semantic_db`:

- `accounts` — core account info and computed scores
- `notes` — call transcripts, emails, internal updates
- `meetings` + `meeting_contacts` + `contacts` — AI-summarized meeting notes
- `topics` — conversation themes and sentiment
- `activities` — product usage events
- `users` — count of active users
- `tasks` — open/closed action items
- `account_metrics_latest` — workspace-defined metric counters
- `needle_movers` — strategic initiatives

### 3. Extract story elements

Before writing a single line of HTML, spend time identifying the right raw material:

- **3–5 key stats** to feature as hero cards. Format `metric_id` names for display: replace
  underscores with spaces, title-case each word, and expand known abbreviations
  (`ci_cd` → "CI/CD", `arr` → "ARR", `mrr` → "MRR", `nps` → "NPS"). So `ci_cd_runs` becomes
  "CI/CD Runs", `user_count` becomes "User Count", `features_count` becomes "Features Count".

- **The strongest outcome** for the headline — lead with what changed for the customer, not what
  product they installed.

- **One specific incident or moment** to open The Challenge. This must be grounded in actual data:
  a topic with negative sentiment, a meeting note describing friction, a spike in support tickets,
  a competitive evaluation mentioned in a note. If no concrete incident exists in the data, describe
  the structural tension the data reveals (e.g. "fragmented tool stack across 3 platforms" if the
  notes mention GitLab, Bitbucket, and GitHub). Do not manufacture a crisis.

- **What they used before** — scan notes and topics for mentions of competitor products,
  previous tools, or "before [product]" references. Include this as "Previous solution" in At a
  Glance if found.

- **Any direct customer quotes** from meeting summaries or notes — verbatim phrases attributed to
  named contacts. If none exist, skip the Pull Quote section entirely. Do not synthesize a quote
  and attribute it to "documented outcomes" or similar — this is misleading.

- **Forward-looking initiatives** — scan needle_movers and meeting notes for concrete next steps
  (specific features, integrations, expansions). These go in What's Next as initiative cards.

### 4. Output a complete HTML artifact

Output a single `<!DOCTYPE html>` document. Claude will render it
as a canvas artifact.

#### Design system

```css
:root {
  --primary: #1a1a2e; /* deep navy — headings, hero bg */
  --accent: #6c63ff; /* purple — stat numbers, borders, highlights */
  --accent-light: #f0eeff; /* light purple — card backgrounds, What's Next bg */
  --text: #2d2d2d;
  --muted: #6b7280;
  --bg: #ffffff;
  --border: #e5e7eb;
}
font-family:
  "Inter",
  system-ui,
  -apple-system,
  sans-serif;
```

#### Page structure

Include these Open Graph tags in `<head>` so the file previews well when shared:

```html
<meta property="og:title" content="{Company} × FunnelStory — Case Study" />
<meta property="og:description" content="{one-sentence outcome summary}" />
<meta property="og:type" content="article" />
```

#### Layout sections

**1. Hero** — Full-width, deep navy gradient (`#1a1a2e` → `#2d2b6b` → `#3a2d7a`). Include:

- A small eyebrow badge ("Customer Story") in the top-left
- The headline (outcome-led, 40–48px, max-width 720px)
- A subtitle (one sentence, muted)
- A horizontal row of 3–5 stat cards. **Important:** the stat row must be inside the hero div
  with `padding: 80px 60px 56px` (do NOT use 0 bottom padding — this causes the stat cards to
  clip against the section boundary). Each stat card: big bold number in `#a8a3ff`, label in
  `rgba(255,255,255,0.55)`. Use a top border on the stat row to divide it from the headline area.

**2. At a Glance** — Two-column grid. Left: a clean table with Company, Domain, Industry, Users on
platform, Customer since, Contract value, Previous solution (if found), Integrations. Right: a
bullet list of 4–6 key outcomes — one sentence each, punchy, no sub-bullets.

**3. The Challenge** — Narrative section. Open with the specific incident or structural tension
identified in step 3. Use a styled `blockquote` (left border in `--accent`) to call out a key
tension sentence from the data if one exists — something with specificity and edge, not a generic
pain statement.

**4. Pull Quote** (only if a real verbatim quote exists) — Place it here, between Challenge and
Solution, for maximum narrative impact. Centered, large italic text, `--accent` decorative
quotation marks, attributed to "First Name, Role, Company". If no real quote, omit this section.

**5. The Solution** — Narrative section. Call out 2–3 specific capabilities the account actually
used (verified from activities or notes), formatted as feature pills in a 3-column grid:
pill label (short, uppercase) + 1–2 sentence description.

**6. The Results** — 2-column grid of result cards. Each card: outcome-led bold heading, 2–3
sentences of context grounded in the data, metric shown in large `--accent` type at the bottom.

**7. What's Next** — Full-width section with `--accent-light` background. Instead of prose
paragraphs, render 2–4 initiative cards in a grid — each card shows an initiative name (from
needle_movers or meeting notes) and a one-sentence description of what it means for the customer.
If no initiatives are found in the data, write 1–2 forward-looking sentences but don't fabricate
specific project names.

**8. Footer** — Dark (`--primary`) background, centered. "FunnelStory · Customer Story · {domain}
· {year}".

#### Hard rules

- Headline leads with outcome, not company or product name
- Challenge must be grounded in at least one specific data point from the data (topic name, meeting
  note, ticket type, metric) — no generic opening paragraphs about industry trends
- Results card headings are outcome statements ("Renewal forecast shifted to quantified confidence"),
  not bare metrics ("88.7% retention")
- No synthesized Pull Quotes — if you can't attribute a quote to a named person in the data,
  omit the section
- What's Next uses initiative cards if needle_movers or meeting data supports it; otherwise short
  prose — never invented project names
- Use metric names exactly as they appear in the data — do not reinterpret them
- Never write: "measurable improvements", "operational efficiency", "seamless integration",
  "robust [anything]", "well-positioned to", "at scale", "streamlined workflows"
- All CSS inline in `<style>` tag. Google Fonts `@import` for Inter is acceptable. No JS
  frameworks. Do not include a print, save, or download PDF button.

After rendering, ask: "Want me to adjust the design, narrative focus, or any section?"
