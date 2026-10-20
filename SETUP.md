<div align="center">

# ar1syr0 Hermes

### A four-agent fleet on Hermes Agent

**Four specialised agents · three models across three providers · one shared task board**
**Network-shared workspaces · mobile control**

*Built and verified end to end. Every command here was run. Every gotcha was hit.*

</div>

---

## Contents

| | |
|---|---|
| [What this builds](#what-this-builds) | architecture and final state |
| [Prerequisites](#prerequisites) | before you start |
| **[Part 1 · Foundation](#part-1--foundation)** | profiles, models, credentials, toolsets |
| **[Part 2 · The board](#part-2--the-board)** | board, orchestration, first card |
| **[Part 3 · Orchestration](#part-3--orchestration)** | delegation and automatic routing |
| **[Part 4 · Access](#part-4--access)** | network share and mobile control |
| [Field notes](#field-notes) | twelve things that cost real time |
| [Command reference](#command-reference) | everything in one place |

---

## What this builds

```
     Telegram (phone)          Dashboard (LAN)          CLI
             │                        │                  │
             └────────────┬───────────┴──────────────────┘
                          ▼
              ┌───────────────────────┐
              │     Kanban board      │   one SQLite board
              │  triage → ready →     │   auto-routes unassigned cards
              │  running → done       │
              └───────────┬───────────┘
                          │  dispatcher claims · spawns a worker process
      ┌──────────┬────────┴────────┬──────────────┐
      ▼          ▼                 ▼              ▼
 ┌─────────┐ ┌──────────┐  ┌───────────┐  ┌──────────┐
 │ manager │ │ analyst  │  │ engineer  │  │   ops    │
 │  plans  │ │ research │  │   code    │  │  files   │
 │ routes  │ │  + web   │  │  + tests  │  │ + shell  │
 ├─────────┤ ├──────────┤  ├───────────┤  ├──────────┤
 │ frugal  │ │  frugal  │  │  capable  │  │   free   │
 │ model   │ │  model   │  │   model   │  │  model   │
 │no hands │ │ no shell │  │ full      │  │  no web  │
 └─────────┘ └──────────┘  └───────────┘  └──────────┘
                          │
            ~/hermes_projects/  ·  shared over SMB
```

**One card has one assignee.** Multi-agent work comes from *several linked cards*
going to different lanes — not from one card reaching many agents.

### Final state

| Profile | Role | Model class | Provider | Capabilities | Must not have |
|:--|:--|:--|:--|:--|:--|
| **manager** | decomposes goals, assigns specialists | frugal flash | *aggregator* | board, memory | shell, files, code |
| **analyst** | research, sources, written synthesis | frugal flash | *aggregator* | web, files | shell, code |
| **engineer** | code, tests, builds | capable frontier | *subscription* | web, shell, files, code | — |
| **ops** | file management, scripts, logs | free tier | *direct API* | shell, files | web |

### Three models, three providers — and why that matters

This build runs **three different models across three different providers**, each with
its own base URL, its own credential and its own billing model: a frontier model on a
**subscription** (allowance-limited, not metered), a frugal flash model through an
**aggregator** (metered, cheap), and a **free-tier** model direct from its vendor
(quota-limited, no cost).

**That combination is the reason this is built from profiles rather than subagents.**
Hermes resolves a single delegation model for an entire child fleet, so anonymous
subagents cannot each carry a different provider. And a routing layer that rewrites
only the model string **cannot cross providers** — the request's base URL and key
belong to whichever provider originated it. Three providers therefore means three
credential domains, and the only boundaries in Hermes that carry a whole credential
bundle are the **profile** and the **per-task override**.

Assignment is deliberate, and it is about *matching capability to cost*:

| Lane | Model | Reasoning |
|:--|:--|:--|
| **engineer** | the most capable available | Code is where a weak model costs more in rework than a strong one costs in budget |
| **ops** | free tier | Mechanical work, verifiable by exit code. A frontier model adds nothing |
| **analyst** | frugal flash | High-volume text output, tolerant of a cheap model |
| **manager** | frugal flash | Counter-intuitive, and **tested correct** — see [ADR](#adr--the-frugal-orchestrator) |

> **The payoff:** a mechanical file-rename costs nothing, a research sweep costs
> fractions of a cent, and the expensive model is reserved for the work that actually
> benefits from it. One model for everything would either overpay for trivial work or
> underperform on hard work.


---

> ### 📋 A note on naming
>
> **Specific model names, provider names and version ids are deliberately omitted
> throughout this guide.** Models are described by *class* — frugal flash, capable
> frontier, free tier — and providers by *type* — aggregator, subscription, direct API.
>
> This is intentional, for three reasons:
>
> **The reasoning outlives the products.** Which model is "the capable one" changes
> every few months. The argument for putting your most capable model on the code lane
> and a free one on mechanical work does not.
>
> **Named ids rot, and rot dangerously here.** As [Field note 2](#2--model-ids-are-not-what-the-dashboard-shows)
> and [Field note 3](#3--free-tier-and-preview-ids-expire-silently) document, a stale
> model id produces a 404 that Hermes treats as **terminal** — the card blocks after one
> attempt. A guide full of hardcoded ids would actively mislead anyone reading it six
> months from now.
>
> **It keeps the architecture portable.** Nothing here depends on a particular vendor.
> Substitute any three models from any three providers and the design holds — that is
> the point of routing by role rather than by name.
>
> **What you should do:** run `hermes model` on your own install, take the exact strings
> it offers you, and record them somewhere you maintain. Never transcribe a model id
> from a dashboard or from a document like this one.

---

## Prerequisites

- **Hermes Agent** installed, with **one working chat** before you start.
  If `hermes --tui` cannot hold a conversation, fix that first — nothing below will
  work and you will debug the fleet instead of the provider.
- **API access** for each model you intend to use.
- **Linux or macOS.** On systemd:
  `loginctl show-user "$USER" | grep -i linger` must report `Linger=yes`, or worker
  spawns are refused.

> **Conventions.** Replace `<LAN_SUBNET>` with your subnet (e.g. `10.0.0.0/24`),
> `<HOST_IP>` with the host address, and `<USER>` with the account Hermes runs as.
> Angle brackets are never part of a value.

---

# Part 1 · Foundation

## Step 1 — Inventory

```bash
hermes --version
hermes model          # read the exact model ids, then quit without changing anything
```

Write the **exact** model id strings down. Never transcribe from a dashboard — display
names are truncated and namespaced ids get mangled. See [Field note 2](#2--model-ids-are-not-what-the-dashboard-shows).

## Step 2 — Create the four profiles

A profile is a self-contained agent: its own config, model, memory, credentials and
toolsets, under `~/.hermes/profiles/<name>/`.

```bash
hermes profile create manager  --description "Orchestrator. Decomposes a goal into child kanban cards using kanban_create and assigns each to the specialist that can do it: analyst (research, web, writing), engineer (code, tests, builds), ops (files, scripts, logs). You have no terminal and no file tools by design — delegation is your only mechanism. Never attempt work yourself."

hermes profile create analyst  --description "Research and analysis. Web search, gathering and comparing sources, reading documents, written synthesis and summaries. Produces text, not code. No shell access, no code execution."

hermes profile create engineer --description "Software implementation. Writes and modifies code, runs tests, debugs, refactors, works in git worktrees and runs build commands."

hermes profile create ops      --description "Operational tasks. File management, moving and renaming files, running existing scripts, inspecting logs, scheduled maintenance. Mechanical work that is verifiable by exit code."
```

> ### Descriptions are functional, not decorative
> Hermes' decomposer reads the profile roster **and these descriptions** to decide
> where a child card goes. Write them as routing criteria with explicit negatives
> ("Produces text, not code") so no two profiles can plausibly claim the same card.
>
> The manager's description states **how it works**, not only what it must not do.
> A description that says only "never implement" produces an agent that tries to
> implement and then gets stuck — see [Step 9](#step-9--prove-delegation).

**Verify:** `hermes profile list` → `default`, `manager`, `analyst`, `engineer`, `ops`.

## Step 3 — One model per role

```bash
hermes -p analyst model
hermes -p manager model
hermes -p ops model
# engineer keeps the root model — no change needed
```

> **Use the interactive picker. Never type model ids by hand.**
> It writes the exact string, including namespacing.

### Credential each profile separately

The step that costs people an afternoon. **Credentials do not propagate from root** —
see [Field note 1](#1--credentials-are-per-profile-and-env-pointers-dangle).

```bash
hermes -p <profile> auth add <provider>
```

Then **restart the affected gateways** — a running gateway captured its environment at
startup and cannot see a credential added afterwards.

**Verify every profile reaches its own provider.** Non-negotiable: a credential failure
is *terminal* in Hermes — the dispatcher blocks the card after a single attempt, days
later, and you will be debugging the board instead of the key.

```bash
for p in manager analyst engineer ops; do
  echo "=== $p"; hermes -p "$p" --tui    # ask: reply with exactly OK
done
```

## Step 4 — Shape the toolsets

Models gave the roles different brains. Toolsets give them different **hands**.

```bash
hermes -p <profile> tools
```

Enter to accept **Configure CLI**, type each number to toggle, Enter to confirm, then
**Done**.

| Profile | Turn OFF | Turn ON | Final |
|:--|:--|:--:|:--:|
| **manager** | 1 2 3 4 5 6 8 11 18 20 21 23 | 14 | 6 |
| **analyst** | 2 3 5 6 8 11 18 20 21 23 | 14 | 8 |
| **engineer** | 2 6 8 11 18 20 21 23 | 14 | 10 |
| **ops** | 1 2 5 6 8 11 18 20 21 23 | 14 | 8 |

<details>
<summary><b>Toolset reference (CLI platform)</b></summary>

`1` Web Search · `2` Browser · `3` Terminal · `4` File Operations · `5` Code Execution ·
`6` Vision · `7` Video Analysis · `8` Image Gen · `9` Video Gen · `10` X Search ·
`11` TTS · `12` Skills · `13` Task Planning · **`14` Kanban** · `15` Memory ·
`16` Context Engine · `17` Session Search · `18` Connections · `19` Clarify ·
**`20` Task Delegation** · `21` Cron · `22` Yuanbao · **`23` Computer Use** · `24` A2A

</details>

### Three decisions worth understanding

> **Why the manager has no hands.**
> An orchestrator told "never implement" will still, when a task looks small, do it
> itself. Instruction is not enforcement — **removing the capability is.** A manager
> with no terminal and no file toolset has no option but to delegate. One config
> change, zero runtime cost, cannot be prompted around.

> **Why Task Delegation (20) comes off everywhere.**
> `delegate_task` spawns anonymous children sharing one model — no board record, no
> resumability, no audit trail. Disabled, the board becomes the only road.

> **Why Kanban (14) goes on all four.**
> It is opt-in and off by default. Enabling it needlessly costs a few tool schemas; a
> worker that *cannot* call `kanban_complete` leaves every card hanging until it
> auto-blocks. Asymmetric risk.

**Toolsets are per platform.** Enabling for CLI does not enable for Telegram. Changes
require a **new chat** — existing sessions keep their tool schemas.

---

# Part 2 · The board

## Step 5 — Create the board

```bash
hermes kanban init          # NOTE: no -p flag
```

> **The missing `-p` is deliberate.** A profile is a full Hermes home. Running this
> under `-p manager` creates a board the other agents cannot see — the manager would
> file cards into a board nobody reads. **One board at root**; profiles are assignees
> on it.

**Verify:**

```bash
ls ~/.hermes/profiles/*/kanban.db 2>/dev/null || echo "none per-profile — correct"
hermes kanban assignees
hermes gateway restart      # restart if it started before the board existed
```

## Step 6 — Configure orchestration

```bash
hermes config set kanban.orchestrator_profile manager
hermes config set kanban.default_assignee manager
hermes config set kanban.max_in_progress 2
hermes config set kanban.max_in_progress_per_profile 1
hermes config set auxiliary.free_only true
hermes config set dashboard.show_token_analytics true
```

| Setting | Why |
|:--|:--|
| `orchestrator_profile` | manager owns the root card after fan-out |
| `default_assignee: manager` | `default` is a valid assignee with no role. A typo would land a card there silently — this makes mistakes visible |
| `max_in_progress: 2` | rate-limited lanes; don't burst |
| `max_in_progress_per_profile: 1` | stops two cards in one lane tripping each other's 429s |
| `auxiliary.free_only` | restricts auxiliary fallbacks to free models — **see caveat** |

> ⚠️ **`auxiliary.free_only` caveat.** The Kanban decomposer *is* an auxiliary task.
> If no free model is reachable, decomposition may fail outright. **If routing or
> decomposition breaks, revert this first.**

> ℹ️ **There is no `kanban.default_workdir`.** The default workspace is set **per
> board** in the dashboard. See [Field note 5](#5--theres-no-default_workdir-key).

## Step 7 — First card, end to end

```bash
hermes kanban create "Write the current date into hello.txt in the workspace, then list the directory contents" \
  --assignee ops --workspace dir:/absolute/path/to/workspace

hermes kanban dispatch
```

**Ops is the right first lane:** mechanical, verifiable by exit code, free model,
unambiguous success criterion.

```
Spawned: 1
  - t_xxxxxxxx  ->  ops  @ /absolute/path/to/workspace
```

`Spawned: 1` with a claim lock and a pid is the whole chain validating itself —
profiles, models, credentials, toolsets, board, dispatcher and workspace.

<details>
<summary><b>If it fails</b></summary>

```bash
hermes kanban log <id>              # the worker's own log
hermes kanban runs <id>             # attempt history
hermes kanban dispatch --dry-run    # why is a ready card not spawning?
```

Most likely: the worker finished but never called `kanban_complete`, parking the card
as a protocol violation. That is what the Kanban toolset grant prevents.

</details>

---

# Part 3 · Orchestration

## Step 9 — Prove delegation

The fleet is only multi-agent once one agent hands work to another.

### The failure that teaches the most

A card phrased **"Review X and produce Y"** made the manager try to do the work itself.
Finding no file tools, it **blocked rather than fabricating**:

```
blocked {'kind': 'capability', 'reason': "This profile's runtime has no terminal/file
tools, so I cannot read the workspace or write the markdown summary file. Draft summary
is saved as a comment on this card..."}
```

> **This is correct behaviour.** It diagnosed its own capability gap, preserved its
> draft as a durable comment, named the fix, and explicitly separated what it knew from
> what it could not verify. It refused to invent a summary about a system it could not
> inspect.

But it created **zero child cards** — and delegation is its entire purpose. Asking it
directly in a chat session proved `kanban_create` was available the whole time.

### The fix: brief the card to orchestrate

```bash
hermes kanban unblock <id>

hermes kanban comment <id> "You are the orchestrator and you have no file or terminal access by design. Do not attempt this work yourself. Use kanban_create to create child cards with parent links, assigning each to the specialist that can do it: analyst (research/writing), engineer (code), ops (file and shell work). Then complete your own card summarising what you created."

hermes kanban dispatch
```

Comments are durable on the card thread and the worker reads them via `kanban_show` on
its next run — **correct a card, don't recreate it**.

### Result — full fan-out

| Card | Assignee | Deps |
|:--|:--|:--:|
| parent goal | manager | 0/3 |
| Research the system and its wiring | **analyst** | 2/2 |
| Write the summary file | **engineer** | 1/1 |
| Verify the deliverable | **ops** | — |

Right lanes, sensible granularity, and an unprompted verification step. Dependency
gating confirmed by the fraction badges.

> ### ADR — the frugal orchestrator
> **Decision:** run the manager on an inexpensive model.
> **Verdict after testing: correct.** A frugal flash-class model orchestrates
> competently — right lanes, good granularity, honest reporting of its own uncertainty.
> **Do not upgrade the manager; the bottleneck is card briefing, not the model.**

> ### The limit of any automatic routing
> **Routing happens after briefing.** A vague card gets *confidently* routed to the
> wrong lane. Card quality is upstream of routing and remains a human responsibility.

### Two behaviours to expect

- **Re-running an orchestrator card re-decomposes rather than resumes.** A second wave
  of children appears alongside the first.
- **Comments can be interpreted as work.** A correction comment became its own card.

## Step 10 — Automatic routing

Hermes has native auto-routing. **Try it before building anything.**

On the Kanban page, flip the **Orchestration** pill from `Manual` to `Auto`. Create a
card from the **`+`** on the Triage column, **leaving the assignee empty**, and press
**Nudge dispatcher**.

**Result: routes unassigned triage cards correctly, with no script, key or cron.**

> ℹ️ The Orchestration pill and `kanban.auto_decompose` are **the same setting on two
> surfaces**. Flipping the pill overrides what was set by CLI.

### Nearly everything is front end

Card creation, assignee, **per-card model override**, drag-and-drop status changes, the
dependency editor, comments, attachments, `⚗ Decompose` and `✨ Specify`, the
orchestrator picker, and per-profile description editors are all in the dashboard.

**CLI-only exceptions:** `profile create`, `auth add`, the `tools` picker, `kanban init`.

### Do you need an external classifier?

**If the built-in routing is good enough, no.** Test it for a week first.

| | Built-in decomposer | Decision model |
|:--|:--:|:--:|
| Routes unassigned cards | ✅ | ✅ |
| **Calibrated confidence** | ❌ it just decides | ✅ plus full probability distribution |
| **Shadow mode** | ❌ acts immediately | ✅ log, compare, then grant control |
| **Defer to a human when unsure** | ❌ | ✅ gate below a threshold |
| **Capability-conflict detection** | ❌ | ✅ separate questions expose the conflict |
| Maintenance | none | a script and a key |

The distinction is not *automatic routing* — Hermes already does that. It is
**routing with a number you can gate on**. A shadow-mode implementation ships in
[`router/route.py`](router/route.py).

---

# Part 4 · Access

## Step 11 — Shared project workspaces over SMB

One folder per project, shared to the LAN, with agents and humans editing the same
files.

```bash
sudo apt update && sudo apt install -y samba

mkdir -p ~/hermes_projects/_scratch
chmod 755 "$HOME"                     # SMB must traverse into home
chmod 775 ~/hermes_projects
```

Append to `/etc/samba/smb.conf`, substituting `<USER>` and `<LAN_SUBNET>`:

```ini
[hermes_projects]
   comment = Hermes agent project workspaces
   path = /home/<USER>/hermes_projects
   browseable = yes
   read only = no
   force user = <USER>
   force group = <USER>
   create mask = 0664
   directory mask = 0775
   veto files = /._*/.DS_Store/.AppleDouble/.Trashes/.TemporaryItems/
   delete veto files = yes
   hosts allow = <LAN_SUBNET>
   hosts deny = 0.0.0.0/0
```

> **Three lines doing real work.**
> **`force user`** makes every file land owned by the account agents run as — without
> it, SMB-written files are owned by whoever wrote them and agents can read but not
> modify, surfacing as baffling worker failures.
> **`veto files`** suppresses macOS `.DS_Store` and `._` resource forks that otherwise
> clutter directories an agent globs.
> **`hosts allow/deny`** restricts at the Samba layer, independent of the firewall.

```bash
sudo smbpasswd -a <USER>                                        # SMB password ≠ Unix password
sudo ufw allow from <LAN_SUBNET> to any port 445 proto tcp
sudo ufw allow from <LAN_SUBNET> to any port 139 proto tcp
testparm -s                                                     # validate before restarting
sudo systemctl restart smbd nmbd && sudo systemctl enable smbd nmbd
```

### Wire the path into Hermes — one board per project

Dashboard → Kanban → **Settings** → set **project directory**. Cards on that board
inherit it. Use **+ New board** for each project.

| Board | Project directory |
|:--|:--|
| `Default` | `~/hermes_projects/_scratch` |
| `project-a` | `~/hermes_projects/project-a` |

**Create folders first** — Hermes will not invent them, and a missing directory fails
at dispatch. Paths must be **absolute**.

**Verify both directions.** Agent writes → SMB edits, and SMB writes → agent modifies:

```bash
ls -la ~/hermes_projects/<project>/
# want: -rw-rw-r-- <USER> <USER>
```

> ⚠️ **An open share is a prompt-injection path.** Agents read files from here.
> Anyone who can write to the share can place instructions in a document an agent will
> later read as part of a task. Use an authenticated share, never guest.

## Step 12 — Mobile control via Telegram

The gateway is "messaging platforms + cron scheduler", so this is a first-class
feature. **Works anywhere** — Telegram is the transport, so no VPN and no open port.

<details>
<summary><b>1 · Create the bot</b></summary>

In Telegram, message **`@BotFather`** → `/newbot` → give it a display name → give it a
username ending in `bot`. It replies with a token.

**The token is a password.** Anyone holding it controls the bot. BotFather can revoke
and reissue with `/token`.

</details>

<details>
<summary><b>2 · Get your numeric user id</b></summary>

Message **`@userinfobot`** and send anything. It replies with your id.

</details>

**3 · Wire it in:**

```bash
hermes -p manager config set TELEGRAM_BOT_TOKEN '<token>'
hermes -p manager config set TELEGRAM_ALLOWED_USERS '<your-numeric-id>'
hermes gateway restart
```

Confirm it connected:

```bash
journalctl --user -u hermes-gateway -n 30 --no-pager | grep -i telegram
# want: [Telegram] Connected to Telegram (polling mode)
```

> **Why the manager owns the bot.** Platform credentials are per-profile. Messaging the
> bot talks to your orchestrator — describe a goal in plain language and it decomposes
> and routes. And because the manager has **no terminal and no file tools**, a bot
> reachable from the internet physically cannot be talked into running a shell command.
> That toolset decision does security work it was not designed for.

### Two modes

| Input | Behaviour |
|:--|:--|
| **Plain language** | a conversation with the manager — it can create cards, read the board, answer questions |
| **`/kanban ...`** | a direct board command. Instant, exact, no LLM turn |

`/kanban` is **exempt from the running-agent guard**, so it lands even while an agent is
mid-task. Use `--triage` so auto-routing picks the card up.

> 📱 **Turn off autocapitalisation.** Phones capitalise `/kanban` to `/Kanban`, which
> fails to parse.

### Lock it to your account

Deny-by-default is the shipped behaviour, but verify rather than trust:

```bash
hermes -p manager config get TELEGRAM_ALLOWED_USERS      # must be YOUR id
hermes -p manager config get GATEWAY_ALLOW_ALL_USERS     # must be unset
hermes config get GATEWAY_ALLOW_ALL_USERS                # must be unset
hermes -p manager config get dm_policy                   # must not be 'open'
hermes -p manager config get group_policy                # must not be 'open'
```

> ### Test the denial, not your access
> Your messages working proves the allowlist **admits you**. It proves nothing about
> whether it **denies others** — separate code paths. Bot usernames are publicly
> searchable, so the deny path is the one that matters.
>
> Temporarily set the allowlist to a fake id, restart, and message the bot. **Expect
> silence.** Then restore. That exercises the exact path an intruder would hit, using
> your own account as the unauthorised one — no second account needed.
>
> ✅ **Verified in both directions on this build:** allowed sender receives a reply,
> unauthorised sender receives nothing.

**Groups use a separate policy from DMs.** Keep the bot in a private chat.

---

# Field notes

> The things that cost real time. All hit on a clean install.

### 1 · Credentials are per-profile, and `env:` pointers dangle

`hermes -p <profile> auth list` shows a **source**:

```
gemini      #1  GOOGLE_API_KEY   api_key  priority=0  env:GOOGLE_API_KEY
openrouter  #2  api-key-2        api_key  priority=1  manual  ←
```

- **`env:VARNAME` is a pointer, not a key.** If the variable is absent from the
  process environment, the pointer dangles and Hermes reports *"no usable credentials
  found"*. The error is literal — it wants the **variable**.
- **`manual` is a stored value.** `auth add` creates these. The reliable fix.
- **`←` marks the credential in use**, in `priority` order.

**Scope:** each profile gets a *copy* of the root store at creation and is independent
after. Root changes do not propagate.

**Selection follows priority, not recency.** Duplicate `auth add` calls stack as inert
entries — a bad earlier key keeps failing while the list looks fully configured.
Remove duplicates rather than stacking.

### 2 · Model ids are not what the dashboard shows

Dashboards truncate and omit namespacing. A provider id may be `vendor/model-name`
while the dashboard shows `model-name`. Typing the display name 404s — and Hermes
treats a 404 model-not-found as **terminal**, blocking the card after one attempt.

### 3 · Free-tier and preview ids expire silently

Preview and free-period ids get retired or renamed with no notice. Keep a model
registry and treat an unexplained 404 as id drift first.

### 4 · "Free" via an aggregator usually isn't

Routing a model that has a free direct tier *through* an aggregator means the
aggregator bills it. Hermes warns on every gateway start when an auxiliary fallback is
not a free SKU. Levers: `auxiliary.free_only: true`, or pin
`auxiliary.openrouter_model` to a free SKU.

### 5 · There's no `default_workdir` key

Unrecognised config keys are **saved anyway** with a warning, so they sit in your
config looking authoritative. `hermes config unset <key>` to clear.

**Workspace kinds:** `scratch` (default, **deleted on completion** — only files
declared via `kanban_complete(artifacts=[...])` survive), `dir:/abs/path` (preserved),
`worktree` (preserved). `dir:` **must be absolute**.

### 6 · The dashboard's plugin API is unauthenticated by design

Hermes' dashboard auth middleware **deliberately skips `/api/plugins/*`** because it
assumes a localhost bind. App-level auth does not cover them — **the control is
network-level**. Never bind to `0.0.0.0` without a firewall rule scoped to your subnet.

`ufw allow <port>` means **any source address**. Delete the permissive rule *before*
adding a restricted one, or the broad rule still matches.

### 7 · `hermes update` can orphan a systemd unit

`status=203/EXEC` means systemd could not **execute the binary** — the update moved the
launcher. A loose process serving the same port can mask this for days.

```bash
systemctl --user daemon-reload
systemctl --user reset-failed <unit>    # clears "start request repeated too quickly"
systemctl --user start <unit>
```

> **Operational lesson:** do not kill a process masking a broken unit while it is the
> only thing providing a service you are about to need.

### 8 · Running gateways capture their environment at startup

A credential added after a gateway starts is invisible to it. Same for a board created
after the gateway came up. **Restart after any credential or board change** — the most
common reason a correct fix appears to have failed.

### 9 · Shell environment is not Hermes' credential store

`auth add` writes to Hermes' own store, **not** to a `.env` file. For anything outside
Hermes, export explicitly — and check the length, because these all give the same
opaque `401`:

| Symptom | Cause |
|:--|:--|
| `len=0` | new shell since the export |
| `len=12` | a placeholder exported literally |
| `len=75` when expecting 73 | `<` `>` markup brackets pasted with the key |

**Cron has almost no environment.** Scheduled scripts must read secrets from a file.

### 10 · API keys: cap them, reset them, scope them

Most providers show a key **once, at creation**. Create one key per consumer so
revocation is surgical and a chatty agent cannot exhaust another's budget.

Set a credit limit **and a monthly reset**. A cap with no reset is a one-shot — the key
stops permanently once consumed.

> **At least one provider returns `404 "requires available credits"` rather than `402`
> on exhaustion.** Hermes treats 404 as terminal, so budget exhaustion is
> indistinguishable from a renamed model id. **If a lane blocks and nothing changed
> locally, check the credit balance before hunting model ids.**

### 11 · Ensemble presets are opt-in, but expensive when selected

A "mixture of agents" preset appears as a selectable model: reference models each
produce a perspective, then an aggregator answers and calls tools. **It only runs if a
profile's model is set to that preset.**

One turn becomes **three LLM calls**, with the most capable model doing the
aggregation. A deliberate hard-problem tool — never a default on a worker profile.

### 12 · Toolsets are per platform

Enabling a toolset for CLI does not enable it for Telegram, Discord or Slack. Each
platform has its own list under `platform_toolsets.<platform>`.

---

# Command reference

```bash
# ─── profiles ────────────────────────────────────────────────
hermes profile list
hermes -p <p> model                      # pick model interactively
hermes -p <p> auth list | auth add <provider>
hermes -p <p> tools                      # toolsets, per platform
hermes -p <p> config get | set <key> [value]

# ─── board ───────────────────────────────────────────────────
hermes kanban init
hermes kanban create "<title>" --assignee <p> --workspace dir:/abs/path [--triage] [--parent <id>]
hermes kanban list [--status triage|todo|ready|running|review|blocked|done]
hermes kanban show | log | runs <id>
hermes kanban assign <id> <profile>
hermes kanban comment <id> "<text>"      # correct a card without recreating it
hermes kanban unblock | promote <id>
hermes kanban decompose | specify <id>
hermes kanban dispatch [--dry-run]
hermes kanban stats | assignees | diagnostics | watch
hermes kanban swarm "<goal>" --workers a,b --verifier c --synthesizer d

# ─── gateway ─────────────────────────────────────────────────
hermes gateway start | restart | status
journalctl --user -u hermes-gateway -n 60 -l --no-pager
```

### Exit codes — what the dispatcher does

| Exit | Meaning | Behaviour |
|:--:|:--|:--|
| `0` while `running` | protocol violation | 2 nudges → bounded retry → auto-block |
| `1` | ordinary failure | counts toward `failure_limit` → auto-block |
| `75` | rate-limited / overloaded / timeout | requeued, **no failure counted**, cooldown |
| `78` | credential 401/403, model 404, TLS | **circuit breaker trips on first occurrence** |

> **A `78` is always configuration.**

---

<div align="center">

**ar1syr0**

Hermes Agent is MIT (Nous Research). This repository is documentation and glue.

</div>
