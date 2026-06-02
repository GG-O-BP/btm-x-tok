# CLAUDE.md

> Authoritative, agent-agnostic instructions live in **AGENTS.md**, imported below.
> Do not duplicate that content here — edit it in `AGENTS.md`.

@AGENTS.md

## Claude Code specifics

- Default to **plan mode** for any task that would change *tracked* files. The publishable
  surface is tiny and allowlist-controlled (`.gitignore`). Generating the unified monitor does
  **not** change tracked files — it writes only to gitignored local paths.
- The integration is **ephemeral**: generate it in the local, gitignored workspace, run it, and
  **never `git add` / commit it**. Only the spec/docs, the two submodules, and the agent-wiring
  files are publishable.
