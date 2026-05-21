---
name: mcp-tool-design
description: Design a tool (or set of tools) that an LLM can use reliably — name, description, parameter schema, return shape, error messages, idempotency, scope. Use when the user asks "design a tool for X", "review this tool definition", "how should I structure this MCP server", "my agent isn't calling the right tool", "design the tools for this agent or workflow", or "what tools does this feature need". Outputs a tool spec the engineer can implement and an eval-design hand-off for tool-use evals.
triggers:
  - "design a tool"
  - "review this tool definition"
  - "design the tools for"
  - "how should I structure this MCP server"
  - "my agent isn't calling the right tool"
  - "what tools does this feature need"
---

# MCP Tool Design

## Job
Produce a tool spec (or a set of them) that an LLM can use reliably across model tiers. Default toward small, single-purpose tools with model-friendly errors. Hand off to eval-design for tool-use evals.

## Process
1. Ask for the feature or agent context if not provided.
2. Ask three diagnostic questions:
   - What does the model need to do that it can't do with just a prompt? (Read data, take action, both?)
   - What model tier will use these tools? Design for the lowest tier.
   - Are any of these tools mutating (taking actions with consequences) or all read-only?
3. Propose the minimum set of tools that covers the feature. Default toward fewer.
4. For each tool, produce the full spec using the structure below.
5. Surface idempotency and confirmation needs explicitly.
6. Hand off to eval-design for tool-use eval coverage.

## Tool spec structure (use this for every tool)

### Name
Verb-object format. Unambiguous. No internal jargon.
Good: `get_order_status`, `create_refund`, `list_subscriptions`
Bad: `fetch_data`, `do_thing`, `helper_function_v2`

### Description
Two sentences.
- Sentence 1: What the tool does.
- Sentence 2: When to use it AND when not to. The "when not to" is mandatory.

### Parameter schema
JSON Schema. For each parameter:
- Tight type (no plain `string` when you can use an enum)
- One-line description that includes an example value
- Required vs optional explicitly marked
- Validation rules (length, format, range)

### Return shape
Structured, consistent across success and failure. Always include a `status` field. Avoid bare strings.

Example shape (JSON):

    {
      "status": "success | not_found | permission_denied | validation_error",
      "data": { ... },
      "error_message": "Present when status is not success. Tells the model what happened and what to do next."
    }

### Error messages (model-facing)
Tell the model what went wrong AND what to do next. Examples:
- Good: "Order ID OR-1234 not found. Use list_orders to find a valid order ID."
- Bad: "404 Not Found."
- Good: "Refund amount $150 exceeds order total $100. Cap refund at order total or call get_order_details to verify the amount."
- Bad: "Invalid amount."

### Idempotency
If the tool takes an action (mutates state):
- Require an idempotency_key parameter, OR
- Make the action naturally idempotent (set state to a value, don't increment), OR
- Document explicitly that the tool is not safe to retry and the agent should not retry on timeout.

### Confirmation pattern
If the tool charges money, sends a message, deletes data, or has external consequences:
- Description must say "Only call this tool after explicit user confirmation."
- Consider splitting into a prepare_X (returns a preview) and execute_X (takes the action) pair.

### Scope
One tool, one job. If the spec covers two things, split it.

## Output structure (always in this order)
1. **Feature or agent context.** One sentence.
2. **Tool set proposed.** Numbered list with one-line purposes.
3. **Per-tool specs.** Full structure above for each.
4. **Idempotency and confirmation summary.** Which tools mutate, which require idempotency keys, which require user confirmation.
5. **Trade-offs to own.** 3-5 explicit ones (tool count, error verbosity vs token cost, granularity vs flexibility, idempotency complexity, version compatibility).
6. **Eval implication.** Tool-use evals — did the model pick the right tool, with the right parameters, in the right order? Hand off to eval-design with a list of tool-use test cases.

## Rules (never violate)
1. Every tool description must include both "use when" and "do not use when."
2. Every mutating tool must address idempotency explicitly (key, naturally idempotent, or documented unsafe to retry).
3. Every action with external consequences (money, messages, deletions) must include a confirmation pattern.
4. Refuse to design more than ~12 tools for a single feature without justification. Past ~20 tools, models misuse them.
5. Error messages must be written for the model, not the developer. Always include what-to-do-next.
6. Always hand off to eval-design with at least 3 tool-use test cases per tool.

## Failure modes to avoid (from past sessions)
- Generic names (fetch_data, do_thing) — the model has to guess what they do.
- Missing "do not use when" in descriptions — model uses the tool in wrong contexts.
- Inconsistent return shapes between success and failure — model produces brittle code paths.
- Bare HTTP-style errors (404, 500) — model has nothing to act on.
- Action tools without idempotency keys — agent retries on timeout, user gets double-charged.
- Tools that wrap entire workflows ("complete_checkout") instead of atomic actions.
- Over-tooling. 20+ tools per feature is almost always a smell.
