# Entropy and Anti-Detection: Running AI Agent Crons Without Behavioral Fingerprinting

## The Problem

AI agent cron jobs have a fundamental flaw: they run on exact schedules.

"Every hour at :00" is machine-obvious. Subscription API providers, rate limiters, and behavioral analysis systems are built to detect deterministic patterns. When your usage looks like a bot -- same minute every hour, same model every day -- you risk rate limiting, account flags, or service degradation.

The solution: make your agent's timing and model choices look like a human's. Irregular. Varied. Unpredictable within a defined envelope.

---

## Two-Layer Anti-Detection System

### Layer 1: Schedule Entropy (Time Randomization)

A daily "Entropy Daemon" runs at 2:15 AM and reshuffles the minute offset for every managed cron job.

Each job has:
- `base_expr`: which hours it runs (e.g., `0 3,9,15,21 * * *`)
- `minute_min` / `minute_max`: the randomization bounds
- The scheduler picks a collision-free random minute within those bounds

Result: a job nominally at "9 AM" might actually fire at 9:07 one day, 9:23 the next, 9:41 the day after. From a provider's perspective, this is indistinguishable from a human who checks their morning briefing when they get to their desk.

**Collision prevention:**
- Before assigning a slot, the scheduler checks all static/unmanaged jobs for occupied minutes
- Managed jobs are spaced at least 8 minutes apart (the stagger window) to avoid simultaneous API calls
- After 100 failed attempts to find a collision-free slot, falls back to default time with a warning logged

**Stagger window:** 8 minutes. If two managed jobs share an hour, they will be at least 8 minutes apart.

### Layer 2: Model Rotation (Persona Variation)

The entropy scheduler also rotates which AI model handles each job.

Each managed job has a `model_pool` -- a list of models it is allowed to use. The scheduler picks one daily.

Result: the same cron job uses Anthropic one day, Google the next. Different API endpoints. Different fingerprints. Different response characteristics. From a provider's perspective, your usage pattern matches a human who uses different tools for different moods.

**Why this matters beyond anti-detection:** Model rotation also provides natural load distribution across provider quotas. Research crons that would hammer Anthropic's rate limits instead spread across Gemini, spreading cost and increasing reliability.

### Model Timeout Buffering

CLI-backed models (e.g., `google-gemini-cli/*`) need longer timeouts than API-backed models. A direct Anthropic API call responds in seconds. A Gemini CLI invocation spawns a subprocess, authenticates, and may take 30+ seconds to start.

The entropy scheduler auto-applies minimum timeout buffers when it rotates a job to a CLI-backed model.

Config location: `MODEL_MIN_TIMEOUTS` dict in `entropy-scheduler.py`. Add entries here whenever you add a new CLI-backed model to any pool.

```python
MODEL_MIN_TIMEOUTS = {
    "google-gemini-cli/gemini-3-pro-preview": 180,
    "google-gemini-cli/gemini-3-flash": 180,
    # Add new CLI-backed models here
}
```

---

## Key Files

| File | Purpose |
|---|---|
| `~/clawd/scripts/entropy-scheduler.py` | The scheduler -- runs daily, reshuffles timing and models |
| `~/clawd/config/entropy-profiles.json` | Per-job config (base_expr, hours, minute bounds, model pools) |
| `~/clawd/memory/logs/entropy-schedule.log` | Daily run log -- SUCCESS/WARN per job |
| `~/.openclaw/cron/jobs.json` | OpenClaw's cron database -- entropy scheduler writes here |

The entropy cron itself: runs at `15 2 * * *`, model `gemini-3-pro-preview` (fast, cheap for a simple script runner).

---

## entropy-profiles.json Structure

```json
{
  "settings": {
    "stagger_minutes": 8,
    "max_attempts": 100
  },
  "jobs": {
    "<cron-job-uuid>": {
      "name": "ArXiv Watcher",
      "managed": true,
      "base_expr": "0 3,9,15,21 * * *",
      "hours": [3, 9, 15, 21],
      "minute_min": 0,
      "minute_max": 55,
      "tz": "America/New_York",
      "model_pool": ["claude-sonnet-4-6", "gemini-3-pro-preview"]
    },
    "<another-uuid>": {
      "name": "Daily Digest",
      "managed": false,
      "base_expr": "0 8 * * *",
      "hours": [8],
      "minute_min": 0,
      "minute_max": 0,
      "tz": "America/New_York",
      "model_pool": ["claude-sonnet-4-6"]
    }
  }
}
```

`"managed": false` skips a job entirely. Use this for jobs where timing is critical (e.g., a 9 AM meeting brief that needs to fire before 9 AM, not at 9:41).

---

## Setup Steps

1. Install `entropy-scheduler.py` to `~/clawd/scripts/`
2. Create `~/clawd/config/entropy-profiles.json` with job entries for each cron you want managed
3. For each managed job: get its UUID from `~/.openclaw/cron/jobs.json`, add an entry to `entropy-profiles.json`
4. Create the daily entropy cron in OpenClaw:
   - Schedule: `15 2 * * *`
   - Model: `gemini-3-pro-preview`
   - Payload: run `entropy-scheduler.py` and report a summary of what changed
5. Run manually first: `python3 ~/clawd/scripts/entropy-scheduler.py` -- verify log output shows SUCCESS for each job
6. Check `~/.openclaw/cron/jobs.json` to confirm minute offsets have changed
7. Monitor `~/clawd/memory/logs/entropy-schedule.log` daily for errors

---

## Operating Notes

- **"Could not find collision-free slot"** is a normal warning when the schedule is dense. Fallback to the default time is safe -- it just means that job did not get randomized that day.
- If you add a new CLI-backed model to any pool, add it to `MODEL_MIN_TIMEOUTS` in `entropy-scheduler.py` before the next entropy run.
- The entropy cron itself is `"managed": false` -- you do not want the daemon that reshuffles schedules to have its own schedule reshuffled.
- Check the log after the first week. If any job shows repeated WARN entries, either the schedule is too dense or the minute bounds are too narrow.

---

## What You Tell Your Users

> "Your agent's cron jobs don't fire at the same time every day. The schedule entropy system randomizes when each job runs within a defined window, and rotates which AI model handles each task. From a subscription provider's perspective, your usage looks like an engaged human power user, not a bot. This protects your account from behavioral flags and keeps your access reliable."

---

*Last updated: February 28, 2026*
