# Model Selection Strategy: Token Economics for a Production Personal AI

## The Token Economics Reality

A production personal AI deployment runs ~800M tokens/month. Without deliberate model selection:
- You hit Anthropic's rate limits (fastest, most capable, most expensive)
- You underutilize Google's generous quotas (slower on some tasks, excellent for research/summarization)
- You ignore local models entirely (free, private, no rate limits, good enough for many tasks)

The goal: route each task to the cheapest capable model, not the most capable available model.

---

## The Rule (Two Sentences)

**New code -> Anthropic. Fix code -> Codex.**

- Greenfield features, new scripts, new services: use Anthropic (Haiku for cost, Sonnet for complexity, Opus for architecture)
- Bug fixes, debugging, patching existing code: use Codex

Everything else follows the routing table below.

---

## Task-to-Model Routing Table

| Task type | Model | Reason |
|---|---|---|
| Complex reasoning, architecture decisions | claude-opus-4-6 | Depth required, low volume |
| General tasks, coding, analysis | claude-sonnet-4-6 | Best capability/cost balance |
| Simple tasks, quick lookups | claude-haiku-4-5 | Fast, cheap, sufficient |
| Research, summarization, web-grounded | gemini-3-pro-preview | 1M context, Google Search grounding |
| Background crons, monitoring | gemini-3-pro-preview or gemini-3-flash | Shift load off Anthropic quota |
| Code fixes, debugging | openai-codex/gpt-5.3-codex | Specialized for code tasks |
| Private/sensitive data | ollama-local/qwen3:30b-a3b | Never leaves machine |
| Confabulation/hallucination tasks | lmstudio-local/qwen/qwen3-next-80b | Local 80B for quality confab |
| Web search with citations | perplexity/sonar-pro | Real-time web, source citations |

**Always verify available models with `openclaw models list` before configuring.** The fleet changes. Never assume model availability based on training data or memory of what was available last month.

---

## Provider Reference

### Anthropic
Highest capability. Rate-limited. Most expensive.

Reserve for:
- Direct user-facing conversation
- Complex reasoning and architecture decisions
- Anything requiring tool use (tool calling is most reliable here)
- New code generation (greenfield)

Background crons should default away from Anthropic. Every background token you save here is available for the conversation you actually care about.

Cache hits are ~90% cheaper. Heavily repeated context (SOUL.md, AGENTS.md, memory files) benefits enormously from prompt caching -- structure prompts so stable content leads.

### Google Gemini CLI
1M context window. Generous quotas. Free tier available.

Best for:
- Document summarization (long context is the killer feature)
- Research crons (Google Search grounding built in)
- Anything that needs broad context without token anxiety
- Background monitoring crons (shift load off Anthropic)

**Important:** CLI-backed models need minimum 180s timeout in cron config. A Gemini CLI invocation is not an API call -- it spawns a subprocess. Entropy scheduler handles this automatically via `MODEL_MIN_TIMEOUTS`, but manual cron configs need this set explicitly.

### NVIDIA NIM
Access to Kimi K2, DeepSeek V3.2, GLM-5. Small volume, mostly used for:
- Cross-model validation (Council of the Wise runs)
- Alternative reasoning styles when Anthropic gives a confident-but-wrong answer
- Breaking out of Anthropic quota ceiling when needed

### Local (Ollama / LMStudio on .152)
Zero cost. Zero latency constraints. Fully private.

Use for:
- Sensitive data that cannot leave the machine
- Confabulation pipelines (gap detection via controlled hallucination)
- High-volume repetitive tasks where quality bar is "good enough"
- Tasks where privacy matters more than speed

The 80B local model (a large local model on LMStudio) is surprisingly capable for confabulation tasks -- better than smaller cloud models for detecting what the agent does not know.

### OpenAI Codex
Specialized for code. GPT-5.x family. Use specifically for:
- Bug fixes and debugging
- Code review
- Patching existing code

Not a general-purpose model for this deployment. Codex routes go: code fix request -> Codex -> done. Do not route general conversation, research, or planning here.

---

## Quota Monitoring

Set up a daily token meter cron (9 AM, model: `gemini-3-flash` -- cheap, no irony in using Gemini to monitor Anthropic):

```python
# ~/clawd/scripts/token-meter.py
# Reads usage from provider APIs + local logs
# Reports: tokens by provider, % of monthly quota used, projected month-end
# Alert threshold: 80% quota usage triggers Telegram notification
```

Key metrics to track:
- **Anthropic:** tokens in / tokens out / cache hits (cache hits at ~10% of full token cost)
- **Google:** tokens in / tokens out (generous quota, rarely a concern)
- **NVIDIA NIM:** tokens (small volume, mostly Council runs)
- **Local:** tokens (free, track for capacity planning and model sizing)

Add a quota guardian cron: every 2 hours, check Anthropic usage. Alert at 80%. Use `everyMs: 7200000` in OpenClaw cron config.

---

## The Load-Shifting Rule

When Anthropic quota approaches 80%:

1. Shift all research/summarization crons to `gemini-3-pro-preview`
2. Shift all monitoring crons to `gemini-3-flash`
3. Keep Anthropic for: direct user conversation, complex reasoning, tool use
4. Use NVIDIA NIM (Kimi K2, DeepSeek) as cross-provider fallback for reasoning tasks

This is manual today. The entropy scheduler's model rotation partially automates it -- jobs with mixed pools will naturally spread load. But the 80% rule requires conscious intervention until a quota-aware rotation system is built.

---

## Setup Steps

1. Run `openclaw models list` at session start -- always know what is actually available
2. Set default model in `openclaw.json` to your primary workhorse (`claude-sonnet-4-6`)
3. Configure per-cron model selection in `entropy-profiles.json` `model_pool` (see entropy-and-anti-detection.md)
4. Set up `token-meter.py` as a daily 9 AM cron
5. Add quota guardian: every 2h check, alert at 80% (`everyMs: 7200000` cron type)
6. Apply `MODEL_MIN_TIMEOUTS` in `entropy-scheduler.py` for any CLI-backed model pools

---

## What You Tell Your Users

> "Not every task needs the most powerful model. Your morning briefing runs on Gemini. Your code debugging runs on Codex. Your conversation with me runs on Claude. Each task gets the right tool for the job -- which means your quota goes further, your costs stay predictable, and the expensive models are available when you actually need them."

---

*Last updated: February 28, 2026*
