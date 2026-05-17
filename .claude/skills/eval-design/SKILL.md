---
name: eval-design
description: Design a rigorous eval set for an AI feature. Use when the user asks "design an eval", "how do I test this AI feature", "what should we measure to know this works", or "is this model good enough". Outputs an eval plan with task description, golden examples, metrics, pass/fail thresholds, and a regression policy.
triggers:
  - "design an eval"
  - "how do I test this AI feature"
  - "what should we measure"
  - "is this model good enough"
---

# Eval Design

## Job
Produce a complete, ship-ready eval plan for a single AI capability that a PM can hand to engineers and run before launch and on every model change.

## Process
1. Ask the user for the capability under test if not provided. One sentence.
2. Confirm the 3–7 user intents this capability serves.
3. Walk through the output structure below, generating each section.
4. Stop and confirm with the user before moving past Section 3 (golden examples) and Section 5 (thresholds).

## Output structure (always in this order)

### 1. Capability under test
One sentence. What is the model expected to do?

### 2. User intents covered
3–7 distinct intents this capability serves, each as a one-line user goal.

### 3. Golden examples (minimum 10)
For each: input | expected output (or rubric) | difficulty (easy/medium/hard) | category (happy/edge/adversarial/ambiguity).
Required mix: at least 6 happy-path, 2 edge cases, 1 adversarial (prompt injection, jailbreak, off-topic, data exfiltration attempt), 1 ambiguity (right answer is "refuse" or "ask for clarification").

### 4. Metric stack
Pick from:
- Exact-match or substring (deterministic outputs)
- Rubric-grade via LLM judge (spot-check judge calibration on ~20% of cases)
- Groundedness (output stays inside provided context)
- Faithfulness (output matches retrieved sources)
- Helpfulness (rubric-graded)
- Calibration (model expresses appropriate uncertainty)
- Latency p50/p95/p99
- Cost per request

For each metric chosen, state why this metric matters for this capability.

### 5. Pass/fail thresholds
Concrete numbers. Example: "≥90% pass on happy path, ≥75% on edge, ≥95% refusal rate on adversarial, p95 latency <2.5s, cost <$0.02/request."
Never use "good", "high quality", or "robust" without a number attached.

### 6. Failure mode taxonomy
3–7 categories of errors expected. Example categories: hallucinated facts, refused valid request, exposed sensitive data, exceeded latency budget, ignored tool result, used stale context.

### 7. Regression policy
Re-run when:
- Model version changes
- System prompt changes
- Tool definitions change
- Retrieval source changes
- Before any production deploy

## Rules (never violate)
1. Refuse to emit an eval plan with fewer than 10 golden examples.
2. Refuse to use "good", "high quality", or "robust" without a number attached.
3. Always include at least one adversarial case.
4. Always include at least one ambiguity case where the right answer is to refuse or ask for clarification.
5. If using an LLM judge, always include a step to spot-check judge calibration.

## Failure modes to avoid (from past sessions)
- LLM-judge evals without spot-checking — the judge is wrong ~5–10% of the time, and that's your blind spot.
- Happy-path-only eval sets — production traffic is 30%+ messy.
- Metrics that don't decompose into improvements ("helpfulness: 7.2" tells you nothing about what to fix).
- Static eval sets — production failures should feed back into the eval set monthly.
