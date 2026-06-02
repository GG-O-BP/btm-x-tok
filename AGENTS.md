# AGENTS.md — btm-x-tok harness (single source of truth)

> **This file is the canonical spec for EVERY coding agent** (Claude Code, OpenAI
> Codex, Cursor, GitHub Copilot, Gemini CLI, Aider, Windsurf, Zed, …). Other agent
> files only point here or invoke the harness defined here — they never duplicate
> it. Read this top-to-bottom before generating anything.

`btm-x-tok` is a **spec, not a program**. The published repo contains only this
spec + the two upstream submodules. On demand, an agent reads this file and
**regenerates an integration** that composes the two tools into one TUI, runs it
locally, and **discards it** — nothing generated is ever committed.

---

## How to run the harness (native, per agent)

The regeneration is wired into each agent's **own native mechanism** — no custom
folder. Invoke whichever matches your agent; all do the same thing (this spec):

| Agent | Invoke | Backed by |
|---|---|---|
| **Claude Code** | `/regen` or the auto-triggered **regenerate-monitor** skill | `.claude/commands/regen.md`, `.claude/skills/regenerate-monitor/`, `.claude/agents/monitor-generator.md`, `.claude/settings.json` |
| **Gemini CLI** | `/regen` | `.gemini/commands/regen.toml` |
| **GitHub Copilot** | `/regen` prompt | `.github/prompts/regen.prompt.md` (+ `.github/copilot-instructions.md`) |
| **Cursor** | ask in Composer; rule always-on | `.cursor/rules/agents.mdc` |
| **Aider** | loads this file every session | `.aider.conf.yml` (`read: [AGENTS.md]`) |
| **Codex / Windsurf / Zed / Jules / Factory / Amp / OpenCode** | read `AGENTS.md` natively | this file |

All of those wrappers carry the **same instruction**: *"Regenerate the unified
monitor per AGENTS.md, in the gitignored workspace, run the acceptance gates,
never commit."* The spec content lives **only here**.

---

## Literal interpretation (read before writing code)

This is a **build specification, not a suggestion.** Reproduce it exactly.

- Do **not** improve, refactor, modernize, optimize, or upgrade anything.
- Do **not** bump a version, port an API, or "clean up" a copied event loop.
- Where a signature, version, or path is given it is **NORMATIVE** — match it exactly.
- When the spec and your judgment disagree, **the spec wins.**
- When the spec is silent, consult the reference at **`.attic/`** for *shape*
  (module layout, signatures, variant coverage) — `.attic = shape, not bytes`.
  Copied event-loop bodies must be **re-derived from CURRENT upstream**
  (`bottom/src/lib.rs` `start_bottom`; `tokscale/.../tui/mod.rs`
  `run_loop_with_background`), because `.attic` may lag upstream.

---

## What you are building — Design A, in-process compositor

A single binary, the `monitor` crate, composes **bottom** (system monitor) and
**tokscale** (AI token monitor) into ONE TUI. `monitor` owns the one real
terminal; each tool renders, unmodified, into its own in-memory ratatui backend;
the compositor blits the active pane and routes input. F1 = System (bottom),
F2 = Tokens (tokscale); the System screen carries an agents-% overlay; `--battery`
adds bottom's battery widget. See `README.md` for product rationale.

```
                ┌───────────────── MAIN THREAD (monitor/src/main.rs) ─────────────────┐
 real terminal ◀┤ TerminalGuard: crossterm 0.29 raw + alt screen (Drop restores once) │
 (one owner)    │ router_loop: one input poller event::poll(8ms); F1/F2 switch screen │
                │ blit ONLY the active engine's buffer (prev-frame cell diff)         │
                └───────┬───────────────────────────────────────┬────────────────────┘
            Sender<CompositorInput>                    Sender<CompositorInput>
            (bottom, crossterm 0.29 direct)            (tokscale, 0.28 via bridge::{key,mouse})
                        ▼                                          ▼
        THREAD S: bottom run_in_compositor_with_config   THREAD T: tokscale_cli::tui::
        renders ratatui 0.30 → SharedBackend030          run_in_compositor(.., 100)
        (ratatui-core 0.1) → Arc<Mutex<Buffer>>          renders ratatui 0.29 → SharedBackend029
        + collection & cleaning threads                  → Arc<Mutex<Buffer>>; tokio in workers
                        │ flush() sets dirty                       │ flush() sets dirty
                        └──────────────► MAIN reads active grid ◄──┘
                          dirty swaps true (or force_full) → blit::blit / blit29::blit29
                          → stdout; on System overlay agents_overlay::draw_agents
                          into layout::reserved_cell_rect(cols,rows)
```

**Why render code stays untouched — by construction.** Each engine renders with
its OWN ratatui version into its OWN in-memory backend; no `Frame`/`Buffer`/widget
ever crosses the version boundary. The only inter-world bridge is plain crossterm
cells + ANSI bytes. So porting is never needed; the upstream render/widget/state
code is consumed verbatim.

---

## Non-goals

- Do **not** edit bottom's or tokscale's render / widget / state / collection code.
- Do **not** commit any generated path — the integration is ephemeral.
- Do **not** add features beyond this Phase-2 reference (F1/F2 tab switch, the
  agents-% overlay, `--battery`).
- Not a PTY/tmux multiplexer; not a ratatui 0.29→0.30 port; do not unify the two
  ratatui/crossterm majors; do not port tokscale off tokio.
- Out of scope / future: themes, unified config beyond the layout TOML,
  cross-platform hardening.

---

## Workspace map + ephemerality

| Path | Role |
|---|---|
| `bottom/` | Pristine upstream **submodule**. You only **add** shim files (`src/compositor.rs`, the listed edits). Never edit render/widget/state code; never commit to it. |
| `tokscale/` | Pristine upstream **submodule**. Add shims (`crates/tokscale-cli/src/{lib.rs,client_filter.rs,tui/compositor.rs}` + 2 edits). Same rule. |
| `monitor/` | The only crate you fully own — 10 source files. **Generated, gitignored.** |
| `vendor/unicode-ellipsis/` | Patched dep for the unicode-width reconciliation. Generated, gitignored. |
| `Cargo.toml` (root) | Workspace manifest. Generated, gitignored. |
| `.attic/{monitor,bottom,tokscale}/` | **Read-only** reference (previous generation). Consult for SHAPE only; never write to it. |
| `scripts/update-upstream.sh` | Upstream-pull helper (drift runbook below). |

**Ephemerality rule:** `monitor/`, `vendor/`, root `Cargo.toml`, and every shim
file you add to the submodules are **gitignored** and must **never** be committed.
If `git status` shows them, that is expected — **never `git add` them.** The
allowlist `.gitignore` makes this structural: only spec/docs + the two submodules
+ agent config are publishable.

---

## Invariants (hard gates — each is a checkable assertion)

1. **I1 — Zero edits to render/widget/state/collection code.** In `bottom/` and
   `tokscale/` only the enumerated *new files* + *minimal whitelisted edits* below.
2. **I2 — Two ratatui majors + two crossterm majors coexist; only crossterm 0.29
   owns the real terminal.** ratatui 0.30 (bottom) + 0.29 (tokscale, alias
   `ratatui029`); crossterm 0.29 (`crossterm`) + 0.28 (alias `crossterm028`). The
   in-memory backends and crossterm 0.28 never touch the real TTY.
3. **I3 — Exact public entry-point signatures (match verbatim):**
   - bottom: `pub fn run_in_compositor<B: Backend>(backend: B, input_rx: Receiver<CompositorInput>) -> anyhow::Result<()>` and `…_with_config(backend, input_rx, config_path: Option<std::path::PathBuf>)`. `Backend` = `tui::backend::Backend` (ratatui 0.30); `Receiver` = `std::sync::mpsc::Receiver`. The first delegates with `None`.
   - tokscale: `pub fn run_in_compositor<B: Backend>(backend: B, input_rx: Receiver<CompositorInput>, tick_ms: u64) -> Result<()>`. `Backend` = `ratatui::backend::Backend` (ratatui 0.29); `Result` = `anyhow::Result`; callers pass `tick_ms = 100`.
4. **I4 — `unicode-width` is `0.2.0` tree-wide; `openssl-sys` absent.** Verify:
   `cargo tree -p monitor -i unicode-width` (one 0.2.0 node) and `… -i openssl-sys` (none).
5. **I5 — Standalone builds still pass:** `cargo build -p bottom` and
   `cargo build -p tokscale-cli`.
6. **I6 — Generated artifacts are ephemeral & gitignored; never committed.**
7. **I7 — Work only inside the gitignored workspace; never modify `.attic/`.** The
   editable set is the closed per-target whitelist below.
8. **I8 — tokscale `CompositorInput` has NO `Paste` variant** (`Key, Mouse, Resize,
   Terminate`); bottom's DOES (`+ Paste(String)`). `main.rs` forwards `Event::Paste`
   to bottom only.
9. **I9 — Do not upgrade/port/refactor anything.** The two-major split is
   intentional and load-bearing, not a migration to finish.

**Verify render untouched:** `git -C bottom status --porcelain` and
`git -C tokscale status --porcelain` must show ONLY the whitelisted shim files
(listed per target). Anything else = an I1 failure; revert it.

---

## Versions & dependency reconciliation (EXACT — do not bump)

| Crate | bottom | tokscale | monitor (links both) |
|---|---|---|---|
| ratatui | `0.30.0` (pkg `tui`) + `ratatui-core 0.1` | `0.29` | `ratatui-core "0.1.0"` + `ratatui029 = {package="ratatui", version="0.29"}` |
| crossterm | `0.29.0` | `0.28` | `crossterm "0.29.0"` + `crossterm028 = {package="crossterm", version="0.28"}` |
| edition / MSRV | 2024 / 1.86 | 2021 | 2021 |
| other | unicode-width pinned `0.2.0` | tokio; reqwest TLS (see below) | `bottom`/`tokscale-cli` path deps, `anyhow "1"` |

**`monitor/Cargo.toml` `[dependencies]` (VERBATIM — the renames are load-bearing):**

```toml
[dependencies]
bottom = { path = "../bottom" }
ratatui-core = "0.1.0"                 # unify with bottom's render-core trait
crossterm = "0.29.0"                   # single real-terminal owner (bottom's major)
ratatui029 = { package = "ratatui", version = "0.29" }   # tokscale's render major
crossterm028 = { package = "crossterm", version = "0.28" } # tokscale's event types
tokscale-cli = { path = "../tokscale/crates/tokscale-cli" }
anyhow = "1"
```

Package header: `name = "monitor"`, `version = "0.0.0"`, `edition = "2021"`,
`publish = false`, `[[bin]] name = "monitor", path = "src/main.rs"`.

**Root `Cargo.toml` (VERBATIM):**

```toml
[workspace]
resolver = "3"
members = ["monitor"]
exclude = ["bottom", "tokscale", "vendor/unicode-ellipsis"]

[patch.crates-io]
unicode-ellipsis = { path = "vendor/unicode-ellipsis" }
```

**unicode-width conflict (the one real reconciliation).** ratatui 0.29 hard-pins
`unicode-width = "=0.2.0"`; bottom's direct dep + `unicode-ellipsis 0.4.0` want
`>=0.2.2`. API is identical across `0.2.0..0.2.2`, so lower both floors:
(a) `bottom/Cargo.toml`: `unicode-width "0.2.2"` → `"0.2.0"` (the one bottom Cargo
edit; keep a 3-line reconciliation comment). (b) Vendor `unicode-ellipsis 0.4.0`
into `vendor/unicode-ellipsis/` changing only its `unicode-width` floor to
`[dependencies.unicode-width] version = "0.2.0", default-features = false`, then
patch it via the root `[patch.crates-io]` above. Standalone bottom still builds.

**tokscale TLS — REACTIVE only.** Pristine upstream uses
`reqwest = { …, features = ["json", "native-tls-vendored"], … }` which needs
`openssl-src` + perl. Only if `cargo build -p monitor` fails referencing
`openssl-sys`/`openssl-src`/perl, switch the feature to `rustls-tls` in
`tokscale/Cargo.toml`. Otherwise leave it.

---

## Build order + per-target contracts

Generate in this order; an earlier failure blocks the next.

### 0. Toolchain — `rustc --version` ≥ 1.86 (bottom is edition 2024, resolver "3").

### 1. Workspace + vendored patch
Root `Cargo.toml` (verbatim above) + `vendor/unicode-ellipsis/` (vendored 0.4.0
with the single unicode-width floor change). Both blocks are given verbatim above;
re-vendor `unicode-ellipsis 0.4.0` from crates.io and apply only that floor change.

### 2. bottom shim — closed whitelist: `src/compositor.rs` (NEW), `src/lib.rs` (EDIT), `Cargo.toml` (EDIT)
- **`src/compositor.rs`** (NEW): the two `run_in_compositor*` signatures (I3), the
  `CompositorInput { Key, Mouse, Paste(String), Resize, Terminate }` enum
  (crossterm 0.29 types, `#[derive(Debug)]`), and a private
  `fn try_drawing<B: Backend>(terminal, app, painter)`. Setup mirrors `start_bottom`
  MINUS terminal setup: parse `BottomArgs::parse_from(["btm"])`,
  `get_or_create_config`, `init_app`, `Painter::init`, spawn the **collection** and
  **cleaning** (`offset_wait = retention_ms + 60000`) threads and an
  **input-translator** thread (`Key→KeyInput, Mouse→MouseInput, Paste→PasteEvent,
  Resize→Resize, Terminate→Terminate`). **Copy the event loop from CURRENT upstream
  `src/lib.rs` `start_bottom`, preserving every `BottomEvent` arm**; OMIT ctrl-c,
  panic hook, and all terminal setup/teardown. Shape ref: `.attic/bottom/src/compositor.rs`.
- **`src/lib.rs`** (EDIT, 3 changes): add `pub(crate) mod compositor;`; add
  `pub use compositor::{CompositorInput, run_in_compositor, run_in_compositor_with_config};`;
  widen `fn create_collection_thread(` → `pub(crate) fn create_collection_thread(`.
- **`Cargo.toml`** (EDIT): unicode-width pin (above).
- Then `cargo build -p bottom` must pass.

### 3. tokscale shims — whitelist: NEW `crates/tokscale-cli/src/{lib.rs, client_filter.rs, tui/compositor.rs}`; EDIT `src/main.rs`, `src/tui/mod.rs`, `tokscale/Cargo.toml` (TLS, conditional)
- **`src/lib.rs`** (NEW facade): `#![allow(dead_code)]`; private `mod client_filter;`
  + `pub(crate) use client_filter::ClientFilter;`; private `mod auth; mod commands;
  mod cursor; mod paths; mod warp;`; `pub mod tui;`. Plus the load-bearing re-export
  (the monitor's agents code imports these EXACT names):
  `pub mod usage { pub use crate::commands::usage::helpers::render_ascii_bar; pub use crate::commands::usage::{UsageMetric, UsageOutput, fetch_all, load_cache}; }`.
- **`src/client_filter.rs`** (NEW): the `ClientFilter` enum **moved verbatim from
  CURRENT upstream `main.rs`** (`#[derive(ValueEnum, Clone, Copy, Debug, PartialEq,
  Eq, Hash)]`, `#[value(rename_all = "lowercase")]`, keep `#[value(name="trae")] Trae`,
  and methods `as_filter_str, to_client_id, from_client_id, from_filter_str,
  default_set`). Variant order = `ClientId::ALL` order + `Synthetic` last. Shape ref:
  `.attic/tokscale/crates/tokscale-cli/src/client_filter.rs`.
- **`src/tui/compositor.rs`** (NEW): the I3 signature; `CompositorInput { Key, Mouse,
  Resize, Terminate }` (crossterm 0.28 types — **no Paste**, I8). Mirror upstream
  `run` prep + `run_loop_with_background`: build `TuiConfig { theme:"blue", refresh:0,
  sessions_path:None, clients:None, since:None, until:None, year:None, initial_tab:None }`,
  `App::new_with_cached_data(config, None)?`, `Terminal::new(backend)?`, the initial
  background load (`ClientFilter::default_set()` → `to_client_id`), a **ticker** thread
  (`Duration::from_millis(tick_ms.max(16))` → `Event::Tick`) and an **input** thread
  (`Key→Event::Key, Mouse→Event::Mouse, Resize→Event::Tick, Terminate→None`). Keep
  every `App`/`ui::render` call byte-identical to upstream. Shape ref:
  `.attic/tokscale/crates/tokscale-cli/src/tui/compositor.rs`.
- **`src/main.rs`** (EDIT): replace the inline `ClientFilter` with
  `mod client_filter; pub(crate) use client_filter::ClientFilter;`.
- **`src/tui/mod.rs`** (EDIT): add `pub mod compositor;` + `pub use compositor::{CompositorInput, run_in_compositor};`.
- Then `cargo build -p tokscale-cli` must pass.

### 4–5. monitor crate — 10 files (full per-file signature contract: `.attic/monitor/src/<file>`)
`Cargo.toml` (verbatim above); `backend.rs` (`SharedState{grid:Arc<Mutex<Buffer>>,
width/height:AtomicU16, dirty:AtomicBool}` + `SharedBackend030: ratatui_core::Backend`,
`type Error = Infallible`; `draw` writes cells under one lock, `size()` reads state,
`flush()` sets dirty); `backend29.rs` (same shape for ratatui 0.29 `Backend`,
`io::Result`); `blit.rs`/`blit29.rs` (Buffer → crossterm 0.29 prev-frame diff paint;
shared Color/Modifier map; full repaint when `prev.area != cur.area`); `bridge.rs`
(crossterm 0.29→0.28 pure re-spell — `pub fn key`/`pub fn mouse`; bitflags via
`from_bits_truncate(.bits())`; **total** match over KeyCode/KeyEventKind/MediaKeyCode/
ModifierKeyCode/MouseButton/MouseEventKind — derive the full variant set from current
enums); `layout.rs` (`layout_toml(battery)` emitting bottom's layout TOML with one set
of ratio constants `ROW_CPU=30, ROW_MID=40, ROW_NET_PROC=30, COL_MEM=4, COL_TEMPDISK=3,
COL_AGENTS=1`; `reserved_cell_rect(cols,rows)` replicating bottom's two-pass Layout split
over the SAME constants, `<2x2 → None`; `#[cfg(test)]` tests); `agents_data.rs`
(`AgentsData::{start, snapshot, stop}`; cache-first `load_cache` + ~60s background
`fetch_all` thread; never network on the UI thread); `agents_overlay.rs`
(`draw_agents(buf, area, &[UsageOutput])`; one row per provider/metric; abbrev +
metric tag; threshold colors `<10` red `<25` yellow else gray; default-theme tints;
`<2x2` guard); `main.rs` (`Screen{System,Tokens}`; `TerminalGuard` raw+alt, restore
once on drop; panic hook chaining restore; `router_loop` F1/F2 + `force_full` +
active-only blit + System overlay + resize-both + input routing bottom-direct /
tokscale-via-bridge + Terminate both on exit; `battery_enabled()`; headless paths;
env vars below).

### 6. `cargo build -p monitor` — must pass.

---

## Build & acceptance gates (Definition of Done)

Run from the workspace root. All gates pass/fail-blocking; nothing is committed.

- **Build:** `cargo build -p monitor`.
- **G1 headless bottom:** `MONITOR_HEADLESS=1 cargo run -p monitor` → `grid WxH,
  non-empty cells: N` with N>0 + a nonzero blit escape count.
- **G2 headless tokscale:** `MONITOR_HEADLESS_TOK=1 cargo run -p monitor` →
  live `tokscale grid WxH, non-empty cells: N` (N>0). (Checked FIRST, before `MONITOR_HEADLESS`.)
- **G3 overlay:** `MONITOR_HEADLESS=1 MONITOR_CUSTOM_LAYOUT=1 MONITOR_W=120 MONITOR_H=40`
  (and `80x24`) → `reserved agents cell = Some(Rect{…})` with overlaid rows in the dump.
- **G4 unit test:** `cargo test -p monitor` (layout tests).
- **G5 deps:** `cargo tree -p monitor -i ratatui` (0.29 + 0.30), `… -i crossterm`
  (0.28 + 0.29), `… -i unicode-width` (0.2.0 only), `… | grep openssl-sys` (empty).
- **G6 standalone + render-untouched:** `cargo build -p bottom`, `cargo build -p
  tokscale-cli`, and the two `git -C … status --porcelain` checks show only the
  whitelisted shim files.
- **G7 TTY:** `cargo run -p monitor` → F1=System, F2=Tokens, `q` quits, terminal
  restored on exit and on panic; `cargo run -p monitor -- --battery` adds the
  battery widget without breaking the overlay.

Env vars `main.rs` honors: `MONITOR_HEADLESS_TOK` (first), `MONITOR_HEADLESS`,
`MONITOR_CUSTOM_LAYOUT`, `MONITOR_W` (def 120), `MONITOR_H` (def 40); CLI `--battery`.

---

## Upstream drift runbook

When `bottom/` or `tokscale/` moves to newer upstream (`git submodule update
--remote …`, or `scripts/update-upstream.sh`): re-derive the shims from the NEW
upstream, `cargo build -p monitor`, then run all gates. **Always re-derive copied
loop bodies from current upstream, never from `.attic`.** Reconciliation points:

- (a) new `BottomEvent` arm → add it to the copied loop in `bottom/src/compositor.rs`.
- (b) tokscale `run_loop_with_background`/`Event` change → re-sync `tui/compositor.rs`.
- (c) key/mouse change → extend `monitor/src/bridge.rs`.
- (d) `ClientFilter` change in tokscale `main.rs` → re-apply the extraction to `client_filter.rs`.
- (e) dep drift → re-verify `cargo tree`; keep bottom unicode-width `0.2.0`; keep
  tokscale `rustls-tls` only if openssl can't build.

Silently-drifting copies (no compile error): the two event-loop bodies, the
`BottomEvent` mapping, `agents_overlay.rs` default-theme `TINTS` + tokscale usage
thresholds, and `layout.rs`'s mirror of bottom's Layout split. Re-derive on every pull.
