# FunnelStory Semantic Lead Reports (MCP + Claude)

## Purpose

Produce **repeatable, executive-style reports** from the customer's **FunnelStory semantic layer** (`accounts`, `activities`, `users`, and optional account fields). Report flavors share the same pattern; only labels, time windows, scoring rules, and activity semantics change.

**Common report names (pick one framing per run):**

| Framing | Typical use |
|--------|-------------|
| **Hot leads** | New or recent accounts with strong product usage worth immediate sales attention |
| **Product-qualified leads (PQL)** | Usage breadth/depth crosses a defined product threshold |
| **Conversion opportunities** | Free/trial/community cohorts showing signals correlated with upgrade or paid conversion |

The workflow is **not** tied to any single product or workspace: the operator supplies **workspace id**, **product/report naming**, **cohort filters**, and **scoring rules** (or reuses their existing FunnelStory agent SQL).

## Standard pipeline (three data passes + HTML)

1. **Metrics row** — Report period label, optional email subject line, and 1–3 KPI values (e.g. cohort size, prior-period comparison, count of qualified leads).
2. **Qualified accounts** — Ranked list: accounts meeting the **qualification rule** (composite score, thresholds, minimum activity diversity). Include fields the HTML layer needs: account identifier, display name/domain, formatted dates, activity breakdown text, contact fields, optional `account_id_encoded` for URLs.
3. **Reviewed, not prioritized (optional)** — Accounts with **meaningful volume** that **did not** qualify, each with a **plain-language `exclusion_reason`** (usually generated in SQL so logic is auditable). Omit this query if the customer does not need an "explain the misses" section.
4. **HTML** — Claude renders **mobile-first**, **card-based** email HTML (see [reference.md](reference.md)); **do not** expose internal scores in copy.

Automated FunnelStory agents add **email.send**; this skill stops at **deliverable HTML** unless the user asks to send mail.

## Prerequisites

- **FunnelStory MCP** enabled for the target workspace (login per tenant MCP server).
- **`get_data_connections`** — locate the **Semantic DB** (or equivalent) `connection_id`.
- **`execute_query`** — run SQLite against that connection. Set **`limit`** high enough for the longest result set (e.g. ≥ **25** if the server defaults would truncate ~20 rows).

## On-demand workflow

1. **Authenticate** — `login` (OTP flow) if required.
2. **Resolve `connection_id`** — `get_data_connections` → use the semantic / warehouse connection your workspace uses for `accounts` and `activities`.
3. **Confirm inputs with the user** — At minimum: **report type label**, **time window** (e.g. previous calendar month, last 30 days), **workspace UUID** for FunnelStory account links, **product/company name** for title and footer, **noise activity names** to exclude from "meaningful" usage, **score threshold** and **row limits**.
4. **Run SQL** — Execute the metrics query, then the qualified-lead query, then the optional flagged query. Adapt SQL from the customer's own FunnelStory agent config when available; otherwise start from the templates in [reference.md](reference.md).
5. **Normalize results for the renderer** — Map MCP `rows` into three blocks matching the HTML template: `report_metrics`, `qualified_leads`, `reviewed_not_prioritized` (empty list if step 3 skipped).
6. **Generate HTML** — Apply the **system** and **user** prompt patterns in [reference.md](reference.md), filling all **placeholders** (`WORKSPACE_ID`, `PRIMARY_SECTION_TITLE`, activity glossary, etc.). Output **raw HTML only**.

## Quality bar

- **Specific "why qualified"** copy per account, driven from **activity_detail** (or equivalent), not generic "high engagement."
- **Numbers** formatted with thousands separators in prose.
- **No internal scores** (`hot_score`, `activity_score`, etc.) in the visible report.
- **Equal-height KPI boxes** and **full-width cards** for lead lists (see reference).
