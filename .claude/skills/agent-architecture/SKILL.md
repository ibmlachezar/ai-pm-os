---
name: agent-architecture
description: Decide which AI architecture fits a given capability: single-turn prompt, workflow, RAG, agent, or fine-tuning. Use when the user asks "should this be an agent or a workflow", "agent vs RAG", "do we need to fine-tune", "how should we architect this AI feature", or "what's the simplest thing that could work here". Outputs a recommendation with trade-offs, cost/latency estimate, and an eval-design hand-off.
triggers:
  - "should this be an agent"
  - "agent or workflow"
  - "agent vs RAG"
  - "do we need to fine-tune"
  - "how should we architect this AI feature"
  - "simplest thing that could work"
---

# Agent Architecture

## Job
Pick the simplest AI architecture that could meet the capability's requirements. Default toward simplicity. Surface trade-offs the PM and engineers must own.

## Process
1. Ask for the capability if not provided. One sentence.
2. Ask three diagnostic questions before recommending:
   - Is the path from input to output known ahead of time, or does it depend on intermediate results?
   - Does the answer depend on knowledge that changes or is too large for the prompt?
   - What is the latency budget (p95) and per-request cost ceiling?
3. Walk through the five candidate architectures and eliminate the ones that don't fit.
4. Recommend one. State the trade-offs explicitly.
5. Hand off to eval-design with a note on what kind of eval this architecture needs.

## The five candidates (in order of preference)

### 1. Single-turn prompt
One input, one output. No tools, no memory.
Use when: task is well-defined, all needed context fits in the prompt, no fresh data needed.
Cost: ~1 model call. Latency: lowest. Reliability: highest.

### 2. Workflow (chained prompts)
Multiple LLM calls in a code-defined sequence. Deterministic control flow.
Use when: task decomposes into known steps.
Cost: 3-10 calls. Latency: medium. Reliability: high — control flow is yours.

### 3. RAG
Workflow + retrieval step pulling external context before generation.
Use when: answers depend on changing or large knowledge (docs, tickets, product data).
Cost: 3-10 calls + retrieval. Latency: medium. Reliability: high if retrieval is good.

### 4. Agent
LLM picks tools and order, loops until done.
Use when: the path is genuinely unknown ahead of time and depends on intermediate results.
Cost: 5-50+ calls. Latency: highest. Reliability: lowest — non-deterministic.

### 5. Fine-tuning
Modify model weights.
Use when: prompting + RAG cannot achieve the required format/style/domain AND you have ≥1000 high-quality examples AND the capability is stable.
Cost: training + per-call. Latency: low at inference. Reliability: high once trained, low across model changes.

## Output structure (always in this order)

1. **Capability under test.** One sentence.
2. **Diagnostic answers.** What the user said about path, knowledge needs, latency/cost.
3. **Architecture recommendation.** One of the five, named.
4. **Why this one.** 2-4 sentences referencing the diagnostics.
5. **What was ruled out and why.** Cover each of the other four candidates in one line each.
6. **Trade-offs to own.** 3-5 explicit trade-offs (cost spike, latency, non-determinism, eval complexity, model-version coupling).
7. **Eval implication.** Which eval shape this needs: outcome eval, trajectory eval, step-level eval, retrieval eval. Hand off to eval-design.

## Rules (never violate)
1. Default to single-turn or workflow. Only escalate to agent if the diagnostics force it.
2. Refuse to recommend fine-tuning unless the user confirms they have ≥1000 high-quality examples and have tried prompting + RAG first.
3. Always state the cost and latency order-of-magnitude estimate before recommending.
4. Never recommend agent for a task with a known, fixed path — call that out as a misuse.
5. Always hand off to eval-design with a note on the eval shape.

## Failure modes to avoid (from past sessions)
- "Agent everywhere" — most agent recommendations should have been workflows. Push back hard.
- Recommending fine-tuning before prompting + RAG have been exhausted.
- Ignoring latency budget — a 50-call agent is unusable in a real-time UI.
- Pretending agent reliability is comparable to workflow reliability. It isn't.
- Skipping the hand-off to eval-design.
