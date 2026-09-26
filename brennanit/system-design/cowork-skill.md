# Cowork Skill

## Clarify
- Trigger: user message, scheduled, webhook, API call?
- Input: free text, structured payload, context from conversation?
- Output: response to user, action (create ticket, update record), or both?
- Data classification of inputs/outputs?
- Human approval required for actions? Threshold?
- Idempotency: can the same trigger fire twice safely?
- Latency target for user-facing vs background?

## Flow
```
Trigger Received
  → Validate input schema + auth
  → Check idempotency key (if provided)
  → Classify data → route to regional model if restricted
  → Retrieve context (conversation history, user profile, KB)
  → Construct versioned prompt (skill-specific system prompt)
  → Call model (temp low, structured output schema)
  → Validate output (schema, allowed actions, confidence)
  → If action + confidence < threshold → Human approval queue
  → Execute action (via tool gateway, least-privilege creds)
  → Return response to user / ack to trigger source
  → Emit telemetry: latency, tokens, actions, approvals, errors
```

## Components
| Component | Responsibility |
|---|---|
| Trigger Listener | HTTP, queue, scheduler, chat platform webhook |
| Input Validator | Schema, sanitization, classification |
| Context Retriever | Conversation, user, KB, RAG if needed |
| Prompt Manager | Skill-specific, versioned, template |
| Model Endpoint | Approved, regional, token budget |
| Output Validator | Schema, action allowlist, confidence |
| Approval Gate | Human review UI, SLA, audit |
| Tool Gateway | Allowlisted connectors, validation, rate limit |
| Action Executor | Idempotent, audited, compensating action on failure |
| Observability | Per-skill metrics, traces, cost, quality |

## Non-functional
- Idempotency: every mutating action carries idempotency key.
- Rate limiting: per-skill, per-user, global.
- Secrets: tool creds from Key Vault at runtime, rotated.
- Audit: every trigger, decision, action, approval logged.
- Human-in-loop: configurable thresholds, escalation path.

## Trade-offs
| Decision | Option A | Option B |
|---|---|---|
| Inline vs queued | Lower latency | Better resilience, decoupling |
| Auto-approve low-risk | Faster | Risk of unintended actions |
| Single skill vs composable | Simpler | Reusable building blocks |

## Risks & Mitigations
- Duplicate action → idempotency key + deduplication at executor.
- Unauthorized action → tool allowlist + least-privilege creds + approval gate.
- Skill drift → evaluation harness per skill, regression gate.
- Low adoption → worked examples, discoverability, feedback channel.
- Prompt injection → input sanitization, instruction/data separation, output validation.