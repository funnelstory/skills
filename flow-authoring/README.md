# Flow authoring

Create or change a FunnelStory flow (an agent or workflow). The server already ships the guides. Read them. Do not reconstruct a flow from memory or from this file.

## When to use this

The user wants to create, edit, run, or debug a flow. A question about an account's data is not this skill.

## Before any JSON

Call `read_resource` on these, in order:

1. `file://workspace/profile.json` — terminology and what must not be surfaced. Required when an AGENT step writes customer-facing copy.
2. `file://flow/guide.md` — the entry point. Follow it.

Then only the ones the flow needs:

- `file://flow/ops.md` — step operation types
- `file://flow/functions.md` — CALL functions
- `file://flow/patterns.md` — which shape fits
- `file://flow/pptx.md` — the flow uses `pptx.create`. Also call `list_templates` and `read_template`.
- `file://flow/pdf.md` — the flow uses `pdf.create`
- `file://flow/outcome_agent.md` — the flow is an outcome program

## Tools

- `get_flows` — list flows, or fetch one by `flow_id`
- `save_flow` — create a flow, or replace one wholesale. Omitting a field on a replace clears it. Do not use this to change one step.
- `edit_flow` — every targeted change after the flow exists
- `preview_flow_triggers` — who a trigger would select
- `run_flow` — run it
- `debug_flow_step` — run one step

`file://flow/guide.md` owns the gates (validate SQL with `query_semantic_db` before save, draft versus published, chat flows storing `@.response`). Follow those gates. Do not invent a shorter path.
