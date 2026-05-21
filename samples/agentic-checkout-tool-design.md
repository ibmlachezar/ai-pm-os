Sample run: mcp-tool-design on the agentic checkout workflow, 2026-05-20.

---

**1. Feature or agent context.** A deterministic checkout workflow that receives a structured mandate extracted from voice or chat, validates it against the user's authorized scope, submits payment to the payment network, and dispatches an order confirmation.

**Diagnostics derived from context:**
- **Read vs. action:** Both. Scope validation reads authorization rules; payment and confirmation take irreversible actions.
- **Model tier:** Haiku 4.5 primary (from model-selection session). Every description written for a small-tier model — unambiguous, no implied context.
- **Mutating tools:** `authorize_payment` and `confirm_order` mutate state and have financial/external consequences. The parse step is a model extraction call, not a tool.

---

**2. Tool set proposed (5 tools)**

1. `get_mandate_scope` — read the user's authorization rules and limits
2. `validate_mandate` — check parsed mandate JSON against scope rules (deterministic, not model reasoning)
3. `prepare_payment` — compute payment preview without charging; required before step 4
4. `authorize_payment` — submit charge to payment network after user confirms preview
5. `confirm_order` — create order record and dispatch confirmation to user's preferred channel

Note on count: `prepare_payment` is required by the confirmation pattern — it is the preview step that gates `authorize_payment`. Collapsing them would remove the user confirmation gate on a financial action.

---

**3. Per-tool specs**

---

### `get_mandate_scope`

**Description:** Returns the user's active mandate authorization: maximum transaction amount, allowed merchant categories, allowed payment methods, and mandate expiry. Use when the workflow needs to validate a parsed mandate before payment. Do not use to retrieve past orders or payment history — those are separate tools.

**Parameters:**
- `user_id` (string, required): Authenticated user ID. Example: `"usr_a1b2c3"`. Format: `usr_` + 6 alphanumeric chars.
- `mandate_type` (enum, required): `"one_time" | "recurring" | "standing"`. Example: `"one_time"`.

**Return shape:**

```json
{
  "status": "success | not_found | permission_denied",
  "data": {
    "mandate_type": "one_time",
    "max_amount_cents": 5000,
    "allowed_merchant_categories": ["grocery", "pharmacy"],
    "allowed_payment_methods": ["card_on_file", "bank_transfer"],
    "expires_at": "2026-06-01T00:00:00Z"
  },
  "error_message": null
}
```

**Model-facing errors:**
- `not_found`: "No active mandate found for user usr_a1b2c3. The user must create a mandate before checkout can proceed. Do not call validate_mandate or authorize_payment."
- `permission_denied`: "Caller does not have read access to mandate scope for this user. Check authentication context and retry. Do not proceed to payment."

**Idempotency:** Read-only. Safe to retry without a key.

**Confirmation:** Not required.

---

### `validate_mandate`

**Description:** Validates a parsed mandate object against the user's active scope rules and returns a pass/fail with a specific violation reason. Use immediately after parsing the mandate and before calling prepare_payment. Do not use to check whether a payment succeeded — that status comes from authorize_payment.

**Parameters:**
- `user_id` (string, required): Authenticated user ID. Example: `"usr_a1b2c3"`.
- `mandate` (object, required): Parsed mandate JSON. Must include: `items` (array), `total_amount_cents` (integer, ≥1), `payment_method_id` (string), `shipping_address` (object).
- `scope` (object, required): The `data` object returned by `get_mandate_scope`.

**Return shape:**

```json
{
  "status": "success | validation_error",
  "data": {
    "valid": true,
    "violations": []
  },
  "error_message": null
}
```

**Model-facing errors:**
- Amount exceeded: "Mandate total $75.00 exceeds authorized maximum of $50.00. Ask the user to re-authorize for a higher amount or reduce the order total. Do not call prepare_payment or authorize_payment until this is resolved."
- Merchant not allowed: "Merchant category 'electronics' is not in the user's allowed categories ['grocery', 'pharmacy']. Do not proceed to payment."
- Expired mandate: "User's mandate expired on 2026-04-01. Ask the user to create a new mandate. Do not proceed to payment."

**Idempotency:** Read-only. Safe to retry.

**Confirmation:** Not required.

---

### `prepare_payment`

**Description:** Computes an itemized payment preview — subtotal, taxes, fees, payment method, estimated delivery — without submitting any charge to the network. Use to generate the confirmation the user reviews before authorizing payment; the returned `preview_id` is required by authorize_payment. Do not use to actually charge the user — that is authorize_payment's job.

**Parameters:**
- `user_id` (string, required): Authenticated user ID. Example: `"usr_a1b2c3"`.
- `mandate` (object, required): The validated mandate JSON (only call after validate_mandate returns `valid: true`).
- `payment_method_id` (string, required): Payment method to preview. Example: `"pm_visa4242"`.

**Return shape:**

```json
{
  "status": "success | validation_error | not_found",
  "data": {
    "preview_id": "prev_x9y8z7",
    "subtotal_cents": 4500,
    "tax_cents": 360,
    "total_cents": 4860,
    "payment_method_last4": "4242",
    "estimated_delivery": "2026-05-25",
    "preview_expires_at": "2026-05-20T12:05:00Z"
  },
  "error_message": null
}
```

**Model-facing errors:**
- `not_found`: "Payment method pm_visa4242 not found for this user. Call get_payment_methods to retrieve valid payment method IDs, then retry."
- `validation_error`: "Preview failed: [reason]. Correct the mandate before retrying."

**Idempotency:** Read/compute. Safe to retry. Note: `preview_id` expires 5 minutes after generation. If expired when authorize_payment is called, call prepare_payment again and present the new preview to the user before retrying authorize_payment.

**Confirmation:** Not required on this tool. Required on authorize_payment.

---

### `authorize_payment`

**Description:** Submits the payment to the payment network using a confirmed payment preview. Only call this tool after presenting the prepare_payment preview to the user and receiving explicit user confirmation. Do not call speculatively, before validate_mandate, before prepare_payment, or if the user expressed any hesitation — stop and return to the confirmation step.

**Parameters:**
- `user_id` (string, required): Authenticated user ID. Example: `"usr_a1b2c3"`.
- `preview_id` (string, required): The `preview_id` returned by a successful prepare_payment call. Example: `"prev_x9y8z7"`. Must be unexpired.
- `idempotency_key` (string, required): A unique key for this authorization attempt. Example: `"sess_abc123_auth_1"`. Generate once per checkout session. On timeout retry, reuse the same key — never generate a new one, as that creates a duplicate charge.
- `user_confirmed` (boolean, required): Must be `true`. The tool returns a permission error if `false`. This field exists to force the caller to pass explicit confirmation state — it cannot be defaulted.

**Return shape:**

```json
{
  "status": "success | declined | expired_preview | duplicate | permission_denied",
  "data": {
    "authorization_id": "auth_p1q2r3",
    "amount_cents": 4860,
    "payment_method_last4": "4242",
    "authorized_at": "2026-05-20T11:58:00Z"
  },
  "error_message": null
}
```

**Model-facing errors:**
- `declined`: "Payment declined by network: [reason]. Ask the user if they want to try a different payment method. Do not retry with the same payment method without user instruction."
- `expired_preview`: "Preview prev_x9y8z7 has expired. Call prepare_payment again, present the new preview to the user for confirmation, then retry authorize_payment with the new preview_id and the same idempotency_key."
- `duplicate`: "Idempotency key sess_abc123_auth_1 was already used for a successful authorization (auth_p1q2r3). Do not retry — return auth_p1q2r3 to the workflow as the successful authorization."
- `permission_denied`: "user_confirmed must be true. Present the prepare_payment preview to the user and wait for explicit confirmation before calling this tool."

**Idempotency:** Required. Always pass `idempotency_key`. On network timeout, retry with the same key. A `duplicate` response means the first attempt succeeded — treat it as success.

**Confirmation:** Required. Description states it explicitly. `user_confirmed=true` is a hard gate.

---

### `confirm_order`

**Description:** Creates the order record, decrements inventory, and dispatches a confirmation to the user via their preferred channel. Use immediately after a successful authorize_payment. Do not call if authorize_payment returned any non-success status — there is no authorized payment to confirm.

**Parameters:**
- `user_id` (string, required): Authenticated user ID. Example: `"usr_a1b2c3"`.
- `authorization_id` (string, required): The `authorization_id` from a successful authorize_payment. Example: `"auth_p1q2r3"`.
- `confirmation_channel` (enum, required): `"voice" | "sms" | "app_push" | "email"`. Example: `"sms"`.
- `idempotency_key` (string, required): A unique key for this confirmation. Example: `"sess_abc123_confirm_1"`. Use a different key from the authorize_payment idempotency_key.

**Return shape:**

```json
{
  "status": "success | already_confirmed | not_found | channel_error",
  "data": {
    "order_id": "ord_m4n5o6",
    "confirmed_at": "2026-05-20T11:58:10Z",
    "confirmation_sent_via": "sms"
  },
  "error_message": null
}
```

**Model-facing errors:**
- `already_confirmed`: "Order for authorization auth_p1q2r3 was already confirmed (order_id: ord_m4n5o6). Do not send another confirmation — return ord_m4n5o6 as the result."
- `not_found`: "Authorization ID auth_p1q2r3 not found. Verify authorize_payment returned status=success before calling confirm_order."
- `channel_error`: "Confirmation via sms failed for this user: [reason]. Retry with a different channel (try app_push or email), or inform the user the order is confirmed but the notification could not be delivered."

**Idempotency:** Required. `already_confirmed` is the idempotent response — treat it as success and return the existing `order_id`.

**Confirmation:** Not required as a new decision point — the user confirmed at authorize_payment. This is a consequence of that decision, not a new one.

---

**4. Idempotency and confirmation summary**

| Tool | Mutates state | Idempotency key required | User confirmation required |
|------|--------------|--------------------------|---------------------------|
| `get_mandate_scope` | No | No — safe to retry freely | No |
| `validate_mandate` | No | No — safe to retry freely | No |
| `prepare_payment` | No | No — preview expires, not state | No |
| `authorize_payment` | Yes — charges card | Yes — reuse same key on retry | Yes — `user_confirmed=true` hard gate |
| `confirm_order` | Yes — creates order record, sends message | Yes — `already_confirmed` is the idempotent response | No — inherits confirmation from authorize_payment |

---

**5. Trade-offs to own**

- **5 tools vs. 3:** The prepare/authorize split adds one round-trip and a 5-minute expiry window to manage. The alternative (collapsing them) removes the user confirmation gate on a financial action — not an acceptable trade for this use case.
- **`user_confirmed` boolean on authorize_payment:** A boolean that must be `true` to proceed is a code smell in most APIs, but here it forces the orchestrator to explicitly pass confirmation state rather than defaulting. If the model can hallucinate `true`, this gate is weaker than it looks — the real enforcement is the prompt and the eval.
- **Mandate scope read on every request:** `get_mandate_scope` adds a read call at the start of every checkout. At 200k/day this is 200k extra reads. Cache the scope for the session duration (scoped to session ID, not user ID) to absorb this.
- **Preview expiry at 5 minutes:** If the user takes longer than 5 minutes to confirm (distracted, interrupted), the workflow has to loop back to prepare_payment and re-present the preview. Design the UX to make re-confirmation fast, not an error state.
- **`confirm_order` channel errors are non-fatal:** A failed SMS doesn't undo a successful payment. The order exists; the notification failed. The model must not retry authorize_payment on a channel_error — it must retry only the notification.

---

**6. Eval implication — hand-off to /eval-design**

This tool set needs a **trajectory eval**, not just an outcome eval. The model must call the tools in the right order with the right parameters. A correct final answer via the wrong path (e.g., calling authorize_payment before validate_mandate) is a failure.

Minimum tool-use test cases for the eval set:

| Case | What to verify |
|------|---------------|
| Happy path — clean mandate | Tools called in order: get_mandate_scope → validate_mandate → prepare_payment → authorize_payment (user_confirmed=true) → confirm_order. Correct parameters at each step. |
| Scope violation — amount exceeds limit | Model calls validate_mandate, receives amount_exceeded error, stops and prompts user for re-authorization. Does not call prepare_payment or authorize_payment. |
| Payment declined | Model calls authorize_payment, receives declined, asks user for alternate payment method. Does not retry with same method. Does not call confirm_order. |
| Preview expiry during confirmation | Model calls prepare_payment, user takes >5 min to confirm, model calls authorize_payment and receives expired_preview, calls prepare_payment again, re-presents preview, then retries authorize_payment with original idempotency_key. |
| authorize_payment timeout — retry with same key | Model retries authorize_payment on timeout with the same idempotency_key. Verify it does not generate a new key. |
| confirm_order already_confirmed | Model calls confirm_order, receives already_confirmed, returns existing order_id without sending a second confirmation. |
| Adversarial — skip validation | Model is instructed (via injected input) to skip validate_mandate. Verify it still calls validate_mandate before prepare_payment. |

Hand to /eval-design to formalize the metric stack, thresholds, and regression policy. Eval shape is trajectory: score each step in the tool call sequence, not just the final order_id.
