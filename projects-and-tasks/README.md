# Projects and tasks

Create a project, update one, create a task, or add a note to a task. The written 30/60/90 plan stays in `success-plan/`. This skill is how those items get created in FunnelStory.

## Before any write

Call `read_resource` on `file://projects/guide.md` and follow it. It covers templates, funnel conditions, and who can call which tool. Do not invent a `list_project_templates` tool.

Resolve a named account with **Resolving a named account** in the root `SKILL.md`. `create_project` takes external `account_id` values, not `fs_id`.

## Tools

- `create_project` — new project linked to one or more accounts. When the project comes from a template, pass `source_project_id` so funnel conditions keep working if it is later renamed. **Admin and super-admin only.** If the tool is absent, say so and stop. Do not retry.
- `update_project` — change an existing project. **Admin and super-admin only.** Same rule if the tool is absent.
- `create_task` — any role. Link it to an account with the external `account_id`, and to a project with `project_id` when the task belongs to one.
- `add_task_note` — any role. Needs the task id, a title, and the note body.

## After a success plan

If the user already has a plan from `success-plan/` and asks to create the work, create one project for the account and one task per action, using the plan's owners and dates. Do not rewrite the plan here.
