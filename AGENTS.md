# Agent Instructions

Shared instructions for all coding agents (Claude Code, OpenAI Codex, etc.) working in this repository. This file is the canonical source; do not duplicate it. If a Claude Code setup does not load `AGENTS.md` directly, add a `CLAUDE.md` that contains only `@AGENTS.md`.

## Repository

- The repository is a single standalone Ansible role, `process_exporter`, with the role contents at the repository root (`tasks/`, `handlers/`, `vars/`, `files/`). There is no `meta/`, `defaults/` or `templates/`.
- `README.md` is the user documentation; keep it consistent with the role's behaviour.

## Project context

- For substantial work, read `/PROJECT_CONTEXT.md` first.
- Keep `PROJECT_CONTEXT.md` current during long or multi-step work, and update it before handoff or completion when durable handoff state has changed.
- Store only non-obvious information useful to a fresh agent: important decisions and rationale, non-obvious constraints, current or incomplete state, blockers, next steps, and relevant non-obvious run/verification commands.
- Remove or revise stale information.
- Do not store conversation history, trivial edits, information easily rediscovered from the repository, documentation already maintained elsewhere, or secrets/credentials.

## Working rules

- Follow existing project architecture, conventions, and tooling.
- Keep changes scoped to the task.
- Run applicable existing validation (see `PROJECT_CONTEXT.md`); never claim checks that were not run or did not pass.

## Maintaining this file

- Keep this file concise and general. Do not put task-specific instructions here; use `PROJECT_CONTEXT.md` for task state.
