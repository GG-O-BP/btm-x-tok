# 12 — Target: the `monitor/` crate

The owned crate. This is the only code you author from scratch and the only place the two
TUIs are composed. Everything here lives at `/home/superset/btm-toks/monitor/` and is
**gitignored** — if `git status` shows it, that is expected, never `git add` it.

This file is the **signature contract** for the 10 files under `monitor/src/`. Generate each
file to satisfy the contract another file depends on; re-derive bodies from the reference
`.attic/monitor/src/<file>` (and `.attic/{bottom,tokscale}` for the embedded engines), never invent APIs.

- Versions / dep specs: see `harness/02-versions-and-deps.md` (do not bump anything — EXACT).
- Hard gates: see `harness/01-invariants.md`. Architecture: see `harness/00-overview.md`.
- Engine shims this crate calls into: see `harness/10-target-bottom-shim.md` and
  `harness/11-target-tokscale-shim.md`.

Reference for every file below: `.attic/monitor/src/<file>` (read-only, authoritative for shape).
`.attic/{bottom,tokscale}` is consulted for the engine internals the embedded loops mirror.

## File map

```
monitor/Cargo.toml
monitor/src/main.rs          router + entry + headless harness
monitor/src/backend.rs       ratatui-core 0.1 capturing backend (bottom)
monitor/src/backend29.rs     ratatui 0.29 capturing backend (tokscale)
monitor/src/blit.rs          ratatui-core 0.1 Buffer -> crossterm 0.29 paint
monitor/src/blit29.rs        ratatui 0.29  Buffer -> crossterm 0.29 paint
monitor/src/bridge.rs        crossterm 0.29 event -> 0.28 event re-spell
monitor/src/layout.rs        layout TOML + reserved overlay Rect (shared constants)
monitor/src/agents_data.rs   cache-first background tokscale usage poller
monitor/src/agents_overlay.rs draw_agents overlay onto bottom's reserved cell
```

`main.rs` declares the modules (no `agents_*` after `bridge`? — declare all):

```rust
mod agents_data;
mod agents_overlay;
mod backend;
mod backend29;
mod blit;
mod blit29;
mod bridge;
mod layout;
```

---

## `monitor/Cargo.toml`

Verbatim block: **see harness/02-versions-and-deps.md** (§1a there). Do not edit the version
specs. Key invariants restated: `bottom`/`tokscale-cli` are path deps; `ratatui-core = "0.1.0"`
must unify with bottom's; `crossterm = "0.29.0"` is the single real-terminal owner;
`ratatui029`/`crossterm028` are the renamed `package = "ratatui"` 0.29 / `package = "crossterm"`
0.28 deps. Reference: `.attic/monitor/Cargo.toml`.

---

## `backend.rs` — bottom's capturing backend (ratatui-core 0.1)

A headless `Backend` that bottom's `Terminal` draws into; instead of touching a real terminal
it writes cells into an in-memory `Buffer` behind a lock. The router blits that buffer.

**`pub struct SharedState`** `#[derive(Clone)]` — shared between the engine thread (writer) and
the router thread (reader):

```rust
pub grid: Arc<Mutex<Buffer>>,   // ratatui_core::buffer::Buffer
width: Arc<AtomicU16>,          // private
height: Arc<AtomicU16>,         // private
pub dirty: Arc<AtomicBool>,
```

Methods (contract — other files depend on these):
- `pub fn new(width: u16, height: u16) -> Self` — starts with `dirty = true`.
- `pub fn size(&self) -> Size` — **single source of truth** for the pane size; the backend's
  `size()` reads from here so a resize is observed atomically by the engine.
- `pub fn set_size(&self, width: u16, height: u16)` — router calls on `Event::Resize`.

**`pub struct SharedBackend030`** — private fields `state: SharedState`, `cursor_pos: Position`,
`cursor_visible: bool`. `pub fn new(state: SharedState) -> Self`.

`impl Backend for SharedBackend030` (`ratatui_core::backend::Backend`),
`type Error = core::convert::Infallible`. Implement **exactly** these methods:

| method | behavior |
|---|---|
| `draw<'a, I>(&mut self, content: I) -> Result<(), Self::Error> where I: Iterator<Item = (u16, u16, &'a Cell)>` | take the `grid` lock **once**, write each `(x, y, cell)` into the buffer, drop the lock |
| `hide_cursor` / `show_cursor` | toggle `cursor_visible` |
| `get_cursor_position(&mut self) -> Result<Position, Self::Error>` | return `cursor_pos` |
| `set_cursor_position<P: Into<Position>>(&mut self, position: P)` | store `cursor_pos` |
| `clear(&mut self)` | reset buffer cells |
| `clear_region(&mut self, _clear_type: ClearType)` | **delegates to `clear()`** |
| `size(&self) -> Result<Size, Self::Error>` | read from `state.size()` |
| `window_size(&mut self) -> Result<WindowSize, Self::Error>` | cols/rows from state, `pixels: Size::ZERO` |
| `flush(&mut self) -> Result<(), Self::Error>` | **set `state.dirty`** so the router knows a frame is ready |

Reference: `.attic/monitor/src/backend.rs`.

---

## `backend29.rs` — tokscale's capturing backend (ratatui 0.29)

Same shape as `backend.rs` for the **ratatui 0.29** `Backend` trait, which has **no associated
`Error` type** — every method returns `io::Result<_>`.

**`pub struct SharedState29`** `#[derive(Clone)]`: `pub grid: Arc<Mutex<Buffer>>`
(`ratatui029::buffer::Buffer`), private `width`/`height: Arc<AtomicU16>`, `pub dirty: Arc<AtomicBool>`.
Methods: `pub fn new(width, height) -> Self`; `pub fn size(&self) -> Size`;
`#[allow(dead_code)] pub fn set_size(&self, width, height)`.

**`pub struct SharedBackend029`**; `pub fn new(state: SharedState29) -> Self`.

`impl Backend for SharedBackend029` (`ratatui029::backend::Backend`):

| method | notes |
|---|---|
| `draw<'a, I>(&mut self, content: I) -> io::Result<()> where I: Iterator<Item = (u16, u16, &'a Cell)>` | one-lock cell write |
| `hide_cursor` / `show_cursor` -> `io::Result<()>` | |
| `get_cursor_position(&mut self) -> io::Result<Position>` | |
| `set_cursor_position<P: Into<Position>>(&mut self, position: P) -> io::Result<()>` | |
| `clear(&mut self) -> io::Result<()>` | **no `clear_region`** in the 0.29 impl |
| `size(&self) -> io::Result<Size>` | reads state |
| `window_size(&mut self) -> io::Result<WindowSize>` | |
| `flush(&mut self) -> io::Result<()>` | sets `dirty` |

Reference: `.attic/monitor/src/backend29.rs`.

---

## `blit.rs` — ratatui-core 0.1 `Buffer` -> crossterm 0.29 paint

Diff-paints the captured `Buffer` to a `Write` sink (the real stdout, owned by the router) using
the previous frame to emit only changed cells.

Public surface (the only thing other files call):
- `pub fn blit<W: Write>(out: &mut W, prev: &Buffer, cur: &Buffer) -> io::Result<()>`

Private helpers: `to_ct_color(c: Color) -> CtColor` (`ratatui_core::style::Color` ->
`crossterm::style::Color`), `apply_modifiers<W: Write>(out: &mut W, m: Modifier) -> io::Result<()>`,
`same(a: &Cell, b: &Cell) -> bool`.

Behavior:
- Full repaint when `prev.area != cur.area`; otherwise emit only cells where `!same(prev, cur)`.
- `same` compares `symbol()`, `fg`, `bg`, `modifier`.
- For each emitted cell, move cursor, apply modifiers, set fg/bg, write the symbol.
- bottom's own colors already arrive as ratatui-core `Color`; map them with `to_ct_color`
  (the hand color map — see the shared table below). Note bottom internally uses
  `into_crossterm`-style conversion; this crate re-implements the equivalent map locally so it
  works against `ratatui-core` `Color` rather than a 0.30 `ratatui` type.

**Shared color map** (identical in `blit.rs` and `blit29.rs`):

```
Reset→Reset  Black→Black  Red→DarkRed  Green→DarkGreen  Yellow→DarkYellow
Blue→DarkBlue  Magenta→DarkMagenta  Cyan→DarkCyan  Gray→Grey  DarkGray→DarkGrey
LightRed→Red  LightGreen→Green  LightYellow→Yellow  LightBlue→Blue
LightMagenta→Magenta  LightCyan→Cyan  White→White
Rgb{r,g,b}→Rgb  Indexed(i)→AnsiValue(i)
```

**Shared modifier order** (`apply_modifiers`, identical in both blit files):

```
BOLD→Bold  DIM→Dim  ITALIC→Italic  UNDERLINED→Underlined  SLOW_BLINK→SlowBlink
RAPID_BLINK→RapidBlink  REVERSED→Reverse  HIDDEN→Hidden  CROSSED_OUT→CrossedOut
```

Reference: `.attic/monitor/src/blit.rs`.

---

## `blit29.rs` — ratatui 0.29 `Buffer` -> crossterm 0.29 paint

Identical structure to `blit.rs`, but the input `Buffer`/`Cell`/`Color`/`Modifier` are the
**ratatui 0.29** types (`ratatui029::...`). Output target is still **crossterm 0.29** (the single
terminal owner).

Public surface:
- `pub fn blit29<W: Write>(out: &mut W, prev: &Buffer, cur: &Buffer) -> io::Result<()>`

Private helpers: `to_ct_color(c: Color) -> CtColor` (`ratatui029::style::Color`),
`apply_modifiers<W: Write>(out: &mut W, m: Modifier) -> io::Result<()>`, `same(a, b) -> bool`.
Color map, modifier order, `same` fields, and full-repaint-on-area-change rule are **exactly the
same** as `blit.rs` (table above). The `Modifier -> Attributes` mapping is the per-attribute
write loop in the order listed.

Reference: `.attic/monitor/src/blit29.rs`.

---

## `bridge.rs` — crossterm 0.29 event -> crossterm 0.28 event

A **pure, mechanical re-spell**. tokscale's handlers are typed against crossterm 0.28; the real
terminal emits 0.29 events. This module rebuilds each event field-for-field across the version
gap. Nothing here is semantic — it is a total match over every variant. **Enumerate ALL variants;
derive the exact set from the CURRENT crossterm 0.29/0.28 enums** (`ct29 = crossterm::event`,
`ct28 = crossterm028::event`).

Public functions (contract used by `main.rs`):
- `pub fn key(e: ct29::KeyEvent) -> ct28::KeyEvent`
- `pub fn mouse(e: ct29::MouseEvent) -> ct28::MouseEvent`

Private helpers: `modifiers`, `state`, `kind`, `media`, `modifier_key`, `code`, `mouse_button`,
`mouse_kind`.

Rules:
- `KeyEvent` carries `{ code, modifiers, kind, state }`; `MouseEvent` carries
  `{ kind, column, row, modifiers }`.
- **Bitflags** (`modifiers`, `state`) round-trip via `ct28::X::from_bits_truncate(e.bits())` —
  re-spell, do not re-derive.
- Every other field is a total `match`. Cover **all** of these (from current enums):

| field | complete variant list |
|---|---|
| `kind` (`KeyEventKind`) | `Press, Repeat, Release` |
| `code` (`KeyCode`) | `Backspace, Enter, Left, Right, Up, Down, Home, End, PageUp, PageDown, Tab, BackTab, Delete, Insert, F(n), Char(ch), Null, Esc, CapsLock, ScrollLock, NumLock, PrintScreen, Pause, Menu, KeypadBegin, Media(m), Modifier(m)` |
| `media` (`MediaKeyCode`) | `Play, Pause, PlayPause, Reverse, Stop, FastForward, Rewind, TrackNext, TrackPrevious, Record, LowerVolume, RaiseVolume, MuteVolume` |
| `modifier_key` (`ModifierKeyCode`) | `LeftShift, LeftControl, LeftAlt, LeftSuper, LeftHyper, LeftMeta, RightShift, RightControl, RightAlt, RightSuper, RightHyper, RightMeta, IsoLevel3Shift, IsoLevel5Shift` |
| `mouse_button` (`MouseButton`) | `Left, Right, Middle` |
| `mouse_kind` (`MouseEventKind`) | `Down(b), Up(b), Drag(b), Moved, ScrollDown, ScrollUp, ScrollLeft, ScrollRight` |

If current crossterm adds a variant not listed, add the matching arm — the contract is *total
coverage*, not this frozen list. Reference: `.attic/monitor/src/bridge.rs`.

---

## `layout.rs` — layout TOML + reserved overlay Rect (shared constants)

Two outputs derived from **one set of ratio constants** so the bottom layout and the overlay Rect
**cannot drift**: the TOML feeds bottom's layout engine, and the Rect math replicates that same
split to find where the agents overlay lands.

**Ratio constants** (the single source both outputs read):

```rust
const ROW_CPU: u16 = 30;
const ROW_MID: u16 = 40;
const ROW_NET_PROC: u16 = 30;
const COL_MEM: u16 = 4;
const COL_TEMPDISK: u16 = 3;
const COL_AGENTS: u16 = 1;
```

### `pub fn layout_toml(battery: bool) -> String`

Returns `format!("{cpu_row}\n{rest}")`. The ratio numbers in the emitted TOML are the constants
above (interpolated, not hard-typed twice).

CPU row when `battery == false`:

```toml
[[row]]
ratio = 30
  [[row.child]]
  type = "cpu"
```

CPU row when `battery == true` (CPU splits 2:1 cpu:battery; **row-1 geometry is unchanged**, so
`reserved_cell_rect` stays valid):

```toml
[[row]]
ratio = 30
  [[row.child]]
  ratio = 2
  type = "cpu"
  [[row.child]]
  ratio = 1
  type = "battery"
```

`rest` (appended after `"\n"`):

```toml
[[row]]
ratio = 40
  [[row.child]]
  ratio = 4
  type = "mem"
  [[row.child]]
  ratio = 3
    [[row.child.child]]
    type = "temp"
    [[row.child.child]]
    type = "disk"
  [[row.child]]
  ratio = 1
  type = "empty"

[[row]]
ratio = 30
  [[row.child]]
  type = "net"
  [[row.child]]
  type = "proc"
  default = true
```

### `pub fn reserved_cell_rect(cols: u16, rows: u16) -> Option<Rect>`

Replicates bottom's two-pass `Layout` split using `ratatui_core::layout::{Constraint, Layout, Rect}`:
1. Vertical split over `(cols, rows)` with `[Fill(ROW_CPU), Fill(ROW_MID), Fill(ROW_NET_PROC)]`.
2. Horizontal split on the **middle** band `rows3[1]` with
   `[Fill(COL_MEM), Fill(COL_TEMPDISK), Fill(COL_AGENTS)]`.
3. Return `cols3[2]` (the agents/`empty` cell).
4. Return `None` if `cell.width < 2 || cell.height < 2`.

This is the rect into which `main.rs` overlays the agents widget. It corresponds to the `type =
"empty"` child in the TOML's middle row — the TOML reserves the slot, the Rect math finds it.

**Unit tests** (`#[cfg(test)]`): `reserved_cell_matches_reference_and_bounds`,
`tiny_terminal_yields_none`; grid sizes exercised `(80,24)`, `(120,40)`, `(200,50)`, `(100,30)`.
Test internals beyond these names: consult `.attic/monitor/src/layout.rs`.

Reference: `.attic/monitor/src/layout.rs`.

---

## `agents_data.rs` — cache-first background usage poller

Owns the tokscale usage data shown in the overlay. **Never** does network I/O on the UI/router
thread: it seeds from cache instantly and refreshes on a background thread.

Imports `tokscale_cli::usage::{UsageOutput, fetch_all, load_cache}`.

**`pub struct AgentsData`** `#[derive(Clone)]` — both fields private:

```rust
inner: Arc<Mutex<Vec<UsageOutput>>>,
cancel: Arc<AtomicBool>,
```

Methods (contract):
- `pub fn start() -> Self` — seeds `inner` from `load_cache().unwrap_or_default()`, then spawns the
  refresh thread.
- `fn spawn_refresh(&self)` (private) — loop: call `fetch_all()`, and **only if non-empty** replace
  `inner`. Refresh roughly every 60s, sleeping in ~1s slices (60 × 1s) so `cancel` is observed at
  1s granularity.
- `pub fn snapshot(&self) -> Vec<UsageOutput>` — cheap clone of `inner`; **never blocks on
  network**. The router calls this each frame.
- `pub fn stop(&self)` — sets `cancel` (`Release`); teardown calls it before joining.

Reference: `.attic/monitor/src/agents_data.rs`.

---

## `agents_overlay.rs` — `draw_agents` onto bottom's reserved cell

Renders the agents-% widget directly into bottom's captured `Buffer`, inside the Rect from
`layout::reserved_cell_rect`. Pure draw — no I/O, no allocation of engine state.

**`pub fn draw_agents(buf: &mut Buffer, area: Rect, data: &[UsageOutput])`**
(`ratatui_core` `Buffer`/`Rect`; `tokscale_cli::usage::UsageOutput`).
Private helpers: `abbrev`, `metric_tag`, `pct_color`, `put`, `put_right`.

Constants (bottom DEFAULT theme palette — can drift from upstream, see harness/30-drift-runbook.md):

```rust
const BORDER: Color = Color::Gray;
const HEADER: Color = Color::LightBlue;
const TINTS: [Color; 8] = [LightMagenta, LightYellow, LightCyan, LightGreen,
                           LightBlue, Cyan, Green, Blue]; // per-provider, index i % len
```

**Guard:** skip entirely if `area.width < 2 || area.height < 2`.

**Border:** `BorderType::Plain` (`┌ ┐ └ ┘`, `─`, `│`), no title.

**Layout:** `wide = inner_w >= 12`.
- Header row: `Style::new().fg(LightBlue).add_modifier(BOLD)`; left label `"Agent"` (wide) /
  `"Agt"` (narrow); right column `"Rem%"` (wide) / `"%"` (narrow).
- Empty `data` -> `"no data"` in `DarkGray`.
- Otherwise **one row per `(provider, metric)`**: left label is `"{abbrev} {tag}"` (wide) /
  `"{abbrev}{tag}"` (narrow); right cell is the percent `"{:.0}%"`, right-aligned, styled
  `fg(pct_color(remaining_percent)).add_modifier(BOLD)`. Per-provider tint from `TINTS[i % 8]`.

**Abbreviation rules** (`fn abbrev(provider) -> String`):

```
"Claude"→"CLA"  "Codex"→"CDX"  "Z.ai"→"ZAI"  "Amp"→"AMP"  "Copilot"→"CPL"
"Kimi"→"KMI"  "MiniMax"→"MMX"  "Warp/Oz"→"WRP"
otherwise → other.chars().take(3).collect().to_uppercase()
```

**Metric tag** (`fn metric_tag(label, wide) -> String`): `wide` -> first 4 chars
(`Sess` / `Week` / `Opus`); narrow -> first char (`S` / `W` / `O`).

**Threshold colors** (`fn pct_color(remaining: f64) -> Color`, mirrors tokscale `usage.rs`):

```
remaining < 10.0  → Red
remaining < 25.0  → Yellow
else              → Gray
```

Reference: `.attic/monitor/src/agents_overlay.rs`.

---

## `main.rs` — router, entry point, headless harness

The owner of the real terminal and the single router thread. Mirrors the two engines' threads,
routes input, and blits **only the active** engine.

### `enum Screen` `#[derive(Clone, Copy, PartialEq)]`

```rust
enum Screen { System, Tokens }   // System = bottom, Tokens = tokscale
```

### `struct TerminalGuard` — restore exactly once on drop

- `fn enter() -> Result<Self>`: `enable_raw_mode()` + `execute!(stdout(), EnterAlternateScreen, Hide)`.
- `impl Drop`: `execute!(stdout(), LeaveAlternateScreen, Show)` + `disable_raw_mode()`.
- Restores the terminal exactly once, on drop — covering normal return, `?` early-exit, and panic
  (the guard must be entered **before** spawning engine threads).

### `fn router_loop(...)`

```rust
#[allow(clippy::too_many_arguments)]
fn router_loop(
    sys_handle: &JoinHandle<Result<()>>,
    tok_handle: &JoinHandle<Result<()>>,
    sys_tx: &Sender<CompositorInput>,
    tok_tx: &Sender<tokscale_cli::tui::CompositorInput>,
    sys_state: &SharedState,
    tok_state: &SharedState29,
    agents: &AgentsData,
    mut cols: u16,
    mut rows: u16,
) -> Result<()>
```

Behavior:
- Poll `event::poll(Duration::from_millis(8))`.
- **Exit** if either `sys_handle.is_finished()` or `tok_handle.is_finished()`.
- **F1 → System, F2 → Tokens**; on switch set `force_full = true`.
- Other keys route to the **active** engine:
  - System: `sys_tx.send(CompositorInput::Key(k))` (bottom — **direct**, crossterm 0.29).
  - Tokens: `tok_tx.send(CompositorInput::Key(bridge::key(k)))` (tokscale — **via bridge** to 0.28).
  - Key gate: `matches!(k.kind, KeyEventKind::Press | KeyEventKind::Repeat)`.
- Mouse routes per active screen; for Tokens use `bridge::mouse(m)`.
- `Event::Paste(s)` → **only System** (`CompositorInput::Paste(s)`); tokscale's `CompositorInput`
  has no `Paste` variant (see harness/11-target-tokscale-shim.md). Comment in source:
  `// tokscale's CompositorInput has no Paste; forward only to bottom.`
- `Event::Resize(c, r)` → update `cols`/`rows`, call `sys_state.set_size` / `tok_state.set_size`,
  rebuild the prev buffers, send `Resize` to **both** engines, set `force_full = true`.
- **Blit only the active engine** when its `dirty` swaps true or on `force_full`. On System,
  overlay `agents_overlay::draw_agents(..., agents.snapshot()?, layout::reserved_cell_rect(cols, rows))`
  before blitting. `force_full` issues `Clear(ClearType::All)` first.

### Helper fns

- `fn battery_enabled() -> bool` — `std::env::args().any(|a| a == "--battery")`.
- `fn write_layout_toml(battery: bool) -> std::path::PathBuf` — writes
  `layout::layout_toml(battery)` to
  `std::env::temp_dir().join(format!("btm-toks-layout-{}.toml", std::process::id()))`.
- `fn headless_dims() -> (u16, u16)` — `MONITOR_W` (default 120), `MONITOR_H` (default 40).
- `fn headless_check() -> Result<()>` — bottom headless path (see harness/20-build-and-acceptance.md).
- `fn headless_check_tok() -> Result<()>` — tokscale headless path (see harness/20-build-and-acceptance.md).
- `fn main() -> Result<()>`.

### Environment variables honored (complete list)

| var | read via | effect |
|---|---|---|
| `MONITOR_HEADLESS_TOK` | `std::env::var_os` | checked **first**; if set, run `headless_check_tok()` and return |
| `MONITOR_HEADLESS` | `std::env::var_os` | checked **second**; if set, run `headless_check()` and return |
| `MONITOR_CUSTOM_LAYOUT` | `std::env::var_os` | inside `headless_check`: if set, use `run_in_compositor_with_config` with the custom layout TOML; else default `run_in_compositor` |
| `MONITOR_W` | `std::env::var`, parse | headless width, default `120` |
| `MONITOR_H` | `std::env::var`, parse | headless height, default `40` |

Plus the `--battery` **CLI flag** (scanned by `battery_enabled()`, threaded into
`write_layout_toml` → `layout::layout_toml(battery)`).

### `main()` real-TTY path

Install a **panic hook** that restores the terminal then chains the previous hook. Read `size()`;
build `SharedState` + `SharedBackend030` (sys) and `SharedState29` + `SharedBackend029` (tok); make
the channels; `write_layout_toml(battery_enabled())`; `AgentsData::start()`. **Enter `TerminalGuard`
BEFORE spawning.** Spawn:
- `bottom::run_in_compositor_with_config(sys_backend, sys_rx, Some(layout_path))`
- `tokscale_cli::tui::run_in_compositor(tok_backend, tok_rx, 100)` (tick_ms hard-coded `100`)

Run `router_loop`. Teardown: send `CompositorInput::Terminate` to **both** engines, `agents.stop()`,
join both handles, drop the guard (restores the terminal).

Reference: `.attic/monitor/src/main.rs`.

---

See harness/20-build-and-acceptance.md for the build/headless-verify commands, and
harness/30-drift-runbook.md for what in this crate can drift on an upstream pull (the mirrored
engine loops, the `BottomEvent` mapping, the default-theme `TINTS`, and tokscale's usage
thresholds). Out of scope / future: themes, unified config beyond the layout TOML, cross-platform
hardening.
