# Integrations & Resilience

## 1. Build a resilient client for an external REST API (e.g., ServiceNow, Graph)

**Flow:** Resolve credentials → construct request with timeout (connect, read, total) → send with idempotency key if mutating → handle response:
- 2xx: normalize, validate, return
- 401: refresh token, retry once
- 403: stop, alert if unexpected
- 429: honor Retry-After, backoff with jitter, retry
- 5xx: retry with backoff + jitter, max attempts, circuit breaker

**Resilience controls (per dependency):**
- Timeouts (connect/read/total)
- Retry: max attempts, exponential backoff, jitter, retry budget, only transient errors
- Circuit breaker: closed → open → half-open → closed
- Rate limiter: token bucket or sliding window, per-tenant + global
- Bulkhead: separate connection pool per integration
- Idempotency keys for mutating calls

**Observability:** Latency p50/p95/p99, success rate, error rate by code, throttle count, retry count, CB state, correlation ID throughout.

## 2. Handle pagination for a large external dataset

**Patterns:**
- Cursor (opaque token): preferred for large/changing data — stable, no drift.
- Offset/limit: simple but slow and inconsistent on large sets.
- Link headers: follow `rel="next"` — flexible, requires parsing.

**Controls:** Max page size, continuation behavior on data change, resume token on failure, timeout per page.

## 3. OAuth2 flows — when to use which

| Flow | Use case |
|---|---|
| Client credentials | Service-to-service, no user context |
| Authorization code | User-delegated access, interactive |
| Device code | Headless/CLI devices |
| Refresh token | Long-lived access; store encrypted, rotate |

**Always:** Least-privilege scopes, short-lived access tokens, secure refresh token storage, validate JWT claims (iss, aud, exp, scopes).

## 4. Microsoft Graph integration specifics

- Use delta queries for incremental sync (users, mail, files).
- Respect throttling (`Retry-After`, 429); backoff with jitter.
- Handle deleted/moved items via delta.
- Least-privilege scopes (e.g., `Mail.Read`, `Sites.Read.All`).
- Never log message bodies or sensitive content.

## 5. CMDB/ITSM integration design

- Map external fields to normalized internal model; preserve source IDs.
- Support incremental sync (updated_since) and full refresh.
- Detect conflicts (concurrent edits) → flag for resolution.
- Respect service ownership and access controls.
- Idempotent upserts using source ID + hash of content.

## 6. Webhook receiver for async events

**Flow:** Verify signature → check idempotency (event ID) → persist event → ACK fast (2xx) → process async (queue) → retry failed processing with backoff → DLQ for poison events.

**Controls:** Signature verification, replay protection, ordering per resource if needed, observability on queue lag.