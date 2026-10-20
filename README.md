<div align="center">

# ar1syr0 Hermes

**A four-agent fleet on [Hermes Agent](https://github.com/NousResearch/hermes-agent)**

Four specialised agents · **three models across three providers** · one shared task board
Network-shared workspaces · mobile control

</div>

---

## What this is

A tested build of a multi-agent system where a **manager** agent decomposes goals and
delegates to three specialists — **analyst**, **engineer** and **ops** — coordinated
through a durable task board.

**Each agent runs a different model, from a different provider.** A capable frontier
model on a subscription handles code. A frugal flash model through an aggregator does
research and orchestration. A free-tier model direct from its vendor does mechanical
file work. Three models, three base URLs, three credentials, three billing models —
routed per role so a file rename costs nothing and the expensive model is reserved for
work that benefits from it.

```
Telegram ─┐
Dashboard ─┼──► Kanban board ──► dispatcher ──► worker process per card
CLI ───────┘                                    (spawned as the assignee profile)

      manager        analyst        engineer         ops
      plans          research       code             files
      routes         + web          + tests          + shell
      no hands       no shell       full access      no web
      ─────────      ─────────      ──────────       ─────────
      frugal         frugal         capable          free tier
      aggregator     aggregator     subscription     direct API
```

Role separation is enforced by **capability, not instruction**: the manager has no
terminal and no file tools, so delegation is its only mechanism.

---

> **A note on naming.** Specific model and provider names are **deliberately omitted**
> throughout this repository. Models are described by class (frugal flash, capable
> frontier, free tier) and providers by type (aggregator, subscription, direct API).
>
> Which model is "the capable one" changes every few months; the argument for putting
> it on the code lane does not. Worse, a stale model id here would produce a 404 that
> Hermes treats as **terminal** — blocking the card after one attempt — so a guide full
> of hardcoded ids would actively mislead later readers. Run `hermes model` on your own
> install and use the exact strings it gives you.

---

## Start here

| | |
|:--|:--|
| **[SETUP.md](SETUP.md)** | The build guide. Twelve steps, every command verified, with twelve field notes on what cost real time. |
| **[HANDOVER.md](HANDOVER.md)** | Self-contained state brief — paste into a fresh session or hand to another engineer. |

The `docs/` directory holds the design reasoning behind the choices: why profiles
instead of subagents, where routing can physically happen, what a decision model adds.
Read SETUP first; read these when you want to know why.

---

## What you get

- **Four agents that delegate to each other**, with dependency gating so a child card
  waits for its parent and inherits its completion summary
- **Cross-provider model routing** — three providers with separate credentials, each
  pinned to the role that needs it, which is the thing anonymous subagents cannot do
- **Automatic routing** of unassigned cards, native to Hermes — no script required
- **Project folders shared over SMB**, one board per project, agents and humans editing
  the same files
- **Mobile control** via Telegram, locked to a single account, working anywhere without
  a VPN or an open port
- **A shadow-mode decision-model router** if you later need routing with calibrated
  confidence you can gate on

---

## Three things worth knowing before you build

> **Three providers means profiles, not subagents.** A routing layer that rewrites only
> the model string cannot cross providers — the base URL and key belong to whoever
> originated the request. Only a profile or a per-task override carries a whole
> credential bundle.

> **Capability is the enforcement.** An orchestrator told "never implement" will still
> try when a task looks small. Removing the toolset is what stops it.

> **Briefing is upstream of routing.** A card phrased "do X" makes the manager attempt
> X. Phrased "decompose X and assign it", it fans out correctly. No classifier fixes a
> badly written card — it will confidently route a vague one to the wrong lane.

> **Try the built-in routing first.** Hermes routes unassigned cards natively. An
> external decision model adds exactly one thing: a confidence number you can gate on.
> Worth it only if some cards should pause rather than be confidently misrouted.

---

## Layout

```
SETUP.md         build guide
HANDOVER.md      state brief
docs/            design reasoning
router/          shadow-mode decision-model router
config/          example configs, root and per profile
scripts/         read-only discovery helper
templates/       routing journal
```

---

<div align="center">

**ar1syr0**

Hermes Agent is MIT (Nous Research). This repository is documentation and glue.

</div>
