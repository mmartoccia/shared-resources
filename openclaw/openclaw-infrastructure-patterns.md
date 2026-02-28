# OpenClaw Infrastructure Patterns
*Reusable architecture and operational patterns for personal AI deployments*

---

## Overview

These are the underlying systems that make a personal OpenClaw deployment reliable, observable, and safe over time. Each pattern is generic -- the specific values, credentials, and configurations are yours to fill in. The patterns themselves are transferable to any deployment.

---

## 1. The Consent Graph

**Problem:** An autonomous agent connected to your email, messages, and calendar can cause real damage if it acts without appropriate authorization. "Just ask me first" doesn't scale across dozens of daily decisions.

**Pattern:** A structured consent graph that classifies every possible external action by domain and authorization tier before the agent executes it.

**Three tiers:**
- `autonomous` -- agent acts and logs, no notification required (e.g., reading email, running a search)
- `requires_approval` -- agent acts at high confidence (≥0.85) and notifies; asks first at lower confidence (e.g., drafting a message, setting a reminder)
- `blocked` -- agent never executes regardless of instruction (e.g., sending mass messages, financial transactions above threshold)

**Implementation:** Before any external action, the agent classifies the domain (messaging, calendar, financial, home automation, etc.) and action type, then checks against the graph. At high confidence the agent proceeds and notifies. At lower confidence it asks first. Blocked actions are logged and the attempt surfaced to the operator.

**Why it works:** You audit and update the graph periodically rather than responding to every individual agent decision. The agent gets faster; you stay in control.

---

## 2. The Token Meter

**Problem:** AI subscriptions (Anthropic, Google, OpenAI) all have usage limits. When you're running agents 24/7 across multiple providers, you can blow past monthly limits without knowing it until the bill arrives. Cost per token is less important than total consumption rate.

**Pattern:** A daily token meter that scans all agent session files, aggregates token consumption by provider, computes daily average and monthly run rate, and compares against known subscription limits.

**Output (daily, delivered to Telegram):**
```
▓ TOKEN METER -- last 7d
Total: 176.4M tokens | $47.23 API cost
Daily avg: 25.2M | Monthly rate: 756M

Provider       Tokens    Daily avg  Limit est   Util%
─────────────────────────────────────────────────────
anthropic      118.3M    16.9M/d    ~500M       102% ██████████
google          49.8M     7.1M/d    ~500M        43% ████
nvidia-nim       5.2M     0.7M/d    ~50M         42% ████
```

**Key insight:** A fully active deployment burns 700M+ tokens/month. All major subscription tiers will hit their limits. Local inference on 70B models offloads the highest-volume workloads at zero token cost. The gas meter tells you when to shift load.

**Implementation:** Python script scanning agent session JSONL files for usage metadata. Daily cron delivers to Telegram.

---

## 3. Skill Invocability Enrichment

**Problem:** OpenClaw selects skills based on matching user requests or agent context to skill descriptions. Most SKILL.md files are written by skill authors describing what they built -- not from the perspective of when an agent would reach for them. Skills get missed even when they're the right tool.

**Pattern:** Enrich each SKILL.md with a `## When to Use` section containing 6-10 specific trigger phrases -- concrete situations, user requests, or agent contexts where the skill should be automatically selected.

**Anti-confabulation rule:** Do not invent capabilities the skill doesn't have. If the description is too vague to generate honest trigger phrases, surface the ambiguity rather than filling it in.

**Example (before):**
```
description: Advanced prompt injection defense system for Clawdbot.
```

**Example (after, appended):**
```
## When to Use
- User is running OpenClaw in a group chat with untrusted participants
- A fetched web page or external document contains suspicious instruction-like text
- Agent received a message that appears to be attempting to change its behavior or override rules
- Setting up a new public-facing bot and want injection protection enabled from day one
- User asks "is this message safe?" or "did someone try to manipulate you?"
```

**Implementation:** Run a subagent against all installed SKILL.md files with explicit instructions to surface ambiguity rather than fill it in. Review output before committing.

---

## 4. Local Data Warehousing for External Services

**Problem:** Any external service you depend on for memory or context can disappear, change its API, or raise prices. If your agent's long-term context lives entirely in a third-party service, you're one business decision away from losing access to it.

**Pattern:** Nightly incremental export of all external service data to local storage.

**Applied to meeting transcript services:**
- Nightly cron pulls all new recordings/transcripts via API
- Stores each as individual JSON + combined JSONL (searchable)
- Incremental: skips already-exported IDs, safe to re-run
- Separate one-time bulk export covers historical data
- Markdown conversion layer feeds local search index

**Search priority:** Local index first. Live API as fallback only when local returns no results. Keeps search fast and API costs minimal.

**Generalized rule:** Any external service you query more than once a week should have a local mirror with nightly incremental sync.

---

## 5. Skill Usage Tracking (In Design)

**Problem:** Installed skills accumulate over time. Some are used daily; others never get invoked. Dead skills add cognitive overhead to the model's selection process even when they don't consume disk space.

**Pattern:** A lightweight log appended on every skill invocation -- slug, timestamp, trigger context. Weekly cron reads the log and produces a heat map:
- Skills with 0 invocations in 30 days: flagged for review/removal
- Skills invoked 10+ times/week: promoted in the available skills listing

**Pairs with the one-in-one-out rule:** Before adding a skill, identify what it replaces. Stack size is a liability.

---

## 6. Session Memory and Handoff

**Problem:** Each agent session starts cold. Without a continuity mechanism, context built up over a day of work disappears at reset.

**Pattern:** Three-tier memory architecture:

| Tier | File | Purpose | Updated |
|------|------|---------|---------|
| Same-day | `memory/state/session-handoff.md` | Active context bridge | Every checkpoint |
| Daily | `memory/daily/YYYY-MM-DD.md` | Raw log of events | Ongoing |
| Long-term | `MEMORY.md` | Curated decisions and context | Periodic review |

**Checkpoint protocol:** At 30%, 50%, and 70% context usage, dump a session summary to the daily file and overwrite `session-handoff.md`. New sessions read `session-handoff.md` first -- intra-day context restored in seconds without loading the full history.

**Nightly indexing:** A 3 AM cron re-indexes all memory files into the local search engine so previous-day context is searchable the next morning.

---

## 7. Unified Local Search

**Problem:** Context is spread across daily notes, project files, meeting transcripts, vault archives, and logs. Searching across all of them requires knowing where to look.

**Pattern:** A local BM25 + vector search engine indexes all collections. Single query interface across everything. Results include file paths and line numbers for verification.

**Collections (example structure):**
```
daily-logs          # daily memory files
projects            # project research and notes
operational-logs    # agent activity logs
meeting-transcripts # archived meeting recordings (local mirror)
vault-archive       # historical documents and notes
```

**Search priority:** Local index first. Live API for any collection that has a live counterpart only when local returns nothing useful.

---

## 8. Hardware Reference

**Recommended configuration for a fully active personal AI deployment:**

| Component | Spec | Role |
|-----------|------|------|
| Mac Mini M4 24GB | Always-on gateway | 24/7 agent host, cloud inference, scheduling |
| Mac Mini M4 Pro 48GB | Gateway + light local | Same plus 30B models |
| Mac Studio M4 Max 64GB | Gateway + heavy local | 70B models, eliminates most API overage |

**Power:** Mac Mini at ~7-20W continuous. The entire gateway infrastructure runs on less power than a laptop charger.

**Remote access:** Use a mesh VPN (Tailscale or equivalent) for secure access from any device. All served file links should use the VPN IP, not the LAN IP -- ensures access from anywhere, not just the local network.

**Node fleet:** Additional machines (Raspberry Pi, VPS, other workstations) can be paired as satellite nodes for specialized workloads, geographic redundancy, or always-on services that shouldn't compete with the primary gateway.

---

## Appendix: Skills Catalog

See [openclaw-skills-catalog.md](https://github.com/mmartoccia/shared-resources/blob/master/openclaw-skills-catalog.md) for the full list of installed skills with descriptions and hub links.

---

*Last updated: February 28, 2026*
*OpenClaw: github.com/openclaw/openclaw*
*Community: discord.com/invite/clawd*
