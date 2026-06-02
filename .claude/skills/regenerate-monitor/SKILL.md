---
name: regenerate-monitor
description: Regenerate the btm-x-tok unified monitor (bottom + tokscale composed into one TUI) from AGENTS.md. Use when asked to build, run, regenerate, or verify the in-process compositor integration. Generates into the gitignored workspace and never commits.
allowed-tools: Read, Edit, Write, Bash(cargo:*), Bash(rustc:*), Bash(git status:*), Bash(git submodule:*), Bash(mkdir:*), Bash(find:*), Bash(grep:*)
---

# Regenerate the btm-x-tok monitor

`AGENTS.md` (repo root) is the single source of truth. Read it fully, then
regenerate the Design-A in-process compositor exactly as specified.

## Where
Everything generates in the **gitignored local workspace**: `monitor/`, `vendor/`,
root `Cargo.toml`, and shim files inside the `bottom/`/`tokscale/` submodules. These
are gitignored — never `git add`/commit them, never commit to the submodules.

## Steps (full contract in AGENTS.md)
1. Root workspace + vendored `unicode-ellipsis` patch.
2. bottom shim → `cargo build -p bottom`.
3. tokscale shims → `cargo build -p tokscale-cli`.
4. monitor crate (10 files) → `cargo build -p monitor`.
5. Acceptance gates G1–G7.

## Rules
- Match every signature/version in `AGENTS.md` exactly; do not upgrade, port, or refactor.
- Consult `.attic/{monitor,bottom,tokscale}` for shape; re-derive copied loop bodies from current upstream.
- Honor invariants I1–I9 — especially: never edit upstream render/widget/state code; tokscale `CompositorInput` has no `Paste`; only crossterm 0.29 owns the real terminal.
