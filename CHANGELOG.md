# Changelog

All notable changes to the FunnelStory Skills are recorded here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versioning follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
