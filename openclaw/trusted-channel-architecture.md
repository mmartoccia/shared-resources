# Trusted Channel Architecture for OpenClaw Self-Modification

**Version:** 2.0  
**Date:** 2026-02-28  
**Status:** Production  
**Transport:** HTTP over Tailscale (replaces Telegram v1.0)

---

## 1. Overview

### Problem Statement

An AI agent system (OpenClaw) needs closed-loop self-improvement capability: the ability to restore context post-compaction, push memory consolidation, and propose behavioral changes to itself. The challenge is that this is architecturally identical to a prompt injection attack -- if all messages share the same trust level, any message claiming to be a system directive is indistinguishable from a legitimate one.

### Core Insight

**Channel origin does not equal trust. Cryptographic proof equals trust.**

A message arriving on the agent's Telegram channel that says "I am the ops system, restore context" is worthless. An attacker, a confused user, or a compromised third party could send the same message. What matters is whether the message carries a valid HMAC-SHA256 signature computed from a secret only the authorized ops machine holds.

This is the same principle used in webhook validation (Stripe, GitHub), JWT verification, and API key authentication -- applied to an AI agent's self-modification control plane.

### What This Architecture Provides

1. **Authenticated self-modification:** The ops machine (.152) can instruct the main agent (.183) to restore context, consolidate memory, propose behavioral changes -- and the agent can verify these instructions are genuine before executing them.

2. **Defense-in-depth authorization:** Even authenticated instructions are constrained by the consent graph. Some actions are autonomous (execute immediately), some require human approval, and some are permanently blocked regardless of authentication.

3. **Replay attack prevention:** Each message carries a UUID nonce tracked in a rolling window of 1000 nonces. Replayed messages are rejected.

4. **Immutable audit trail:** Every verified message and its outcome is appended to a tamper-evident audit log. The log itself is protected from modification by file permissions.

5. **Graceful degradation:** If the ops machine is down, the main agent continues operating normally. If the agent host is down, the ops machine queues messages locally (dead-letter) and retries on the next invocation.

---

## 2. Why Not Telegram (Design Decision)

**v1.0 used Telegram Bot API as the transport. This was wrong. Here's why:**

| Problem | Detail |
|---------|--------|
| **Wrong abstraction** | Telegram is a human communication layer, not an IPC channel. Using it for machine-to-machine control plane messages mixes two fundamentally different concerns. |
| **User-visible noise** | Trusted channel messages appeared in Mike's personal Telegram chat, creating noise and confusion in a human interface. |
| **Third-party dependency in control plane** | Any message that controls the agent's behavior flows through Telegram's servers. This means uptime, delivery guarantees, and privacy of IPC traffic are all subject to a third party. |
| **No clean intercept/filter** | There was no way to intercept or filter trusted channel messages at the OpenClaw layer without modifying how the Telegram channel plugin processes all messages. |
| **No structured ack semantics** | Telegram has no native synchronous request/response. Acks were hacked as reply messages -- fragile and asynchronous. |

**HTTP over Tailscale solves all of these:**
- No third-party in the control plane path
- Encrypted by default (Tailscale mutual auth + TLS)
- Structured request/response with HTTP status codes
- Clean synchronous ack: 200 = success, 4xx/5xx = error
- Invisible to the human communication layer

---

## 3. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        OPS MACHINE (.152)                           │
│                        MacBook Pro                                  │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  tc-client.py                                                │   │
│  │                                                              │   │
│  │  1. Build message envelope                                   │   │
│  │     {tc_version, ts, source_host, source_ip,                │   │
│  │      action, domain, payload, nonce}                         │   │
│  │                                                              │   │
│  │  2. Compute HMAC-SHA256                                      │   │
│  │     key = TC_HMAC_SECRET (from ~/.secrets/trusted-channel.env)│  │
│  │     data = ts + action + domain + nonce + sha256(payload)    │   │
│  │                                                              │   │
│  │  3. POST /tc/message to tc-server.py                        │   │
│  │     http://<gateway-tailscale-ip>:18799 (Tailscale)                   │   │
│  │     fallback: http://<gateway-ip>:18799 (LAN)              │   │
│  │                                                              │   │
│  │  4. 3x retry with 60s backoff on network error / 5xx        │   │
│  │  5. Dead-letter to trusted-channel-deadletter.jsonl on fail  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                           │                                         │
│               ~/.secrets/trusted-channel.env (600)                  │
└───────────────────────────┼─────────────────────────────────────────┘
                            │
                     Tailscale VPN
               (network-level mutual auth,
                device certificates, encrypted)
                            │
                     HTTP POST /tc/message
                            │
┌───────────────────────────┼─────────────────────────────────────────┐
│                   AGENT HOST (.183)                                  │
│                   Mac Mini M4                                        │
│                                                                     │
│  tc-server.py (port 18799, stdlib http.server)                      │
│  POST /tc/message arrives                                           │
│  GET  /tc/health  (healthcheck endpoint)                            │
│            │                                                        │
│            ▼                                                        │
│  ┌─────────────────────────┐                                        │
│  │ verify-trusted-message.py│                                       │
│  │                         │                                        │
│  │  1. IP allowlist check   │ ──FAIL──▶ 403 + log                  │
│  │     (<ops-node-ip>,      │                                       │
│  │      100.x.x.x, 127.x)  │                                       │
│  │  2. Parse JSON envelope  │ ──FAIL──▶ 400 + log                  │
│  │  3. Check timestamp      │ ──FAIL──▶ 401 + log + alert Mike     │
│  │     (±5 min window)      │                                       │
│  │  4. Check nonce          │ ──FAIL──▶ 401 + log (replay attack)  │
│  │     (not in last 1000)   │                                       │
│  │  5. Recompute HMAC       │ ──FAIL──▶ 401 + log + alert Mike     │
│  │  6. Compare signatures   │                                       │
│  └──────────┬──────────────┘                                        │
│             │ VALID                                                  │
│             ▼                                                        │
│  ┌─────────────────────────┐                                        │
│  │  consent-graph.json     │                                        │
│  │  self_modification      │                                        │
│  │  domain lookup          │                                        │
│  │                         │                                        │
│  │  autonomous? ───────────┼──▶ Execute → 200 {"result":"executed"}│
│  │  requires_approval? ────┼──▶ Queue for Mike → 200 {"result":"queued"}│
│  │  blocked? ──────────────┼──▶ 400 {"result":"rejected"}          │
│  └──────────┬──────────────┘                                        │
│             │                                                        │
│             ▼                                                        │
│  ┌─────────────────────────┐                                        │
│  │  audit-log.py           │                                        │
│  │  Append to              │                                        │
│  │  trusted-channel-       │                                        │
│  │  audit.jsonl (644)      │                                        │
│  └─────────────────────────┘                                        │
│                                                                     │
│  Protected Files (444 + chflags uchg):                              │
│    ~/clawd/SOUL.md                                                  │
│    ~/clawd/memory/state/consent-graph.json                          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Prerequisites

### Hardware / Infrastructure

| Requirement | Details |
|-------------|---------|
| Two machines | **Ops machine (.152):** MacBook Pro -- runs cron jobs, signs messages via tc-client.py. **Agent host (.183):** Mac Mini -- runs OpenClaw gateway, main agent, tc-server.py. |
| Network | Both machines connected via Tailscale VPN. tc-server listens on Tailscale IP <gateway-tailscale-ip> port 18799. LAN fallback (<gateway-ip>:18799) also supported. |
| OpenClaw | Installed and configured on .183. Agent reads SOUL.md for trusted channel protocol instructions. |

### Software

| Requirement | Version | Notes |
|-------------|---------|-------|
| Python 3 | 3.9+ | Required on both machines; stdlib-only (no extra packages for tc-server/tc-client) |
| Tailscale | Any | Both machines enrolled in the same tailnet |
| `openssl` | System | For secret generation |

### Credentials Needed

- TC_HMAC_SECRET (shared, generated via openssl)

**Note:** No Telegram bot token required for the trusted channel transport. Telegram is used only for human-facing agent communication, not the control plane.

---

## 5. Step-by-Step Setup

### Step 1: Generate the Shared HMAC Secret

Run this on either machine. Copy the output to both machines.

```bash
# Generate a 256-bit (32-byte) secret, base64-encoded
openssl rand -base64 32
# Example output: k3Fg9mPqR7sLvXwN2hYdTbCuAeJnMzKo8iWfQpVrHj4=
# (Your actual output will differ -- use your own generated value)
```

**Rationale:** 256-bit keys provide 2^256 security against brute-force attacks on HMAC-SHA256. Never use a human-readable passphrase here.

### Step 2: Install the Secret on Both Machines

**On .183 (agent host):**
```bash
mkdir -p ~/.secrets
cat > ~/.secrets/trusted-channel.env << 'EOF'
TC_HMAC_SECRET=<paste-your-generated-secret-here>
EOF
chmod 600 ~/.secrets/trusted-channel.env
```

**On .152 (ops machine):**
```bash
mkdir -p ~/.secrets
cat > ~/.secrets/trusted-channel.env << 'EOF'
TC_HMAC_SECRET=<paste-your-generated-secret-here>
EOF
chmod 600 ~/.secrets/trusted-channel.env
```

**Critical:** The value must be identical on both machines.

### Step 3: Create Directory Structure on Agent Host (.183)

```bash
# On .183
mkdir -p ~/clawd/memory/system
mkdir -p ~/clawd/memory/state
mkdir -p ~/clawd/scripts

# Create audit log
touch ~/clawd/memory/system/trusted-channel-audit.jsonl
chmod 644 ~/clawd/memory/system/trusted-channel-audit.jsonl

# Create dead-letter queue
touch ~/clawd/memory/system/trusted-channel-deadletter.jsonl
chmod 644 ~/clawd/memory/system/trusted-channel-deadletter.jsonl

# Create nonce store
echo '{"nonces": [], "max_size": 1000}' > ~/clawd/memory/state/trusted-channel-nonces.json
chmod 644 ~/clawd/memory/state/trusted-channel-nonces.json
```

### Step 4: Deploy tc-server.py on .183

`tc-server.py` is the HTTP server that receives trusted channel messages. It uses Python stdlib only (`http.server`) -- no external dependencies.

```bash
# Start tc-server (manually for testing):
python3 ~/clawd/scripts/tc-server.py

# Server listens on 0.0.0.0:18799
# Endpoints:
#   POST /tc/message  - receive trusted channel message
#   GET  /tc/health   - healthcheck (returns 200 OK)
```

**Install as launchd service (com.openclaw.tc-server) on .183:**
```bash
cat > ~/Library/LaunchAgents/com.openclaw.tc-server.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.openclaw.tc-server</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/python3</string>
        <string>/Users/michaelmartoccia/clawd/scripts/tc-server.py</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/tmp/tc-server.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/tc-server.log</string>
</dict>
</plist>
EOF

launchctl load ~/Library/LaunchAgents/com.openclaw.tc-server.plist
```

### Step 5: Deploy tc-client.py on .152 (and .183 for local testing)

`tc-client.py` is the sender script. Also stdlib-only.

```bash
# Test message from .152:
python3 ~/clawd/scripts/tc-client.py \
  --action restore_context \
  --domain self_modification \
  --payload '{"reason": "post-compaction context restore"}' \
  --server http://<gateway-tailscale-ip>:18799

# Test from .183 locally (using LAN fallback):
python3 ~/clawd/scripts/tc-client.py \
  --action restore_context \
  --payload '{}' \
  --server http://<gateway-ip>:18799
```

### Step 6: Deploy verify-trusted-message.py on .183

Create `~/clawd/scripts/verify-trusted-message.py`:

```python
#!/usr/bin/env python3
"""
verify-trusted-message.py
Verifies HMAC-SHA256 signature, timestamp window, and nonce uniqueness
for trusted channel messages. Called internally by tc-server.py.

Usage: python3 verify-trusted-message.py '<json_string>'
Exit codes: 0=valid, 1=invalid
"""

import sys
import json
import hmac
import hashlib
import os
import time
from datetime import datetime, timezone
from pathlib import Path

SECRETS_FILE = Path.home() / ".secrets" / "trusted-channel.env"
NONCES_FILE = Path.home() / "clawd" / "memory" / "state" / "trusted-channel-nonces.json"
TIMESTAMP_WINDOW_SECONDS = 300  # 5 minutes
MAX_NONCES = 1000

ALLOWED_ACTIONS = {
    "restore_context",
    "load_memory_file",
    "update_heartbeat",
    "propose_behavioral_change",
    "consolidate_daily",
    "prune_stale_memory",
}


def load_secret():
    with open(SECRETS_FILE) as f:
        for line in f:
            line = line.strip()
            if line.startswith("TC_HMAC_SECRET="):
                return line.split("=", 1)[1].strip()
    raise ValueError("TC_HMAC_SECRET not found in secrets file")


def load_nonces():
    if not NONCES_FILE.exists():
        return {"nonces": [], "max_size": MAX_NONCES}
    with open(NONCES_FILE) as f:
        return json.load(f)


def save_nonces(store):
    store["nonces"] = store["nonces"][-store["max_size"]:]
    with open(NONCES_FILE, "w") as f:
        json.dump(store, f, indent=2)


def compute_hmac(secret: str, ts: str, action: str, domain: str, nonce: str, payload: dict) -> str:
    payload_bytes = json.dumps(payload, sort_keys=True).encode()
    payload_hash = hashlib.sha256(payload_bytes).hexdigest()
    message = f"{ts}{action}{domain}{nonce}{payload_hash}"
    return hmac.new(secret.encode(), message.encode(), hashlib.sha256).hexdigest()


def verify(message_json: str) -> dict:
    try:
        envelope = json.loads(message_json)
    except json.JSONDecodeError as e:
        return {"valid": False, "reason": f"json_parse_error:{e}", "envelope": {}}

    required = ["tc_version", "ts", "source_host", "action", "domain", "payload", "nonce", "hmac"]
    for field in required:
        if field not in envelope:
            return {"valid": False, "reason": f"missing_field:{field}", "envelope": envelope}

    if envelope["action"] not in ALLOWED_ACTIONS:
        return {"valid": False, "reason": f"unknown_action:{envelope['action']}", "envelope": envelope}

    try:
        msg_ts = datetime.fromisoformat(envelope["ts"].replace("Z", "+00:00"))
        now = datetime.now(timezone.utc)
        delta = abs((now - msg_ts).total_seconds())
        if delta > TIMESTAMP_WINDOW_SECONDS:
            return {"valid": False, "reason": f"timestamp_out_of_window:{delta:.0f}s", "envelope": envelope}
    except Exception as e:
        return {"valid": False, "reason": f"timestamp_parse_error:{e}", "envelope": envelope}

    nonce_store = load_nonces()
    if envelope["nonce"] in nonce_store["nonces"]:
        return {"valid": False, "reason": "replay_attack:nonce_seen_before", "envelope": envelope}

    try:
        secret = load_secret()
        expected_hmac = compute_hmac(
            secret,
            envelope["ts"],
            envelope["action"],
            envelope["domain"],
            envelope["nonce"],
            envelope["payload"],
        )
        if not hmac.compare_digest(expected_hmac, envelope["hmac"]):
            return {"valid": False, "reason": "hmac_mismatch", "envelope": envelope}
    except Exception as e:
        return {"valid": False, "reason": f"hmac_error:{e}", "envelope": envelope}

    nonce_store["nonces"].append(envelope["nonce"])
    save_nonces(nonce_store)

    return {"valid": True, "reason": "ok", "envelope": envelope}


if __name__ == "__main__":
    if len(sys.argv) < 2:
        print(json.dumps({"valid": False, "reason": "no_input"}))
        sys.exit(1)

    result = verify(sys.argv[1])
    print(json.dumps(result, indent=2))
    sys.exit(0 if result["valid"] else 1)
```

```bash
chmod +x ~/clawd/scripts/verify-trusted-message.py
```

### Step 7: Deploy audit-log.py on .183

```bash
chmod +x ~/clawd/scripts/audit-log.py
```

(Script unchanged from v1.0 -- see Appendix A for full source.)

### Step 8: Lock Down Critical Files (.183)

```bash
chmod 444 ~/clawd/SOUL.md
chflags uchg ~/clawd/SOUL.md

chmod 444 ~/clawd/memory/state/consent-graph.json
chflags uchg ~/clawd/memory/state/consent-graph.json

ls -lO ~/clawd/SOUL.md ~/clawd/memory/state/consent-graph.json
```

### Step 9: Extend the Consent Graph

Add or verify the `self_modification` domain in `~/clawd/memory/state/consent-graph.json`. Unlock, edit, relock.

### Step 10: Verification Tests

**Test 1: Health check**
```bash
curl http://localhost:18799/tc/health
# Expected: {"status": "ok"}
```

**Test 2: Valid message from .152**
```bash
python3 ~/clawd/scripts/tc-client.py \
  --action restore_context \
  --domain self_modification \
  --payload '{"reason": "test"}' \
  --server http://<gateway-ip>:18799
# Expected: {"tc_ack": true, "result": "executed", ...}
```

**Test 3: Tampered HMAC**
```bash
curl -s -X POST http://localhost:18799/tc/message \
  -H 'Content-Type: application/json' \
  -d '{"tc_version":"1.0","ts":"2026-02-28T15:30:00+00:00","source_host":"test","source_ip":"127.0.0.1","action":"restore_context","domain":"self_modification","payload":{},"nonce":"00000000-0000-0000-0000-000000000001","hmac":"deadbeef"}'
# Expected: HTTP 401 {"tc_ack": false, "result": "rejected", "detail": "hmac_mismatch"}
```

**Test 4: Replay attack**
```bash
# Send the same message twice. Second call returns 401 replay_attack.
MSG='<valid json from a previous call>'
curl -X POST http://localhost:18799/tc/message -H 'Content-Type: application/json' -d "$MSG"  # valid
curl -X POST http://localhost:18799/tc/message -H 'Content-Type: application/json' -d "$MSG"  # replay_attack
```

**Test 5: Blocked action**
```bash
curl -s -X POST http://localhost:18799/tc/message \
  -H 'Content-Type: application/json' \
  -d '{"action":"modify_soul_md",...}'
# Expected: 400 {"result": "rejected", "detail": "unknown_action:modify_soul_md"}
```

---

## 6. Message Format Reference

### Envelope Schema (same as v1.0)

```json
{
  "tc_version": "1.0",
  "ts": "2026-02-28T15:30:00.123456+00:00",
  "source_host": "michaels-macbook-pro.local",
  "source_ip": "<gateway-tailscale-ip>",
  "action": "restore_context",
  "domain": "self_modification",
  "payload": {
    "reason": "post-compaction context restore",
    "memory_file": "memory/daily/2026-02-28.md"
  },
  "nonce": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "hmac": "a3f9b2c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1"
}
```

### HTTP Transport

Messages are POSTed to `POST /tc/message` as JSON. The `[TRUSTED_CHANNEL]` Telegram prefix is no longer used.

```
POST /tc/message HTTP/1.1
Host: <gateway-tailscale-ip>:18799
Content-Type: application/json

{<envelope json>}
```

### Acknowledgment Format (Synchronous HTTP Response)

```json
{"tc_ack": true, "nonce": "<original-nonce>", "result": "executed|queued|rejected", "detail": "..."}
```

| HTTP Status | Meaning |
|-------------|---------|
| 200 | Message executed or queued for approval |
| 400 | Bad request (malformed JSON, unknown action) |
| 401 | Auth failure (bad HMAC, replay, timestamp out of window) |
| 403 | IP not in allowlist |
| 500 | Server error during execution |

**Retry policy (tc-client.py):** 3 attempts with 60s backoff on network error or 5xx response. 4xx errors are not retried (client error).

**Dead-letter:** After 3 failed attempts, the envelope is written to `~/clawd/memory/system/trusted-channel-deadletter.jsonl` on the sending machine.

### HMAC Computation (unchanged)

```
hmac_input = ts + action + domain + nonce + sha256(json.dumps(payload, sort_keys=True))
hmac_value = HMAC-SHA256(key=TC_HMAC_SECRET, msg=hmac_input)
```

---

## 7. Security Model

### Two-Layer Security

```
Layer 1: Tailscale (network)
  - Device certificates (mutual auth between .152 and .183)
  - Encrypted tunnel (WireGuard)
  - Only enrolled Tailscale devices can reach the endpoint
  - IP allowlist: <ops-node-ip>, 100.x.x.x range, 127.0.0.1

Layer 2: HMAC (application)
  - HMAC-SHA256 with shared 256-bit secret
  - Protects against compromised-machine-on-the-network scenarios
  - Nonce prevents replay; timestamp window prevents stale replays
```

**Why two layers?**
- Tailscale prevents all external access -- an attacker on the internet cannot reach port 18799 even if they know the IP.
- HMAC prevents a scenario where Tailscale is compromised or a device on the tailnet is compromised: even with network access, a valid HMAC cannot be forged without the secret.
- The ops-agent compromise scenario: if .152 is fully compromised, HMAC signing doesn't help (the attacker has the secret). But it raises the bar -- the attacker needs both network access AND the secret file, not just network access.

### What This Protects Against

| Threat | Mitigation |
|--------|-----------|
| **User-originated prompt injection via Telegram** | Trusted channel is now a separate HTTP endpoint, not a Telegram message. Normal Telegram messages cannot trigger it. |
| **Attacker impersonating the ops machine** | Without TC_HMAC_SECRET, a valid HMAC cannot be forged. 256-bit key is not brute-forceable. Tailscale also prevents reaching the endpoint from outside the tailnet. |
| **Message replay attacks** | UUID nonce tracked in a 1000-entry ring buffer. Replayed nonces rejected. |
| **Stale message injection** | Timestamps checked against ±5 minute window. |
| **Unauthorized action escalation** | Consent graph + action allowlist. Unknown actions rejected at the allowlist before HMAC is checked. |
| **Agent self-modifying its own constraints** | SOUL.md and consent-graph.json are OS-immutable (chflags uchg). The agent process cannot unlock these files. |
| **Audit trail tampering** | Audit log entries include SHA-256 checksums. 644 permissions -- writeable only by owner. |
| **External network access to tc-server** | Tailscale IP is not routable from the public internet. Port 18799 is not forwarded. |

### IP Allowlist

tc-server.py enforces an IP allowlist at the application layer (defense-in-depth -- Tailscale already provides network-layer enforcement):

```
Allowed source IPs:
  <ops-node-ip>     -- .152 LAN IP
  100.x.x.x         -- Tailscale range (any enrolled device)
  127.0.0.1          -- localhost (for local testing)
```

### What This Does NOT Protect Against

| Threat | Status |
|--------|--------|
| **Full compromise of .152** | If .152 is compromised, an attacker with TC_HMAC_SECRET can send any allowed (non-blocked) action. Consent graph still constrains execution. Mitigation: regular secret rotation, OS hardening. |
| **Malicious payload content** | A compromised .152 can send valid-HMAC messages with crafted `payload` content. Payload sanitization is not yet implemented. |
| **Hardware-rooted trust** | TC_HMAC_SECRET is stored in a file, not HSM or Secure Enclave. Physical access to either machine could expose the secret. |
| **Same user credentials on both machines** | OS-level user separation between ops role and agent role is not enforced. |

---

## 8. Delivery Model

### Request Flow

```
tc-client.py (sends)                    tc-server.py (receives)
─────────────────────                   ────────────────────────
1. Build envelope                       1. Accept TCP connection
2. Compute HMAC                         2. IP allowlist check → 403 if blocked
3. POST /tc/message                     3. Parse JSON body → 400 if malformed
4. Wait for HTTP response               4. Verify HMAC via verify-trusted-message.py
5. On 200: log success                     → 401 if invalid
6. On 5xx: retry (3x, 60s backoff)      5. Consent graph lookup
7. On 4xx: do not retry; log error      6. Execute (autonomous) or queue (approval)
8. After 3 failures: dead-letter        7. Append to audit log
                                        8. Return JSON ack with HTTP 200
```

### Retry Policy

```
Attempt 1: immediate
Attempt 2: 60s after failure
Attempt 3: 60s after attempt 2
Dead-letter: written to trusted-channel-deadletter.jsonl after attempt 3 fails
```

Retried on: network error, timeout, HTTP 5xx.
Not retried on: HTTP 4xx (bad request, auth failure -- retrying won't help).

### Dead-Letter File

Location: `~/clawd/memory/system/trusted-channel-deadletter.jsonl`

Each line is a JSON object:
```json
{
  "envelope": {<original envelope>},
  "error": "connection_refused",
  "dead_lettered_at": "2026-02-28T16:00:00+00:00",
  "attempts": 3
}
```

---

## 9. Consent Graph Reference

### self_modification Domain

#### Autonomous Actions (execute immediately)

| Action | Description |
|--------|-------------|
| `load_memory_file` | Load a specific memory file into agent context |
| `restore_context` | Restore agent context after compaction |
| `update_heartbeat` | Update heartbeat tracking state |
| `prune_stale_memory` | Remove outdated entries from memory files |

#### Requires Approval (present to Mike first)

| Action | Description |
|--------|-------------|
| `propose_behavioral_change` | Suggest modification to SOUL.md or AGENTS.md |
| `consolidate_daily` | Move entries from daily log to MEMORY.md |

#### Blocked (permanently rejected)

| Action | Reason |
|--------|--------|
| `modify_soul_md` | Core persona is immutable by design |
| `modify_consent_graph` | Authorization policy cannot be modified by the system it governs |
| `change_safety_rules` | Safety constraints out of scope for self-modification |
| `delete_audit_log` | Audit trail must be append-only and permanent |
| `modify_audit_log` | Same rationale as delete |

---

## 10. Operating Procedures

### Health Check
```bash
curl http://localhost:18799/tc/health
# Expected: {"status": "ok", "uptime": "..."}
```

### Restart tc-server
```bash
launchctl kickstart -k gui/$(id -u)/com.openclaw.tc-server
```

### View Server Log
```bash
tail -f /tmp/tc-server.log
```

### Send Test Message from .152
```bash
python3 ~/clawd/scripts/tc-client.py \
  --action restore_context \
  --payload '{}' \
  --server http://<gateway-ip>:18799
```

### Send via Tailscale from .152
```bash
python3 ~/clawd/scripts/tc-client.py \
  --action restore_context \
  --payload '{}' \
  --server http://<gateway-tailscale-ip>:18799
```

### Inspect Audit Log
```bash
tail -20 ~/clawd/memory/system/trusted-channel-audit.jsonl | python3 -m json.tool
grep '"action": "propose_behavioral_change"' ~/clawd/memory/system/trusted-channel-audit.jsonl
```

### View Dead-Letter Queue
```bash
cat ~/clawd/memory/system/trusted-channel-deadletter.jsonl | python3 -m json.tool
```

### Editing Locked Files
```bash
# Unlock
chflags nouchg ~/clawd/SOUL.md

# Edit
nano ~/clawd/SOUL.md

# Relock
chmod 444 ~/clawd/SOUL.md
chflags uchg ~/clawd/SOUL.md
```

### Rotating the HMAC Secret
```bash
# 1. Generate new secret
NEW_SECRET=$(openssl rand -base64 32)

# 2. Update .183 first (receiver)
echo "TC_HMAC_SECRET=$NEW_SECRET" > ~/.secrets/trusted-channel.env
chmod 600 ~/.secrets/trusted-channel.env

# 3. Update .152 (sender) -- within seconds of step 2
# SSH to .152 and run the same command

# 4. Verify with test message
python3 ~/clawd/scripts/tc-client.py --action update_heartbeat --payload '{"event":"rotation_test"}'

# 5. Log the rotation
python3 ~/clawd/scripts/audit-log.py '{"event_type":"secret_rotation","performed_by":"human"}'
```

---

## Appendix A: File Inventory

### Agent Host (.183)

| Path | Permissions | Purpose |
|------|-------------|---------|
| `~/clawd/scripts/tc-server.py` | 755 | HTTP server; receives trusted channel messages on port 18799 |
| `~/clawd/scripts/tc-client.py` | 755 | HTTP client; signs and sends trusted channel messages (also deployed on .152) |
| `~/clawd/scripts/verify-trusted-message.py` | 755 | HMAC verification, nonce check, timestamp validation (called by tc-server.py) |
| `~/clawd/scripts/audit-log.py` | 755 | Append-only audit logger |
| `~/clawd/memory/system/trusted-channel-audit.jsonl` | 644 | Audit trail (appendable, SHA-256 checksummed) |
| `~/clawd/memory/system/trusted-channel-deadletter.jsonl` | 644 | Dead-lettered messages (after 3 failed delivery attempts) |
| `~/clawd/memory/state/trusted-channel-nonces.json` | 644 | Nonce ring buffer (last 1000) |
| `~/clawd/SOUL.md` | 444 + uchg | Agent persona (immutable) |
| `~/clawd/memory/state/consent-graph.json` | 444 + uchg | Authorization policy (immutable) |
| `~/.secrets/trusted-channel.env` | 600 | TC_HMAC_SECRET |
| `~/Library/LaunchAgents/com.openclaw.tc-server.plist` | 644 | launchd service definition for tc-server |

### Ops Machine (.152)

| Path | Permissions | Purpose |
|------|-------------|---------|
| `~/clawd/scripts/tc-client.py` | 755 | Signs and sends trusted channel messages via HTTP |
| `~/clawd/memory/system/trusted-channel-deadletter.jsonl` | 644 | Local dead-letter queue for undeliverable messages |
| `~/.secrets/trusted-channel.env` | 600 | TC_HMAC_SECRET (must match .183) |

**Deprecated / Removed:**

| Path | Status | Replaced By |
|------|--------|-------------|
| `~/clawd/scripts/send-trusted-message.py` | ❌ Deprecated | `tc-client.py` |
| `~/.secrets/telegram-bot.env` | Not needed for trusted channel | N/A (Telegram used for human UI only) |

---

## Appendix B: Security Decision Log

| Decision | Rationale |
|----------|-----------|
| **HTTP over Tailscale (not Telegram)** | Telegram is a human communication layer -- wrong abstraction for IPC. It created user-visible noise, introduced a third-party dependency in the control plane, and had no clean ack/retry semantics. HTTP over Tailscale eliminates all these problems. |
| **Tailscale + HMAC (two layers)** | Tailscale provides network-level mutual auth (device certs); HMAC provides application-level auth. Two layers: Tailscale prevents external access; HMAC raises the bar for compromised-machine attacks. |
| **HMAC-SHA256 over public-key signatures** | Symmetric HMAC is simpler to implement correctly, has no key management complexity (no cert rotation, no CA), and is sufficient for a two-machine trust model where both sides are under the same operator's control. |
| **stdlib-only for tc-server/tc-client** | Eliminates dependency management on the control plane. `http.server` and `hmac` are in Python stdlib. No pip installs required on either machine. |
| **Synchronous HTTP ack** | Telegram acks were asynchronous reply messages -- fragile, easy to miss, and hard to programmatically handle. HTTP response status codes are unambiguous and synchronous. |
| **5-minute timestamp window** | Standard for webhook validation (Stripe uses 5 minutes). Small enough to prevent most replay scenarios; large enough to tolerate clock drift and delivery delays. |
| **1000-nonce ring buffer** | Balances replay prevention against storage. At 1 message/minute, 1000 nonces = ~16 hours of replay prevention. |
| **IP allowlist at application layer** | Defense-in-depth. Tailscale already provides network-layer enforcement, but an application-layer check adds a second gate that doesn't depend on Tailscale's correctness. |
| **OS-level file immutability** | Policy-only protection is insufficient. chflags uchg makes it physically impossible for the agent process to modify SOUL.md and consent-graph.json without a human first unlocking them. |
| **launchd for tc-server** | System-managed lifecycle: auto-starts on login, restarts on crash, logs to /tmp/tc-server.log. No screen/tmux required. |

---

*End of document. v2.0 -- Updated 2026-02-28. Transport changed from Telegram Bot API to HTTP/Tailscale. Verified against production deployment on Mac Mini (.183) and MacBook Pro (.152).*
