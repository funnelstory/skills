# FunnelStory Skills

[Agent Skills](https://agentskills.io) for the FunnelStory MCP server. Each skill is a folder with a `SKILL.md`.

The skills today cover customer intelligence and GTM artifacts:

- **Account summary** — health, usage, contacts, tickets, and products for one account.
- **Meeting prep** — a short pre-call brief with recent activity, risks, open work, and context.
- **QBR** — a Quarterly Business Review deck: metrics, notes, activities, renewal readiness.
- **Case study** — a styled, self-contained HTML case study ready to share or publish.
- **Prioritize My Day** — a ranked daily action plan for a CSM using tasks, needle movers, notes, meetings, topics, support, and renewal/health signals.

All of them require the FunnelStory MCP to be connected so the agent can reach your data.

## Usage

#### Claude (Desktop, Web)

1. Connect the FunnelStory MCP server in Claude's MCP settings.
2. Add these skill folders as custom skills — see [Using skills in Claude](https://support.claude.com/en/articles/12512180-using-skills-in-claude).
3. Ask naturally: "account summary for Acme", "prep me for the Contoso call", etc.

#### Claude Code

1. Connect the FunnelStory MCP in Claude Code's MCP config.
2. Copy or symlink each skill folder into `~/.claude/skills/` (personal) or `<repo>/.claude/skills/` (project-scoped):

```bash
ln -s /path/to/funnelstory-skills/account-summary ~/.claude/skills/account-summary
```

3. Claude will auto-select a skill from its description, or you can invoke it directly (e.g. `/account-summary`).

More on Claude Code skills: [docs.anthropic.com](https://docs.anthropic.com/en/docs/claude-code/skills).

#### Cursor

1. Add the FunnelStory MCP server in Cursor's MCP settings.
2. Place each skill folder under `~/.cursor/skills/` (user-wide) or your project's skills path.
3. Chat normally — the `description` in each `SKILL.md` lets the agent match phrases like "meeting prep" or "case study for ...".

## Contributing

Each skill lives in its own folder with a single `SKILL.md` entrypoint. To add or improve a skill:

1. Create a new folder (lowercase, hyphens) with a `SKILL.md` inside.
2. Include YAML frontmatter with at least `name` and `description`. The description should explain what the skill does and when to trigger it.
3. Write step-by-step instructions in the body — SQL patterns, expected tables, output format.
4. Open a PR. Keep one skill per folder and one concern per skill.

For the general skill format, see [How to create custom skills](https://support.claude.com/en/articles/12512198-creating-custom-skills) and [Anthropic's template](https://github.com/anthropics/skills/tree/main/template).

## License

MIT — see [LICENSE](LICENSE)
