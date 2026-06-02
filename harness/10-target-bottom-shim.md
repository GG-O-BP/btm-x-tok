# harness/10-target-bottom-shim.md — bottom shim contract

The `bottom/` submodule is **pinned to upstream latest** and consumed by `monitor/` as a path dependency (`bottom = { path = "../bottom" }`; see harness/02-versions-and-deps.md). You add a thin **in-process compositor** so bottom renders into our custom backend driven by a channel instead of owning the real terminal. You never edit bottom's render/widget/state/data code, and you never commit to the submodule — generated/edited paths there are gitignored; if `git status` shows them, that is expected — **never `git add` them**.

`.attic = shape, not bytes.` Consult `.attic/bottom/src/compositor.rs` for module SHAPE (item layout, signatures, helper names, thread structure). The event-loop body MUST be re-derived from **current** upstream `bottom/src/lib.rs` (`start_bottom`), because `.attic` may lag upstream.

---

## Closed whitelist — the ONLY files you may touch in `bottom/`

| Action | Path | What |
|---|---|---|
| **NEW** | `bottom/src/compositor.rs` | The whole compositor module (entire file is new). |
| **EDIT** | `bottom/src/lib.rs` | Add the two `mod`/`pub use` lines and widen `create_collection_thread` visibility (exact lines below). |
| **EDIT** | `bottom/Cargo.toml` | Pin `unicode-width = "0.2.0"` (see harness/02-versions-and-deps.md). |

Anything outside this whitelist is a spec violation. Do not upgrade, port, refactor, rename, or "improve" any other bottom file.

---

## EDIT — `bottom/src/lib.rs`

Three insertions/changes vs pristine upstream. Match exactly.

**Module declaration** — inserted among the existing `pub(crate) mod ...` block (`.attic` line 25):

```rust
pub(crate) mod compositor;
```

**Re-export** — with a trailing blank line after it (`.attic` line 31):

```rust
pub use compositor::{CompositorInput, run_in_compositor, run_in_compositor_with_config};
```

**Visibility widening** — `create_collection_thread` must be reachable from `compositor.rs` (same crate). Change the pristine declaration:

```rust
fn create_collection_thread(
```

to:

```rust
pub(crate) fn create_collection_thread(
```

(Pristine/live upstream declares it `fn create_collection_thread(`. Line numbers shift across upstream pulls — locate by name, not by line.)

---

## EDIT — `bottom/Cargo.toml`

Set the direct `unicode-width` dependency to **`"0.2.0"`** (EXACT — do not bump). This is the unified-build reconciliation point: ratatui 0.29 (tokscale's TUI) hard-pins `unicode-width = "=0.2.0"`, and the API is identical across 0.2.0..0.2.2. The full reconciliation (this pin + the vendored `unicode-ellipsis` patch) is specified in harness/02-versions-and-deps.md and harness/13-target-workspace-vendor.md. The pristine/live value is `unicode-width = "0.2.2"`. Keep the 3-line reconciliation comment from harness/02-versions-and-deps.md above the pin.

---

## NEW — `bottom/src/compositor.rs`

### Public API (match signatures exactly)

```rust
pub fn run_in_compositor<B: Backend>(
    backend: B,
    input_rx: Receiver<CompositorInput>,
) -> anyhow::Result<()>
```

```rust
pub fn run_in_compositor_with_config<B: Backend>(
    backend: B,
    input_rx: Receiver<CompositorInput>,
    config_path: Option<std::path::PathBuf>,
) -> anyhow::Result<()>
```

- `run_in_compositor` delegates: `run_in_compositor_with_config(backend, input_rx, None)`.
- `Backend` = `tui::backend::Backend`, where `tui` is bottom's `ratatui` **0.30**.
- `Receiver` = `std::sync::mpsc::Receiver`.

### The input enum (5 variants — note `Paste`)

```rust
pub enum CompositorInput {
    Key(KeyEvent),
    Mouse(MouseEvent),
    Paste(String),
    /// The pane was resized; bottom should relayout to the backend's new `size()`.
    Resize,
    /// Stop the loop and return.
    Terminate,
}
```

- `#[derive(Debug)]`.
- `KeyEvent` / `MouseEvent` are **crossterm 0.29** (`crossterm::event::{KeyEvent, MouseEvent}`).
- bottom's `CompositorInput` **has** the `Paste(String)` variant (tokscale's does not — `monitor/src/main.rs` forwards `Event::Paste` only to the System screen; see harness/11-target-tokscale-shim.md and harness/12-target-monitor-crate.md).

### Private helper

```rust
fn try_drawing<B: Backend>(
    terminal: &mut Terminal<B>,
    app: &mut App,
    painter: &mut Painter,
) -> anyhow::Result<()>
```

### What `run_in_compositor_with_config` does (setup)

Mirror `start_bottom` minus terminal setup. In order:

1. Parse args headlessly: `BottomArgs::parse_from(["btm"])`.
2. `get_or_create_config(config_path.as_deref())`.
3. `init_app(...)`, then `Painter::init(...)`.
4. Spawn the **data-collection thread**: `create_collection_thread(...)` (now reachable thanks to the `pub(crate)` widening).
5. Spawn the **cleaning thread** with `offset_wait = retention_ms + 60000`.
6. Spawn an **input-translator thread** that maps `CompositorInput` → `BottomEvent`:
   - `Key` → `KeyInput`
   - `Mouse` → `MouseInput`
   - `Paste` → `PasteEvent`
   - `Resize` → `Resize`
   - `Terminate` → `Terminate`
7. Enter the main loop (below).

For exact construction details (argument values, channel wiring, painter init args, the `try_drawing` body), consult the reference at `.attic/bottom/src/compositor.rs` for SHAPE, and re-derive the live values from current upstream `bottom/src/lib.rs`.

### KEY INSTRUCTION — the event loop

**Copy the event-loop body from the CURRENT upstream `bottom/src/lib.rs` `start_bottom`** (the `.attic` copy mirrors `lib.rs:410-481` — that range is a hint, not normative; line numbers drift). When copying:

- **Remove all terminal setup/teardown** — `compositor.rs` does NOT own the terminal: no `enable_raw_mode`, no alternate-screen enter/leave, no cursor hide/show. The real terminal is owned solely by `monitor/src/main.rs`'s `TerminalGuard` (see harness/12-target-monitor-crate.md).
- **Deliberately OMIT**: any ctrl-c handler, any panic hook, and any terminal/event-stream initialization. Those belong to the monitor binary only.
- **Preserve EVERY `BottomEvent` match arm present in current upstream.** Do not drop, merge, or reorder arms. Drawing routes through `try_drawing(&mut terminal, &mut app, &mut painter)` into the supplied `Backend` instead of the real stdout terminal.
- Events arrive on bottom's normal `BottomEvent` receiver, fed by the input-translator thread above plus the collection/cleaning threads — identical to how `start_bottom` consumes them upstream.

### Things this module must NOT do

- No terminal setup or teardown (raw mode, alternate screen, cursor visibility).
- No panic hook.
- No ctrl-c / signal handler.
- No input device reading — input arrives only via `input_rx`. (The translator thread maps it; it does not poll a real device.)

---

## Per-target drift note

On every upstream pull (`scripts/update-upstream.sh`; see harness/30-drift-runbook.md), re-reconcile bottom against these copied surfaces:

- **`bottom/Cargo.toml`** — `unicode-width` must stay `"0.2.0"` (ratatui 0.29 pins `=0.2.0`).
- **`bottom/src/lib.rs`** — keep `pub(crate) mod compositor;` + the `pub use` re-export + `create_collection_thread` as `pub(crate)`.
- **`compositor.rs` is a COPY** of upstream surfaces and can drift:
  - **If upstream adds a new `BottomEvent` variant**, add a matching arm to the copied event loop in `compositor.rs`. A non-exhaustive match is a regression.
  - If upstream changes `create_collection_thread`'s signature/body, the cleaning-thread `offset_wait`, or the `init_app` / `get_or_create_config` / `Painter::init` / `BottomArgs` APIs, re-derive the copied bodies from current upstream.
- New files (`compositor.rs`) never conflict on subtree pulls; only the two edited files do.

---

**Anything outside this whitelist is a spec violation.** Touch only `bottom/src/compositor.rs` (new), `bottom/src/lib.rs` (the three exact edits above), and `bottom/Cargo.toml` (the unicode-width pin). Do not edit any other bottom file, and never commit to the `bottom/` submodule.
