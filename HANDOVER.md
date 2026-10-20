<div align="center">

# Handover

### ar1syr0 Hermes — four-agent fleet

*Self-contained state brief. Paste as context into a fresh session, or read as a
takeover document. Assumes no prior conversation.*

</div>

---

## 1 · Status

**Complete and verified.** The fleet runs end to end: agents delegate to each other,
unassigned cards route themselves, project folders are network-shared, and work can be
filed from a phone.

| Capability | State |
|:--|:--|
| Four agents · three models · **three providers** | ✅ built, each separately credentialed and tested |
| Role separation enforced by toolsets | ✅ manager has no shell or file access |
| Shared task board + dispatcher | ✅ proven — claims cards, spawns workers |
| Agent-to-agent delegation | ✅ proven — fan-out to three specialists with dependency gating |
| Automatic routing of unassigned cards | ✅ Hermes native, Orchestration = Auto |
| Project folders shared over SMB | ✅ bidirectional read/write verified |
| Mobile control | ✅ Telegram, allowlisted to one account — deny path tested |
| External decision model (optional) | ⚪ verified by API call, **not wired in** |

---

> ⚠️ **Model and provider names are deliberately omitted from this document.**
> Classes and provider types are given instead. Do **not** infer, invent or substitute
> specific model ids from this brief — they rot, and in Hermes a stale id returns a 404
> that is treated as **terminal**, blocking the card after a single attempt. Read the
> live values from the install with `hermes profile list` and `hermes -p <p> model`.

---

## 2 · Architecture

```
Telegram ─┐
Dashboard ─┼──► Kanban board ──► dispatcher ──► worker process per card
CLI ───────┘    (one SQLite DB)                 (spawned as the assignee profile)
```

A **profile** is a full Hermes home — its own config, model, credentials, memory and
toolsets, under `~/.hermes/profiles/<name>/`.

| Profile | Model class | Provider type | Toolsets | Deliberately lacks |
|:--|:--|:--|:--|:--|
| `manager` | frugal flash | aggregator (metered) | board, memory, skills, planning, session search, clarify | **shell, files, code** |
| `analyst` | frugal flash | aggregator (metered) | + web search, file ops | shell, code |
| `engineer` | capable frontier | subscription (allowance) | + terminal, code execution | — |
| `ops` | free tier | direct API (quota) | terminal, file ops | web |

**Three providers, three credential domains, three billing models.** This is the
constraint that shapes the whole architecture: Hermes resolves one delegation model for
an entire child fleet, and a model-string rewrite cannot cross providers. Only a
**profile** or a **per-task override** carries a complete credential bundle — hence
profiles + board rather than subagents.

**One card has one assignee.** Multi-agent work comes from several linked cards, not
one card reaching many agents.

### Key configuration

```yaml
kanban:
  orchestrator_profile: manager
  default_assignee: manager        # a typo lands visibly, not on `default`
  auto_decompose: true             # == Orchestration "Auto" pill in the dashboard
  max_in_progress: 2
  max_in_progress_per_profile: 1   # stops one lane tripping its own 429s
  dispatch_in_gateway: true
  dispatch_interval_seconds: 60
  failure_limit: 2
auxiliary:
  free_only: true                  # ⚠ see Risk R1
```

**Board** lives in the root Hermes home — **not** per profile. Project folders live
under `~/hermes_projects/<project>/`, one board per project with its project directory
set in board Settings.

---

## 3 · Hard-won findings

> Do not rediscover these. Each cost real time.

**Credentials**

1. **Per-profile.** Each profile gets a *copy* of the root credential store at creation
   and is independent after. Root changes do not propagate.
2. **`env:VARNAME` credentials are pointers, not keys.** If the variable is absent from
   the process environment, the pointer dangles and Hermes reports *"no usable
   credentials found"*. `auth add` creates a `manual` (stored) credential — the fix.
3. **Selection follows `priority`, not recency.** `←` marks the one in use. Duplicate
   `auth add` calls stack as inert entries; a bad earlier key keeps failing while the
   list looks configured.
4. **A running gateway captures its environment at startup.** Restart after any
   credential or board change — the most common reason a correct fix looks like it
   failed.

**Models**

5. **Model ids are namespaced**; dashboards truncate them. Always use
   `hermes -p <p> model` to write the exact string.
6. **A 404 model-not-found is terminal** — the card blocks after one attempt. It is
   ambiguous between a renamed id, a retired model, and **credit exhaustion** (some
   providers return 404, not 402). **Check credit balance before hunting model ids.**
7. **Preview and free-period ids are retired or renamed with no notice.**
8. **Routing a model through an aggregator loses its free tier.**

**Board and tools**

9. **Toolsets are per platform.** Enabling for CLI does not enable for Telegram.
10. **Kanban toolset is opt-in.** A worker that cannot call `kanban_complete` hangs
    until it auto-blocks as a protocol violation.
11. **Unrecognised config keys are saved anyway** with a warning, so they sit in config
    looking authoritative. There is no `kanban.default_workdir`; workspace defaults are
    **per board** in the dashboard.
12. **Workspace kinds:** `scratch` is **deleted on completion** (only declared artifacts
    survive); `dir:` and `worktree` are preserved. `dir:` must be **absolute**.
13. **Correct a card with a comment, don't recreate it.** Comments are durable on the
    thread and the worker reads them via `kanban_show` on its next run.

**Security**

14. **The dashboard's `/api/plugins/*` routes are unauthenticated by design** — it
    assumes a localhost bind. App-level auth does not cover them; the control is
    network-level.
15. **`hermes update` can orphan a systemd unit** (`203/EXEC`, moved binary). A loose
    process on the same port can mask it for days.
16. **Cron has almost no environment.** Scheduled scripts must read secrets from a file.

---

## 4 · Design decisions

**Profiles + Kanban, not `delegate_task`.** Hermes resolves one delegation model for
the entire child fleet, so three roles on three models requires profiles. Gains durable
state, resumability, human-in-the-loop and an audit trail. `delegate_task` is disabled
on all profiles so the board is the only road.

**Capability is the enforcement.** An orchestrator told "never implement" will still try
when a task looks small. Removing the terminal and file toolsets leaves delegation as
its only mechanism. This cannot be prompted around, and it doubles as the security
control on the Telegram bot.

**The frugal orchestrator — tested, and correct.** A flash-class model orchestrates
competently: right lanes, sensible granularity, honest reporting of its own
uncertainty. **Do not upgrade the manager.** The bottleneck is card briefing.

**Briefing is upstream of routing.** A card phrased "do X" makes the manager attempt X;
phrased "decompose X and assign it" it fans out correctly. **No classifier fixes a
badly written card** — it will confidently route a vague card to the wrong lane.

**Built-in routing before an external classifier.** Hermes' native auto-routing assigns
unassigned triage cards correctly with no script, key or cron. An external decision
model adds exactly one thing: **calibrated confidence you can gate on**, plus shadow
mode to verify before granting control. Worth it only if some cards should pause rather
than be confidently misrouted.

---

## 5 · Risks and open items

| | Item | Why it matters | Priority |
|:--|:--|:--|:--|
| **R1** | `auxiliary.free_only: true` | The Kanban decomposer **is** an auxiliary task. If routing or decomposition breaks, **revert this first** | High if symptomatic |
| **R2** | Dashboard systemd unit broken (`203/EXEC`) | Dashboard runs as a loose process; will not survive a reboot | Medium |
| **R3** | API keys with a credit cap and no monthly reset | Stop **permanently** when consumed rather than recovering. Poor fit for recurring work. Failure presents as a 404 | Medium |
| **R4** | Open ports beyond the intended services | Review `sudo ss -tlnp` against firewall rules. A VPN overlay would solve the class | Review |
| **R5** | SMB share is a prompt-injection path | Agents read files from it; anyone who can write can plant instructions. Keep it authenticated, never guest | By design |
| **R6** | Telegram bot usernames are publicly searchable | The allowlist is the only control. Deny path **tested in both directions** — unauthorised sender receives no response | ✅ Verified |
| **R7** | Re-running an orchestrator card re-decomposes | Duplicate children appear rather than resuming | Known |

---

## 6 · Next actions

1. **Run it for a week.** The open question is whether native routing is good enough.
   Watch for cards routed confidently to the wrong lane, and for the decomposer
   splitting work that only needed assigning.
2. **If some cards should pause rather than be misrouted**, wire the decision model.
   Implementation in `router/route.py` — shadow mode by default, fail-open, reads its
   key from a file. Run `--shadow` first and compare its calls against your own before
   granting control. **The threshold values in the example config are placeholders, not
   findings.**
3. **Fix R2 and R3** — both are small and both bite at the worst moment.
4. **Consider the fleet's own suggestion**: a mandatory review-gate child card on every
   implementation card, with explicit acceptance criteria and required test evidence.

---

## 7 · Repository

```
SETUP.md       full build guide · 12 field notes          ← the one to share
HANDOVER.md    this file
README.md      index
docs/          design reasoning: architecture, providers, routing, ops, risks
router/        shadow-mode decision-model router
config/        example configs for root and each profile
scripts/       read-only discovery helper
templates/     routing journal
```

---

<div align="center">

**ar1syr0**

</div>
