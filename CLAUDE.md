# CLAUDE.md

> Authoritative, agent-agnostic instructions live in **AGENTS.md**, imported below.
> Do not duplicate that content here — edit it in `AGENTS.md`.

@AGENTS.md

## Claude Code specifics

- **Regenerate** the unified monitor with the `/regen` command or the
  **regenerate-monitor** skill (or delegate to the `monitor-generator` subagent).
  Each reads `AGENTS.md`, generates into the gitignored workspace, runs the
  acceptance gates, and never commits.
- Default to **plan mode** for any task that would change *tracked* files.
  Generating the monitor does NOT change tracked files — it writes only gitignored
  local paths.
- The integration is **ephemeral**: generate locally, run, and **never `git add` /
  commit it**. Only spec/docs + the two submodules + agent config are publishable.
