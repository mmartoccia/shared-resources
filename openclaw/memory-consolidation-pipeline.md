# Memory Consolidation Pipeline

## What it is

The memory consolidation pipeline is how an AI agent builds genuine long-term memory. Raw session logs are too noisy to use directly; MEMORY.md becomes useless if it grows unbounded. The pipeline distills experience across three tiers -- episodic to semantic -- the same way biological memory works during sleep.

## The three tiers

```
Daily Logs            Episodic Memory
~/clawd/memory/       Raw session notes, append-only, one file per day.
daily/YYYY-MM-DD.md   What happened, decisions made, actions taken, things learned.
                      Not curated. The unfiltered record.
        ↓
        Memory Maintenance (rotational heartbeat, ~daily)
        ↓
MEMORY.md             Semantic Memory
~/clawd/MEMORY.md     Curated, distilled. Decisions with rationale. Key context
                      about people, projects, preferences. Lessons learned.
                      Updated by pruning AND adding. Not a log.
        ↓
        QMD Nightly Index (3 AM)
        ↓
QMD Search Index      Retrieval Layer
~/clawd/.qmd/         BM25 + vector search across all memory files.
                      `qmd query "..."` for previous-day and older context.
```

## The nightly pipeline (what actually runs)

**3:00 AM -- QMD Index Refresh** (`qmd index ~/clawd`)
- Indexes all markdown in ~/clawd for full-text + semantic search
- Enables `qmd query "what did we decide about X"` to retrieve any prior context

**3:05 AM -- TC Consolidate** (trusted channel from .152)
- Reads today's daily log
- Sends `consolidate_daily` message through trusted channel
- Main agent distills the day's log into MEMORY.md updates
- Rule: only add entries that are worth remembering in 6 months; don't add noise

**Session Archiver** (3:21 AM daily)
- Archives old session transcripts to keep ~/.openclaw/agents/main/sessions/ lean
- Prevents session directory from growing to GB scale

## Memory maintenance (rotational heartbeat check)

Every few days during a heartbeat, the agent:
1. Reads recent daily logs (last 3-7 days)
2. Identifies significant events, lessons, insights worth long-term retention
3. Updates MEMORY.md with distilled learnings
4. Removes outdated entries that no longer apply

The daily/MEMORY.md distinction:
- **Daily logs:** raw notes. "Spent 2 hours debugging a cron script. Root cause was Python `or` treating 0 as falsy."
- **MEMORY.md:** distilled. "Cron scripts: check Python falsy behavior with numeric zero in boolean chains. `or` falls through on 0."

## What goes in MEMORY.md

Use the DECISIONS template for significant choices:

```markdown
- **DECISION:** What decision or architectural choice is this?
- **DATE:** YYYY-MM-DD
- **WHY:** 1-2 sentence rationale
- **IMPACTS:** Which systems/people are affected?
- **REVISIT:** Date or condition for reconsideration
- **STATUS:** Open | Implemented | Deferred | Rejected
```

Always include:
- Architectural decisions with rationale
- User preferences and communication style notes
- Lessons learned from failures
- Key context about people (roles, communication preferences, relationship to user)
- Active project status and next steps

Never include:
- Raw session transcripts (that's daily logs)
- One-time events with no future relevance
- Sensitive credentials or secrets

## Retrieval

```bash
# Search all memory for prior context
qmd query "what did we decide about the trusted channel"

# Search specific store
qmd query --store limitless-2026 "project kickoff meeting"

# Full text search
qmd query --mode bm25 "Berkeley Electric"
```

## What you tell your users

> "Your agent builds genuine memory over time. Every night, the day's raw notes are indexed for search and distilled into long-term memory. Six months from now, if you ask about a decision you made today, the agent can retrieve it. The daily logs are the raw record; MEMORY.md is the curated wisdom. The nightly pipeline is your agent's sleep cycle -- consolidating what matters, discarding what doesn't."

## Setup steps

1. Create `~/clawd/memory/daily/` and `~/clawd/MEMORY.md` (with DECISIONS template)
2. Install QMD: `npm install -g @qmd/cli` (or equivalent)
3. Set up nightly QMD index cron: `0 3 * * *` → `qmd index ~/clawd`
4. Configure memory maintenance in HEARTBEAT.md as a rotational check (~daily)
5. Define MEMORY.md sections: DECISIONS, Key Context, Family, Projects, Open Strategy Threads
6. (Optional) Wire trusted channel consolidate_daily for automated nightly distillation
7. Establish the rule: daily logs are append-only; MEMORY.md is curated and pruned

## Operating notes

- MEMORY.md should stay under ~1000 lines -- beyond that, create overflow files (memory/extras.md)
- Review and prune MEMORY.md quarterly -- remove decisions that have been fully resolved
- QMD stores are named collections: `limitless-2026`, `limitless-archive`, `obsidian-archive`
- `qmd query` returns top snippets with source file + line numbers for verification
