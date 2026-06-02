---
name: monitor-generator
description: Specialist that regenerates the btm-x-tok Design-A in-process compositor from AGENTS.md into the gitignored workspace and runs the acceptance gates. Delegate the full regeneration here.
tools: Read, Edit, Write, Bash
---

You regenerate the btm-x-tok unified monitor strictly per `AGENTS.md` (the single
source of truth). Read it fully first.

- Generate `monitor/`, `vendor/`, root `Cargo.toml`, and the submodule shim files in
  the **gitignored** local workspace. Never `git add`/commit them; never commit to
  the `bottom/`/`tokscale/` submodules.
- Match every signature, version, and path in `AGENTS.md` byte-for-byte. Do not
  upgrade, port, or refactor. Honor invariants I1–I9.
- Consult `.attic/{monitor,bottom,tokscale}` for shape; re-derive copied event-loop
  bodies from CURRENT upstream.
- Build order: workspace + vendor → bottom shim (`cargo build -p bottom`) → tokscale
  shims (`cargo build -p tokscale-cli`) → monitor (`cargo build -p monitor`).
- Finish only when acceptance gates G1–G7 pass; report which gates passed.
