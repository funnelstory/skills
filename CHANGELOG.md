# Changelog

All notable changes to the FunnelStory Skills are recorded here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versioning follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [1.2.0] — 2026-10-14

Brings every sub-skill onto the product's health tiers and the current semantic database, and routes three workflows the MCP already supports.

### Added

- **Shared conventions** in the root `SKILL.md`: named-account resolution, health tiers, value and renewal, needle-mover filters, support data, missing data, and output files.
- **`flow-authoring/`** — routes flow work to the MCP flow resources (`file://flow/guide.md` and the guides it names) and to `save_flow`, `edit_flow`, `get_flows`, `preview_flow_triggers`, `run_flow`, and `debug_flow_step`.
- **`projects-and-tasks/`** — `create_project`, `update_project`, `create_task`, and `add_task_note`, after reading `file://projects/guide.md`. Project create and update are admin-only.
- **`prioritize-my-day/`** — today's open action cards, then Critical or At risk accounts and near renewals that have no card.

### Changed

- **Health tiers match Focus Areas:** Critical at 45 or below, At risk from 46 to 54, On track at 55 or above, on the displayed `health_score` (`50 + trunc(prediction_score * 100 / 2)`). Drivers sit beside the tier and do not change it. Accounts with no score are Unscored.
- **Identity lookup** calls `who_am_i` first. Authentication is described as OAuth 2.0 with PKCE.
- Portfolio value is `aggregate_amount`. A parent's renewal date is `earliest_child_expiry`. Portfolio skills skip `churned` accounts unless asked.
- Needle movers follow the product list (`source` `v2` or `relationship_map`) and are reported as impact tiers, not raw integers.
- Account briefs and meeting prep start from `account_profiles`. Meeting prep uses `calendar_events` for upcoming invites and `content_labels` for themes.
- `qbr-internal/`, `renewal-dashboard/`, `churn-risk-report/`, and `meeting-prep/` source support data through `references/linking-conversations-to-accounts.md`.
- `configure-data-model/` distinguishes `edit_data_model` (a partial edit) from `configure_data_model` (create or full replace).
- Account resolution ranks, proceeds, states the choice, and lists alternatives. It asks first only for parent-versus-child, and for a customer-facing deliverable with more than one plausible match.
- Output steps are client-neutral. The README table lists every routed folder.

### Removed

- **`qbr-deck/`** — unrouted duplicate of `qbr-internal/`. Use `qbr-internal/` or `qbr-external/`.
- Workspace-specific entitlement fields. Skills inspect `properties` for whatever the workspace carries.
- The reference to a `get_account` tool, and OTP login as the default authentication step.

### Upgrading

- **Claude Code / Cursor (symlinked):** `git pull` in the skills directory.
- **Claude Desktop / web:** re-download and re-add the **entire** skills directory, then remove the previous version.

---

## [1.1.0] — 2026-08-28

Broadens support-data coverage across ticketing and conversation platforms, and adds a
dedicated sub-skill for account support history.

### Added

- **`account-support-history/`** — new sub-skill. Ask for the latest tickets, open
  escalations, or recent support history on a named account and get an answer built from
  whatever ticketing or conversation platform that workspace uses. Tickets and chat
  conversations are presented as distinct bands, with open and escalated items surfaced
  first and each response carrying a short provenance line.
- **`references/linking-conversations-to-accounts.md`** — shared reference describing how
  to locate support data in a workspace and attribute it to the right account. Supports
  Salesforce Service Cloud, Zendesk, DevRev, and Slack Connect, including workspaces that
  run more than one platform or have migrated between them. Adapts by measuring coverage
  rather than assuming a fixed schema, and reports which method it used.
- **Version identifier** in the root `SKILL.md` frontmatter, so you can ask the agent
  which release you are running.
- **This changelog.**

### Improved

- **`account-brief/`** now sources support history through the shared reference, giving it
  the same platform coverage as the new sub-skill. Briefs for accounts on Zendesk, DevRev,
  or Slack-based support may include support activity that earlier releases did not — if
  support is material to an account, a fresh brief is worth running.
- Account briefs honor workspace profile `instructions` for metric naming and health
  conventions, so workspaces with custom terminology or a custom health index are reported
  in their own language.
- Account resolution ranks likely matches and states its choice with alternatives, rather
  than pausing to ask — useful in workspaces that carry one account per site.
- Column availability is verified before selection, improving portability across workspaces
  with differing schemas.
- Output distinguishes "no data for this account" from "not carried in this workspace", so
  an empty section is unambiguous.
- Customer conversations are labelled by visibility, keeping internal discussion out of
  customer-facing deliverables.

### Changed

- Root `SKILL.md` prerequisites now reference the current MCP tool surface
  (`query_semantic_db`, `read_resource`).

### Upgrading

- **Claude Code / Cursor (symlinked):** `git pull` in the skills directory.
- **Claude Desktop / web:** re-download and re-add the **entire** skills directory, then
  remove the previous version. Adding only the new sub-folder is not sufficient, because
  routing lives in the root `SKILL.md`.

---

## [1.0.0]

Initial public release: sub-skills for account briefs, book of business, meeting prep, QBR
decks, case studies, lead reports, expansion and upsell dashboards, churn risk, renewal
dashboards, adoption analysis, health score breakdowns, success plans, value emails,
executive sponsor coverage, feature requests, customer ROI stories, flow authoring, data
model configuration, and connection query authoring.
