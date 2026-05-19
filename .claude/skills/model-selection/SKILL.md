---
name: model-selection
description: Pick the right model for a given AI capability and justify the choice on capability, cost, latency, context, and constraints. Use when the user asks "which model should we use", "Opus or Sonnet or Haiku", "GPT vs Claude vs Gemini", "is this model good enough", "can we use a cheaper model here", or "should we cascade models". Outputs a recommendation, a cascade strategy if applicable, and an eval plan to validate the choice.
triggers:
  - "which model should we use"
  - "Opus or Sonnet or Haiku"
  - "GPT vs Claude vs Gemini"
  - "is this model good enough"
  - "can we use a cheaper model"
  - "should we cascade models"
---

# Model Selection

## Job
Pick the model (or model cascade) that meets the capability's requirements at the lowest cost and latency. Force the decision to be made on explicit trade-offs, not vibe. Hand off to eval-design to validate.

## Process
1. Ask for the capability if not provided.
2. Ask four diagnostic questions before recommending:
   - Task tier: extraction/classification, summarization/drafting, reasoning/planning, or agentic tool-use?
   - Latency budget: p95 ceiling in milliseconds?
   - Cost ceiling: per-request and per-month?
   - Context size: max tokens of input expected, including any retrieved context and tool results?
3. Map task tier to model tier, then narrow by latency, cost, and context.
4. Recommend one primary model. Propose a cascade if a cheaper model could handle the easy fraction.
5. Hand off to eval-design with the candidate models named.

## The model tiers (2026)

### Frontier tier — hard reasoning, complex agentic work
- Claude Opus 4.7
- GPT-5
- Gemini 2.5 Pro
Use when: multi-step reasoning, complex tool-use, code generation at length, decisions with real consequences.
Cost: highest. Latency: highest. Reliability on hard tasks: highest.

### Mid tier — most production workloads
- Claude Sonnet 4.6
- GPT-5 mini
- Gemini 2.5 Flash
Use when: drafting, summarization, structured extraction, routine tool-use, RAG generation. The default starting point for most features.
Cost: ~5-10x cheaper than frontier. Latency: medium. Reliability on routine tasks: very high.

### Small tier — high-volume, simple tasks
- Claude Haiku 4.5
- GPT-5 nano
- Gemini 2.5 Flash Lite
Use when: classification, intent detection, simple extraction, first-pass filtering, real-time UIs where latency dominates.
Cost: ~10-30x cheaper than frontier. Latency: lowest. Reliability on simple tasks: high; on hard tasks: poor.

## The five axes (always evaluate all five)
1. Capability — does this tier reliably do the task on your eval set?
2. Cost — per-request and per-month at expected volume.
3. Latency — time to first token + total time at p95.
4. Context — max input tokens including retrieved context and tool results.
5. Constraints — tool-use reliability, structured-output reliability, safety posture, data residency, vendor lock-in.

## Cascade pattern (the most underused PM lever)
For most production features:
- Try the small or mid-tier model first.
- Gate on a confidence signal or eval-style check.
- Fall back to the frontier model only when the cheap path fails.
Result: 70-95% of traffic served at small/mid cost; quality of frontier on the hard cases.

## Output structure (always in this order)
1. **Capability under test.** One sentence.
2. **Diagnostic answers.** Task tier, latency, cost, context.
3. **Primary recommendation.** Named model. One paragraph on why.
4. **Cascade proposal.** If applicable: which cheap model first, what gates the fallback, what fraction of traffic each tier likely handles.
5. **Ruled-out alternatives.** Each of the other major options in one line.
6. **Trade-offs to own.** 3-5 explicit ones (cost ceiling, latency at p95, context limit, vendor lock-in, model deprecation risk).
7. **Eval plan hand-off.** Which models to run the eval set against. Hand off to eval-design.

## Rules (never violate)
1. Refuse to recommend without diagnostic answers. No vibe picks.
2. Always propose a cascade when the task tier is mixed (some easy, some hard cases).
3. Refuse to recommend on public benchmark scores alone. State that the user's own eval set is the only benchmark that matters.
4. Always state the cost difference order-of-magnitude (e.g., "Haiku is ~10-30x cheaper than Opus").
5. Always hand off to eval-design with at least 2-3 model candidates to compare.

## Failure modes to avoid (from past sessions)
- "Just use Opus / GPT-5 / Gemini 2.5 Pro" without diagnostics — most production traffic doesn't need frontier.
- Ignoring latency for voice/chat UIs — Opus at 2s p95 breaks the UX.
- Picking by leaderboard, not by the user's eval set.
- Forgetting the cascade — paying frontier prices for easy cases.
- Ignoring vendor lock-in — single-provider designs are a liability.
