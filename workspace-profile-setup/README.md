# Workspace Profile Setup

Help a workspace admin draft — and test — the workspace profile that every other sub-skill reads first (`file://workspace/profile.json`). The output is a set of values the admin applies on [Workspace Grounding](https://app.funnelstory.ai/internal/console).

**Audience:** workspace admins (`admin` or `super_admin` from `who_am_i`). Typically run once at onboarding, then whenever the data model, sources, or conventions change. Anyone else is stopped in Step 0.

**This skill never writes workspace configuration.** It reads, measures, asks, drafts, and tests. The admin applies the result. Update replaces the whole profile.

---

## Why the profile matters

The profile is the workspace's standing context. Its `instructions` field overrides the defaults in every sub-skill README and in `file://semantic/usage.md`. A good profile is the difference between a brief that says "amount: 125000" and one that says "ARR $125K, renews in 41 days, owned by the Enterprise CSM team."

Fields in the profile:

| Field | Holds |
|---|---|
| `company_name`, `domain`, `additional_domains` | Who the workspace belongs to. Additional domains matter: they tell skills which email domains are internal (vendor) rather than customer. |
| `description` | A short business description. |
| `offerings` | Product lines, each with the features customers name. |
| `competitors`, `competitor_domains` | Who customers compare you with. The console form edits competitor names. It has no competitor-domains field, so leave `competitor_domains` as fetched. |
| `instructions` | Free-text rules every skill must follow: terminology, value labels, ownership fields, support-data findings, and customer-facing guardrails. |

**What the profile cannot change:** health. `health_score` and its tiers (Critical ≤ 45, At risk 46–54, On track ≥ 55) are product-defined so that Claude and the FunnelStory UI always show the same thing, and skills ignore profile rules that try to redefine them. The profile can still restrict *where* health appears — for example, never in customer-facing output.

---

## Step 0 — Set up

1. Call `who_am_i`. Continue only when `role` is `admin` or `super_admin`. Otherwise stop. Do not read the profile, measure the workspace, or draft anything. Tell the user these settings are limited to a FunnelStory workspace admin, and they should ask an admin to run this setup.
2. Read `file://workspace/profile.json`. Summarize which fields are filled and which are empty.
   - If `instructions` already has content, treat it as the baseline. Propose changes as edits to it. Never drop an existing rule without asking.
3. Read `file://semantic/usage.md` and `file://semantic/schema.sql`.
4. Tell the user the plan in two lines: measure the data (about 15–25 read-only queries), ask a short set of questions the data cannot answer, then draft and test.

---

## Step 1 — Measure the workspace

All queries are read-only. Run them, keep the results, and do not show raw output unless asked — summarize in Step 2. Confirm columns with `pragma_table_info` before relying on any column not in the schema resource.

### 1a — What is populated

```sql
SELECT name FROM sqlite_master WHERE type = 'table' ORDER BY name;
```

Then count rows in the tables skills depend on (skip any that do not exist):

```sql
SELECT 'accounts' AS t, COUNT(*) AS n FROM accounts
UNION ALL SELECT 'contacts', COUNT(*) FROM contacts
UNION ALL SELECT 'tickets', COUNT(*) FROM tickets
UNION ALL SELECT 'chat_threads', COUNT(*) FROM chat_threads
UNION ALL SELECT 'meetings', COUNT(*) FROM meetings
UNION ALL SELECT 'calendar_events', COUNT(*) FROM calendar_events
UNION ALL SELECT 'notes', COUNT(*) FROM notes
UNION ALL SELECT 'activities', COUNT(*) FROM activities
UNION ALL SELECT 'needle_movers', COUNT(*) FROM needle_movers
UNION ALL SELECT 'account_profiles', COUNT(*) FROM account_profiles
UNION ALL SELECT 'action_cards', COUNT(*) FROM action_cards
UNION ALL SELECT 'account_metrics_latest', COUNT(*) FROM account_metrics_latest
UNION ALL SELECT 'content_labels', COUNT(*) FROM content_labels
UNION ALL SELECT 'audiences', COUNT(*) FROM audiences;
```

Empty tables become "Not available in this workspace" for every skill. Note them — they are often a connector or data-model gap worth fixing at the source rather than in the profile.

### 1b — Account shape

```sql
SELECT
  COUNT(*) AS accounts,
  SUM(CASE WHEN parent_account_id IS NULL THEN 1 ELSE 0 END) AS top_level,
  SUM(CASE WHEN parent_account_id IS NOT NULL THEN 1 ELSE 0 END) AS children,
  SUM(CASE WHEN churned = 1 THEN 1 ELSE 0 END) AS churned,
  SUM(CASE WHEN health_score IS NULL THEN 1 ELSE 0 END) AS unscored,
  SUM(CASE WHEN aggregate_amount > 0 THEN 1 ELSE 0 END) AS with_value,
  SUM(CASE WHEN json_array_length(assignees) > 0 THEN 1 ELSE 0 END) AS with_assignees
FROM accounts;
```

```sql
SELECT domain, COUNT(*) AS accounts
FROM accounts GROUP BY domain HAVING accounts > 1
ORDER BY accounts DESC LIMIT 20;
```

Children and shared domains tell you what a child account represents (site, subsidiary, business unit, product instance). Ask the admin to name it in Step 2.

### 1c — Commercial values

```sql
SELECT
  MIN(aggregate_amount) AS min_value,
  MAX(aggregate_amount) AS max_value,
  AVG(aggregate_amount) AS avg_value,
  SUM(CASE WHEN parent_account_id IS NULL AND amount = 0 AND aggregate_amount > 0 THEN 1 ELSE 0 END) AS parents_rolled_up,
  SUM(CASE WHEN earliest_child_expiry IS NOT NULL OR expires_at IS NOT NULL THEN 1 ELSE 0 END) AS with_renewal_date
FROM accounts
WHERE churned = 0 OR churned IS NULL;
```

The range is a hint, not an answer. Whether the value is ARR, ACV, TCV, or MRR — and in what currency — comes from the admin.

### 1d — Ownership and custom properties

Inventory the keys in `properties`:

```sql
SELECT j.key, COUNT(*) AS accounts
FROM accounts a, json_each(a.properties) j
GROUP BY j.key ORDER BY accounts DESC LIMIT 80;
```

For every key that looks like ownership (contains `csm`, `owner`, `manager`, `am`, `tam`, `renewal`, `success`), measure coverage and value shape — email, name, or id:

```sql
SELECT
  COUNT(*) AS total,
  SUM(CASE WHEN json_extract(properties, '$.<key>') IS NOT NULL
            AND json_extract(properties, '$.<key>') <> '' THEN 1 ELSE 0 END) AS filled,
  SUM(CASE WHEN json_extract(properties, '$.<key>') LIKE '%@%' THEN 1 ELSE 0 END) AS looks_like_email
FROM accounts WHERE churned = 0 OR churned IS NULL;
```

Compare against `assignees` coverage from 1b. Also look at who uses the workspace:

```sql
SELECT user_role, COUNT(*) AS users FROM workspace_users GROUP BY user_role ORDER BY users DESC;
```

Keys that look like segment, tier, region, or industry are candidates for the Segments section. Record **key names and coverage only** — never copy individual people's names or emails into the draft.

### 1e — Products, metrics, and usage vocabulary

```sql
SELECT metric_id, product_id, COUNT(*) AS accounts
FROM account_metrics_latest
GROUP BY metric_id, product_id ORDER BY accounts DESC LIMIT 60;
```

```sql
SELECT activity_name, COUNT(DISTINCT account_id) AS accounts, SUM(count) AS events
FROM activities
WHERE timestamp >= datetime('now', '-90 days')
GROUP BY activity_name ORDER BY accounts DESC LIMIT 40;
```

Compare `product_id` values and activity names with the `offerings` in the profile. Mismatched naming (the data says `pkg_analytics_v2`, people say "Analytics") belongs in the Terminology section.

### 1f — Health and prediction

```sql
SELECT
  CASE
    WHEN health_score IS NULL THEN 'Unscored'
    WHEN health_score <= 45 THEN 'Critical'
    WHEN health_score <= 54 THEN 'At risk'
    ELSE 'On track'
  END AS tier,
  COUNT(*) AS accounts
FROM accounts
WHERE churned = 0 OR churned IS NULL
GROUP BY tier;
```

```sql
SELECT prediction, COUNT(*) AS accounts FROM accounts GROUP BY prediction;
```

This is how the tiers the UI shows distribute across this workspace — share it as context, not as a decision. Health score and tiers are product-defined and are not set in the profile. If the admin's teams rely on a different health measure (a property or a source metric), note it as feedback for their FunnelStory contact; do not put it in `instructions`. Unscored accounts are worth flagging as a data gap.

### 1g — Support data

Run Steps 1–4 of `references/linking-conversations-to-accounts.md`. Keep its findings per source: table, linkage mechanism, coverage, exclusions, and internal vs external channel conventions. These go into the Support data section so other skills can skip discovery.

### 1h — Needle movers, themes, and segments

```sql
SELECT source, label, state, COUNT(*) AS n
FROM needle_movers GROUP BY source, label, state ORDER BY n DESC LIMIT 40;
```

```sql
SELECT a.name, COUNT(aa.account_id) AS accounts
FROM audiences a LEFT JOIN audience_accounts aa ON aa.audience_id = a.id
GROUP BY a.id ORDER BY accounts DESC LIMIT 30;
```

Confirm the `audience_accounts` join columns with `pragma_table_info` first. Saved audiences are often the segments people already talk about ("Enterprise", "Strategic 50").

---

## Step 2 — Share findings, then ask

Open with a short findings summary: what is populated, what is empty, how accounts are structured, ownership coverage, health distribution, support linkage coverage, and anything surprising. Flag data gaps separately from profile decisions.

Then ask only what the data cannot answer. **Ask at most three questions per turn**, and offer the answer the data suggests as the default wherever there is one.

**Company and products**
- Is the `description` accurate? What should it add or drop?
- Which product lines and features should `offerings` list, in the names customers use?
- Who are the competitors, and their domains? Any additional internal domains (subsidiaries, acquired brands)?

**Value and renewal**
- What does `aggregate_amount` represent — ARR, ACV, TCV, or MRR — and in what currency? What label should reports use?
- What is a child account here (site, subsidiary, business unit, product instance)? Should portfolio views report parents only?

**Health visibility** (health itself is product-defined and not asked about)
- May health tiers and scores appear in internal deliverables only, or also in customer-facing ones? (The usual answer is internal only.)

**Ownership and segments**
- Which field is authoritative for the CSM? For the account manager, renewal owner, and executive sponsor, if they exist?
- How are segments defined — a property, an audience, or a value band?

**Support**
- Do the measured linkage findings match how your support teams work? Any source that should be excluded?

**Guardrails**
- What must never appear in customer-facing output? (Common answers: health scores, prediction outcomes, internal notes and channels, competitor mentions, pricing discussions, other customers' names.)
- Is anything never to be surfaced at all, even internally?
- Any terminology rules? (For example "customer" vs "account", product naming, words to avoid.)

---

## Step 3 — Draft

Produce two parts.

### Part 1 — Structured fields

A table per field: current value, proposed value, and source (`Data`, `Admin`, or `Both`). Leave a field unchanged unless there is a reason.

### Part 2 — `instructions`

Write it in the format below so every sub-skill can find what it needs. Rules for the text:

- **Imperative and specific.** Name fields exactly (`properties.csm_owner`, not "the CSM field").
- **Short.** It is read on every request. Aim for under 500 words; move anything longer to a team skill.
- **No personal data or secrets.** No individual names, emails, phone numbers, API keys, or URLs to internal systems. Refer to roles and fields.
- **Only what differs from the defaults.** If the workspace uses the product health tiers, a single line saying so is enough.

```
WORKSPACE INSTRUCTIONS — v<n>, <YYYY-MM-DD>, maintained by <team or role>

TERMINOLOGY
- <term in data> means <term people use>. Always use the latter in output.

VALUE AND RENEWAL
- `aggregate_amount` is <ARR|ACV|TCV|MRR> in <currency>. Label it "<label>".
- A child account is a <site|subsidiary|business unit>. Portfolio views report <parents only|parents and children>.

OWNERSHIP
- CSM: <`assignees`|properties.<key>>. Account manager: properties.<key>. Renewal owner: properties.<key>.

SEGMENTS
- Segment comes from <properties.<key>|audience "<name>">. Values: <list>.

PRODUCTS AND USAGE
- <metric_id or product_id> measures <product line>. <activity_name> means <plain-language action>.

SUPPORT DATA
- <table>, source <source>: linkage <mechanism>, coverage <n>% (measured <date>). Exclude <noise>. Channels prefixed <prefix> are internal.

CUSTOMER-FACING RULES
- Never show: <list, e.g. health score and tier, prediction outcome>.
- <Any tone or framing rule>.

ALWAYS / NEVER
- <Any remaining workspace-wide rule.>
```

Show a review copy with each line's source (`Data` / `Admin`) beside it, and a clean copy for pasting.

---

## Step 4 — Test before saving

The draft is not saved yet. Tell the admin that for this test the draft `instructions` take precedence over the saved profile, then run these three prompts. Pick a real parent account and a real CSM with the admin.

| Prompt | What it checks |
|---|---|
| "Tell me about <large parent account>" | Value label, parent/child handling, ownership; health matches the UI |
| "How's the book for <CSM>?" | Ownership field |
| "Draft a value email to <account>" | Nothing on the never-show list appears |

For each prompt, say what the draft changed compared with the defaults and anything that still looks wrong. Revise the draft and re-test until the admin is satisfied. Do not store the prompts. A later "check whether our profile is still right" remeasures and compares with the saved profile.

---

## Step 5 — Apply

Give the admin the values field by field, then walk them through applying them:

1. Open [Internal Console](https://app.funnelstory.ai/internal/console) and select **Workspace Grounding**.
2. Click **Fetch Grounding** so the form holds the current profile.
3. Change only the drafted fields. The form has Company Name, Domain, Description, Instructions, Offerings (JSON array of `{ "name", "features" }`), Competitors (JSON array of names), and Additional Domains (JSON array). Leave `competitor_domains` as fetched.
4. Click **Update Grounding**. The confirm dialog shows the full payload, and the update replaces the entire profile. Read that payload with the admin and confirm every field they did not mean to change is still there before they confirm.
5. After they say it is saved, read `file://workspace/profile.json` again and confirm it matches the draft. Re-run one of the three prompts with no draft in context.
6. Recommend one named owning team for the profile, and re-running this skill when a source is added or migrated, the data model changes, recorded support coverage stops matching spot checks, or at least quarterly.

---

## Quality bar

- Every rule in the draft traces to a measurement or an admin answer.
- Nothing in the draft identifies an individual or exposes a credential or internal URL.
- Data gaps are reported as gaps, not papered over with instructions.
- Nothing in the draft redefines health score or tiers.
- The admin sees the effect of the draft on real prompts before saving it.
