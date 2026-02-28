# Heartbeat Protocol

## What it is

The heartbeat is OpenClaw's proactive monitoring system. Instead of waiting for the user to ask "anything urgent?", the agent runs a structured check every ~30 minutes and surfaces what matters. It is the difference between a reactive assistant and a proactive one.

## How it works (production implementation)

- OpenClaw sends a heartbeat poll message to the main agent on a configurable interval (every 30 min by default)
- The agent reads HEARTBEAT.md -- a user-editable file defining what to check
- The agent runs the checks, evaluates results, and either replies with `HEARTBEAT_OK` (nothing needs attention) or sends a substantive alert
- Key principle: **quality over quantity**. If nothing needs surfacing, the agent says nothing. No "all clear" messages.

## HEARTBEAT.md structure (actual production format)

The file has numbered sections:

```markdown
## 0. [Priority watch] (Every Run)
Critical items that always get checked -- e.g., active geopolitical situation, pending deadlines

## 1. Critical Checks (Every Run)
- Calendar: next 2h, alert if event <15 min, skip all-day events
- VIP Email: unread from priority contacts in last 4h
- iMessage: check VIP chat IDs for inbound <30 min

## 2. Quota Check (every 2h)
- Run: ~/clawd/scripts/claude-usage-check.sh --alert-only
- Alert if output non-empty (>=80% quota)

## 3. Rotational Checks (>4h since last -> do ONE per heartbeat)
- Limitless sync: fetch last 5 lifelogs, extract signals
- Home Assistant anomaly check
- Memory maintenance (~daily)

## 4. Idle Work
- Check TASK_QUEUE.md for top uncompleted item, work one task

## 5. Housekeeping
- State file: memory/state/heartbeat-state.json
- Nothing needs attention -> HEARTBEAT_OK
```

## Heartbeat state file

`~/clawd/memory/state/heartbeat-state.json` tracks last check time per domain:

```json
{
  "lastChecks": {
    "email": 1703275200,
    "calendar": 1703260800,
    "imessage": 1703275200,
    "limitless_sync": 1703260800,
    "home_assistant": null,
    "memory_maintenance": 1703260800,
    "claude_quota": 1703275200
  },
  "last_imsg_seen": {
    "+1XXXXXXXXXX": "msg-id-xyz"
  }
}
```

## When to alert vs. stay quiet

**Alert when:**
- Important email arrived from VIP contact
- Calendar event <2h away
- Something new discovered during rotational checks
- Quota approaching limit

**Stay quiet (HEARTBEAT_OK) when:**
- Late night (23:00-08:00) unless urgent
- Human is clearly busy (in a meeting)
- Nothing new since last check
- Checked <30 min ago

## Heartbeat vs. Cron: when to use each

| Use heartbeat | Use cron |
|---|---|
| Multiple checks that batch together | Exact timing matters |
| Need conversational context from recent messages | Task needs isolation |
| Timing can drift slightly | One-shot reminders |
| Reduce API calls by combining periodic checks | Output delivers directly to channel |

## What you tell your users

> "Your agent doesn't wait to be asked. Every 30 minutes, it checks what matters -- your calendar, email from people who matter, messages from family. If something needs your attention, it tells you. If nothing does, it says nothing. You set the rules in a plain text file; the agent follows them."

## Setup steps

1. Create `~/clawd/HEARTBEAT.md` with your check definitions (see structure above)
2. Create `~/clawd/memory/state/heartbeat-state.json` with empty lastChecks object
3. In OpenClaw config, set heartbeat prompt: `"Read HEARTBEAT.md if it exists. Follow it strictly. If nothing needs attention, reply HEARTBEAT_OK."`
4. Set heartbeat interval in openclaw.json (default: 1800000ms = 30 min)
5. Test: trigger a heartbeat manually and verify HEARTBEAT_OK response when nothing is pending
6. Add VIP contacts to the iMessage check section with their chat IDs
7. Add any time-sensitive recurring items (bill due dates, medication reminders) to Section 0

## Operating notes

- Keep HEARTBEAT.md small -- every heartbeat loads it. Larger file = more tokens per check.
- Use rotational checks for expensive operations (Limitless API, Home Assistant) -- not every heartbeat
- The `HEARTBEAT_OK` response is special -- OpenClaw may discard it from conversation history to reduce noise
- Idle work (Section 4) is how the agent does useful background work between conversations
