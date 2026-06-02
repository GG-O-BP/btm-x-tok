# btm-x-tok

> **One TUI that composes `btm` × `tok`: bottom (system monitor) + tokscale
> (AI token & cost) — built from a spec, not checked in. A coding agent generates
> the integration on demand.**

`btm-x-tok` cross-wires two existing terminal tools into a single dashboard:

- **[bottom]** (`btm`) — a cross-platform **system** monitor: CPU, memory,
  network, processes, disk, temperatures, battery.
- **[tokscale]** (`tok`) — an **AI-agent token & cost** monitor: usage across
  Claude, Codex, Copilot, and other coding agents.

The result is one terminal pane where you watch **the machine and the model at
the same time** — hardware health beside what your AI agents are spending.

But the interesting part isn't the tool — it's **how this repository ships it.**

## Spec-as-source

`btm-x-tok` is a practical experiment in **spec-as-source** — a term from the
emerging (2025–) [spec-driven-development][sdd] discourse for its most aggressive
rung: the **specification is the single source of truth, and the implementation
code is a derived, regenerable artifact** an AI agent produces — you edit the spec,
never the code. Martin Fowler frames the ladder as *spec-first → spec-anchored →
spec-as-source*; GitHub's [Spec Kit][speckit] puts it as moving from "code is the
source of truth" to "intent is the source of truth." (The term is new and has no
settled single origin.)

Here's the honest nuance: most spec-driven tools (Spec Kit, Kiro, Tessl) still
**commit** the generated code — they just mark it `// GENERATED FROM SPEC — DO NOT
EDIT`. **This repo takes spec-as-source to its limit: it commits no integration
code at all.** What is version-controlled is the spec ([`AGENTS.md`](AGENTS.md))
plus links to the two upstreams; an agent reads the spec, **generates** the
integration locally, you run it, and it is **discarded**. That "never committed"
stance is *this repo's* stricter choice, not the SDD norm — and it is enforced
structurally by an allowlist [`.gitignore`](.gitignore) and a
[CI guard](.github/workflows/spec-guard.yml).

|                    | Conventional | Spec-driven dev (Spec Kit / Kiro / Tessl) | This repo — spec-as-source, to the limit    |
| ------------------ | ------------ | ----------------------------------------- | ------------------------------------------- |
| Source of truth    | the code     | the spec                                  | **the spec only**                           |
| Generated code     | —            | **committed**, marked read-only           | **never committed** (ephemeral, gitignored) |
| To change behavior | edit code    | edit spec → regenerate → commit           | **edit spec → regenerate**                  |

> Like a `Dockerfile` or `Makefile` made the *recipe* the deliverable — except the
> "compiler" is an LLM, so it is **non-deterministic**: re-derivation reproduces the
> *behavior*, not the exact bytes.

What this buys — and what it costs:

- **Stays current, as long as the spec is maintained.** The integration is
  re-derived against whatever the upstreams are *now*, instead of rotting against a
  vendored snapshot. The catch: that only holds if [`AGENTS.md`](AGENTS.md) is kept
  in sync as the upstreams move, and a capable model is used.
- **Nothing to vendor or patch downstream.** No glue to keep in a fork.
- **The intent is the artifact.** The durable part — *how* the two tools fit
  together — is written down; the mechanical part is regenerated.
- **No license/maintenance burden of a fork.** The repo redistributes no upstream
  code; it only links to it.

## Design

```
   .claude/  .cursor/  .gemini/  .github/  .aider.conf.yml   ← each agent's NATIVE config
        └──────────────┬───────────────────────┘               (a thin "/regen" wrapper)
                       │ all invoke the same harness
              ┌────────▼─────────┐
              │    AGENTS.md     │   ← THE single source of truth (the full spec)
              └────────┬─────────┘
       reads the spec  │  + the two upstream submodules (current source)
              ┌────────▼─────────┐
              │  coding agent    │   ← Claude Code / Codex / Cursor / Copilot / Gemini / Aider …
              └────────┬─────────┘
                       │ generates into a GITIGNORED local workspace
              ┌────────▼─────────┐
              │ unified monitor  │   ← monitor/ + shims, built & run locally,
              └──────────────────┘      then discarded. Never committed.
```

- **[`AGENTS.md`](AGENTS.md) is the one source of truth.** It is the open
  [AGENTS.md](https://agents.md) standard, read natively by most coding agents. It
  contains the entire harness: architecture, invariants, exact versions, per-target
  contracts, build order, and acceptance gates. Every other config file only
  *points to* or *invokes* it — content lives in one place, so it cannot drift.
- **The two upstreams are git submodules** (`bottom/`, `tokscale/`), pinned to
  upstream and left pristine. The agent adds shim files to them *locally*; those
  edits are never committed.
- **Generation is ephemeral.** The generated `monitor/` crate, vendored patches,
  and the workspace manifest all live in **gitignored** paths. An allowlist
  [`.gitignore`](.gitignore) makes this structural — only the spec, the submodules,
  and agent config are publishable; integration code *cannot* be committed.
- **A CI guard** ([`.github/workflows/spec-guard.yml`](.github/workflows/spec-guard.yml))
  fails the build if any integration code is ever tracked or if an agent file stops
  pointing at `AGENTS.md`.

## Repository layout

| Path | Tracked? | What |
| ---- | -------- | ---- |
| `AGENTS.md` | ✅ | the harness specification — the single source of truth |
| `CLAUDE.md`, `GEMINI.md` | ✅ | thin bridges that import/point to `AGENTS.md` |
| `.claude/`, `.cursor/`, `.gemini/`, `.github/`, `.aider.conf.yml` | ✅ | each agent's **native** config + a `/regen` harness |
| `bottom/`, `tokscale/` | ✅ | the two upstreams, as git submodules |
| `.github/workflows/spec-guard.yml` | ✅ | enforces "spec only; code never committed" |
| `monitor/`, `vendor/`, `Cargo.*` | 🚫 gitignored | the **generated** integration — ephemeral, local-only |

## Get the repo

```sh
git clone --recursive https://github.com/GG-O-BP/btm-x-tok.git
# already cloned without --recursive?
git submodule update --init --recursive
# advance the upstreams to their latest commits whenever you like:
git submodule update --remote
```

## Generate & run — per coding agent

Open the repo in your agent and trigger the harness. Each agent uses its **own
native mechanism**, but they all do the same thing: *read `AGENTS.md` → generate
the monitor into the gitignored workspace → run the acceptance gates → never
commit.* (Regenerating ~2k LOC is a real task; use a capable model.)

| Agent | Setup (already in the repo) | Run it |
| ----- | --------------------------- | ------ |
| **Claude Code** | `.claude/commands/regen.md`, `.claude/skills/regenerate-monitor/`, `.claude/agents/monitor-generator.md`, `CLAUDE.md` (`@AGENTS.md`) | `/regen` (or `/regen --battery`); or just ask — the **regenerate-monitor** skill auto-triggers |
| **OpenAI Codex** | reads `AGENTS.md` natively | "Regenerate the monitor per AGENTS.md." |
| **Cursor** | `.cursor/rules/agents.mdc` (always-on) | ask in Composer/Agent: "regenerate per AGENTS.md" |
| **GitHub Copilot** | `.github/prompts/regen.prompt.md`, `.github/copilot-instructions.md` | run the `/regen` prompt in Copilot Chat |
| **Gemini CLI** | `.gemini/commands/regen.toml`, `GEMINI.md` | `/regen` |
| **Aider** | `.aider.conf.yml` (`read: [AGENTS.md]`) | ask: "regenerate the monitor per AGENTS.md" |
| **Windsurf / Zed / OpenCode / Jules / Factory / Amp …** | read `AGENTS.md` natively | ask them to follow `AGENTS.md` |

Once generated, run and drive the unified monitor:

```sh
cargo run -p monitor            # F1 = System (bottom), F2 = Tokens (tokscale), q = quit
cargo run -p monitor -- --battery
```

The harness also runs headless acceptance checks (e.g. `MONITOR_HEADLESS=1`,
`MONITOR_HEADLESS_TOK=1`) — see the **Build & acceptance gates** section of
[`AGENTS.md`](AGENTS.md). Nothing it produces is committed; `git status` listing
`monitor/` etc. is expected — never `git add` them.

## The upstreams

| Tag   | Project                       | What it watches                                        | License |
| ----- | ----------------------------- | ------------------------------------------------------ | ------- |
| `btm` | **[bottom]** by Clement Tsang | system: CPU / mem / net / proc / disk / temp / battery | MIT     |
| `tok` | **[tokscale]** by Junho Yeo   | AI agents: token usage & cost                          | MIT     |

Please support the upstream projects — `btm-x-tok` is nothing without them. See
[NOTICE](NOTICE) for full attributions.

## License

MIT © 2026 GG-O-BP. See [LICENSE](LICENSE).

`bottom` and `tokscale` remain under their own MIT licenses; their notices travel
in [NOTICE](NOTICE).

[bottom]: https://github.com/ClementTsang/bottom
[tokscale]: https://github.com/junhoyeo/tokscale
[sdd]: https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html
[speckit]: https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/
