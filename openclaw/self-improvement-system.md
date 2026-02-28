# Self-Improvement System Architecture

An AI agent that can only do what it was configured to do at setup is a fixed tool. An agent that improves its own behavior based on outcomes, patterns, and operator feedback is infrastructure. This document specifies how to build the latter -- a system that learns from experience, consolidates patterns during off-peak cycles, prunes what's no longer useful, and pre-stages what's likely to be needed next.

**Prerequisite:** The trusted channel must be live before building any component of this system. Self-modification without a verified, cryptographically authenticated transport is a prompt injection vulnerability, not a feature. See [Trusted Channel Architecture](trusted-channel-architecture.md).

**Standing rule:** Before implementing any new component here, bring the design to the Council of the Wise. See [Council of the Wise](council-of-the-wise.md). The failure mode for self-improvement systems is not "it doesn't work" -- it's "it works in a direction you didn't intend."

---

## The Four Components

### 1. Hebbian Linking (Association)

**The neuroscience:** "Neurons that fire together wire together." Concepts that co-activate repeatedly become associated.

**The concrete mechanism (not a metaphor):**

When the agent successfully resolves a task, it writes a structured outcome record:

```json
{
  "timestamp": "2026-02-28T14:00:00Z",
  "task_type": "debug_cron",
  "context_tags": ["python", "falsy_zero", "boolean_chain"],
  "resolution": "check_or_operator_with_numeric_zero",
  "outcome": "success",
  "source_file": "memory/daily/2026-02-28.md#cron-fix-example"
}
```

A nightly linking pass scans outcome records and writes associations to `memory/system/hebbian-links.json`:

```json
{
  "python + boolean_chain": {
    "co_occurrences": 3,
    "resolution_pattern": "check_or_operator_with_numeric_zero",
    "confidence": 0.85,
    "source_outcomes": ["id-1", "id-3", "id-7"]
  }
}
```

**Retrieval:** Before attempting a task, query the hebbian index by tag. If confidence ≥ 0.7, surface the associated resolution pattern as a prior.

**What this is NOT:** It is not LLM fine-tuning. It is not weight modification. It is structured retrieval of prior successful resolutions -- a lookup table built from outcomes, not a black box.

**Key files:**
- `~/clawd/memory/system/hebbian-links.json` -- the association index
- `~/clawd/memory/system/outcome-records.jsonl` -- append-only outcome log
- `~/clawd/scripts/hebbian-linker.py` -- nightly linking pass

---

### 2. Sleep Consolidation (Nightly Distillation)

**The neuroscience:** During sleep, the hippocampus replays experiences and transfers important patterns to long-term cortical storage. Noise is discarded; signal is compressed.

**The concrete mechanism:**

A nightly cron (3:05 AM via trusted channel) runs the consolidation pipeline:

1. Reads today's daily log (`memory/daily/YYYY-MM-DD.md`)
2. Reads new outcome records added since last consolidation
3. Identifies patterns: recurring task types, recurring failures, recurring resolutions
4. Proposes MEMORY.md updates via `consolidate_daily_to_memory` (requires_approval, trusted_channel_required)
5. Writes a consolidation report to `memory/system/consolidation-log.jsonl`

**The approval gate:**

The consolidation does not write to MEMORY.md directly. It writes a proposal to `memory/state/tc-consolidate-pending.json`:

```json
{
  "proposed_additions": [
    {
      "section": "Lessons",
      "content": "Python `or` treats 0 as falsy -- check boolean chains with numeric zero explicitly.",
      "source": "memory/daily/2026-02-28.md",
      "confidence": 0.92
    }
  ],
  "proposed_removals": [
    {
      "section": "Open Strategy Threads",
      "entry": "Grafana Cloud -- safe to delete March 3",
      "reason": "Date passed, resolved"
    }
  ]
}
```

The main agent reviews this proposal at session start, applies approved changes, and logs the decision to the audit trail.

**What gets consolidated:**
- Lessons learned from resolved failures
- Decision rationale that should survive session resets
- Context about people and projects that is now stale
- Patterns identified in the hebbian linker

**What does NOT get consolidated:**
- Raw event logs (those stay in daily files)
- One-time events with no future relevance
- Anything in the blocked list of the consent graph

**Key files:**
- `~/clawd/scripts/tc-consolidate-daily.py` -- runs on .152, sends via trusted channel
- `~/clawd/memory/state/tc-consolidate-pending.json` -- pending proposals
- `~/clawd/memory/system/consolidation-log.jsonl` -- audit trail of what was consolidated

---

### 3. Synaptic Pruning (Decay and Removal)

**The neuroscience:** The brain eliminates synaptic connections that are rarely used. This is not degradation -- it is optimization. Removing noise makes signal retrieval faster and more accurate.

**The concrete mechanism:**

A weekly pruning pass (Sunday 3 AM) scans three targets:

**Target 1: Hebbian links**
Links with `co_occurrences < 2` and `last_seen > 60 days` are candidates for removal. The pruner writes a removal proposal, not an immediate delete.

**Target 2: MEMORY.md entries**
Entries marked `STATUS: Resolved` with a resolution date > 90 days ago are candidates for archival to `memory/archive/YYYY.md`.

**Target 3: Daily logs**
Daily logs older than 30 days are compressed: the raw file is replaced with a 10-line summary. The QMD index retains searchability; the full text is moved to `memory/archive/daily/`.

**The approval gate:**

Same pattern as consolidation -- pruning proposes, never executes autonomously:

```json
{
  "prune_candidates": [
    {
      "type": "hebbian_link",
      "key": "vb6 + migration",
      "reason": "co_occurrences=1, last_seen=2025-11-14, confidence=0.3",
      "proposed_action": "delete"
    },
    {
      "type": "memory_entry",
      "section": "DECISIONS",
      "entry_id": "grafana-cloud-2026-02",
      "reason": "STATUS: Resolved, resolved > 90 days ago",
      "proposed_action": "archive"
    }
  ]
}
```

**What is never pruned:**
- The audit log (`trusted-channel-audit.jsonl`) -- append-only, immutable, not subject to pruning
- Entries in the consent graph blocked list
- Any outcome record referenced by an active hebbian link with confidence ≥ 0.7

**Key files:**
- `~/clawd/scripts/synaptic-pruner.py` -- weekly pruning pass
- `~/clawd/memory/state/tc-prune-request.json` -- pending pruning proposals
- `~/clawd/memory/archive/` -- destination for archived content

---

### 4. Predictive Staging (Pre-loading)

**The neuroscience:** The brain does not retrieve memories on demand -- it pre-activates likely-relevant context based on situational cues. You walk into a kitchen and kitchen-related knowledge becomes more accessible.

**The concrete mechanism:**

At session start, the restore_context script (running on .152, sent via trusted channel) pre-stages context beyond just the session handoff:

1. Checks the calendar for events in the next 4 hours
2. Queries QMD for documents related to those events
3. Checks the hebbian index for tags associated with the current day's known task types
4. Assembles a staged context block and sends it via `restore_context` trusted channel action

The staged context lands in `memory/state/tc-restore-pending.md` and is loaded at session start before any user interaction.

**Example staged context:**
```
Pre-staged for 2026-02-28 session:
- Project review meeting at 2 PM → loaded project-brief-draft.md summary
- Hebbian: "project X + proposal draft" → prior: "[owner] manages shared doc, owner@example.com"
- Hebbian: "python + cron" → prior: "check MODEL_MIN_TIMEOUTS when adding CLI-backed models"
- Open from handoff: Berkeley Electric bill due today
```

**What this is NOT:** It is not the agent making decisions before being asked. It is retrieving relevant context early so that when the user asks, the agent already has the right information loaded -- not searching for it in the middle of a response.

**Key files:**
- `~/clawd/scripts/tc-restore-context.py` -- runs on .152 every 30 min
- `~/clawd/memory/state/tc-restore-pending.md` -- staged context landing zone
- `~/clawd/config/staging-rules.json` -- rules for what to pre-stage and when

---

## The Five Blockers (What This System Needs to Work)

The Council identified five things that must exist before any component of this system is reliable. These are not optional:

| Blocker | Why it matters | Resolution |
|---|---|---|
| **Outcome signal** | Without knowing whether a task succeeded or failed, you can't build outcome records. The hebbian linker has nothing to learn from. | Every resolved task writes a structured outcome record. Failures are explicit, not inferred. |
| **Modification rights governance** | Self-modification without an authorization layer is prompt injection waiting to happen. | Consent graph `self_modification` domain + trusted channel requirement. Must be live before any other component. |
| **Cross-session pattern recognition** | Single-session patterns are noise. Multi-session patterns are signal. | QMD index + hebbian linker running across all daily logs, not just the current session. |
| **Compression layer** | Raw logs accumulate without bound. After 30 days, the daily log directory becomes a retrieval liability. | Synaptic pruning + daily log compression + archive pipeline. |
| **Hypothesis testing** | Without a way to evaluate whether a behavioral change actually improved outcomes, the system can drift in any direction. | Outcome records + consolidation log. Every MEMORY.md update from consolidation gets tagged with its source outcome records. If you add a lesson and the same failure recurs, the lesson was wrong -- update it. |

---

## What the Council Found

The Council review (Gemini security red-team + Opus 4-perspective) flagged three things worth building explicitly into the architecture:

**1. Neuroscience metaphors must not substitute for boring infrastructure.**
"Hebbian linking" is a useful frame, but it must resolve to: a JSON file, a cron script, a lookup function, and an approval gate. If any of those four are missing, the component doesn't exist -- it's just a description.

**2. Same-machine privilege separation is cosmetic without OS-level enforcement.**
The consent graph's `modify_consent_graph` blocked entry is a soft rule unless the file is actually locked. `chflags uchg` (macOS) or `chattr +i` (Linux) is what makes the block real. Apply it to: `consent-graph.json`, `SOUL.md`, `AGENTS.md`, and the audit log.

**3. The human gate is not a bureaucratic step -- it is the system's error-correction mechanism.**
Behavioral proposals that bypass human review remove the only check on whether the system is improving in a direction the operator actually wants. The cooling period (2 hours), the diff requirement, and the undo window (7 days) are not friction -- they are the feedback loop.

---

## Build Order

If you're implementing from scratch, build in this sequence:

1. **Audit logging** -- before anything else; you need a record of what happened
2. **Consent graph + OS-level locks** -- before any self-modification component
3. **Trusted channel** -- before any cross-machine self-modification
4. **Outcome records** (append-only JSONL) -- before hebbian linker
5. **Sleep consolidation** (nightly distillation) -- first self-modification component; lowest risk
6. **Hebbian linker** -- depends on outcome records; adds retrieval value
7. **Synaptic pruning** -- depends on consolidation; reduces noise
8. **Predictive staging** -- depends on hebbian index + QMD; highest value, highest complexity

Do not skip steps. Do not build 6 before 4. The system fails silently if the dependencies are missing.

---

## What You Tell Your Users

> "Your agent gets better over time. When it resolves a problem, it records how. A nightly process distills those records into patterns. The patterns inform what gets pre-loaded next session. Every week, stale patterns are proposed for removal. None of this modifies your agent's behavior without your review -- every proposed change requires your approval, has a 2-hour cooling period, and can be reversed for 7 days. The improvement loop is real; the human gate is also real."

---

## Key Files (Full Map)

| File | Purpose |
|---|---|
| `~/clawd/memory/system/outcome-records.jsonl` | Append-only task outcome log |
| `~/clawd/memory/system/hebbian-links.json` | Association index (tag pairs → resolution patterns) |
| `~/clawd/memory/system/consolidation-log.jsonl` | Audit trail of consolidation decisions |
| `~/clawd/memory/state/tc-consolidate-pending.json` | Pending consolidation proposals (awaiting approval) |
| `~/clawd/memory/state/tc-prune-request.json` | Pending pruning proposals (awaiting approval) |
| `~/clawd/memory/state/tc-restore-pending.md` | Pre-staged context for session start |
| `~/clawd/memory/archive/` | Compressed old content (logs, resolved decisions) |
| `~/clawd/scripts/hebbian-linker.py` | Nightly association pass |
| `~/clawd/scripts/tc-consolidate-daily.py` | Nightly consolidation (runs on .152, sends via TC) |
| `~/clawd/scripts/synaptic-pruner.py` | Weekly pruning pass |
| `~/clawd/scripts/tc-restore-context.py` | Session pre-staging (runs on .152 every 30 min) |
| `~/clawd/config/staging-rules.json` | Pre-staging rules (what to load, when) |
| `~/clawd/memory/system/trusted-channel-audit.jsonl` | Immutable audit log (never pruned) |
