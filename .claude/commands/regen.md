---
description: Regenerate the btm × tok unified monitor from AGENTS.md (ephemeral; never commit)
argument-hint: "[--battery]"
allowed-tools: Read, Edit, Write, Bash(cargo:*), Bash(rustc:*), Bash(git status:*), Bash(git submodule:*), Bash(mkdir:*), Bash(find:*), Bash(grep:*)
---

Read @AGENTS.md — the single source of truth — and regenerate the Design-A
in-process compositor exactly as specified.

Generate **only into the gitignored local workspace**: `monitor/`, `vendor/`, root
`Cargo.toml`, and the shim files inside the `bottom/`/`tokscale/` submodules. These
are gitignored — **never `git add`/commit them, never commit to the submodules.**

Build order (full contract in AGENTS.md):
1. Root workspace + vendored `unicode-ellipsis` patch.
2. bottom shim (`src/compositor.rs` + `lib.rs`/`Cargo.toml` edits) → `cargo build -p bottom`.
3. tokscale shims (`lib.rs`, `client_filter.rs`, `tui/compositor.rs` + edits) → `cargo build -p tokscale-cli`.
4. monitor crate (10 files) → `cargo build -p monitor`.
5. Run acceptance gates G1–G7. Pass `$ARGUMENTS` through (e.g. `--battery`).

Consult `.attic/{monitor,bottom,tokscale}` for shape; re-derive copied loop bodies
from current upstream. Match every signature/version in AGENTS.md exactly — do not
upgrade, port, or refactor. Honor invariants I1–I9.
