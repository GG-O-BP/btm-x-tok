---
description: "Regenerate the btm × tok unified monitor from AGENTS.md (ephemeral; never commit)"
mode: "agent"
---

Read [AGENTS.md](../../AGENTS.md) — the single source of truth — and regenerate the
Design-A in-process compositor exactly as specified.

Generate only into the gitignored local workspace (`monitor/`, `vendor/`, root
`Cargo.toml`, and shim files inside the `bottom/`/`tokscale/` submodules). Never
commit them; never commit to the submodules.

Build order: workspace + vendored `unicode-ellipsis` patch → bottom shim
(`cargo build -p bottom`) → tokscale shims (`cargo build -p tokscale-cli`) → the
monitor crate (`cargo build -p monitor`) → acceptance gates G1–G7.

Consult `.attic/{monitor,bottom,tokscale}` for shape; re-derive copied loop bodies
from current upstream. Match every signature/version exactly; do not upgrade,
port, or refactor. Honor invariants I1–I9.
