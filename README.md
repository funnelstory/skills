# FunnelStory CS Skills

[Agent Skills](https://agentskills.io) for the FunnelStory MCP server.

## Get started

Download: https://github.com/funnelstory/skills/archive/main.zip

## Layout

- **`SKILL.md`** (repo root) — main skill entrypoint. It covers MCP setup, how to scope requests (named account vs portfolio), and a routing table to the right sub-skill.
- **Sub-folders** — one deliverable or workflow each, with a **`README.md`** the agent reads after routing. Some folders have extra reference files (for example `lead-reports/reference.md`).

Everything requires the FunnelStory MCP so the agent can query your Customer Intelligence Graph.

### Sub-skills

| Folder | What it’s for |
| --- | --- |
| `account-brief/` | Single-account summary / health deep-dive (internal) |
| `account-support-history/` | Latest tickets, open escalations, support history for one account |
| `health-score-breakdown/` | Why the health score is what it is — factors, weights, what to fix |
| `success-plan/` | Forward-looking action plan for one account |
| `meeting-prep/` | Pre-call brief for a named account |
| `qbr-internal/` | Internal QBR prep deck (risks, honest metrics) — not for the customer |
| `qbr-external/` | Customer-facing QBR focused on value delivered |
| `case-study/` | Publishable styled HTML case study |
| `value-email/` | Customer-facing ROI / value recap email |
| `book-of-business/` | “My book” — assigned accounts / portfolio snapshot |
| `prioritize-my-day/` | Today’s action cards, plus Critical and At risk accounts with no card |
| `churn-risk-report/` | At-risk accounts, retention drivers |
| `renewal-dashboard/` | Renewals in the next N days, ranked by risk, with pre-renewal actions |
| `expansion-dashboard/` | Land-and-expand: seats, users, sites, geography |
| `upsell-dashboard/` | Higher tier, premium modules, add-on SKUs |
| `adoption-gap/` | Low usage vs entitlements / shelfware |
| `exec-sponsor/` | Executive sponsor coverage and gaps |
| `lead-reports/` | Hot leads, PQL, trial conversion (see `reference.md` too) |
| `feature-requests/` | What customers are asking for, ranked across accounts (PM) |
| `adoption-funnel/` | Feature-level adoption across the customer base (PM) |
| `customer-roi-stories/` | Batch ROI snippets across accounts (marketing) |
| `connection-queries/` | Connection query syntax (SQL, SOQL, HubSpot, Attio blocks) |
| `configure-data-model/` | Data model configuration via MCP (mappings, joins, preview, save) |
| `flow-authoring/` | Create or edit a flow; reads the MCP flow guides |
| `projects-and-tasks/` | Create a project or task, or add a note to a task |

Shared material: `templates/` (HTML dashboard and one-pager layouts) and `references/` (support-data linkage, troubleshooting, persona use cases). Conventions that every sub-skill follows — account resolution, health tiers, value and renewal, needle movers, support data, missing data, output files — live in the **Shared conventions** section of the root `SKILL.md`.

## Usage

#### Claude (Desktop, Web)

1. Connect the FunnelStory MCP server in Claude's MCP settings.
2. Add this repo (or the folder containing root `SKILL.md` and the sub-folders) as a custom skill — see [Using skills in Claude](https://support.claude.com/en/articles/12512180-using-skills-in-claude).
3. Ask naturally; the root `description` and routing table steer the agent to the right `README.md`.

#### Claude Code

1. Connect the FunnelStory MCP in Claude Code's MCP config.
2. Copy or symlink the **whole** skills directory so root `SKILL.md` stays next to the sub-folders — for example:

```bash
ln -s /path/to/funnelstory-skills ~/.claude/skills/funnelstory-skills
```

3. The agent uses the root skill’s routing instructions, or you can reference the skill by name from frontmatter (`funnelstory`).

More on Claude Code skills: [docs.anthropic.com](https://docs.anthropic.com/en/docs/claude-code/skills).

#### Cursor

1. Add the FunnelStory MCP server in Cursor's MCP settings.
2. Place the repo (root `SKILL.md` + sub-folders) under `~/.cursor/skills/` or your project skills path.
3. Chat normally — the root skill describes when to use FunnelStory data; sub-folder `README.md` files hold the detailed steps.

## Contributing

1. Prefer **new workflows** as a new sub-folder with a clear **`README.md`** (steps, SQL patterns, output expectations).
2. Update root **`SKILL.md`**: add a row to the routing table and extend the `description` if new triggers are needed. Follow its **Shared conventions** rather than restating or overriding them.
3. Keep examples generic. This is a public repository — no customer names, workspace-specific field names, real emails, or data samples from a real workspace.
4. Use YAML frontmatter on root `SKILL.md` with at least `name` and `description`, and bump `version`.
5. Add a `CHANGELOG.md` entry.
6. Open a PR. Keep one concern per sub-folder.

For the general skill format, see [How to create custom skills](https://support.claude.com/en/articles/12512198-creating-custom-skills) and [Anthropic's template](https://github.com/anthropics/skills/tree/main/template).

## License

MIT — see [LICENSE](LICENSE)
