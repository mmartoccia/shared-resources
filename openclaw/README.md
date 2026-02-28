# OpenClaw: Implementation Guide

This folder documents how to build and operate a production personal AI agent system using OpenClaw. Everything here is distilled from a live deployment -- not theory.

Each document is a spec you can follow to replicate a specific capability from scratch.

---

## Before You Start

**What you need:**

- [OpenClaw](https://openclaw.ai) installed and running on your primary machine (the gateway host)
- A second machine (laptop, Pi, VPS) connected via [Tailscale](https://tailscale.com) -- this becomes your ops/cron node
- Python 3.9+ on both machines
- A Telegram bot token (for user-facing chat; NOT for infrastructure -- see Trusted Channel)
- `~/.secrets/` directory on both machines, permissions locked (`chmod 700 ~/.secrets`)

**What you're building:**

A personal AI agent that runs continuously, monitors your world, executes tasks autonomously, and -- critically -- can improve its own behavior over time without becoming a security liability.

The three foundational capabilities everything else builds on:

1. **Secure self-modification** -- the agent can update its own memory and context without being vulnerable to prompt injection
2. **Multi-machine orchestration** -- ops crons run on a separate machine, keeping the trust boundary real
3. **Authorization governance** -- every action the agent can take on itself is explicitly classified: autonomous, requires approval, or permanently blocked

---

## Documents

---

### [Trusted Channel Architecture](trusted-channel-architecture.md)
**Start here if:** You want your agent to restore context after compaction, consolidate memory nightly, or propose behavioral changes to itself -- without that mechanism being an attack surface.

**The problem it solves:**

A self-improving agent needs to send instructions to itself. But an instruction from a machine looks identical to a prompt injection attack from a bad actor -- if you're treating all messages with the same trust level.

**The solution in one sentence:**

Run a signed HTTP server on your gateway machine (port 18799). Only your ops node (authenticated via Tailscale + HMAC) can POST to it. Everything else is ignored.

**What you tell your users:**

> "Your agent can now restore its own context after a session reset, consolidate daily notes into long-term memory overnight, and propose behavioral improvements -- all without any message appearing in your chat. It happens in the background, verified cryptographically, logged to an audit trail you can inspect at any time."

**What you need to implement it:**

- Two machines connected via Tailscale
- A shared HMAC secret (generated once, never transmitted -- copy via `scp`)
- Python 3.9+ stdlib only -- no external dependencies
- `tc-server.py` running on the gateway (launchd keeps it alive)
- `tc-client.py` on the ops node calling it on a schedule

**Time to implement:** ~2 hours from scratch. Verification included in the spec.

---

### [OpenClaw Infrastructure Patterns](openclaw-infrastructure-patterns.md)
**Start here if:** You're designing a new OpenClaw deployment and want to avoid the mistakes that come from learning these patterns the hard way.

**The problem it solves:**

Personal AI infrastructure has a specific failure mode: it works fine for the first few weeks, then accumulates technical debt (stale crons, sprawling secrets, no audit trail, skills that conflict with each other) until it becomes unreliable and hard to reason about.

**Eight patterns that prevent that:**

| Pattern | The rule |
|---|---|
| Multi-node fleet | Gateway host is sacred. Crons and heavy workloads run on satellite nodes. |
| Secrets management | One file per secret. `chmod 600`. Never in code or committed config. |
| Consent graph | Every action the agent can take is pre-classified. No ambient authority. |
| Memory architecture | Three tiers: session handoff (working), daily logs (episodic), MEMORY.md (semantic). |
| Cron orchestration | OpenClaw-native for simple jobs. launchd for cross-machine and timing-critical. |
| Audit trail | Append-only JSONL. Written before the action executes, not after. |
| Sub-agent orchestration | Spawn-and-announce. Never poll. Parallel when independent, sequential when dependent. |
| Skill management | One-in-one-out. Every new skill retires something or justifies net-new surface area. |

**What you tell your users:**

> "Before we add anything new, we check what it replaces. Before any action that modifies the agent's own behavior, it's logged. Before any sensitive external action, it's classified against the consent graph. The system is designed so that when something goes wrong -- and it will -- you can trace exactly what happened and why."

**Time to implement:** Patterns are applied incrementally. Start with secrets management and consent graph on day one. Add the rest as your deployment grows.

---

### [OpenClaw Skills Catalog](openclaw-skills-catalog.md)
**Start here if:** You're setting up a new deployment and want to know what skills are worth installing, in what order, and what each one actually does in practice.

**The problem it solves:**

ClawdHub has 100+ skills. Most new deployments install too many too fast, end up with conflicts, and can't tell which skill handled what. This catalog is the curated starting set from a production deployment -- 23 skills, each with a clear trigger, a clear "when not to use," and integration notes from real use.

**What you tell your users:**

> "We don't install skills speculatively. Each one has a documented trigger -- a specific phrase or context that activates it. If two skills could handle the same request, we've picked one and documented why. You'll always know what handled your request."

**Recommended install order for a new deployment:**

1. `imsg` or `bluebubbles` (iMessage -- your most important communication channel)
2. `github` (if you have active repos)
3. `perplexity` (web-grounded research)
4. `council-of-the-wise` (multi-model review for major decisions)
5. `clawbrain` (persistent memory layer)
6. Everything else as specific needs arise

---

### [Entropy and Anti-Detection](entropy-and-anti-detection.md)
**Start here if:** Your agent runs cron jobs against subscription APIs and you want to avoid behavioral fingerprinting, rate limiting, or bot detection.

**The problem it solves:**

Deterministic cron schedules are machine-obvious. "Every hour at :00" is a bot pattern. Subscription API providers can detect and flag accounts that hit their APIs on exact, repeating intervals with the same model every time.

**Two-layer solution:**

1. **Schedule entropy:** A daily daemon reshuffles the minute offset for every managed cron job within configurable bounds. A job "at 9 AM" fires at 9:07 today, 9:23 tomorrow, 9:41 the day after. Human-plausible variation.
2. **Model rotation:** The same daemon rotates which AI model handles each job from a configured pool. Anthropic today, Google tomorrow. Different endpoints, different fingerprints.

**What you tell your users:**

> "Your agent's cron jobs don't fire at the same time every day. The schedule entropy system randomizes when each job runs within a defined window, and rotates which AI model handles each task. From a subscription provider's perspective, your usage looks like an engaged human power user, not a bot. This protects your account from behavioral flags and keeps your access reliable."

**What you need to implement it:**

- `entropy-scheduler.py` in `~/clawd/scripts/`
- `entropy-profiles.json` in `~/clawd/config/` with per-job bounds and model pools
- One daily cron at 2:15 AM that runs the scheduler (model: `gemini-3-pro-preview`)
- UUIDs for each managed job from `~/.openclaw/cron/jobs.json`

---

### [Model Selection Strategy](model-selection-strategy.md)
**Start here if:** You're running a high-volume personal AI agent and hitting quota limits, or want to understand how to route tasks across Anthropic, Google, NVIDIA NIM, OpenAI, and local models.

**The problem it solves:**

Without deliberate routing, everything defaults to the most capable (most expensive, most rate-limited) model. A production deployment running ~800M tokens/month needs intentional load distribution -- or Anthropic quota runs out by the 20th of the month.

**The rule in two sentences:**

**New code -> Anthropic. Fix code -> Codex.** Everything else routes by task type: research to Gemini (1M context, Search grounding), sensitive data to local models (never leaves the machine), background crons to Gemini to preserve Anthropic quota for user-facing conversation.

**What you tell your users:**

> "Not every task needs the most powerful model. Your morning briefing runs on Gemini. Your code debugging runs on Codex. Your conversation with me runs on Claude. Each task gets the right tool for the job -- which means your quota goes further, your costs stay predictable, and the expensive models are available when you actually need them."

**What you need to implement it:**

- `openclaw models list` run at session start (always verify what's actually available)
- Per-cron model selection configured in `entropy-profiles.json` model pools
- `token-meter.py` daily cron tracking usage by provider
- Quota guardian cron (every 2h, alert at 80% Anthropic usage)

---


### [Mission Control Dashboard](mission-control-dashboard.md)
**Start here if:** You want a real-time Kanban view of everything your agent fleet is doing -- active sub-agents, pending tasks, recent completions -- without adding a database or external service to your stack.

**The problem it solves:**

When an agent is running multiple sub-agents in parallel, managing a backlog of tasks, and operating on a cron schedule, you lose visibility. You don't know what's in flight, what's blocked, or what completed while you weren't looking.

**The solution in one sentence:**

A Next.js app scans your markdown files for task blocks every 30 seconds and renders them as a live Kanban board -- the agent writes tasks to files, Mission Control reads them, no database required.

**What you tell your users:**

> "Mission Control is your agent's task board. Every task the agent is working on -- or plans to work on -- is tracked in plain markdown files. The dashboard reads those files and shows you a live Kanban view. When a sub-agent completes something, it marks the task done; you see it update in real time. No database, no sync issues -- the files are the source of truth, the dashboard is just the view."

**Time to implement:** ~1 hour. Node.js app, no database, runs on localhost:3001. Add to launchd for persistence.

---

### [Heartbeat Protocol](heartbeat-protocol.md)
**Start here if:** You want your agent to proactively monitor email, calendar, and notifications without being explicitly asked -- and to know when to reach out vs. when to stay quiet.

**The problem it solves:**

A purely reactive agent misses things. Important emails arrive, calendar events approach, and the agent says nothing because no one asked. But a chatty agent that checks in constantly is worse than useless.

**The solution in one sentence:**

A configurable heartbeat schedule drives periodic checks against email, calendar, and notification sources; a suppress logic layer decides whether findings are worth surfacing based on time of day, recency, and urgency threshold.

**Time to implement:** ~1 hour. HEARTBEAT.md configuration + heartbeat state tracking in JSON.

---

### [Session Handoff and Context Continuity](session-handoff-and-continuity.md)
**Start here if:** You want context to survive session compaction -- so when a session resets, the next session knows what was in flight without you having to re-explain.

**The problem it solves:**

LLM sessions have finite context windows. When a session compacts or resets, the agent loses everything it was working on. Without a handoff mechanism, every reset is a cold start.

**The solution in one sentence:**

`memory/state/session-handoff.md` is overwritten at each context checkpoint (30%, 50%, 70% usage) with current active context; the next session reads it before greeting, restoring intra-day state in seconds.

**Time to implement:** ~30 minutes. One file, one convention, one nightly QMD cron to index previous-day memory.

---

### [Council of the Wise](council-of-the-wise.md)
**Start here if:** You're about to make a significant architectural decision, security model change, or novel technical commitment -- and want independent multi-model review before committing.

**The problem it solves:**

A single model has a single perspective and consistent blind spots. For decisions that are hard to reverse -- security models, infrastructure choices, behavioral changes -- you want adversarial review, not confirmation.

**The solution in one sentence:**

Spawn sub-agents across frontier models (Opus thinking + Gemini Pro + one more), each analyzing the proposal from a defined expert perspective, then synthesize the independent findings before deciding.

**Time to implement:** ~2 hours. Sub-agent orchestration pattern + persona definitions + synthesis prompt.

---

### [Memory Consolidation Pipeline](memory-consolidation-pipeline.md)
**Start here if:** Your daily log files are growing unbounded and you want a principled way to decide what to keep, what to compress, and what to discard -- without losing anything important.

**The problem it solves:**

Raw daily logs are verbose and unsearchable. Long-term memory (MEMORY.md) needs to be curated, not appended to indefinitely. Without a consolidation step, the agent either keeps everything (expensive) or loses context across sessions (worse).

**The solution in one sentence:**

A nightly pipeline reads recent daily logs, extracts significant events and lessons via an LLM pass, updates MEMORY.md with distilled findings, and indexes everything via QMD for semantic search.

**Time to implement:** ~2 hours. Nightly cron + consolidation script + QMD integration.

---

### [Consent Graph Reference](consent-graph-reference.md)
**Start here if:** You want every action your agent takes on the world -- email, messages, calendar, home automation, self-modification -- to be explicitly authorized before it fires.

**The problem it solves:**

An agent with access to your email, iMessage, and home automation is powerful. It's also a liability if there's no classification layer between "agent wants to do X" and "agent does X." The consent graph pre-classifies every possible action into three buckets: autonomous (just do it), requires approval (ask first), or blocked (never, period). The graph is locked at the OS level -- the agent cannot modify it autonomously.

**What you tell your users:**

> "Before your agent does anything in the real world, it checks the consent graph. Every action is pre-classified. The graph decays toward requiring approval -- not permissiveness -- if you skip your monthly review. The agent cannot modify the graph; only you can."

**What you need to implement it:**

- `~/clawd/memory/state/consent-graph.json` with all domains and actions classified
- `consent.py` CLI for pre-flight checks
- `chflags uchg` (macOS) or `chattr +i` (Linux) to lock the file
- A 30-day calendar reminder to review and reset the decay clock

**Time to implement:** 30 minutes. The reference spec includes the full production graph you can fork.


---

### [Self-Improvement System Architecture](self-improvement-system.md)
**Start here if:** You want your agent to improve its own behavior over time based on outcomes -- without becoming a prompt injection liability.

**The problem it solves:**

An agent configured once and never updated degrades in usefulness as your context evolves. An agent that modifies itself without constraints is a security risk. This spec resolves both: a four-component system (Hebbian linking, sleep consolidation, synaptic pruning, predictive staging) that learns from outcomes, distills patterns nightly, removes noise weekly, and pre-stages relevant context at session start -- all behind a human approval gate with a 2-hour cooling period and 7-day undo window.

**What you tell your users:**

> "Your agent gets better over time. When it resolves a problem, it records how. A nightly process distills those records into patterns. None of this modifies your agent's behavior without your review -- every proposed change requires your approval, can be reversed for 7 days, and must include an explicit before/after diff."

**What you need to implement it:**

- Trusted channel live (hard prerequisite -- see Trusted Channel Architecture)
- Consent graph with `self_modification` domain configured
- QMD indexed and running nightly
- Python 3.9+ on both machines

**Time to implement:** 4-6 hours for all four components. Build in order: audit logging → consent graph → trusted channel → outcome records → consolidation → hebbian linker → pruning → predictive staging.


---

## Specs In Progress

These capabilities exist in the production deployment but don't have their own spec yet. PRs welcome.

| Pattern | What it is |
|---|---|
| **Self-Improvement System Architecture** | How to wire Hebbian memory linking, nightly sleep consolidation, synaptic pruning of stale context, and predictive staging. The neuroscience-to-code translation that actually works vs. the metaphors that lead you astray. |
| **Civil liberties and geopolitical monitoring** | Automated monitoring of policy developments, legal changes, and geopolitical signals relevant to the operator. Distinct from general news monitoring in its focus and alert thresholds. |

---

## What's Coming

| Document | What it covers |
|---|---|

---

## The Four Rules

Every document in this folder is built on four principles. If you find yourself violating one, stop and ask why.

**1. Verified over asserted.**
Every security claim is tested with an expected output. "The HMAC rejects tampered messages" is not a claim -- it's a test result: `{"valid": false, "reason": "hmac_mismatch"}`.

**2. Replicable over idealized.**
Every command is copy-pasteable. macOS vs. Linux differences are called out explicitly (`chflags` vs. `chattr`). Known limitations are in the spec, not hidden in the implementation.

**3. One-in-one-out.**
Before adding any tool, skill, API, cron, or dependency: identify what it replaces. If nothing gets retired, the bar is high -- it must be genuinely additive, not just interesting. Stack size is a liability.

**4. No third parties in the control plane.**
Infrastructure the system depends on for *correctness* -- not convenience -- must not route through external services. Telegram for user chat: correct tool. Telegram for machine-to-machine IPC: wrong tool, wrong abstraction, build the HTTP layer instead.

---

*Last updated: February 28, 2026 | [github.com/mmartoccia/shared-resources](https://github.com/mmartoccia/shared-resources)*
