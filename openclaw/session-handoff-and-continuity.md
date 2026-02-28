# Session Handoff and Continuity

## What it is

AI agent sessions have a hard constraint: context windows are finite. When a session compacts or resets, the agent loses everything that wasn't persisted to files. Session handoff is the system that ensures continuity -- the agent wakes up in the next session knowing what it was doing, what decisions were made, and what's pending.

## The three-tier memory architecture

```
Session handoff (working memory)     -- intra-day, hours
  ↓ distilled at session end
Daily logs (episodic memory)         -- days to weeks
  ↓ distilled during memory maintenance
MEMORY.md (semantic memory)          -- months to years
```

## Session handoff (memory/state/session-handoff.md)

- The intra-day bridge -- overwrites on each checkpoint, not appended
- Written at 30%, 50%, and 70% context usage thresholds
- Contains: what we're working on, decisions made, open threads, any user requests still pending
- New session reads this BEFORE greeting -- it's the fastest way to restore intra-day context
- Reset/cleared at the start of each new calendar day

**Checkpoint protocol:**
- At 30% context: write session summary to daily log, update handoff
- At 50% context: update handoff with current state
- At 70% context: update handoff, proactively suggest /new reset to user
- At 75%: always suggest reset

**Handoff file format:**

```markdown
# Session Handoff -- YYYY-MM-DD

**Last updated:** HH:MM EST
**Session:** Main (Telegram)

## Critical -- Action Required
- [time-sensitive items]

## Infrastructure State
- [anything running, deployed, or in an intermediate state]

## Active Work
### [Project Name]
- Status: ...
- Next step: ...

## Pending Backlog
- [items not yet started]
```

## Daily logs (memory/daily/YYYY-MM-DD.md)

- Raw session notes -- append-only during the day
- Created at session start if it doesn't exist
- Captures: decisions made, actions taken, things learned, open threads
- Not curated -- this is the unfiltered record

## MEMORY.md (long-term semantic memory)

- Curated, distilled -- not a log
- Contains: decisions (with rationale and revisit dates), key context about people/projects, lessons learned, standing preferences
- Updated during memory maintenance (rotational heartbeat check ~daily)
- Never grows unboundedly -- outdated entries are removed or compressed

## The nightly consolidation pipeline

1. **3 AM: QMD index refresh** -- indexes all markdown files in ~/clawd for BM25 + vector search
2. **3:05 AM: TC consolidate** (trusted channel, from .152) -- sends daily log summary to main agent for MEMORY.md distillation
3. **Session archiver cron** -- archives old session transcripts to keep sessions/ directory lean

## How context survives compaction

When a session compacts:
1. The pre-compaction hook fires: agent writes current state to daily log + session handoff
2. New session starts fresh
3. Agent reads session-handoff.md before responding
4. Trusted channel restore_context fires (from .152 via tc-restore-context.py) with handoff content

## What you tell your users

> "Your agent remembers what it was doing. When a session resets, it reads a handoff file before responding -- so it knows what you were working on, what decisions you made, and what's still pending. The nightly process distills that day's work into long-term memory. You don't have to re-explain your context every morning."

## Setup steps

1. Create `~/clawd/memory/state/session-handoff.md` (empty initially)
2. Create `~/clawd/memory/daily/` directory
3. Add to agent startup protocol: read session-handoff.md before greeting
4. Add context checkpoint rule: at 30/50/70% usage, update handoff + daily log
5. Set up nightly QMD index cron: `0 3 * * *`, runs `qmd index ~/clawd`
6. (Optional) Wire trusted channel restore_context to automatically send handoff content on each tc-restore run

## Operating notes

- Handoff file overwrites each checkpoint -- it's the current state snapshot, not a log
- Daily log appends -- it's the full record
- MEMORY.md is curated -- don't let it grow unbounded; prune quarterly
- `qmd query "..."` is the retrieval interface for previous-day and older context
