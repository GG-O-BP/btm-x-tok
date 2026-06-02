# harness/13 — Target: Workspace + Vendored Patch + Scripts

The pieces that bind `monitor/`, `bottom/`, and `tokscale/` into one buildable tree:
the **root `Cargo.toml`** (workspace + dependency-reconciliation patch), the
**`vendor/unicode-ellipsis`** patched crate, and **`scripts/update-upstream.sh`**.

> **All of these live in the GITIGNORED workspace and are NEVER committed.** The
> root `Cargo.toml`, `vendor/`, `monitor/`, and the shim files are generated
> ephemerally. If `git status` shows them, that is expected — **never `git add`
> them.** See harness/01-invariants.md.

Cross-references: harness/02-versions-and-deps.md (the two-ratatui-major lock
table and the unicode-width reconciliation), harness/10-target-bottom-shim.md and
harness/11-target-tokscale-shim.md (the per-submodule shims this workspace
consumes), harness/30-drift-runbook.md (the upstream-pull runbook that drives the
script below). Canonical entry: AGENTS.md.

---

## 1. Root `Cargo.toml` (gitignored: `/home/superset/btm-toks/Cargo.toml`)

Write this file **VERBATIM**. `resolver`, `members`, `exclude`, and the
`[patch.crates-io]` entry are all NORMATIVE — match exactly; do not add members,
do not drop the excludes, do not change the patch path.

```toml
# Unified monitoring tool workspace.
# bottom/ and tokscale/ are vendored as squashed git SUBTREES (remotes
# bottom-upstream / tokscale-upstream) and kept OUT of the workspace (excluded),
# consumed as path dependencies. Each subtree is a pristine upstream import with
# our changes isolated to a single "<name>: monitor integration shim" commit on
# top. Pull new releases with `scripts/update-upstream.sh` (wraps `git subtree
# pull --squash`); conflicts are confined to the shimmed files.
[workspace]
resolver = "3"
members = ["monitor"]
exclude = ["bottom", "tokscale", "vendor/unicode-ellipsis"]

# Dependency reconciliation for the two-ratatui-major build (bottom=0.30,
# tokscale=0.29): ratatui 0.29 hard-pins unicode-width "=0.2.0", while
# unicode-ellipsis 0.4.0 (a bottom dep) pinned ">=0.2.2". We vendor
# unicode-ellipsis with its unicode-width floor lowered to 0.2.0 (API-identical)
# so the whole tree unifies on unicode-width 0.2.0. This crate is third-party, so
# it does not add churn to bottom/tokscale upstream merges.
[patch.crates-io]
unicode-ellipsis = { path = "vendor/unicode-ellipsis" }
```

Why each line is load-bearing:

- **`resolver = "3"`** — required; do not downgrade.
- **`members = ["monitor"]`** — only the `monitor` crate is a workspace member.
  `bottom` and `tokscale` are consumed strictly as **path dependencies** from
  `monitor/Cargo.toml` (see harness/12-target-monitor-crate.md), not as members.
- **`exclude = ["bottom", "tokscale", "vendor/unicode-ellipsis"]`** — the two
  submodules and the vendored crate must stay OUT of the workspace so their own
  `[workspace]`/lock semantics and pinned deps do not collide with the unified
  build. Excluding `vendor/unicode-ellipsis` keeps the patch target from being
  pulled in as a member.
- **`[patch.crates-io] unicode-ellipsis = { path = "vendor/unicode-ellipsis" }`**
  — redirects every crates.io `unicode-ellipsis` reference (it is a transitive
  bottom dep) to the patched copy in §2. This is the reconciliation point that
  lets the whole tree unify on `unicode-width 0.2.0`. Do not change the path; do
  not patch any other crate here.

---

## 2. `vendor/unicode-ellipsis/` — the patched crate

A **vendored copy of `unicode-ellipsis` 0.4.0** with **exactly one change**: the
`unicode-width` floor is lowered from `0.2.2` to `0.2.0`. This is API-identical
across `0.2.0..0.2.2` — **no logic change, no source edits.** Being third-party,
this vendored crate adds no churn to bottom/tokscale upstream merges.

Obtain the pristine `unicode-ellipsis 0.4.0` source (e.g. from crates.io / the
crate's repo), place it under `vendor/unicode-ellipsis/`, then make the single
Cargo.toml edit below.

### 2a. The `unicode-width` dependency block (VERBATIM)

In `vendor/unicode-ellipsis/Cargo.toml`, the `unicode-width` dependency must read
exactly:

```toml
[dependencies.unicode-width]
version = "0.2.0"
default-features = false
```

### 2b. Everything else stays pristine

From the facts sheet, the vendored manifest is otherwise the upstream 0.4.0
manifest:

- Package: `unicode-ellipsis`, version `0.4.0`, edition `2021`.
- `[features]`: `default = ["fish"]`, `fish = []`.
- `[dependencies.unicode-segmentation]`: `version = "1.13.2"`.

Do not bump the package version, change edition, alter features, or touch the
`unicode-segmentation` pin. The **only** delta from pristine 0.4.0 is the
`unicode-width` `version` floor (`0.2.0`) and `default-features = false` shown in
§2a. For any manifest detail not listed here, consult the pristine
`unicode-ellipsis 0.4.0` upstream manifest (and `.attic` for the exact vendored
copy) — do not invent.

> **Reconciliation context:** `ratatui 0.29` (tokscale's TUI) hard-pins
> `unicode-width = "=0.2.0"`; `unicode-ellipsis 0.4.0` originally pinned
> `>= 0.2.2`. Together with lowering bottom's own direct `unicode-width` pin to
> `"0.2.0"` (see harness/10-target-bottom-shim.md and harness/02-versions-and-deps.md),
> this patch lets the unified tree resolve `unicode-width 0.2.0` everywhere. The
> full reconciliation is owned by harness/02-versions-and-deps.md.

---

## 3. `scripts/update-upstream.sh` — exists; not regenerated here

`scripts/update-upstream.sh` **exists** and is the upstream-pull driver. The spec
does **NOT** regenerate it — the **drift runbook (harness/30-drift-runbook.md)
references and explains it**. This file only states what it is and what it does.

### 3a. What it does

- Pulls new upstream releases into the `bottom/` and `tokscale/` subtrees via
  `git subtree pull --squash`, then re-merges the shims, then rebuilds.
- Configured remotes:
  - `bottom-upstream` = `https://github.com/ClementTsang/bottom.git`
  - `tokscale-upstream` = `https://github.com/junhoyeo/tokscale.git`
- Pull commands (squashed subtree pulls):
  - `git subtree pull --prefix=bottom --squash bottom-upstream main -m "vendor: pull bottom upstream"`
  - `git subtree pull --prefix=tokscale --squash tokscale-upstream main -m "vendor: pull tokscale upstream"`
- Bootstraps the `git-subtree` contrib script if it is missing.
- After pulling, rebuilds to verify the merge:

  ```sh
  cargo build -p monitor
  ```

  Progress line: `Building monitor to verify the merge...`; success line:
  `OK: upstream pulled and monitor still builds. Review 'git log' and push.`

### 3b. Where conflicts land (the script's reconciliation-points comment)

The script documents that merge conflicts are confined to the shimmed files
(verbatim list, lines 44–50):

- `bottom/Cargo.toml` — unicode-width must stay `"0.2.0"` (ratatui 0.29 pins `=0.2.0`)
- `bottom/src/lib.rs` — keep `pub(crate) mod compositor;` + re-exports + `create_collection_thread` pub(crate)
- `tokscale/Cargo.toml` — reqwest feature must stay `rustls-tls` (not `native-tls-vendored`)
- `tokscale/.../main.rs` — ClientFilter lives in `client_filter.rs` (re-apply the move if upstream edits the enum)
- `tokscale/.../tui/mod.rs` — keep `pub mod compositor;` + re-exports
- (new files `compositor.rs` / `client_filter.rs` / `lib.rs` never conflict)

The reconciliation handling for each of these is owned by
harness/30-drift-runbook.md (with the per-shim contracts in
harness/10-target-bottom-shim.md and harness/11-target-tokscale-shim.md). For the
exact script body, consult the reference at `.attic` — do not regenerate it from
this spec.

---

## 4. Gitignore / commit invariant (recap)

The root `Cargo.toml`, `vendor/unicode-ellipsis/`, `monitor/`, and all shim files
are part of the **ephemeral, gitignored workspace**. They are built on demand and
**never committed**. `bottom/` and `tokscale/` are git submodules pinned to
upstream latest; the agent adds shim files to them locally and **never commits to
them**. If `git status` lists any of these generated paths, that is expected —
**never `git add` them.** See harness/01-invariants.md for the full determinism
contract.
