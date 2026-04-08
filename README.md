# FunnelStory Skills

[Agent Skills](https://agentskills.io) for the FunnelStory MCP server.

## Layout

- **`SKILL.md`** (repo root) — main skill entrypoint. It covers MCP setup, how to scope requests (named account vs portfolio), and a routing table to the right sub-skill.
- **Sub-folders** — one deliverable or workflow each, with a **`README.md`** the agent reads after routing. Some folders have extra reference files (for example `lead-reports/reference.md`).

Everything requires the FunnelStory MCP so the agent can query your Customer Intelligence Graph.

### Sub-skills

| Folder | What it’s for |
| --- | --- |
| `account-brief/` | Single-account summary / health deep-dive (internal) |
| `book-of-business/` | “My book” — assigned accounts / portfolio snapshot |
| `prioritize-my-day/` | Ranked daily priorities for a CSM across risk, renewal, and expansion |
| `meeting-prep/` | Pre-call brief for a named account |
| `qbr-deck/` | Quarterly Business Review deck (e.g. PowerPoint) |
| `case-study/` | Publishable styled HTML case study |
| `lead-reports/` | Hot leads, PQL, trial conversion (see `reference.md` too) |
| `expansion-dashboard/` | Land-and-expand: seats, users, sites, geography |
| `upsell-dashboard/` | Higher tier, premium modules, add-on SKUs |
| `churn-risk-report/` | At-risk accounts, retention drivers |
| `adoption-gap/` | Low usage vs entitlements / shelfware |
| `value-email/` | Customer-facing ROI / value recap email |
| `exec-sponsor/` | Executive sponsor coverage and gaps |
| `flow-authoring/` | Workflow / automation design |

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
2. Update root **`SKILL.md`**: add a row to the routing table and extend the `description` if new triggers are needed.
3. Use YAML frontmatter on root `SKILL.md` with at least `name` and `description`.
4. Open a PR. Keep one concern per sub-folder.

For the general skill format, see [How to create custom skills](https://support.claude.com/en/articles/12512198-creating-custom-skills) and [Anthropic's template](https://github.com/anthropics/skills/tree/main/template).

## License

MIT — see [LICENSE](LICENSE)
