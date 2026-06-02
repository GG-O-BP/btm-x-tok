# 11 — Target: tokscale shim contract

Local, additive shim files you ADD to the `tokscale/` submodule so its TUI can be
driven in-process by the `monitor` compositor. **Never edit tokscale's
render/widget/state/event code, never commit to the submodule.** All paths in
this file are relative to `tokscale/crates/tokscale-cli/`. The submodule is
pinned to upstream latest; you regenerate these shims ephemerally on top of
whatever the pin currently is.

See `harness/00-overview.md` for the thread/data-flow model, `harness/01-invariants.md`
for the hard gates, `harness/02-versions-and-deps.md` for the version lock-table,
`harness/10-target-bottom-shim.md` for the symmetric bottom side, and
`harness/12-target-monitor-crate.md` for the consumers of these symbols.

## Closed whitelist (exactly these touchpoints — nothing else)

| # | File | Action |
|---|------|--------|
| 1 | `src/lib.rs` | **NEW** — library facade for in-process embedding |
| 2 | `src/client_filter.rs` | **NEW** — `ClientFilter` enum extracted from `main.rs` |
| 3 | `src/tui/compositor.rs` | **NEW** — `run_in_compositor` entry point |
| 4 | `src/main.rs` | **EDIT** — replace inline `ClientFilter` with the extracted module |
| 5 | `src/tui/mod.rs` | **EDIT** — +1 `pub mod` + +1 re-export line |
| 6 | `../../Cargo.toml` (tokscale workspace) | **EDIT** — TLS feature reconciliation (only if the build requires it) |

**Whitelist-violation rule:** if a change does not fit one of these six rows,
do NOT make it. Do not port, refactor, rename, reformat, "improve", or upgrade
any other tokscale code. The three NEW files never conflict on an upstream pull;
the three EDITs are the only reconciliation surface (see `harness/30-drift-runbook.md`).

---

## 1. NEW `src/lib.rs` — library facade

Additive facade exposing tokscale's TUI for in-process embedding. The standalone
`tokscale` binary (`main.rs`) keeps its own module tree and does NOT depend on
this lib, so upstream merges of `main.rs` stay clean. The lib declares only the
transitive module closure the TUI needs (`tui` → `commands`, `paths`, `warp`;
plus `ClientFilter` and `auth`/`cursor`) and re-exports the compositor entry
point and the usage helpers the monitor's agents overlay consumes.

**`#![allow(dead_code)]`** at the crate top — the lib drives only the TUI path,
so the many CLI-only helpers in these shared modules are unused here. That is
expected; silence the noise. Do not delete the "unused" code.

**Module declarations (EXACT — match this set and visibility):**

```rust
#![allow(dead_code)]

mod client_filter;
pub(crate) use client_filter::ClientFilter;

mod auth;
mod commands;
mod cursor;
mod paths;
mod warp;
pub mod tui;
```

- `client_filter`, `auth`, `commands`, `cursor`, `paths`, `warp` are private `mod`.
- `tui` is `pub mod`.
- `pub(crate) use client_filter::ClientFilter;` lifts the extracted enum to the
  crate root so `tui::compositor` can name it as `crate::ClientFilter`.

**`pub mod usage` re-export (EXACT names — these are a load-bearing API
contract; the monitor crate's `agents_data.rs` and `agents_overlay.rs` import
exactly these):**

```rust
pub mod usage {
    pub use crate::commands::usage::helpers::render_ascii_bar;
    pub use crate::commands::usage::{UsageMetric, UsageOutput, fetch_all, load_cache};
}
```

Re-export EXACTLY these five names and no more: `render_ascii_bar` (from
`commands::usage::helpers`); `UsageMetric`, `UsageOutput`, `fetch_all`,
`load_cache` (from `commands::usage`). `commands` itself stays private. If
upstream moves any of these symbols, re-point the path here (drift note below) —
do not change the exported names, because the monitor crate consumes
`tokscale_cli::usage::{UsageOutput, fetch_all, load_cache}` (agents_data) and
`tokscale_cli::usage::UsageOutput` (agents_overlay).

> **Drift:** the declared module set must equal the transitive closure the
> current upstream `tui` needs. If a new upstream `tui` submodule pulls in
> another top-level module, add it here as a private `mod`. Reference shape:
> `.attic/tokscale/crates/tokscale-cli/src/lib.rs`.

---

## 2. NEW `src/client_filter.rs` — extracted `ClientFilter`

Upstream `main.rs` defines `pub enum ClientFilter` inline. The shim **moves that
definition verbatim** into `client_filter.rs` so it can live at the crate root of
BOTH the `tokscale` binary and the library facade. Extract from the CURRENT
upstream `main.rs` — the definition must be byte-identical to upstream apart from
being relocated. `.attic` shows the extraction boundary, not necessarily the
current bytes.

Header note for the file: *"Definition is otherwise byte-identical to upstream."*

**Enum (derive + attrs, EXACT):**

```rust
#[derive(ValueEnum, Clone, Copy, Debug, PartialEq, Eq, Hash)]
#[value(rename_all = "lowercase")]
pub enum ClientFilter { /* variants, in upstream order */ }
```

Variant order mirrors `ClientId::ALL` declaration order, with `Synthetic`
appended last (it has no `ClientId` counterpart). Reference order (from
`.attic`, in case upstream is unreadable):

> `Opencode, Claude, Codex, Cursor, Gemini, Amp, Droid, Openclaw, Pi, Kimi,
> Qwen, Roocode, Kilocode, Mux, Kilo, Crush, Hermes, Copilot, Goose, Codebuff,
> Antigravity, Zed, Kiro, #[value(name = "trae")] Trae, Warp, Synthetic`

Keep the `#[value(name = "trae")]` attribute on `Trae`. Needs `use clap::ValueEnum;`.

**Methods — copy all from upstream (do not drop, rename, or re-implement):**

- `pub fn as_filter_str(&self) -> &'static str` — canonical lowercase id per
  variant (e.g. `Self::Synthetic => "synthetic"`).
- `pub fn to_client_id(self) -> Option<tokscale_core::ClientId>` — `Synthetic => None`.
- `pub fn from_client_id(client: tokscale_core::ClientId) -> Self`.
- `pub fn from_filter_str(s: &str) -> Option<Self>`.
- `pub fn default_set() -> std::collections::HashSet<Self>` — every variant
  EXCEPT `Synthetic`.

`to_client_id` and `default_set` are directly required by `tui/compositor.rs`
(initial background load). Reference full bodies:
`.attic/tokscale/crates/tokscale-cli/src/client_filter.rs`.

> **Drift:** if upstream edits the enum (adds a client, changes a `#[value]`
> rename, adds a method), re-apply the **move** against the new upstream
> definition. The enum body always comes FROM current upstream; only its
> location changes.

---

## 3. NEW `src/tui/compositor.rs` — `run_in_compositor`

Embeddable entry point mirroring upstream `run`'s prep + the
`run_loop_with_background` loop, but: draws into a caller-supplied ratatui 0.29
`Backend` (an in-memory capturing backend) instead of the real terminal;
receives input via a public `CompositorInput` channel instead of its own
`EventHandler`/crossterm poller; and does NOT touch raw mode / alt screen /
panic hook / SIGCONT — the compositor owns the real terminal. **Additive and
isolated: do NOT modify `run` / `run_loop_with_background` / any ui/render/state
code.**

### `CompositorInput` — 4 variants, NO `Paste`

```rust
#[derive(Debug)]
pub enum CompositorInput {
    Key(KeyEvent),
    Mouse(MouseEvent),
    /// Pane resized; bottom-style relayout happens via the backend's `size()`,
    /// so this just forces a redraw.
    Resize,
    /// Stop the loop and return.
    Terminate,
}
```

`KeyEvent`/`MouseEvent` are **crossterm 0.28** (tokscale's version):
`use crossterm::event::{KeyEvent, MouseEvent};` (in the tokscale crate,
`crossterm` IS 0.28). The compositor translates from its real crossterm 0.29
events via the monitor's `bridge` module before sending.

> **Paste asymmetry (NORMATIVE):** tokscale's `CompositorInput` has **NO**
> `Paste` variant. Only bottom's does. The monitor forwards `Event::Paste(s)`
> to bottom only — never construct a `Paste` case here. See
> `harness/10-target-bottom-shim.md` and `harness/12-target-monitor-crate.md`.

### Signature (EXACT — match exactly)

```rust
pub fn run_in_compositor<B: Backend>(
    backend: B,
    input_rx: Receiver<CompositorInput>,
    tick_ms: u64,
) -> Result<()>
```

`Backend` = `ratatui::backend::Backend` (ratatui **0.29**); `Result` =
`anyhow::Result`. `Receiver` is `std::sync::mpsc::Receiver`. The monitor calls
this with `tick_ms = 100` (hard-coded) in both real and headless paths — do not
change the call sites; this function just honors whatever it is passed.

### Body — derive from CURRENT upstream `run` + `run_loop_with_background`

Re-derive the prep and loop from the current upstream `tui/mod.rs` / `app.rs`;
keep all `App` / `ui::render` calls **byte-identical** to upstream. Reference
shape (may lag upstream): `.attic/tokscale/crates/tokscale-cli/src/tui/compositor.rs`.

**Prep:**

1. Build `TuiConfig` with these EXACT field values:
   ```rust
   TuiConfig {
       theme: "blue".to_string(),
       refresh: 0,
       sessions_path: None,
       clients: None,
       since: None,
       until: None,
       year: None,
       initial_tab: None,
   }
   ```
2. `let mut app = App::new_with_cached_data(config, None)?;`
3. `let mut terminal = Terminal::new(backend)?;`
4. **Initial background load** (gives live data; mirrors `run`'s
   `needs_background_load`): make a `mpsc::channel::<Result<UsageData>>()`; call
   `app.set_background_loading(true)`; compute `let enabled =
   ClientFilter::default_set();` then `let bg_clients: Vec<ClientId> =
   enabled.iter().filter_map(|f| f.to_client_id()).collect();` and
   `include_synthetic = enabled.contains(&ClientFilter::Synthetic)`; spawn a
   thread that runs `super::background_data_loader(None, None, None, minutely)`
   `.load(&bg_clients, &group_by, include_synthetic)` and sends the result. The
   in-app filter-change reload path is intentionally omitted (Phase 2 scope).

**Threads (merge ticks + injected input into one `Option<Event>` channel; `None`
means terminate):**

- **Ticker thread** — tokscale draws every loop iteration, so a periodic `Tick`
  drives the redraw cadence and background-data pickup. `let tick =
  Duration::from_millis(tick_ms.max(16));` then loop `sleep(tick)` and send
  `Some(Event::Tick)`, breaking on send error.
- **Input thread** — `while let Ok(input) = input_rx.recv()` map:
  - `Key(k)   → Some(Event::Key(k))`
  - `Mouse(m) → Some(Event::Mouse(m))`
  - `Resize   → Some(Event::Tick)` (forces a redraw; relayout is via `size()`)
  - `Terminate → None` (then break the input thread)

  Send the mapped message; break on send error or after a `None`.

**Loop (re-derive from `run_loop_with_background`, keep render calls identical):**

1. `terminal.draw(|f| super::ui::render(f, &mut app))` — keep this `render` call
   byte-identical to upstream; map the draw error into `anyhow`.
2. Drain background results via `bg_rx.try_recv()`: on `Ok(result)` clear
   loading and `update_data` / `set_status` (or `set_error` on `Err`); on
   `Disconnected` clear loading if still set; `Empty` is a no-op.
3. `match ev_rx.recv()`:
   - `Tick → app.on_tick()`
   - `Key(key) → if app.handle_key_event(key) { break }`
   - `Mouse(mouse) → app.handle_mouse_event(mouse)`
   - `Resize(w, h) → app.handle_resize(w, h)`
   - `None | Err(_) → break`
4. `if app.should_quit { break }`.

Return `Ok(())`.

> **Drift:** the loop mirrors current upstream `run_loop_with_background`; the
> background-load wiring mirrors `run`. If upstream changes `App::new_with_cached_data`,
> `background_data_loader`, `TuiConfig` fields, the `ui::render` signature, the
> `Event` variant set, or the `update_data`/`set_status`/`handle_*` API, re-derive
> the body to match. Do not "modernize" beyond matching upstream.

---

## 4. EDIT `src/main.rs` — use the extracted `ClientFilter`

Replace the inline `pub enum ClientFilter` definition with a `mod` declaration +
re-export, so the binary now consumes the extracted file (item 2). Keep
top-of-file module declarations unchanged (`mod antigravity; mod auth; mod
commands; mod cursor; mod device; mod paths; mod trae; mod tui; mod warp;`).

At the point where the inline enum + its `impl` block previously lived, the
replacement is EXACTLY:

```rust
mod client_filter;
pub(crate) use client_filter::ClientFilter;
```

> **Drift / reconciliation:** `ClientFilter` lives in `client_filter.rs` — if an
> upstream pull edits the inline enum in `main.rs`, re-apply the move: copy the
> new enum/impl into `client_filter.rs` and re-insert these two lines. See the
> reconciliation list in `harness/30-drift-runbook.md`.

---

## 5. EDIT `src/tui/mod.rs` — register the compositor module

Exactly two insertions vs pristine upstream:

- Add the module declaration line (among the other `mod`/`pub mod` lines, where
  `compositor` sorts):
  ```rust
  pub mod compositor;
  ```
- Add to the `pub use` block:
  ```rust
  pub use compositor::{CompositorInput, run_in_compositor};
  ```

No other line in `mod.rs` changes.

> **Drift / reconciliation:** keep `pub mod compositor;` + the re-export across
> upstream pulls. Conflicts here are confined to these two added lines.

---

## 6. EDIT tokscale workspace `Cargo.toml` — TLS reconciliation (conditional)

Pristine/upstream value:

```toml
reqwest = { version = "0.12", features = ["json", "native-tls-vendored"], default-features = false }
```

If the build environment lacks the `openssl-src`/perl build dependency that
`native-tls-vendored` pulls in, switch TLS to pure-Rust `rustls-tls`:

```toml
# Reconciliation for the unified monitor build: switched TLS from
# native-tls-vendored to rustls-tls (pure Rust). Avoids the openssl-src/perl
# build dependency (which some build envs lack) and is better for the
# cross-compilation native-tls-vendored was originally chosen for.
reqwest = { version = "0.12", features = ["json", "rustls-tls"], default-features = false }
```

Change ONLY the feature list (`native-tls-vendored` → `rustls-tls`); keep
`version = "0.12"`, `"json"`, and `default-features = false`. This is the only
allowed edit to the tokscale workspace `Cargo.toml`.

> **Drift / reconciliation:** the reqwest feature must stay `rustls-tls` (not
> `native-tls-vendored`) once applied. See `harness/30-drift-runbook.md`.

---

## Determinism recap

- The three NEW files (`lib.rs`, `client_filter.rs`, `tui/compositor.rs`) are
  additive and never conflict on upstream pulls.
- The three EDITs (`main.rs` ClientFilter move, `tui/mod.rs` +2 lines,
  `Cargo.toml` TLS feature) are the entire reconciliation surface.
- These shim files are gitignored and built in the local workspace only. If
  `git status` shows them under `tokscale/`, that is expected — **never `git add`
  them, never commit to the submodule.**
- Match every signature, field value, and re-export name above EXACTLY. Do not
  bump versions, port APIs, or refactor. Where a detail is not pinned here,
  derive it from CURRENT upstream and consult `.attic/tokscale/...` for shape.
