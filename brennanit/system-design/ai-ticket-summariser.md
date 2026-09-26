# AI Ticket Summariser

## Clarify
- Ticket sources (ServiceNow, Jira, custom), data classification, volume, latency target.
- Output: advisory summary only, or auto-update ticket fields?
- Human review threshold? What fields are high-risk?
- Success metrics: time saved, human edit rate, user satisfaction.

## Flow
```
Request (ticket_id)
  → Validate auth + input
  → Retrieve ticket (resilient client: timeout, retry, CB, idempotency)
  → Classify data (restricted? → regional model, no logging)
  → Retrieve related context (similar tickets, KB articles) if needed
  → Construct versioned prompt (system + user + context)
  → Call approved model endpoint (token budget, temp=0)
  → Validate structured output (schema, citations, confidence)
  → If confidence < threshold → route to human review queue
  → Return summary or apply update (with audit)
  → Emit telemetry: latency, tokens, cost, quality, escalation
```

## Components
| Component | Responsibility |
|---|---|
| API Gateway | Auth, rate limit, request validation, correlation ID |
| Ticket Integration Client | Resilient ITSM API calls (auth, paging, retry, CB) |
| Data Classifier | Tag request/data with classification |
| Retrieval Service | Vector/keyword search for related context |
| Prompt Manager | Versioned prompts, template rendering |
| Model Endpoint | Approved LLM, regional, token-limited |
| Output Validator | Schema, citation check, faithfulness, confidence |
| Human Review Queue | UI + SLA for reviewer actions |
| Cache | Recent summaries (TTL, tenant isolation) |
| Observability | Logs, metrics, traces, audits |

## Non-functional
- Timeouts: 2s connect, 10s read, 30s total per external call.
- Retries: max 3, exp backoff 200ms→2s, jitter ±25%.
- Circuit breaker: open after 5 failures in 10s, half-open after 30s.
- Idempotency: key per (ticket_id, prompt_version).
- Security: least-privilege scopes, secrets in Key Vault, no PII in logs.
- Data residency: restricted data → approved regional model only.

## Trade-offs
| Decision | Option A | Option B |
|---|---|---|
| Sync vs async | Sync for <5s tickets | Async + status polling for large |
| Advisory vs auto-update | Safer, human-in-loop | Faster, needs higher confidence |
| Context retrieval | Always | Only for complex tickets |

## Risks & Mitigations
- Wrong summary → citations + faithfulness eval + human review for high-risk.
- Hallucinated fields → strict schema validation, no free-text output.
- Missing ticket → 404 with clear message, not 500.
- Model latency spike → circuit breaker + cached fallback + alert.
- Data leak → classification gate before model call, regional enforcement.