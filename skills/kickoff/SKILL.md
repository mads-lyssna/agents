---
name: kickoff
description: Turn a Linear ticket into an executable implementation plan. Reads the ticket and all its context, sets it to In Progress, explores the codebase, then produces a written plan via the plan skill. Starting point is a Linear ticket (URL, short ID, or the current branch's issue); output is a ready-to-execute plan.
---

# Kickoff

Take a Linear ticket and produce a written implementation plan an executor can pick up cold.

This skill **orchestrates** two others — it does not reimplement them:

- **Linear access** goes through the linear skill. Read `skills/linear/SKILL.md` and use `linear` as described there. Don't encode `linear` command shapes here.
- **Plan authoring** goes through the plan skill. Read `skills/plan/SKILL.md` and follow its process and templates. Don't reimplement clarification, shape choice, location, or validation here.

Kickoff's job is the glue: resolve the ticket, gather its context, confirm it's worth starting, mark it started, do ticket-grounded exploration, then hand a well-briefed problem to the plan skill.

## 1. Identify the ticket

- If the user gave a `linear.app` URL or short ID (e.g. `ENG-123`), use it.
- Otherwise resolve the current branch's issue before asking. (linear skill: `linear issue id`.)

## 2. Read the ticket and gather context

Via the linear skill, pull the full issue with comments, then follow the threads that matter: linked documents, sub-issues, related/blocking issues, attachments, and project context. Read enough to understand the problem on its own terms.

Synthesize what you learn — don't paste raw command output back to the user.

## 3. Sufficiency check (lightweight)

Confirm two things only:

- You're working the **right** ticket (matches what the user intended).
- The ticket describes a **real, actionable kernel** — a problem worth starting on, not an empty placeholder.

If the ticket is essentially empty or you can't tell what it's asking for, stop and ask the user before going further.

**Do not run a full requirements interrogation here.** Rigorous clarification of scope, edge cases, and tradeoffs is the plan skill's Clarify step — defer to it rather than duplicating it. This gate is just "is there enough to start, and is it the right thing".

## 4. Mark the ticket In Progress

Once the sufficiency check passes, transition the ticket to started via the linear skill (e.g. `linear issue start <ID>`, or `linear issue update <ID> -s "In Progress"`).

**This is the only write Kickoff makes to Linear.** Do not post comments, change assignees, or write anything else back to the ticket.

## 5. Explore the codebase

Do ticket-grounded exploration: find existing patterns and integration points the work will touch, and verify the thing isn't already implemented. Never assume absence — check.

For non-trivial discovery, use Explore subagents to keep context lean. Keep the output informal — notes, file pointers, identified patterns.

This is a head-start, not a replacement, for the plan skill's own Explore step. It exists so the handoff carries real findings instead of a bare ticket.

## 6. Hand off to the plan skill

Invoke the plan skill to produce the artifact. Carry forward everything you've gathered so it doesn't get re-derived:

- The ticket's problem statement and any acceptance / done criteria (map them into the plan's Acceptance Criteria).
- Constraints, decisions, and links surfaced in the ticket and its comments.
- Your exploration findings — file pointers and patterns.

Seed the plan's Context with a link back to the ticket and an "Implements `<ID>`" reference so the artifact traces to its source.

The plan skill owns the rest: depth of clarification, single-file vs. index shape, **location (defer to its detection and confirmation)**, templates, and cold-read validation. Don't override those choices from here.
