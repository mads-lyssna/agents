# agents

Personal configuration for coding agents: shared working conventions in [`AGENTS.md`](AGENTS.md) and task-specific skills under [`skills/`](skills/).

## Repository layout

- `AGENTS.md` defines shared rules and preferences for editing, tools, commits, testing, language conventions, and devcontainers.
- `skills/<name>/SKILL.md` describes when a skill applies and the workflow an agent should follow.
- `skills/plan/templates/` contains reusable templates for single-file and multi-file implementation plans.

## Skills

Skills are loaded based on the task context rather than invoked for every request.

| Skill                                  | Purpose                                                                                                                                                  |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [datadog](skills/datadog/SKILL.md)     | Use the `pup` CLI for Datadog observability work across logs, metrics, traces, monitors, incidents, dashboards, security, and related APIs.              |
| [github](skills/github/SKILL.md)       | Use the `gh` CLI for GitHub repository, issue, PR, review, search, and Actions work.                                                                     |
| [kickoff](skills/kickoff/SKILL.md)     | Turn a Linear ticket into an implementation-ready plan by gathering context, starting the issue, exploring the codebase, and invoking the plan workflow. |
| [linear](skills/linear/SKILL.md)       | Use the `linear` CLI to work with issues, projects, cycles, milestones, initiatives, documents, and comments.                                            |
| [notion](skills/notion/SKILL.md)       | Use the `ntn` CLI for Notion pages, data sources, files, and authenticated API work.                                                                     |
| [plan](skills/plan/SKILL.md)           | Produce a written implementation plan that a future executor can use without the original conversation.                                                  |
| [pr-review](skills/pr-review/SKILL.md) | Collaboratively review a pull request by gathering context, independently reviewing the diff, and triaging reviewer feedback.                            |
| [sentry](skills/sentry/SKILL.md)       | Use the `sentry` CLI for read-only Sentry work across issues, events, traces, spans, logs, replays, releases, and related APIs.                          |

## Plan templates

The [plan skill](skills/plan/SKILL.md) can produce either a compact single-file plan or a multi-file plan for larger work:

- [`single-file.md`](skills/plan/templates/single-file.md) — self-contained plan template.
- [`index.md`](skills/plan/templates/index.md) — entry point and task checklist for a multi-file plan.
- [`task.md`](skills/plan/templates/task.md) — detailed template for an individual task.
- [`supporting-context.md`](skills/plan/templates/supporting-context.md) — optional shared research, decisions, and references.
