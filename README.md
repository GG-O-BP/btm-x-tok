# btm-x-tok

> **One TUI that composes `btm` × `tok`: bottom (system monitor) + tokscale
> (AI token & cost) — built from a spec, not checked in. The LLM generates the
> integration on demand.**

`btm-x-tok` cross-wires two existing terminal tools into a single dashboard:

- **[bottom]** (`btm`) — a cross-platform **system** monitor: CPU, memory,
  network, processes, disk, temperatures, battery.
- **[tokscale]** (`tok`) — an **AI-agent token & cost** monitor: usage across
  Claude, Codex, Copilot, and other coding agents.

The result is one terminal pane where you watch **the machine and the model at
the same time** — hardware health beside what your AI agents are spending.

## A specification, not an implementation

`btm-x-tok` is not a program you clone and build. Its source of truth is a
**harness specification** — a precise description of *how* the two upstream tools
are composed onto one screen.

The composition itself is **generated on demand by an LLM** (e.g. Claude Code)
directly from that specification, and is deliberately **never committed**. The
spec is the source of truth; the running tool is an artifact the model re-derives
each time. Change the spec, regenerate — there is nothing to keep in sync in
between.

> The way a `Dockerfile` or a `Makefile` made the *recipe* the deliverable, here
> the recipe is a harness specification and the "compiler" is an LLM.

## How it works

```
              bottom (btm)        tokscale (tok)      ← two existing upstream tools
                    └──────────────────┘
                             │
                   ┌─────────▼──────────┐
                   │  harness spec      │             ← the source of truth
                   └─────────┬──────────┘
                             │   read by an LLM
                   ┌─────────▼──────────┐
                   │  unified monitor   │             ← generated on demand,
                   └────────────────────┘                ephemeral, never committed
```

Hand the harness to an LLM coding agent and ask it to realize the composition for
your platform. The agent reads the specification and the two upstream tools,
produces the unified monitor locally, and you run it — then discard it. Nothing
it produces is stored here.

## Why a spec instead of an implementation

- **Stays current for free.** The integration is re-derived against whatever the
  upstreams are *now*, instead of drifting from them release after release.
- **Nothing to maintain downstream.** There is no glue to keep patched and in
  sync as the upstreams evolve.
- **The intent is the artifact.** The durable, interesting part — *how* the two
  tools should fit together — is what gets written down; the mechanical part is
  regenerated.

## Cloning

`bottom/` and `tokscale/` are tracked as **git submodules** pointing at their
upstreams — this repository carries no source for them; it is fetched from the
upstreams on clone:

```sh
git clone --recursive https://github.com/GG-O-BP/btm-x-tok.git
# already cloned without --recursive?
git submodule update --init --recursive
```

Advance the upstreams to their latest commits whenever you like:

```sh
git submodule update --remote
```

## The upstreams

Both are wired in as submodules — cloning `btm-x-tok` (with `--recursive`) brings
them in:

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
