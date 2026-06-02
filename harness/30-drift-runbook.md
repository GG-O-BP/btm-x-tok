# 30 — Upstream Drift Runbook

Reconciliation procedure for regenerating Design A after `bottom/` or `tokscale/`
submodules move to a newer upstream commit. The integration is **ephemeral**:
`monitor/`, `vendor/`, the root `Cargo.toml`, and every shim file are gitignored
and never committed. Re-deriving from new upstream means editing the shim files
in the local workspace again, not patching `.attic`.

The canonical entry is `AGENTS.md`. See `harness/02-versions-and-deps.md` for the
version lock-table, `harness/10-target-bottom-shim.md` and
`harness/11-target-tokscale-shim.md` for the shim contracts, `harness/12-target-monitor-crate.md`
for the monitor crate, and `harness/20-build-and-acceptance.md` for the acceptance gates.

---

## Golden rule: re-derive, never copy `.attic`

> **`.attic = shape, not bytes.`**

`.attic/{bottom,tokscale}/` is the **previous** generation's shimmed reference. It
shows the SHAPE of each shim — module layout, signatures, variant coverage — but
its copied event-loop bodies may LAG the upstream the submodules now point at.

- Use `.attic` to learn **what** to add (which functions, which enum variants,
  which re-exports) and **where** (file paths, line neighborhoods).
- **Always re-derive the copied loop bodies from CURRENT upstream**, specifically:
  - bottom's `start_bottom` event loop (`bottom/src/lib.rs`, ref region `410-481`)
    that `run_in_compositor_with_config` mirrors.
  - tokscale's `run_loop_with_background` in `tokscale/crates/tokscale-cli/src/tui/mod.rs`
    that `tui/compositor.rs` mirrors.
- Never `git add` any regenerated path. If `git status` lists `monitor/`,
  `vendor/`, root `Cargo.toml`, or shim files, that is expected.
- Do not upgrade, port, refactor, or "improve" anything while reconciling. Match
  the signatures and versions exactly.

---

## Procedure

1. **Move upstream.** Either `git submodule update --remote bottom tokscale`, or
   run the subtree flow in `scripts/update-upstream.sh` (wraps
   `git subtree pull --squash` against remotes `bottom-upstream`
   = `https://github.com/ClementTsang/bottom.git` and `tokscale-upstream`
   = `https://github.com/junhoyeo/tokscale.git`). Conflicts are confined to the
   shimmed files listed below.

2. **Re-apply the shims onto NEW upstream** (see the reconciliation table). Open
   each shimmed file at its current upstream state and re-derive the edit; cross-check
   only the SHAPE against `.attic`.

3. **Build.** `cargo build -p monitor` (the only build command; also emitted by
   `scripts/update-upstream.sh` with "Building monitor to verify the merge..." →
   "OK: upstream pulled and monitor still builds. Review 'git log' and push.").

4. **Run all acceptance gates** from `harness/20-build-and-acceptance.md`
   (headless gates `MONITOR_HEADLESS_TOK=1`, `MONITOR_HEADLESS=1`,
   `MONITOR_CUSTOM_LAYOUT=1`, the `MONITOR_W`/`MONITOR_H` sizing, and the
   `--battery` flag). Do not consider the regeneration usable until they pass.

---

## Reconciliation points (where conflicts land)

The shim touches exactly these files. Verbatim list from
`scripts/update-upstream.sh` (lines 44–50), mapped to detection + fix:

| ID | File | Invariant to re-apply |
|----|------|------------------------|
| (a) | `bottom/Cargo.toml` | unicode-width must stay `"0.2.0"` (ratatui 0.29 pins `=0.2.0`) |
| (b) | `bottom/src/lib.rs` | keep `pub(crate) mod compositor;` + re-exports + `create_collection_thread` pub(crate) |
| (c) | `tokscale/Cargo.toml` | reqwest feature must stay `rustls-tls` (not `native-tls-vendored`) |
| (d) | `tokscale/.../main.rs` | ClientFilter lives in `client_filter.rs` (re-apply the move if upstream edits the enum) |
| (e) | `tokscale/.../tui/mod.rs` | keep `pub mod compositor;` + re-exports |

New files `compositor.rs` (bottom + tokscale), `client_filter.rs`, and the
tokscale `lib.rs` facade **never conflict** — but their COPIED bodies can drift.
The table below covers both the conflict points and the silently-drifting copies.

---

## Drift table: detection → fix

### (a) New `BottomEvent` arm in bottom

- **Detection:** `cargo build -p monitor` errors in `bottom/src/compositor.rs` on a
  non-exhaustive `match` against `BottomEvent`, or the System pane stops reacting to
  an input class. The input-translator thread maps `CompositorInput → BottomEvent`
  (`Key→KeyInput`, `Mouse→MouseInput`, `Paste→PasteEvent`, `Resize→Resize`,
  `Terminate→Terminate`).
- **Fix:** Re-derive the event loop body in `bottom/src/compositor.rs` from the
  CURRENT `start_bottom` (`bottom/src/lib.rs`, ref `410-481`). Add handling for the
  new `BottomEvent` arm to the copied loop and, if it corresponds to a new
  `CompositorInput` source, extend the translator. Keep the explicit absence of any
  ctrl-c handler, panic hook, or terminal setup. The cleaning thread keeps
  `offset_wait = retention_ms + 60000`. Do not copy this body from `.attic`.

### (b) tokscale `run_loop_with_background` / `Event` variant change

- **Detection:** `cargo build -p monitor` errors in
  `tokscale/crates/tokscale-cli/src/tui/compositor.rs` on `Event`, `App`,
  `TuiConfig`, `App::new_with_cached_data`, `ui::render`, or the background loader;
  or the Tokens pane stops updating. The input thread maps `Key→Event::Key`,
  `Mouse→Event::Mouse`, `Resize→Event::Tick`, `Terminate→None`; a ticker thread
  emits `Event::Tick` at `Duration::from_millis(tick_ms.max(16))`.
- **Fix:** Re-sync `tui/compositor.rs` against the CURRENT
  `run_loop_with_background` in `tokscale/crates/tokscale-cli/src/tui/mod.rs`. Rebuild
  the `TuiConfig { theme: "blue", refresh: 0, sessions_path: None, clients: None,
  since: None, until: None, year: None, initial_tab: None }`, the
  `App::new_with_cached_data(config, None)` call, the initial background load
  (`ClientFilter::default_set()` → `to_client_id`), and the draw/drain/dispatch loop
  to match new `Event` variants. The signature is NORMATIVE — do not change it:
  `pub fn run_in_compositor<B: Backend>(backend: B, input_rx: Receiver<CompositorInput>, tick_ms: u64) -> Result<()>`
  (`Backend` = ratatui 0.29; `tick_ms` is hard-coded `100` by callers). Re-derive from
  upstream, not `.attic`.

### (c) Key/mouse handling change (crossterm version or variant)

- **Detection:** `cargo build -p monitor` errors in `monitor/src/bridge.rs` on a
  non-exhaustive `match` over `KeyCode`, `KeyEventKind`, `MediaKeyCode`,
  `ModifierKeyCode`, `MouseEventKind`, or `MouseButton`; or Tokens-pane keys/mouse
  misbehave. `bridge.rs` translates crossterm **0.29** (`ct29 = crossterm::event`) →
  **0.28** (`ct28 = crossterm028::event`) via `pub fn key`/`pub fn mouse`.
- **Fix:** Extend `monitor/src/bridge.rs`. Add the new variant to the matching helper
  (`code`, `kind`, `media`, `modifier_key`, `mouse_kind`, `mouse_button`,
  `modifiers`, `state`). `modifiers`/`state` are rebuilt via
  `from_bits_truncate(.bits())`. If a crossterm MAJOR moves on either side, that is a
  version change — reconcile against `harness/02-versions-and-deps.md`
  (bottom/System = crossterm `0.29`; tokscale handlers expect `0.28`). Do not bump
  either pin to make the build pass; the two-major split is intentional. For the full
  variant coverage, consult the reference at
  `.attic/bottom`/`.attic/tokscale` and `harness/12-target-monitor-crate.md`.

### (d) `ClientFilter` change in tokscale `main.rs`

- **Detection:** Upstream `git diff` touches the inline `pub enum ClientFilter` in
  `tokscale/crates/tokscale-cli/src/main.rs` (pristine inline definition near line
  820), adding/removing/renaming a provider variant; or `cargo build -p monitor`
  errors on `ClientFilter` in `client_filter.rs` / the `usage` overlay path.
- **Fix:** Re-apply the extraction. The enum lives in `client_filter.rs` (header note:
  *"Definition is otherwise byte-identical to upstream."*). In `main.rs`, the inline
  definition is replaced by:
  ```rust
  mod client_filter;
  pub(crate) use client_filter::ClientFilter;
  ```
  Re-copy the CURRENT upstream enum body into `client_filter.rs` (keep the
  `#[derive(ValueEnum, Clone, Copy, Debug, PartialEq, Eq, Hash)]`,
  `#[value(rename_all = "lowercase")]`, the `#[value(name = "trae")] Trae` special
  case, and methods `as_filter_str`, `to_client_id`, `from_client_id`,
  `from_filter_str`, `default_set`). If a variant changed, that flows through
  `default_set()` (all variants except `Synthetic`) into the overlay's initial load
  — verify the Tokens pane and the System overlay still populate. See
  `harness/11-target-tokscale-shim.md`; for the exact variant order and method
  bodies, consult the reference at
  `.attic/tokscale/crates/tokscale-cli/src/client_filter.rs`.

### (e) Dependency drift (unicode-width / ratatui / TLS)

- **Detection:** `cargo build -p monitor` fails with a unicode-width version conflict,
  a ratatui-core/ratatui trait mismatch on either `Backend` impl, or a TLS/openssl
  build failure (openssl-src/perl). Inspect with `cargo tree`
  (e.g. `cargo tree -p monitor -i unicode-width`).
- **Fix — unicode-width:** Keep it pinned `"0.2.0"` across the tree. ratatui 0.29
  (tokscale's TUI) hard-pins `unicode-width = "=0.2.0"`. Two anchors must hold:
  1. `bottom/Cargo.toml` direct dep stays `unicode-width = "0.2.0"` (pristine upstream
     ships `"0.2.2"` — re-lower it; API is identical across `0.2.0..0.2.2`).
  2. `vendor/unicode-ellipsis/Cargo.toml` keeps
     `[dependencies.unicode-width] version = "0.2.0", default-features = false`, applied
     via root `[patch.crates-io] unicode-ellipsis = { path = "vendor/unicode-ellipsis" }`.
     Re-vendor if upstream bottom moves `unicode-ellipsis` (its crates.io `0.4.0`
     floors `>=0.2.2`). See `harness/13-target-workspace-vendor.md` and
     `harness/02-versions-and-deps.md`.
- **Fix — ratatui / ratatui-core:** Do NOT bump. `monitor/Cargo.toml` pins
  `ratatui-core = "0.1.0"` (must unify with bottom 0.30) and
  `ratatui029 = { package = "ratatui", version = "0.29" }` (tokscale). If a trait on
  either `Backend` impl breaks, re-derive the impl in `backend.rs` / `backend29.rs`
  against the version actually resolved, but keep the pins fixed. If upstream bottom
  moves off ratatui 0.30 / ratatui-core 0.1, treat it as a version-lock change and
  reconcile via `harness/02-versions-and-deps.md` before touching the monitor crate.
- **Fix — TLS:** `tokscale/Cargo.toml` `reqwest` stays on `rustls-tls` ONLY if the
  build env lacks openssl-src/perl (pristine upstream ships
  `features = ["json", "native-tls-vendored"]`). Switch to `rustls-tls` (pure Rust)
  when native-tls/openssl fails to build:
  `reqwest = { version = "0.12", features = ["json", "rustls-tls"], default-features = false }`.
  If native-tls builds cleanly in your env, the pristine value may be left as-is — the
  shim exists only to avoid the openssl-src/perl build dependency.

---

## Copied bodies that can silently drift (no compile error)

These track upstream by VALUE, not type, so a clean build does not prove they are
current. Re-derive from upstream on every pull:

- bottom's `create_collection_thread` body (reused by `compositor.rs`; its visibility
  must stay `pub(crate)`), the `BottomEvent` mapping, and the `start_bottom` loop
  (ref `bottom/src/lib.rs:410-481`), plus `init_app` / `get_or_create_config` /
  `Painter::init` / `BottomArgs` usage.
- tokscale's `App` / `Event` / `TuiConfig` / `UsageData` / `background_data_loader` /
  `ui::render` / `App::new_with_cached_data` usage in `tui/compositor.rs`, and the
  `commands::usage::{UsageMetric, UsageOutput, fetch_all, load_cache, helpers::render_ascii_bar}`
  re-exports in the `lib.rs` facade.
- `monitor/src/agents_overlay.rs` hard-codes bottom's DEFAULT theme palette
  (`TINTS`) and tokscale's usage thresholds (`<10%` red, `<25%` yellow, else gray).
  If upstream tokscale changes its `usage.rs` thresholds, re-sync `pct_color`.
- `monitor/src/layout.rs` mirrors bottom's two-pass `Layout` split for
  `reserved_cell_rect`; if upstream bottom changes its layout engine, re-verify the
  `#[cfg(test)]` tests (`reserved_cell_matches_reference_and_bounds`,
  `tiny_terminal_yields_none`).

> Out of scope / future: this runbook covers only the Phase-2 reference (F1/F2 tab
> switch, agents-% overlay, `--battery`). Themes, unified config beyond the layout
> TOML, and cross-platform hardening are not reconciled here.
