# 20 — Build & Acceptance

Ordered build steps and the pass/fail acceptance gates for **Design A — in-process compositor**. Every command, path, and env var below is NORMATIVE — match it exactly. Do not bump versions, port, refactor, or "improve" anything. See `harness/01-invariants.md` for the determinism contract, `harness/02-versions-and-deps.md` for the version lock-table, and `AGENTS.md` for the canonical entry.

The generated tree (`monitor/`, `vendor/`, root `Cargo.toml`, and the shim files inside `bottom/`/`tokscale/`) is gitignored and ephemeral. If `git status` shows it, that is expected — **never `git add` it.**

---

## Build steps (ordered)

Run from the workspace root `/home/superset/btm-toks/`. Build targets in this order; an earlier failure blocks the next step.

### (0) Toolchain check — MSRV ≥ 1.86

bottom is edition `2024`; the workspace `resolver = "3"`. Confirm a recent stable toolchain before anything else:

```sh
rustc --version   # require >= 1.86
cargo --version
```

If `rustc` is older than 1.86, stop and upgrade the toolchain. Do not edit `edition` or `resolver` to work around an old compiler.

### (1) Root workspace + vendored `unicode-ellipsis` patch

Create the two reconciliation pieces first so every downstream `cargo` invocation resolves a single `unicode-width`:

- Root `Cargo.toml` — verbatim block in `harness/13-target-workspace-vendor.md`: `[workspace] resolver = "3"`, `members = ["monitor"]`, `exclude = ["bottom", "tokscale", "vendor/unicode-ellipsis"]`, and `[patch.crates-io] unicode-ellipsis = { path = "vendor/unicode-ellipsis" }`.
- `vendor/unicode-ellipsis/` — the path-vendored copy of `unicode-ellipsis 0.4.0` whose only change is the `unicode-width` floor: `[dependencies.unicode-width] version = "0.2.0", default-features = false`. See `harness/13-target-workspace-vendor.md`.

### (2) bottom shim → standalone build must pass

Apply ONLY the whitelisted bottom edits (full contract in `harness/10-target-bottom-shim.md`):

- NEW `bottom/src/compositor.rs`.
- `bottom/src/lib.rs`: add `pub(crate) mod compositor;`, the `pub use compositor::{CompositorInput, run_in_compositor, run_in_compositor_with_config};` re-export, and change `fn create_collection_thread(` → `pub(crate) fn create_collection_thread(`.
- `bottom/Cargo.toml`: `unicode-width = "0.2.2"` → `unicode-width = "0.2.0"`.

Then verify bottom still compiles on its own:

```sh
cargo build -p bottom
```

This MUST pass before continuing. The render/widget/state code is untouched.

### (3) tokscale shims → standalone build must pass

Apply ONLY the whitelisted tokscale edits (full contract in `harness/11-target-tokscale-shim.md`):

- NEW `tokscale/crates/tokscale-cli/src/lib.rs` (library facade).
- NEW `tokscale/crates/tokscale-cli/src/client_filter.rs`.
- NEW `tokscale/crates/tokscale-cli/src/tui/compositor.rs`.
- `src/main.rs`: replace the inline `ClientFilter` enum with `mod client_filter; pub(crate) use client_filter::ClientFilter;`.
- `src/tui/mod.rs`: add `pub mod compositor;` and `pub use compositor::{CompositorInput, run_in_compositor};`.
- `tokscale/Cargo.toml`: `reqwest` features `["json", "native-tls-vendored"]` → `["json", "rustls-tls"]`.

Then verify tokscale-cli still compiles on its own:

```sh
cargo build -p tokscale-cli
```

This MUST pass before continuing.

### (4) monitor backends + blit + bridge

Add the low-level monitor glue (signatures in `harness/12-target-monitor-crate.md`):

- `monitor/src/backend.rs` (`SharedState`, `SharedBackend030` over `ratatui-core` 0.1).
- `monitor/src/backend29.rs` (`SharedState29`, `SharedBackend029` over `ratatui029` 0.29).
- `monitor/src/blit.rs` / `monitor/src/blit29.rs` (buffer → crossterm diff writers).
- `monitor/src/bridge.rs` (crossterm 0.29 → 0.28 `key`/`mouse` translation).

### (5) layout + agents + main

Add the composition layer (signatures in `harness/12-target-monitor-crate.md`):

- `monitor/src/layout.rs` (`layout_toml`, `reserved_cell_rect`).
- `monitor/src/agents_data.rs` (`AgentsData`).
- `monitor/src/agents_overlay.rs` (`draw_agents`).
- `monitor/src/main.rs` (`Screen`, `TerminalGuard`, `router_loop`, headless paths, `main`).

### (6) full monitor build

```sh
cargo build -p monitor
```

This is the same command `scripts/update-upstream.sh` runs to verify a merge ("Building monitor to verify the merge…"). It MUST pass.

---

## Acceptance gates

Each gate is **pass/fail-blocking**: a failing gate blocks Definition of Done. Use the exact env vars and commands below. Headless gates need no TTY.

### G1 — Headless bottom pane

```sh
MONITOR_HEADLESS=1 cargo run -p monitor
```

`headless_check()` drives bottom into `SharedBackend030` (~1500 ms), then prints `grid {w}x{h}, non-empty cells: N` and exercises the real `blit::blit` into an in-memory `Vec<u8>`.

- **PASS**: `non-empty cells: N` reports `N > 0`, AND the blit line `blit emitted {N} bytes with {esc} escape sequences (full frame painted)` reports a nonzero escape-sequence count.
- **FAIL**: an empty grid or zero bytes/escape sequences.

### G2 — Headless tokscale pane

```sh
MONITOR_HEADLESS_TOK=1 cargo run -p monitor
```

`headless_check_tok()` drives `tokscale_cli::tui::run_in_compositor(backend, rx, 100)` into `SharedBackend029` (~2500 ms), prints `tokscale grid {w}x{h}, non-empty cells: N` plus the first up-to-10 rows, then `Terminate`s and joins. `MONITOR_HEADLESS_TOK` is checked FIRST, before `MONITOR_HEADLESS`.

- **PASS**: a live grid — `non-empty cells: N` with `N > 0` and rendered rows present.
- **FAIL**: empty grid or a hang past the join.

### G3 — Custom-layout overlay (reserved agents cell)

Run at both reference grid sizes:

```sh
MONITOR_HEADLESS=1 MONITOR_CUSTOM_LAYOUT=1 MONITOR_W=120 MONITOR_H=40 cargo run -p monitor
MONITOR_HEADLESS=1 MONITOR_CUSTOM_LAYOUT=1 MONITOR_W=80  MONITOR_H=24 cargo run -p monitor
```

With `MONITOR_CUSTOM_LAYOUT` set, `headless_check` uses `run_in_compositor_with_config` with the layout TOML, dumps `g.area.height` rows, prints `reserved agents cell = {r:?}`, overlays fake `UsageOutput` data into the reserved cell, and dumps that region.

- **PASS**: `reserved agents cell = Some(Rect { … })` at 120x40 with the overlaid agent rows visible in the dumped region; at 80x24 the reserved cell is still present (or `None` only when the cell genuinely falls below `width < 2 || height < 2`, per `reserved_cell_rect`).
- **FAIL**: overlay paints outside the reserved cell, or the reserved cell collides with bottom's row-1 widgets.

Optional: append `--battery` to confirm the battery variant keeps row-1 geometry valid (see G7).

### G4 — Layout unit test

```sh
cargo test -p monitor
```

- **PASS**: `layout.rs` tests `reserved_cell_matches_reference_and_bounds` and `tiny_terminal_yields_none` pass (grid sizes `(80,24),(120,40),(200,50),(100,30)`).
- **FAIL**: any test failure.

### G5 — Dependency-tree assertions

Confirm the two-major build resolved as intended:

```sh
cargo tree -p monitor -i ratatui
cargo tree -p monitor -i unicode-width
cargo tree -p monitor -i openssl-sys
```

- **PASS**, all of:
  - **ratatui present at BOTH 0.29 and 0.30** (0.30 via bottom's `tui`, 0.29 via `ratatui029`).
  - **crossterm present at BOTH 0.28 and 0.29** (0.29 = real-terminal owner / bottom; 0.28 = `crossterm028` for tokscale's typed events).
  - **`unicode-width` resolves to 0.2.0 ONLY** — no second version in the tree (the `=0.2.0` pin from ratatui 0.29 unifies with the lowered bottom floor and the patched `unicode-ellipsis`).
  - **NO `openssl-sys` / `openssl-src`** — tokscale's `reqwest` is on `rustls-tls`, so the openssl/perl build dependency is absent.
- **FAIL**: a duplicate `unicode-width`, a missing ratatui/crossterm major, or any openssl crate present.

### G6 — Standalone builds unbroken + render-untouched check

Re-confirm the submodules build alone and that ONLY whitelisted shim files changed:

```sh
cargo build -p bottom
cargo build -p tokscale-cli
git -C bottom status --porcelain
git -C tokscale status --porcelain
```

- **PASS**:
  - Both standalone builds succeed.
  - `git -C bottom status` shows ONLY: `src/compositor.rs`, `src/lib.rs`, `Cargo.toml`.
  - `git -C tokscale status` shows ONLY: `crates/tokscale-cli/src/lib.rs`, `crates/tokscale-cli/src/client_filter.rs`, `crates/tokscale-cli/src/tui/compositor.rs`, `crates/tokscale-cli/src/main.rs`, `crates/tokscale-cli/src/tui/mod.rs`, `Cargo.toml`.
- **FAIL**: any change to render/widget/state code, or any file outside the whitelists above. The whitelists match the reconciliation points in `scripts/update-upstream.sh` — see `harness/30-drift-runbook.md`.

### G7 — TTY run (interactive)

```sh
cargo run -p monitor
```

- **PASS**, all of:
  - The TUI opens in the alternate screen (`TerminalGuard::enter` ran `enable_raw_mode` + `EnterAlternateScreen, Hide`).
  - **F1 switches to the System screen (bottom); F2 switches to the Tokens screen (tokscale).** A switch forces a full repaint.
  - **`q` quits** (routed to the active engine), and on exit the terminal is fully restored — `TerminalGuard::drop` runs `LeaveAlternateScreen, Show` + `disable_raw_mode` exactly once (normal return, `?` errors, and panics).
- Battery variant:

```sh
cargo run -p monitor -- --battery
```

  - **PASS**: the System screen's CPU row gains bottom's **battery widget** (CPU row splits 2:1 cpu:battery) WITHOUT changing row-1 geometry, so the agents overlay still lands in the reserved cell.
- **FAIL**: tabs don't switch, `q` doesn't quit, the terminal is left in raw/alt-screen state, or `--battery` shifts row-1 and breaks the overlay.

> The agents-% overlay on the System screen and the `--battery` flag are the full Phase-2 scope. Themes, unified config beyond the layout TOML, and cross-platform hardening are out of scope / future.

---

## Definition of Done

All of the following hold:

- Build steps (0)–(6) succeed, including the two standalone builds (`cargo build -p bottom`, `cargo build -p tokscale-cli`).
- Acceptance gates **G1–G7 are all green.**
- The render-untouched check (G6) shows ONLY whitelisted shim files dirty in `bottom/` and `tokscale/`.
- **Nothing is committed.** The generated `monitor/`, `vendor/`, root `Cargo.toml`, and the shim files remain gitignored and uncommitted — if `git status` shows them, that is expected; never `git add` them.
