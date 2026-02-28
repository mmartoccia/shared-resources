# Mission Control Dashboard

**Start here if:** You want a real-time Kanban view of everything your agent fleet is doing -- active sub-agents, pending tasks, recent completions -- without adding a database or external service to your stack.

**The problem it solves:**

When an agent is running multiple sub-agents in parallel, managing a backlog of tasks, and operating on a cron schedule, you lose visibility. You don't know what's in flight, what's blocked, or what completed while you weren't looking. Chat history is the wrong interface for operational status.

**The solution in one sentence:**

A Next.js app scans your markdown files for task blocks every 30 seconds and renders them as a live Kanban board -- the agent writes tasks to files, Mission Control reads them, no database required.

**Core insight:** The agent's task state lives in markdown files -- not a database. Mission Control reads those files directly, which means any file the agent edits is automatically reflected in the dashboard. The agent is the source of truth; Mission Control is the view.

**What you tell your users:**

> "Mission Control is your agent's task board. Every task the agent is working on -- or plans to work on -- is tracked in plain markdown files. The dashboard reads those files and shows you a live Kanban view. When a sub-agent completes something, it marks the task done; you see it update in real time. No database, no sync issues -- the files are the source of truth, the dashboard is just the view."

---

## Architecture

```
~/clawd/**/*.md                     Mission Control (Next.js)
  (task blocks in YAML)    -->      localhost:3001
  | scanned every 30s               |-- /          (Kanban board)
  parser.ts                         |-- /api/tasks  (task index)
  taskIndex.ts                      |-- /api/health (system health)
  |                                 +-- /api/activity (recent events)
  in-memory cache
```

The scanner walks `~/clawd/` recursively on a 30-second interval, finds HTML comment markers, parses the YAML blocks that follow them, and updates an in-memory task index. The Kanban board polls the API and re-renders on change.

No write path from dashboard to disk -- the dashboard is read-only. Agents write to files; the PATCH API updates the in-memory index for immediate UI consistency (see race condition note below).

---

## Task Block Format

Tasks are fenced YAML blocks embedded in any markdown file under `~/clawd/`. The block is identified by an HTML comment prefix on the line immediately before the code fence:

```
<!-- TASK id=my-task uid=01JMQX7A1B2C3D4E5F6G7H8J9K -->
```yaml
task_schema_version: 1
id: my-task
uid: 01JMQX7A1B2C3D4E5F6G7H8J9K
title: "My task title"
status: backlog
owner: main
priority: high
hours: 2
tags: [infrastructure, security]
blockers: []
created: '2026-02-28T14:00:00Z'
updated: '2026-02-28T14:00:00Z'
```

Optional prose description after the block. This text is displayed as the task description in the dashboard.

**Required fields:** `task_schema_version`, `id`, `uid`, `title`, `status`, `owner`, `priority`, `created`, `updated`

**Field reference:**

| Field | Values | Notes |
|---|---|---|
| `status` | `backlog` \| `todo` \| `in-progress` \| `done` \| `blocked` | Controls Kanban column |
| `owner` | `main` \| `ops` \| `architect` \| `mike` \| sub-agent label | Who is responsible |
| `priority` | `high` \| `medium` \| `low` | Affects card sort order |
| `hours` | integer | Estimated effort (optional) |
| `tags` | YAML list | Free-form; used for filtering |
| `blockers` | YAML list | Blocking items; non-empty implies `blocked` status |
| `uid` | ULID or UUID | Must be globally unique across all task files |

**The `uid` field is the primary key.** The `id` field is a human-readable slug. Both are required. Use ULID format (`01JMQX7A...`) for sortable UIDs; UUID v4 is acceptable.

---

## Common Task File Locations

Any file in `~/clawd/` is scanned. Common patterns:

| Location | Use case |
|---|---|
| `~/clawd/TASK_QUEUE.md` | Persistent backlog -- tasks that survive session resets |
| `~/clawd/memory/projects/work/session-YYYY-MM-DD-tasks.md` | Session-specific task files |
| `~/clawd/memory/projects/*/tasks.md` | Project-specific task tracking |

The agent is free to create task blocks in any markdown file. The scanner finds them all.

---

## API Endpoints

```
GET  /api/tasks           -- full task index
GET  /api/tasks?status=in-progress&owner=main&tag=security&priority=high
                          -- filtered task index
GET  /api/health          -- system health check
GET  /api/activity        -- recent task state changes (worklog)
PATCH /api/tasks/:uid     -- update task status/fields
```

**PATCH request body:**
```json
{
  "status": "done",
  "updated": "2026-02-28T18:00:00Z"
}
```

Any top-level task field can be patched. The endpoint validates `status` against the allowed set.

**Race condition note (do not change this behavior):** The PATCH endpoint updates the in-memory index directly from its own response. It does NOT trigger a disk re-scan. This is intentional: a re-scan is async and could return stale data, overwriting a status change the agent just made. If you extend the PATCH handler, preserve this pattern.

---

## Setup

**Prerequisites:** Node.js 18+, npm

**Steps:**

1. Clone or copy the mission-control app to `~/clawd/mission-control/`
2. Install dependencies:
   ```bash
   cd ~/clawd/mission-control
   npm install
   ```
3. Build:
   ```bash
   npm run build
   ```
4. Start:
   ```bash
   npm start -- --port 3001
   ```
   Or add to launchd for persistence (see below).
5. Verify:
   ```bash
   curl http://localhost:3001/api/health
   # {"status":"ok"}
   ```
6. Create your first task file: `~/clawd/TASK_QUEUE.md` with a task block (see format above)
7. Verify task appears: open http://localhost:3001

**Development mode** (hot reload, verbose logging):
```bash
npm run dev -- --port 3001
```

**launchd plist** for persistence on macOS (save to `~/Library/LaunchAgents/ai.openclaw.mission-control.plist`):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>ai.openclaw.mission-control</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/local/bin/node</string>
    <string>/Users/you/clawd/mission-control/.next/standalone/server.js</string>
  </array>
  <key>EnvironmentVariables</key>
  <dict>
    <key>PORT</key>
    <string>3001</string>
  </dict>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <true/>
  <key>StandardOutPath</key>
  <string>/tmp/mission-control.log</string>
  <key>StandardErrorPath</key>
  <string>/tmp/mission-control.err</string>
</dict>
</plist>
```
Load with: `launchctl load ~/Library/LaunchAgents/ai.openclaw.mission-control.plist`

---

## Kanban Board

The board renders four columns based on task status:

| Column | Status values | Notes |
|---|---|---|
| **Backlog** | `backlog`, `todo` | Not yet started |
| **In Progress** | `in-progress` | Actively being worked |
| **Blocked** | `blocked` | Waiting on something; `blockers` field is displayed |
| **Done** | `done` | Complete; auto-archived after 7 days |

Cards sort by priority within each column (high -> medium -> low). Stale tasks (not updated in 30 days) are flagged with a visual indicator.

---

## How the Agent Uses It

**Creating a task:** The agent writes a task block into any markdown file.

**Updating a task:** The agent edits the YAML block directly (status, updated timestamp). Dashboard picks up the change within 30 seconds via the next scan cycle.

**Via PATCH API** (for immediate UI reflection, no wait for scan):
```bash
curl -X PATCH http://localhost:3001/api/tasks/01JMQX7A1B2C3D4E5F6G7H8J9K \
  -H "Content-Type: application/json" \
  -d '{"status": "done", "updated": "2026-02-28T18:00:00Z"}'
```

---

## Sub-Agent Integration

When the main agent spawns a sub-agent for a task:

1. Create a task block with `status: in-progress`, `owner: <sub-agent-label>` (e.g., `owner: spec-mission-control`)
2. Sub-agent completes work, auto-announces result to requester session
3. Main agent updates task to `status: done` via PATCH or direct file edit
4. Dashboard reflects completion within 30 seconds

The `owner` field doubles as a sub-agent label -- when a task is owned by a named sub-agent, it's visible in the dashboard as that sub-agent's work.

---

## Operating Notes

- **Scan interval:** 30 seconds (configurable in `taskIndex.ts` via `SCAN_INTERVAL_MS`)
- **Stale threshold:** Tasks not updated in 30 days are flagged in the UI
- **UID uniqueness:** `uid` must be unique across all task files -- duplicates cause the second task to overwrite the first in the index
- **Marker format:** The HTML comment `<!-- TASK id=... uid=... -->` must be on the line immediately before the opening code fence
- **Prose description:** Markdown text after the closing code fence (before the next TASK marker or end of file) is captured as the task description
- **PATCH is authoritative:** The PATCH endpoint's response is the source of truth for in-memory state -- do not add a disk re-fetch after PATCH without understanding the race condition

---

## Extending Mission Control

To add new fields to task blocks:

1. Add the field to the YAML in your task file
2. Add it to the TypeScript types in `src/types.ts`
3. Add it to the parser in `src/parser.ts`
4. Optionally surface it in the UI

Custom fields are preserved through parse/render cycles even if not displayed. The parser stores the full parsed YAML object, so unrecognized fields are not dropped.

---

## Implementation Files

| File | Purpose |
|---|---|
| `src/parser.ts` | Scans markdown files, extracts task blocks, parses YAML |
| `src/taskIndex.ts` | Manages in-memory task cache, scan loop, change detection |
| `src/types.ts` | TypeScript interfaces for Task and related types |
| `src/app/page.tsx` | Kanban board UI |
| `src/app/api/tasks/route.ts` | GET /api/tasks (index + filtering) |
| `src/app/api/tasks/[uid]/route.ts` | PATCH /api/tasks/:uid |
| `src/app/api/health/route.ts` | GET /api/health |
| `src/app/api/activity/route.ts` | GET /api/activity (worklog) |

---

*Part of the [OpenClaw Implementation Guide](README.md)*
