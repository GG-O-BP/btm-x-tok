# 02 — Versions & Dependency Reconciliation

Normative version pins and the dependency-conflict resolution for the unified
monitor build. Every pin below is **EXACT — do not bump, port, or "modernize".**
The whole point of this integration is that two ratatui majors (0.30 from bottom,
0.29 from tokscale) coexist in one process. See `harness/01-invariants.md` for the
hard gates and `harness/00-overview.md` for the architecture.

The agent ADDS shim files locally; it does NOT upgrade dependencies in `bottom/`
or `tokscale/` beyond the two reconciliation edits called out here.

---

## Version lock-table (EXACT)

### bottom (`bottom/`, pristine upstream + one Cargo edit)

| Item | Value | Notes |
|---|---|---|
| crate version | `0.12.3` | EXACT |
| edition | `2024` | EXACT |
| `tui` (= ratatui) | `0.30.0`, `package = "ratatui"` | features `["unstable-rendered-line-info", "layout-cache", "crossterm"]` |
| `ratatui-core` | `0.1.0` | bottom's render-core trait surface |
| `crossterm` | `0.29.0` | bottom's terminal events |
| MSRV | `1.86` | EXACT |

### tokscale (`tokscale/`, pristine upstream + reconciliation edits)

| Item | Value | Notes |
|---|---|---|
| workspace version | `3.0.0` | EXACT |
| edition | `2021` | EXACT |
| `ratatui` | `0.29`, features `["all-widgets"]` | hard-pins `unicode-width = "=0.2.0"` |
| `crossterm` | `0.28`, features `["event-stream"]` | EXACT |
| tokio | present (async runtime) | consult `.attic/tokscale/Cargo.toml` for the exact spec |

### monitor (`monitor/`, generated crate — the linker)

| Dependency | Spec | Why |
|---|---|---|
| `bottom` | `{ path = "../bottom" }` | renders into our 0.30 backend |
| `ratatui-core` | `"0.1.0"` | EXACT — unify with bottom's render-core trait |
| `crossterm` | `"0.29.0"` | EXACT — single real-terminal owner, same major as bottom |
| `ratatui029` | `{ package = "ratatui", version = "0.29" }` | EXACT — tokscale's render major |
| `crossterm028` | `{ package = "crossterm", version = "0.28" }` | EXACT — build the events tokscale handlers expect |
| `tokscale-cli` | `{ path = "../tokscale/crates/tokscale-cli" }` | tokscale's TUI, in-process |
| `anyhow` | `"1"` | error type for compositor entry points |

**Tree-wide pin:** `unicode-width = "0.2.0"` everywhere (see the conflict section
below). No other unicode-width version may appear in `cargo tree`.

---

## VERBATIM `monitor/Cargo.toml` dependency block

This block is **load-bearing**: the `package = "..."` renames (`ratatui029`,
`crossterm028`) are what let two majors of each crate link into one binary. Match
exactly — do not collapse the renames or drop the comments.

```toml
[dependencies]
# bottom's library, consumed as a path dependency. Renders into our custom backend.
bottom = { path = "../bottom" }
# Must unify with bottom's ratatui-core 0.1 so our Backend impl matches the trait
# bottom's `tui::Terminal` expects.
ratatui-core = "0.1.0"
# Single real-terminal owner. Same major as bottom's crossterm (0.29).
crossterm = "0.29.0"
# tokscale renders with ratatui 0.29; its custom backend lives here.
ratatui029 = { package = "ratatui", version = "0.29" }
# To build the crossterm-0.28-typed events tokscale's handlers expect.
crossterm028 = { package = "crossterm", version = "0.28" }
# tokscale's TUI, embedded in-process (lib facade added to tokscale-cli).
tokscale-cli = { path = "../tokscale/crates/tokscale-cli" }
anyhow = "1"
```

The package header is `name = "monitor"`, `version = "0.0.0"`, `edition = "2021"`,
`publish = false`, with `[[bin]] name = "monitor", path = "src/main.rs"`. The full
file is the live reference at `monitor/Cargo.toml`.

In source, the renamed crates are referenced as `ratatui029::...` /
`crossterm028::...`; bottom's majors are the plain `ratatui-core::...` (via
`ratatui_core`) / `crossterm::...`. The crossterm 0.29→0.28 event translation
lives in `monitor/src/bridge.rs` (see `harness/12-target-monitor-crate.md`).

---

## The `unicode-width` conflict (the one real reconciliation)

**Conflict.** ratatui 0.29 (tokscale's render major) **hard-pins
`unicode-width = "=0.2.0"`**. Meanwhile bottom's direct dependency and
`unicode-ellipsis 0.4.0` (pulled in by bottom) both want `unicode-width >= "0.2.2"`.
A single Cargo resolution cannot satisfy `=0.2.0` and `>=0.2.2` at once. The APIs
are identical across `0.2.0..0.2.2`, so the fix is to lower both floors to `0.2.0`.

**Resolution — two edits, both NORMATIVE:**

**(a) Lower bottom's direct pin.** This is the *only* edit to `bottom/Cargo.toml`.
Change line 104 from the pristine value to `0.2.0` and add the reconciliation
comment. VERBATIM target (from `.attic/bottom/Cargo.toml` lines 104–107):

```toml
# Lowered from "0.2.2" to "0.2.0" so the whole workspace can pin unicode-width
# 0.2.0, which ratatui 0.29 (tokscale's TUI) requires exactly (=0.2.0). API is
# identical across 0.2.0..0.2.2. Reconciliation point for the unified monitor.
unicode-width = "0.2.0"
```

Pristine/live value (`bottom/Cargo.toml` line 104) is `unicode-width = "0.2.2"`
with no comment block. This is reconciliation point **(a)** in
`harness/30-drift-runbook.md` — it must survive every upstream pull.

**(b) Vendor + patch `unicode-ellipsis`.** Vendor `unicode-ellipsis 0.4.0` at
`vendor/unicode-ellipsis/` with its `unicode-width` floor lowered to `0.2.0`, then
patch crates.io to the path copy in the root workspace `Cargo.toml`.

Vendored `vendor/unicode-ellipsis/Cargo.toml` dependency block (VERBATIM):

```toml
[dependencies.unicode-width]
version = "0.2.0"
default-features = false
```

(Vendored package stays `unicode-ellipsis` v`0.4.0`, edition `2021`, with
`[dependencies.unicode-segmentation] version = "1.13.2"` and
`[features] default = ["fish"]`, `fish = []`. The copy differs from crates.io
**only** in the unicode-width floor.) Build details for the vendor dir are in
`harness/13-target-workspace-vendor.md`.

Root workspace `Cargo.toml` patch (VERBATIM):

```toml
[patch.crates-io]
unicode-ellipsis = { path = "vendor/unicode-ellipsis" }
```

Because `unicode-ellipsis` is third-party, vendoring it adds **no churn** to the
bottom/tokscale upstream merges.

**Standalone bottom still builds.** Lowering bottom's floor to `0.2.0` is
API-compatible, so `bottom/` continues to build on its own — the edit does not
fork bottom's behavior, it only relaxes a version floor.

---

## tokscale TLS — REACTIVE only

Pristine upstream tokscale uses `native-tls-vendored`, which drags in
`openssl-src` and requires a working `perl` + C toolchain at build time. Some
build environments lack that. The shim *can* switch to pure-Rust `rustls-tls`,
but **only switch reactively** — do not pre-emptively edit if the default build
succeeds.

**How to detect you need the switch:** `cargo build -p monitor` fails with an
error referencing `openssl-sys`, `openssl-src`, or a missing `perl` during the
openssl build. Only then apply the edit.

**Where.** `tokscale/Cargo.toml` line 43.

Pristine/live (line 43):

```toml
reqwest = { version = "0.12", features = ["json", "native-tls-vendored"], default-features = false }
```

Reactive shim value (from `.attic/tokscale/Cargo.toml`, with the 4-line comment):

```toml
# Reconciliation for the unified monitor build: switched TLS from
# native-tls-vendored to rustls-tls (pure Rust). Avoids the openssl-src/perl
# build dependency (which some build envs lack) and is better for the
# cross-compilation native-tls-vendored was originally chosen for.
reqwest = { version = "0.12", features = ["json", "rustls-tls"], default-features = false }
```

This is reconciliation point **(c)** in `harness/30-drift-runbook.md`: if you
applied it, it must stay `rustls-tls` across upstream pulls.

---

## Acceptance — what `cargo tree` must show

After `cargo build -p monitor` succeeds, the dependency graph must satisfy ALL of:

- **ratatui:** both `0.29` AND `0.30` present (two majors, by design).
- **crossterm:** both `0.28` AND `0.29` present (two majors, by design).
- **unicode-width:** `0.2.0` ONLY — no `0.2.1`/`0.2.2` anywhere in the tree.
- **openssl-sys:** ABSENT only if the `rustls-tls` switch was applied; if you are
  on the pristine `native-tls-vendored` path, `openssl-sys` is expected. The
  acceptance target for an openssl-less environment is **no `openssl-sys`**.

Suggested checks (read-only; do not `git add` any generated paths):

```sh
cargo tree -p monitor -i unicode-width      # one version line: 0.2.0
cargo tree -p monitor -i ratatui            # 0.29 and 0.30
cargo tree -p monitor -i crossterm          # 0.28 and 0.29
cargo tree -p monitor | grep -i openssl-sys # empty when rustls-tls applied
```

Full ordered build + acceptance commands live in
`harness/20-build-and-acceptance.md`. The canonical entry point is `AGENTS.md`.
