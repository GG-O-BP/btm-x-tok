# AGENTS.md — Canonical Entry for All Coding Agents

> **This file is the single source of truth for every coding agent** (Claude Code,
> OpenAI Codex, Cursor, GitHub Copilot, Gemini CLI, Aider, …). `CLAUDE.md`,
> `GEMINI.md`, and any other agent-specific file only point here. Read this file
> first, then follow the pointers into `harness/`.

## Literal interpretation (read this before writing any code)

This is a **build specification, not a suggestion.** Reproduce it exactly.

- **Do not** improve, refactor, modernize, optimize, or upgrade anything.
- **Do not** bump a version, port an API, or "clean up" a copied event loop.
- Where a signature, version, or path is given, it is **NORMATIVE** — match it exactly.
- When the spec and your own judgment disagree, **the spec wins.**
- When the spec is **silent**, consult `.attic/<path>` for SHAPE (module layout,
  signatures, variant coverage) — `.attic = shape, not bytes`. The copied
  event-loop bodies must be **re-derived from CURRENT upstream**
  (`bottom/src/lib.rs` `start_bottom`, `tokscale/.../tui/mod.rs`
  `run_loop_with_background`), because `.attic` may lag upstream.

## What you are building

A single binary, the `monitor` crate, that composes two terminal tools —
**bottom** (system monitor) and **tokscale** (AI token monitor) — into **one
TUI** via **Design A: an in-process compositor**. Each tool keeps its own
renderer; `monitor` owns the real terminal, runs each tool's draw loop against a
custom in-memory capturing backend, and blits the active pane to the screen.
F1 selects the **System** screen (bottom), F2 selects the **Tokens** screen
(tokscale); the System screen carries an agents-% overlay; a `--battery` flag
adds bottom's battery widget. See `README.md` for product rationale; the
integration itself is defined entirely in `harness/`.

## Non-goals

- Do **not** edit bottom's or tokscale's render / widget / state code.
- Do **not** commit any generated path — the integration is ephemeral.
- Do **not** add features beyond this Phase-2 reference (F1/F2 tab switch, the
  agents-% overlay, the `--battery` flag).
- Out of scope / future (do not build): themes, a unified config beyond the
  layout TOML, cross-platform hardening.

## Workspace map + ephemerality rule

| Path | Role |
|---|---|
| `bottom/` | Pristine upstream submodule. You ONLY **add** shim files (e.g. `src/compositor.rs`) plus the minimal edits in `harness/10-target-bottom-shim.md`. Never edit its render/widget/state code; never commit to it. |
| `tokscale/` | Pristine upstream submodule. Same rule — add shims (e.g. `crates/tokscale-cli/src/lib.rs`, `.../tui/compositor.rs`, `.../client_filter.rs`); see `harness/11-target-tokscale-shim.md`. |
| `monitor/` | **The only crate you fully own.** All 10 source files are yours — see `harness/12-target-monitor-crate.md`. |
| `vendor/unicode-ellipsis/` | Vendored dep for the unicode-width reconciliation — see `harness/13-target-workspace-vendor.md`. |
| `Cargo.toml` (root) | Generated workspace manifest — see `harness/13-target-workspace-vendor.md`. |
| `.attic/{monitor,bottom,tokscale}/` | **Read-only** reference (previous generation). Consult for SHAPE only. |
| `scripts/` | `update-upstream.sh` reconciliation runbook — see `harness/30-drift-runbook.md`. |

**Ephemerality rule:** generated paths (`monitor/`, `vendor/`, root `Cargo.toml`,
and every shim file you add to `bottom/`/`tokscale/`) are **gitignored** and must
**never** be committed. If `git status` shows them, that is expected — **never
`git add` them.**

## Hard gates (summary)

Full contract: `harness/01-invariants.md`.

1. bottom/ and tokscale/ stay pristine upstream — only **added** shim files plus the listed edits.
2. Never commit generated paths; never `git add` them.
3. `unicode-width` pins to **`0.2.0`** everywhere (ratatui 0.29 hard-pins `=0.2.0`).
4. The terminal is restored **exactly once** on drop (normal return, `?`, panic) via `TerminalGuard`.
5. `monitor` owns the only real-terminal handle; both panes render into capturing backends.
6. tokscale's `CompositorInput` has **no** `Paste` variant — forward `Event::Paste` to bottom only.
7. F1 → System (bottom), F2 → Tokens (tokscale); switching sets `force_full = true`.
8. `cargo build -p monitor` succeeds; headless (`MONITOR_HEADLESS`, `MONITOR_HEADLESS_TOK`) checks pass.

## Version lock-table — EXACT, do not bump

Full table + reconciliation: `harness/02-versions-and-deps.md`.

| Dependency | Spec |
|---|---|
| `bottom` | `{ path = "../bottom" }` |
| `ratatui-core` | `"0.1.0"` |
| `crossterm` | `"0.29.0"` |
| `ratatui029` | `{ package = "ratatui", version = "0.29" }` |
| `crossterm028` | `{ package = "crossterm", version = "0.28" }` |
| `tokscale-cli` | `{ path = "../tokscale/crates/tokscale-cli" }` |
| `anyhow` | `"1"` |
| `unicode-width` (pinned) | `"0.2.0"` |

Context: bottom `0.12.3` (edition 2024, `tui` = ratatui `0.30.0`, crossterm
`0.29.0`); tokscale `3.0.0` (edition 2021, ratatui `0.29`, crossterm `0.28`).

## ORDERED FILE MANIFEST — the build index

Build in this order. "new" = create the whole file; "edit" = add/modify the
listed lines in an otherwise-pristine upstream file.

| # | Target file | new/edit | Read | `.attic` reference |
|---|---|---|---|---|
| 1 | `vendor/unicode-ellipsis/Cargo.toml` | edit | `harness/13-target-workspace-vendor.md` | `.attic/bottom/Cargo.toml` (pin context) |
| 2 | `Cargo.toml` (root workspace) | new | `harness/13-target-workspace-vendor.md` | — |
| 3 | `bottom/Cargo.toml` (unicode-width `0.2.0`) | edit | `harness/10-target-bottom-shim.md` | `.attic/bottom/Cargo.toml` |
| 4 | `bottom/src/compositor.rs` | new | `harness/10-target-bottom-shim.md` | `.attic/bottom/src/compositor.rs` |
| 5 | `bottom/src/lib.rs` (mod + re-export + `create_collection_thread` pub(crate)) | edit | `harness/10-target-bottom-shim.md` | `.attic/bottom/src/lib.rs` |
| 6 | `tokscale/Cargo.toml` (reqwest `rustls-tls`) | edit | `harness/11-target-tokscale-shim.md` | `.attic/tokscale/Cargo.toml` |
| 7 | `tokscale/crates/tokscale-cli/src/client_filter.rs` | new | `harness/11-target-tokscale-shim.md` | `.attic/tokscale/crates/tokscale-cli/src/client_filter.rs` |
| 8 | `tokscale/crates/tokscale-cli/src/lib.rs` | new | `harness/11-target-tokscale-shim.md` | `.attic/tokscale/crates/tokscale-cli/src/lib.rs` |
| 9 | `tokscale/crates/tokscale-cli/src/main.rs` (ClientFilter move) | edit | `harness/11-target-tokscale-shim.md` | `.attic/tokscale/crates/tokscale-cli/src/main.rs` |
| 10 | `tokscale/crates/tokscale-cli/src/tui/compositor.rs` | new | `harness/11-target-tokscale-shim.md` | `.attic/tokscale/crates/tokscale-cli/src/tui/compositor.rs` |
| 11 | `tokscale/crates/tokscale-cli/src/tui/mod.rs` (`pub mod` + re-export) | edit | `harness/11-target-tokscale-shim.md` | `.attic/tokscale/crates/tokscale-cli/src/tui/mod.rs` |
| 12 | `monitor/Cargo.toml` | new | `harness/12-target-monitor-crate.md` | `.attic/monitor/Cargo.toml` |
| 13 | `monitor/src/backend.rs` (ratatui-core 0.1) | new | `harness/12-target-monitor-crate.md` | `.attic/monitor/src/` |
| 14 | `monitor/src/backend29.rs` (ratatui 0.29) | new | `harness/12-target-monitor-crate.md` | `.attic/monitor/src/` |
| 15 | `monitor/src/blit.rs` | new | `harness/12-target-monitor-crate.md` | `.attic/monitor/src/` |
| 16 | `monitor/src/blit29.rs` | new | `harness/12-target-monitor-crate.md` | `.attic/monitor/src/` |
| 17 | `monitor/src/bridge.rs` (crossterm 0.29 → 0.28) | new | `harness/12-target-monitor-crate.md` | `.attic/monitor/src/` |
| 18 | `monitor/src/layout.rs` | new | `harness/12-target-monitor-crate.md` | `.attic/monitor/src/` |
| 19 | `monitor/src/agents_data.rs` | new | `harness/12-target-monitor-crate.md` | `.attic/monitor/src/` |
| 20 | `monitor/src/agents_overlay.rs` | new | `harness/12-target-monitor-crate.md` | `.attic/monitor/src/` |
| 21 | `monitor/src/main.rs` | new | `harness/12-target-monitor-crate.md` | `.attic/monitor/src/` |

> For each target, consult `.attic/monitor/src/<file>` for shape (and
> `.attic/{bottom,tokscale}` for the embedded engine internals).

## Build & verify gates

Full ordered steps: `harness/20-build-and-acceptance.md`.

- **Build:** `cargo build -p monitor` (the only build command; also run by `scripts/update-upstream.sh`).
- **Headless tokscale pane:** `MONITOR_HEADLESS_TOK=1 cargo run -p monitor` — prints `tokscale grid {w}x{h}, non-empty cells: N` + first up-to-10 rows.
- **Headless bottom pane:** `MONITOR_HEADLESS=1 cargo run -p monitor` — prints `grid {w}x{h}, non-empty cells: N`; with `MONITOR_CUSTOM_LAYOUT=1` also prints `reserved agents cell = {r:?}` and the blit byte/escape summary.
- **Headless sizing:** `MONITOR_W` (default `120`), `MONITOR_H` (default `40`).
- **Battery:** add the `--battery` flag (CPU row splits 2:1 cpu:battery; row-1 geometry unchanged so `reserved_cell_rect` stays valid).
- **Real TTY:** run with a terminal attached; F1 = System, F2 = Tokens; tokscale ticker `tick_ms` is hard-coded `100`.

## Definition of done (closed checklist)

- [ ] All 21 manifest targets created/edited exactly as specified.
- [ ] bottom/ and tokscale/ remain pristine upstream except the listed shim additions/edits.
- [ ] `unicode-width` resolves to `0.2.0` across the tree (vendored `unicode-ellipsis` + bottom pin + `[patch.crates-io]`).
- [ ] `cargo build -p monitor` succeeds.
- [ ] `MONITOR_HEADLESS_TOK=1` and `MONITOR_HEADLESS=1` (incl. `MONITOR_CUSTOM_LAYOUT=1`) checks print their expected non-empty output.
- [ ] Real-TTY run switches System/Tokens via F1/F2 and restores the terminal cleanly on exit and on panic.
- [ ] No generated path is staged or committed (`git status` may list them; never `git add`).

## If upstream changed

If `bottom/` or `tokscale/` has moved (new release pulled, signatures shifted,
event loops changed) → **read `harness/30-drift-runbook.md` FIRST** before
regenerating any shim or copied loop body.
