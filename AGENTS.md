# Agent Instructions

Canonical shared instructions for coding agents working in this repository.

## Project context

- For substantial work, read `PROJECT_CONTEXT.md` first.
- Keep `PROJECT_CONTEXT.md` current during long or multi-step work and update it before handoff or completion when durable handoff state has changed.
- Store only non-obvious information useful to a fresh agent: important decisions and rationale, non-obvious constraints, current or incomplete state, blockers, next steps, and relevant non-obvious run or verification commands.
- Remove or revise stale information.
- Do not store conversation history, trivial edits, information easily rediscovered from the repository, documentation already maintained elsewhere, or secrets and credentials.

## Working rules

- Follow existing project architecture, conventions, and tooling.
- Keep changes scoped to the task.
- Run applicable existing validation; never claim checks that were not run or did not pass.
- Keep production behavior unchanged unless the task explicitly requires changing it.

## Maintaining this file

- Keep these instructions concise and repository-wide.
- Do not put task-specific instructions here.
- Keep implementation-specific details in `PROJECT_CONTEXT.md` when they cannot be reliably recovered from the repository.
