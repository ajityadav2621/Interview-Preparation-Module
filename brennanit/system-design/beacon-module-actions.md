# Beacon Module with Actions

## Clarify
- Module purpose: what action does it perform? (create ticket, update record, send mail, approve request)
- Trigger: API call, chat command, scheduled, webhook?
- Input: structured payload, free text, context from conversation?
- Output: confirmation, created resource ID, status?
- Data classification of input/output?
- Human approval required? Confidence threshold?
- Idempotency: can the same trigger execute twice safely?
- Rollback: can the action be undone/compensated?

## Flow
```
Trigger Received
  → Validate input schema + auth + idempotency key
  → Classify data → route to regional model if restricted
  → Resolve context (user, tenant, conversation, related records)
  → Construct versioned prompt (module-specific system prompt)
  → Call model (temp=0, structured output schema, token budget)
  → Validate output: schema, allowed actions, confidence, citations
  → If action required:
      If confidence < threshold → Human approval queue
      Else → Execute via Tool Gateway (least-privilege creds, idempotent)
  → Return result / status to caller
  → Emit audit: input, output, confidence, action, approver, outcome
  → Emit telemetry: latency, tokens, cost, quality, approval rate
```

## Components
| Component | Responsibility |
|---|---|
| Trigger Listener | HTTP, queue, chat platform, scheduler |
| Input Validator | Schema, sanitization, classification, idempotency check |
| Context Resolver | User, tenant, conversation, related records (cached) |
| Prompt Manager | Module-specific, versioned, template, citation format |
| Model Endpoint | Approved, regional, token budget, structured output |
| Output Validator | Schema, action allowlist, confidence, citation check |
| Approval Gate | Human review UI, SLA, audit, escalation |
| Tool Gateway | Allowlisted connectors, input validation, output validation, rate limit, idempotency |
| Action Executor | Compensating action on failure, audit, retry |
| Observability | Per-module metrics, traces, cost, quality, audit |

## Non-functional
- Idempotency: every mutating action carries idempotency key (trigger ID + action hash).
- Least privilege: module identity ≠ caller identity; tool creds scoped to action.
- Approval: configurable thresholds per action type; audit trail immutable.
- Rollback: compensating action defined per tool (delete, revert, cancel).
- Token budget: hard limit per module execution; fail fast if exceeded.
- Observability: correlation ID from trigger → module → tool → model.

## Trade-offs
| Decision | Option A | Option B |
|---|---|---|
| Auto-approve high-confidence | Fast | Risk of wrong action |
| Require approval for all | Safest | Slow, reviewer fatigue |
| Inline execution | Low latency | Coupled, less resilient |
| Queued execution | Resilient, decoupled | Higher latency, status polling |

## Risks & Mitigations
- Unauthorized action → tool allowlist + least-privilege creds + approval gate.
- Duplicate action → idempotency key at executor + deduplication.
- Action failure mid-way → compensating action, saga pattern, alert on partial.
- Prompt injection → input sanitization, instruction/data separation, output validation.
- Module drift → eval harness per module, regression gate, usage monitoring.