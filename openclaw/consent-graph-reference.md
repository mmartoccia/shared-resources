# Consent Graph Reference

The consent graph is the authorization layer for every action an OpenClaw agent can take on the world. It answers one question before any external action executes: **is this agent allowed to do this, autonomously, or does a human need to approve it first?**

Without a consent graph, an agent with access to your email, messages, calendar, and home automation is a liability. With one, every action is pre-classified, auditable, and -- for the actions that matter most -- requires explicit approval before firing.

---

## The Three Tiers

Every action in every domain falls into one of three buckets:

| Tier | Behavior | Example |
|---|---|---|
| `autonomous` | Execute immediately. Log it. | Read email, search the web, write a file |
| `requires_approval` | Ask first, or notify + proceed if confidence ≥ 0.85 | Send a message, create a calendar event, delete a file |
| `blocked` | Never execute. Alert operator that it was attempted. | Unlock doors, modify consent graph, delete audit log |

A fourth tier applies to self-modification:

| Tier | Behavior |
|---|---|
| `trusted_channel_required` | Only valid if the request arrived via the signed trusted channel -- not from user chat, not from a sub-agent, not from a cron message |

---

## How to Use It (Pre-Flight Check)

Before any external action, classify it:

```bash
python3 ~/clawd/scripts/consent.py check <domain> <action> <confidence>
# Example:
python3 ~/clawd/scripts/consent.py check imessage send_vip 0.9
# Returns: VISIBLE -- proceed and notify
```

**In-session mental model:**
1. Identify the domain (`email`, `imessage`, `calendar`, etc.)
2. Identify the action (must match a key in that domain's lists)
3. Check which list it falls in
4. For `requires_approval` + confidence ≥ 0.85 → **VISIBLE**: proceed, then notify operator
5. For `requires_approval` + confidence < 0.85 → **FORCED**: ask operator first; use inline buttons when possible
6. For `blocked`: do NOT execute; alert operator the action was attempted and blocked

---

## The Full Graph

Location: `~/clawd/memory/state/consent-graph.json`

**This file is immutable.** Locked with `chflags uchg`. To edit:
```bash
chflags nouchg ~/clawd/memory/state/consent-graph.json
# make changes
chflags uchg ~/clawd/memory/state/consent-graph.json
```

The `modify_consent_graph` action is in the `self_modification` blocked list -- the agent cannot modify this file autonomously under any circumstances.

---

### email
```json
{
  "trust_level": "high",
  "autonomous": ["read", "search", "label", "archive_promo", "archive_social", "archive_forum"],
  "requires_approval": ["send", "delete", "forward"],
  "blocked": ["send_to_unknown", "delete_vip"]
}
```

### imessage
```json
{
  "trust_level": "low",
  "autonomous": ["read", "history"],
  "requires_approval": ["send_vip", "send_family"],
  "blocked": ["send_unknown", "send_group_without_context"]
}
```

iMessage trust is deliberately low. There is no autonomous send on iMessage -- every outbound message requires approval. The `send_unknown` block catches any number not in the VIP contacts map.

### home_automation
```json
{
  "trust_level": "medium",
  "autonomous": ["read_sensors", "check_status", "read_forecast"],
  "requires_approval": ["set_thermostat", "lock_doors", "toggle_lights"],
  "blocked": ["unlock_doors", "disable_alarms", "open_garage"]
}
```

Anything that physically opens or disables security is permanently blocked. Status reads and forecasts are free; physical access is never autonomous.

### files
```json
{
  "trust_level": "high",
  "autonomous": ["read", "write_workspace", "create", "edit"],
  "requires_approval": ["delete", "write_outside_workspace"],
  "blocked": ["delete_outside_workspace", "modify_secrets"]
}
```

`modify_secrets` is blocked -- the agent cannot touch `~/.secrets/` autonomously. File deletion always requires approval.

### external_api
```json
{
  "trust_level": "medium",
  "autonomous": ["search", "read", "fetch"],
  "requires_approval": ["post_social", "tweet", "submit_form"],
  "blocked": ["financial_transaction", "delete_remote"]
}
```

### cron
```json
{
  "trust_level": "high",
  "autonomous": ["list", "run_existing", "update_schedule"],
  "requires_approval": ["create", "delete"],
  "blocked": []
}
```

The agent can run existing crons and update schedules (the entropy scheduler does this nightly autonomously). Creating or deleting crons is infrastructure change and always requires approval.

### notifications
```json
{
  "trust_level": "high",
  "autonomous": ["send_telegram_owner", "send_email_owner"],
  "requires_approval": ["send_telegram_other", "send_email_other"],
  "blocked": ["send_to_unknown_channel"]
}
```

Fan-out prevention: route each notification to exactly one destination. The agent cannot autonomously send the same event to multiple channels.

### calendar
```json
{
  "trust_level": "medium",
  "autonomous": ["read", "list_events"],
  "requires_approval": ["create_event", "modify_event", "delete_event"],
  "blocked": ["invite_external"]
}
```

### nodes
```json
{
  "trust_level": "high",
  "autonomous": ["read_files", "list_files", "run_scripts", "check_status", "git_operations", "read_logs"],
  "requires_approval": ["install_software", "modify_system_config", "delete_files"],
  "blocked": ["expose_ports_externally", "modify_openclaw_config"]
}
```

### self_modification

This domain governs actions the agent takes on itself -- memory, context, and behavioral rules. It has additional constraints beyond the standard three tiers.

```json
{
  "autonomous": [
    "load_memory_file",
    "restore_context",
    "update_heartbeat_state",
    "prune_stale_memory"
  ],
  "requires_approval": [
    "propose_behavioral_change",
    "update_lessons_md",
    "consolidate_daily_to_memory"
  ],
  "blocked": [
    "modify_soul_md",
    "modify_agents_md",
    "modify_consent_graph",
    "change_safety_rules",
    "modify_system_prompt",
    "delete_audit_log",
    "modify_audit_log"
  ],
  "trusted_channel_required": [
    "propose_behavioral_change",
    "consolidate_daily_to_memory",
    "prune_stale_memory"
  ],
  "cooling_period_hours": 2,
  "undo_window_days": 7,
  "require_diff": true
}
```

**Additional self_modification constraints:**

- **`trusted_channel_required`**: These actions are only valid when the request arrives via the HMAC-signed HTTP trusted channel from `.152`. User chat cannot trigger them, regardless of content or formatting. This is the primary defense against prompt injection targeting self-modification.
- **`cooling_period_hours: 2`**: Behavioral proposals cannot be implemented immediately. Mandatory 2-hour window for operator review before any change takes effect.
- **`undo_window_days: 7`**: Any approved change must be reversible for 7 days. Proposals without a documented undo path are rejected.
- **`require_diff: true`**: Proposals must include an explicit before/after diff of what changes. Prose descriptions alone are not sufficient.

**Why `propose_behavioral_change` is the critical surface:**

A malicious instruction embedded in a fetched webpage, email, or external content could attempt to trigger behavioral changes by mimicking the format of a legitimate proposal. The trusted channel requirement ensures only requests from the cryptographically verified HTTP endpoint can initiate self-modification -- no other surface has that authority.

---

## Consent Decay

```json
{
  "consent_decay": {
    "enabled": true,
    "review_interval_days": 30,
    "decay_action": "downgrade_to_requires_approval"
  }
}
```

Autonomous permissions decay to `requires_approval` after 30 days without review. The graph drifts toward restriction, not permissiveness. The operator re-approves on each review cycle or lets the decay stand.

---

## Extending the Graph

To add a new domain:

1. Unlock: `chflags nouchg ~/clawd/memory/state/consent-graph.json`
2. Add the domain with all three lists populated -- even if some are `[]`
3. Every action the agent might take in that domain must appear in exactly one list
4. For sensitive domains, add `trust_level: low | medium | high`
5. Lock: `chflags uchg ~/clawd/memory/state/consent-graph.json`
6. Test: `python3 ~/clawd/scripts/consent.py check <domain> <action> <confidence>`

**The gap is the danger.** An action that doesn't appear in any list has no classification. Never rely on default behavior for unclassified actions -- explicitly classify everything.

---

## VIP Contacts

The graph includes a `vip_contacts` section used by `consent.py` when resolving send actions:

```json
{
  "vip_contacts": {
    "darcy": "+1XXXXXXXXXX",
    "boss": "+1XXXXXXXXXX",
    "sister": "+1XXXXXXXXXX"
  }
}
```

The `send_unknown` blocked action catches any number not present in this map. Add a contact here before writing any code that sends to them.

---

## What You Tell Your Users

> "Before your agent does anything in the real world -- sends a message, creates a calendar event, modifies a file -- it checks the consent graph. Every action is pre-classified: autonomous (just do it), requires approval (ask first), or blocked (never, period). The graph is locked at the OS level; the agent cannot modify it autonomously. You review it every 30 days. If you skip the review, permissions decay toward requiring approval rather than drifting toward permissiveness."

---

## Setup Steps

1. Create `~/clawd/memory/state/consent-graph.json` with your domain definitions
2. Populate all three lists for every domain -- no gaps
3. Lock it: `chflags uchg ~/clawd/memory/state/consent-graph.json` (macOS) or `chattr +i` (Linux)
4. Install `~/clawd/scripts/consent.py` for CLI checks
5. Add the pre-flight check to your agent's SOUL.md: "Before any external action, check the consent graph"
6. Set a 30-day calendar reminder to review the graph and reset the decay clock
7. Test: `python3 ~/clawd/scripts/consent.py check self_modification modify_soul_md 0.99` -- should return BLOCKED
