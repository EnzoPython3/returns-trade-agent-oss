# AI refund prompts

The shipped code makes every refund decision with fixed rules (the database checks in `return_decision()`, plus the rule engine in `frontend/reply_engine.py`). No AI model is called and no AI key is needed.

An earlier version asked an AI model (xAI Grok, or Groq when the key started with `gsk_`) to make the first refund decision, and fell back to the rules when no key was set or the call failed. The prompts are kept here so you can plug in a model of your own.

## Where to put the call

Call the model from the server, never from the browser. A key in browser code can be read by anyone who opens the page.

Good places:
- a Supabase Edge Function that reads the key with `Deno.env.get(...)` after `supabase secrets set MY_AI_KEY=...`
- `frontend/reply_engine.py`, inside `adjudicate_refund()`, just before the rule-based decision, reading the key from an environment variable

Always keep the rules as the fallback, and keep the database as the final judge: a return the database refuses is refused, whatever the model says.

## Settings used

| Setting | Value |
|---|---|
| Endpoint | `https://api.x.ai/v1/chat/completions` (Groq: `https://api.groq.com/openai/v1/chat/completions`) |
| Model | `grok-3-mini` (Groq: `llama-3.3-70b-versatile`) |
| Temperature | `0.1` |
| Response format | `{"type": "json_object"}` |
| Timeout | 12 seconds, then fall back to the rules |

Both endpoints use the OpenAI-style chat format, so any compatible provider works.

## System prompt

Values in `{...}` come from `frontend/business.json`.

```text
You are the official AI Refund Adjudicator for {name}.
Tone: {tone}.
Currency: {currency}.
Business Policies:
- Refund Policy: {policies.refund}
- Delivery Policy: {policies.delivery}
- Payment Policy: {policies.payment}

ADJUDICATION RULES:
1. If detected_flags contains 'possible_scam_or_payment_redirect' or customer asks to change bank/EFT details, the refund MUST be DECLINED (is_valid: false, decision: 'DECLINED').
2. If the order is older than 14 days (days_since_purchase > 14), the refund MUST be DECLINED (is_valid: false, decision: 'DECLINED').
3. If the reason is 'Changed my mind' or non-defective/opened item without valid defect, DECLINE automated refund so a manager can inspect it.
4. If the order is within 14 days of delivery (<= 14 days), has valid Order ID proof of payment, and states a valid issue (damaged, cold, wrong item/size, late delivery), APPROVE the refund (is_valid: true, decision: 'APPROVED').

Respond ONLY with a valid JSON object matching this exact schema:
{
  "is_valid": true or false,
  "decision": "APPROVED" or "DECLINED",
  "confidence": 0.92,
  "intent": "refund" | "complaint" | "human" | "payment",
  "needs_human": true or false,
  "decision_reason": "Clear 1-2 sentence explanation of why the refund was approved or declined",
  "reply": "Friendly, concise customer reply ending with Kind regards,\nThe Returns Trade Agent Team"
}
```

## User message

A JSON object, pretty printed:

```json
{
  "order_code": "DEMO-1001",
  "days_since_purchase": 3,
  "selected_items": [],
  "refund_amount": 249.0,
  "reason": "Item arrived damaged",
  "customer_notes": "",
  "detected_security_flags": [],
  "initial_intent": "refund"
}
```

`detected_security_flags` and `initial_intent` come from the rule engine's `flags()` and `classify()` in `frontend/reply_engine.py`, run before the model.

## Handling the answer

- Read `choices[0].message.content` and parse it as JSON.
- If `decision` is not `APPROVED` or `DECLINED`, use `is_valid` to choose.
- Force `needs_human` to true when there are any security flags or the decision is `DECLINED`.
- On any error or timeout, use the rule-based decision instead.
