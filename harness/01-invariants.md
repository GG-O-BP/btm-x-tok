# 01 — Invariants (hard gates / determinism contract)

These are the non-negotiable gates for the regenerated **Design A — in-process compositor**. Each is a checkable assertion plus the failure it prevents. If any fails, the generation is wrong — fix the generation, do not relax the gate. Canonical entry: `AGENTS.md`. See `harness/00-overview.md` for the architecture and `harness/02-versions-and-deps.md` for the lock-table.

> The integration is **ephemeral**: `monitor/`, `vendor/`, the root `Cargo.toml`, and every shim file are gitignored. They are built in a local workspace and **never committed**. `bottom/` and `tokscale/` are git submodules pinned to upstream latest; you only **add** shim files to them locally.

---

## I1 — Zero edits to render / widget / state / collection code

**Assert:** In both `bottom/` and `tokscale/`, no render, widget, state, layout-painting, or data-collection source file is modified. The only changes are (a) **new additive files** and (b) the **enumerated minimal edits** whitelisted per target in `harness/10-target-bottom-shim.md`, `harness/11-target-tokscale-shim.md`, and `harness/12-target-monitor-crate.md`.

The complete set of allowed touches:

- **bottom** (`harness/10-...`): new file `bottom/src/compositor.rs`; edits to `bottom/src/lib.rs` (module decl + re-export + `create_collection_thread` → `pub(crate)`); edit to `bottom/Cargo.toml` (unicode-width pin).
- **tokscale** (`harness/11-...`): new files `crates/tokscale-cli/src/lib.rs`, `src/client_filter.rs`, `src/tui/compositor.rs`; edits to `src/main.rs` (ClientFilter move), `src/tui/mod.rs` (module decl + re-export), `tokscale/Cargo.toml` (reqwest TLS feature).

**Rationale / prevents:** The whole point of Design A is to compose upstream **unmodified**. Editing render/state code makes upstream pulls (`scripts/update-upstream.sh`) conflict everywhere and silently forks behavior.

---

## I2 — Two ratatui majors + two crossterm majors coexist; only crossterm 0.29 owns the real terminal

**Assert:** The build links **ratatui 0.30** (bottom, via `tui`) and **ratatui 0.29** (tokscale, aliased `ratatui029`), plus **crossterm 0.29** (`crossterm`) and **crossterm 0.28** (tokscale, aliased `crossterm028`) — all four simultaneously. Exactly **one** crate writes to the real TTY: **crossterm 0.29**, from `monitor/src/main.rs` (`TerminalGuard`, `event::poll`, `execute!`). Neither ratatui backend (`SharedBackend030`, `SharedBackend029`) nor crossterm 0.28 touches the real terminal — they render into in-memory `Buffer`s.

**Rationale / prevents:** Two terminal owners corrupt the alternate screen and raw-mode state. crossterm 0.28 events exist only as the *typed target* of `bridge::{key,mouse}` for tokscale's handlers, never as a terminal driver.

---

## I3 — Exact public entry-point signatures (NORMATIVE — match verbatim)

**Assert:** The four compositor entry points match these signatures byte-for-byte. Do not rename, reorder params, or change bounds.

bottom (`bottom/src/compositor.rs`):

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

Here `Backend` = `tui::backend::Backend` (ratatui **0.30**); `Receiver` = `std::sync::mpsc::Receiver`. `run_in_compositor` delegates to `run_in_compositor_with_config(backend, input_rx, None)`.

tokscale (`crates/tokscale-cli/src/tui/compositor.rs`):

```rust
pub fn run_in_compositor<B: Backend>(
    backend: B,
    input_rx: Receiver<CompositorInput>,
    tick_ms: u64,
) -> Result<()>
```

Here `Backend` = `ratatui::backend::Backend` (ratatui **0.29**); `Result` = `anyhow::Result`. The `tick_ms` argument is hard-coded `100` at both call sites (real path and headless); do not change it.

**Rationale / prevents:** `monitor/src/main.rs` calls these by exact shape (custom backend types, channel types, `Option<PathBuf>`). A drifted signature fails to compile or silently binds the wrong overload. Re-derive the loop *bodies* from current upstream (see `harness/30-drift-runbook.md`), but keep the *signatures* as written.

---

## I4 — unicode-width is `=0.2.0` tree-wide; openssl-sys absent

**Assert:**
- The entire dependency tree resolves to **unicode-width 0.2.0** (ratatui 0.29 hard-pins `=0.2.0`). This holds via: bottom's direct dep lowered `"0.2.2"` → `"0.2.0"`; `vendor/unicode-ellipsis/Cargo.toml` carrying `[dependencies.unicode-width] version = "0.2.0", default-features = false`; and root `[patch.crates-io] unicode-ellipsis = { path = "vendor/unicode-ellipsis" }`. API is identical across 0.2.0..0.2.2.
- **openssl-sys / openssl-src do not appear** in the resolved tree. tokscale's `reqwest` uses `rustls-tls` (the TLS reconciliation), not `native-tls-vendored`, so no openssl/perl build dependency is pulled.

Verify both:

```sh
cargo tree -p monitor -i unicode-width      # expect a single 0.2.0 node
cargo tree -p monitor -i openssl-sys        # expect: nothing / not found
```

**Rationale / prevents:** Mixed unicode-width majors make ratatui 0.29 refuse to resolve; an `openssl-sys` leak reintroduces the openssl-src/perl build dependency that the `rustls-tls` reconciliation exists to avoid. See `harness/02-versions-and-deps.md` §dep-reconciliation.

---

## I5 — Standalone `cargo build -p bottom` and `-p tokscale` still pass after shimming

**Assert:** After applying the shims, each upstream crate still builds in isolation:

```sh
cargo build -p bottom
cargo build -p tokscale-cli   # tokscale CLI crate
```

The shim files are purely additive plus the minimal whitelisted edits, so neither standalone crate regresses.

**Rationale / prevents:** Proves the shims didn't break upstream's own build — a cheap early signal that I1 was honored and that the new files compile against unmodified upstream APIs.

---

## I6 — Generated artifacts are ephemeral & gitignored; never committed

**Assert:** `monitor/`, `vendor/`, the root `Cargo.toml`, and all shim files are gitignored and are **never** `git add`-ed or committed. If `git status` lists them, that is **expected** — leave them untracked. No commit in this repo (or in the submodules) introduces them.

**Rationale / prevents:** The integration regenerates on demand; committing it forks the canonical spec and rots. "If git status shows them, that is expected — never git add them."

---

## I7 — Work only inside the gitignored workspace; never modify `.attic/`

**Assert:** All generated/edited files live in the live workspace (`monitor/`, `vendor/`, `bottom/`, `tokscale/`, root `Cargo.toml`). `.attic/` is **read-only reference** — never write to it. The set of editable files is the closed whitelist defined per target in `harness/10`–`harness/13`; nothing outside that whitelist is touched.

**Rationale / prevents:** `.attic/` is the previous generation's reference, consulted for **shape** (module layout, signatures, variant coverage) — *not bytes* to copy, since it may lag current upstream. Mutating it destroys the reference and tempts byte-copying stale loop bodies.

---

## I8 — tokscale `CompositorInput` has **no** `Paste` variant — do not add one

**Assert:** tokscale's `CompositorInput` has exactly the variants `Key(KeyEvent)`, `Mouse(MouseEvent)`, `Resize`, `Terminate` (crossterm 0.28 events) — **no `Paste`**. bottom's `CompositorInput` *does* have `Paste(String)` (plus `Key`, `Mouse`, `Resize`, `Terminate`, crossterm 0.29 events). In `monitor/src/main.rs`, `Event::Paste(s)` is forwarded **only** to bottom:

```rust
// tokscale's CompositorInput has no Paste; forward only to bottom.
sys_tx.send(CompositorInput::Paste(s))
```

**Rationale / prevents:** Adding a `Paste` variant to tokscale is an unrequested edit to upstream-shaped code and breaks the asymmetric routing the spec depends on. See the Paste-asymmetry note in `harness/11-target-tokscale-shim.md`.

---

## I9 — Do not upgrade, port, refactor, or "improve" anything

**Assert:** No dependency version is bumped, no render/widget code is ported across ratatui majors, no upstream code is refactored "for cleanliness." The versions in `monitor/Cargo.toml` are EXACT (`ratatui-core "0.1.0"`, `crossterm "0.29.0"`, `ratatui029 = ratatui "0.29"`, `crossterm028 = crossterm "0.28"`, `anyhow "1"`, path deps for `bottom` and `tokscale-cli`) — see `harness/02-versions-and-deps.md`. The two-ratatui / two-crossterm split is **intentional and load-bearing**, not a migration to finish.

**Rationale / prevents:** Any "helpful" upgrade collapses the dual-major design, re-pins unicode-width off `0.2.0`, or rewrites code that must stay pristine — defeating the determinism contract.

---

## How to verify render code is untouched

The submodules must show **only** the whitelisted shim files as changed:

```sh
git -C bottom   status --porcelain
git -C tokscale status --porcelain
```

Expected (and **only**) entries:

- **bottom:** `src/compositor.rs` (new), `src/lib.rs` (modified), `Cargo.toml` (modified).
- **tokscale:** `crates/tokscale-cli/src/lib.rs` (new), `crates/tokscale-cli/src/client_filter.rs` (new), `crates/tokscale-cli/src/tui/compositor.rs` (new), `crates/tokscale-cli/src/main.rs` (modified), `crates/tokscale-cli/src/tui/mod.rs` (modified), `tokscale/Cargo.toml` (modified).

Any **other** path in that output means a render/widget/state/collection file was edited — a hard I1 failure. Revert it. For the exact line-level content of each whitelisted edit, consult the shim contracts in `harness/10`–`harness/11` and the `.attic` reference at `.attic/{bottom,tokscale}/`.
