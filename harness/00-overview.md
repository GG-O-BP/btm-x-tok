# 00 — Architecture Overview

**Design A — in-process compositor.** One TUI process composes two terminal
tools — **bottom** (system monitor) and **tokscale** (AI token monitor) — into a
single screen. The main thread owns the one real terminal; each engine renders,
unmodified, into its own in-memory buffer using its own ratatui version; the
compositor blits whichever engine is active to the real terminal and overlays an
agents-% panel on the System screen.

This file is the map. The canonical entry is `AGENTS.md`; hard gates live in
`harness/01-invariants.md`; the version lock-table in
`harness/02-versions-and-deps.md`.

---

## Control / data-flow diagram

```
                         ┌────────────────────────────────────────────────┐
                         │                MAIN THREAD                      │
                         │  (monitor/src/main.rs)                          │
                         │                                                 │
   real terminal ◀──────▶│  TerminalGuard: crossterm 0.29 raw mode +       │
   (stdout, one owner)   │  EnterAlternateScreen + Hide  (Drop restores)   │
                         │                                                 │
                         │  router_loop():                                 │
                         │    single input poller — event::poll(8ms)       │
                         │    F1 → Screen::System   F2 → Screen::Tokens     │
                         │    blit ONLY the active engine's buffer         │
                         └───────┬───────────────────────────┬─────────────┘
                                 │ Sender<CompositorInput>    │ Sender<CompositorInput>
                                 │ (bottom, crossterm 0.29)   │ (tokscale, crossterm 0.28
                                 │                            │  via bridge::{key,mouse})
                                 ▼                            ▼
            ┌────────────────────────────┐   ┌────────────────────────────────┐
            │  THREAD S = bottom          │   │  THREAD T = tokscale            │
            │  run_in_compositor_         │   │  tokscale_cli::tui::             │
            │  with_config(...)           │   │  run_in_compositor(.., 100)      │
            │                            │   │                                 │
            │  renders w/ ratatui 0.30    │   │  renders w/ ratatui 0.29         │
            │  into SharedBackend030      │   │  into SharedBackend029           │
            │  (ratatui-core 0.1)         │   │  (ratatui 0.29 backend)          │
            │                            │   │  tokio block_on confined to      │
            │  data-collection thread,    │   │  worker threads (bg loader,      │
            │  cleaning thread spawned    │   │  ticker)                         │
            └──────────────┬─────────────┘   └───────────────┬─────────────────┘
                           │ flush() sets dirty               │ flush() sets dirty
                           ▼                                  ▼
            ┌────────────────────────────┐   ┌────────────────────────────────┐
            │ SharedState                 │   │ SharedState29                   │
            │  grid: Arc<Mutex<Buffer>>   │   │  grid: Arc<Mutex<Buffer>>       │
            │  dirty: Arc<AtomicBool>     │   │  dirty: Arc<AtomicBool>         │
            └──────────────┬─────────────┘   └───────────────┬─────────────────┘
                           └───────────────┬──────────────────┘
                                           ▼
                       MAIN THREAD reads active engine's grid:
                         dirty swaps true (or force_full) →
                         blit::blit / blit29::blit29 (prev-frame cell diff)
                         → stdout.  On System: overlay agents_overlay::
                         draw_agents into layout::reserved_cell_rect(cols,rows)
```

The agents-% data feeding the overlay is produced by a separate
`AgentsData::start()` refresh thread (background, ~60s cadence) that the main
thread snapshots cheaply; it never blocks on the network. See
`harness/12-target-monitor-crate.md`.

---

## Thread model & why it is safe

- **Single input poller.** Only `router_loop` on the main thread calls
  `event::poll(Duration::from_millis(8))` and `event::read()`. Neither engine
  reads the real terminal; each receives translated input over its own
  `std::sync::mpsc::Sender<CompositorInput>`. There is exactly one reader of
  stdin and one writer of stdout (the main thread), so no two threads contend
  for the real terminal.

- **No nested terminal setup.** bottom's `compositor.rs` explicitly installs
  **no** ctrl-c handler, **no** panic hook, and does **no** terminal setup — the
  main thread's `TerminalGuard` owns raw/alt mode and restores it exactly once on
  drop (normal return, `?` errors, panics). tokscale's compositor likewise only
  renders into its in-memory backend.

- **tokio confined to workers.** tokscale's async work (`block_on`,
  `background_data_loader`, the ticker) runs on tokscale's own worker threads
  inside Thread T; it never touches the real terminal or the main thread's input
  loop.

- **dirty AtomicBool handshake.** Each backend's `flush()` sets its
  `dirty: Arc<AtomicBool>`. The router blits the active engine only when that
  engine's `dirty` swaps true (or when `force_full` is requested). The grid
  itself is an `Arc<Mutex<Buffer>>`, so the producing engine and the consuming
  blit never read/write the buffer simultaneously. This is the entire
  cross-thread contract — no shared render state beyond grid + dirty.

---

## Screen model

`enum Screen { System, Tokens }` (`#[derive(Clone, Copy, PartialEq)]`).

- **F1 → System** (bottom), **F2 → Tokens** (tokscale). Switching sets
  `force_full = true` so the next blit repaints the whole frame
  (`Clear(ClearType::All)`).
- **Other keys / mouse** route to the active engine only: System gets
  `CompositorInput::Key(k)` (crossterm 0.29 native); Tokens gets
  `CompositorInput::Key(bridge::key(k))` (translated to crossterm 0.28). Mouse
  is bridged the same way for Tokens.
- **Paste asymmetry.** `Event::Paste(s)` is forwarded **only** to System
  (`CompositorInput::Paste(s)`) — tokscale's `CompositorInput` has no `Paste`
  variant. See `harness/11-target-tokscale-shim.md`.
- **Resize.** `Event::Resize(c,r)` updates `cols/rows`, calls
  `sys_state.set_size`/`tok_state.set_size`, rebuilds the prev buffers, sends
  `Resize` to both engines, and sets `force_full = true`.
- **Agents overlay only on System.** When System is active and a repaint
  happens, `agents_overlay::draw_agents` is overlaid into
  `layout::reserved_cell_rect(cols, rows)` — a cell the System layout reserves
  for exactly this purpose. The Tokens screen has no overlay.
- **`--battery` flag.** `battery_enabled()` scans `std::env::args()` for
  `"--battery"`; the result selects bottom's layout TOML
  (`layout::layout_toml(battery)`), which splits the CPU row 2:1 cpu:battery
  **without** changing row-1 geometry, so `reserved_cell_rect` stays valid.

Out of scope / future: themes, unified config beyond the layout TOML, and
cross-platform hardening are not part of this Phase-2 reference.

---

## Why "render code untouched" holds BY CONSTRUCTION

The two tools target **different ratatui majors** — bottom uses ratatui 0.30
(`tui` package) on `ratatui-core` 0.1; tokscale uses ratatui 0.29. The
integration never reconciles those versions at the rendering layer, because:

- **Each engine renders with its OWN ratatui version into its OWN backend.**
  bottom draws through `SharedBackend030` (implements
  `ratatui_core::backend::Backend`); tokscale draws through `SharedBackend029`
  (implements `ratatui029::backend::Backend`). Each backend captures into its own
  `Arc<Mutex<Buffer>>` of the matching ratatui version.
- **Frames are never shared.** No `Frame`, `Buffer`, widget, or `Terminal`
  crosses the version boundary. The only thing the main thread ever touches is
  the captured `Buffer` of each engine, blitted to bytes by the version-specific
  `blit::blit` (ratatui-core 0.1) / `blit29::blit29` (ratatui 0.29). The bridge
  between the worlds is plain `crossterm` cells and ANSI bytes, not ratatui types.
- **Therefore no porting is ever needed.** bottom's and tokscale's render/widget/
  state code is consumed verbatim; the only added code lives in shim files and
  the `monitor` crate. This is why the invariant in `harness/01-invariants.md`
  ("never edit upstream render/widget/state code") is satisfied structurally, not
  by discipline.

Input crosses the boundary in one direction only and is explicitly translated:
`bridge::{key, mouse}` convert crossterm 0.29 events into the crossterm 0.28
types tokscale's handlers expect. See `harness/12-target-monitor-crate.md` §
`bridge.rs` for the complete variant coverage.

---

## Where to read next (per-target files)

- `AGENTS.md` — canonical agent-agnostic entry point.
- `harness/01-invariants.md` — hard gates / determinism contract.
- `harness/02-versions-and-deps.md` — version lock-table + dependency
  reconciliation (two-ratatui-major build, unicode-width / unicode-ellipsis).
- `harness/10-target-bottom-shim.md` — bottom shim contract
  (`compositor.rs`, `lib.rs` edits, `create_collection_thread` visibility).
- `harness/11-target-tokscale-shim.md` — tokscale shim contract
  (lib facade, `client_filter.rs`, `tui/compositor.rs`, `tui/mod.rs` edits).
- `harness/12-target-monitor-crate.md` — the 10 `monitor/src/` files and their
  signature contracts (backends, blits, bridge, layout, agents, `main.rs`).
- `harness/13-target-workspace-vendor.md` — root `Cargo.toml`, vendored
  `unicode-ellipsis`, scripts.
- `harness/20-build-and-acceptance.md` — ordered build steps + acceptance /
  verify commands (incl. `MONITOR_HEADLESS` / `MONITOR_HEADLESS_TOK` paths).
- `harness/30-drift-runbook.md` — upstream-pull reconciliation runbook.

The previous generation's reference is at `.attic/{bottom,tokscale}/`; consult it
for SHAPE (module layout, signatures), but re-derive copied event-loop bodies
from CURRENT upstream. ".attic = shape, not bytes."
