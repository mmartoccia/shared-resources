# Council of the Wise

## What it is

The Council of the Wise is a multi-model review pattern for high-stakes decisions. Before committing to a significant architectural decision, security design, venture strategy, or behavioral change, the agent spawns sub-agents across multiple frontier models to get independent perspectives. The council red-teams the proposal, identifies blind spots, and synthesizes a verdict.

**Standing directive:** Big system ideas always go to the Council before building.

## Why multi-model review works

Different model families have genuinely different priors, training emphases, and failure modes. An architectural decision that looks solid to Claude may have a flaw that Gemini or Kimi K2 catches -- not because one is smarter, but because they approach the problem differently. The value is in the disagreements, not the consensus.

## Trigger conditions

Use the Council when:
- Designing a new security architecture or trust model
- Making a significant infrastructure decision (new persistent service, major cron, new data store)
- Evaluating a venture or business strategy
- Any decision where "if this is wrong, it's expensive to undo"
- Explicit user request: "bring in the council" / "get other opinions"

Do NOT use for:
- Routine tasks, quick lookups, single-session work
- Decisions that are easily reversible
- When speed matters more than thoroughness

## The four-perspective framework

Each council run assigns roles to the spawned sub-agents:

| Role | Focus | Voice |
|---|---|---|
| 👹 Devil's Advocate | Challenge every assumption. Strongest argument against. Specific failure modes paired with paths forward. | Adversarial, specific |
| 🏗️ Architect | Is this the right abstraction? One-way doors? What will you wish you'd decided differently in 6 months? | Structural, long-horizon |
| 🛠️ Engineer | Implementation feasibility. Where will this break in practice? What's missing from the spec? Build order. | Concrete, skeptical |
| 🎨 Artist/UX | How does the human experience this? What breaks the relationship if it goes wrong? Interaction model. | Human-centered |
| ⚖️ Synthesis | Combine best insights. Flag where council members disagree -- that's the gold. 3 critical decisions before building. | Balanced, decisive |

## Model selection for council runs

- **Opus** (anthropic/claude-opus-4-5 with thinking): full 4-perspective council, deep reasoning
- **Gemini 3 Pro** (google-gemini-cli/gemini-3-pro-preview): adversarial red-team, security focus
- **Kimi K2 Thinking** (nvidia-nim/moonshotai/kimi-k2-thinking): alternative reasoning style
- **GPT-5.x** (openai-codex/gpt-5.3-codex): engineering-focused critique
- Always check `openclaw models list` for current fleet before selecting

## How to run a council

```bash
# Via sessions_spawn -- Opus full council
sessions_spawn(
  task="[full council prompt with proposal details + four perspective assignments]",
  model="anthropic/claude-opus-4-5",
  label="council-[topic]",
  mode="run"
)

# Gemini security red-team in parallel
sessions_spawn(
  task="[adversarial security analysis prompt]",
  model="google-gemini-cli/gemini-3-pro-preview",
  label="council-[topic]-security",
  mode="run"
)
```

Both run in parallel (independent tasks, no dependencies). Wait for both before delivering combined verdict.

## Council prompt structure

A good council prompt includes:
1. The proposal in full -- what you're building, why, the key design decisions
2. The four role assignments with explicit instructions
3. A neuroscience/domain validity check if applicable
4. "Be direct, specific, and adversarial where warranted. Vague praise is useless."

## Reading council output

The highest-value content is **where council members disagree**. Consensus = probably fine. Disagreement = that's where the real decision lives.

Look for:
- Security flaws identified by multiple independent reviewers (high confidence)
- Implementation concerns the architect didn't raise (integration risk)
- UX failure modes that pure technical analysis misses
- The "3 critical decisions before building" from the synthesis

## Real example: Trusted Channel Architecture

The council identified:
- Telegram as control plane: "catastrophic" (Gemini) / "cosmetic security" (Opus DA)
- Immutable blocked list as fiction until enforced at OS level
- `propose_behavioral_change` as the primary social engineering attack surface
- Hebbian learning: valid inspiration, but "neurons that fire together wire together" needs a concrete mechanism -- without one it's a metaphor without implementation

Result: redesigned transport from Telegram to HTTP/Tailscale before building.

## What you tell your users

> "Before we build anything significant, we bring in the council. We spawn multiple AI models -- Claude, Gemini, Kimi -- and have them independently red-team the proposal. The value isn't the consensus; it's the disagreements. If three different models from different companies all flag the same flaw, that's a high-confidence signal you'd have missed. We use the council's verdict to make three key decisions before writing a line of code."

## Setup steps

1. Install council skill: `clawdhub install council-of-the-wise`
2. Or use sessions_spawn directly (no skill required)
3. Keep council agents persona files in `~/clawd/skills/council-of-the-wise/agents/`
4. Add standing directive to SOUL.md: "Big system ideas → always bring in the Council before building"
5. After each council run: document the key findings and the decisions made in response
