# Failure Handling & Degraded Mode

## Clarify
- Which failures are critical vs tolerable?
- Acceptable degraded behavior: stale data, reduced functionality, read-only?
- Fallback content: cached, static, or "service unavailable"?
- User communication: inline error, banner, status page?
- Recovery: auto-retry, manual intervention, data reconciliation?

## Flow
```
Normal Operation
  All dependencies healthy → full functionality

Degraded Detection
  Health checks (active + passive) → Circuit breaker opens
  OR latency > threshold → Degraded mode triggered

Degraded Mode
  Classify failure: transient (retry) vs permanent (fallback)
  For each capability:
    - Retrieval failed → Return cached results (with staleness header)
    - Model failed → Return static fallback or cached last-good
    - Action executor failed → Queue for retry, return 202 Accepted
    - Auth failed → Fail closed (401/403), alert
  → Emit degraded-mode metric + alert
  → Status page updated

Recovery
  Health checks pass → Circuit breaker half-open → test requests
  → Success threshold met → Close → Full functionality restored
  → Reconcile any queued work → Verify consistency
```

## Components
| Component | Responsibility |
|---|---|
| Health Checker | Active (synthetic requests) + Passive (error rates) |
| Circuit Breaker | Per dependency: closed/open/half-open, configurable thresholds |
| Fallback Provider | Cached responses, static content, degraded responses |
| Retry Queue | Persistent, idempotent, backoff, DLQ for poison |
| Degraded Mode Manager | Feature flags per capability, user-facing messaging |
| Alerting | On degraded entry, exit, duration, unrecovered queue |
| Status Page | Public/internal, real-time, capability-level |

## Non-functional
- Timeouts on every external call (connect/read/total).
- Bounded connection pools per dependency.
- Idempotent operations safe to retry.
- Fallback freshness: timestamp on cached data, expose to client.
- Graceful degradation: disable non-critical capabilities first.

## Trade-offs
| Decision | Option A | Option B |
|---|---|---|
| Fail fast vs stale data | Correctness | Availability |
| Auto-recover vs manual | Speed | Safety |
| Per-capability vs global | Granular | Simpler |

## Risks & Mitigations
- Silent degradation → synthetic monitoring on critical paths, alert on fallback rate.
- Stale fallback served too long → TTL on cache, max-age header, alert on age.
- Queue backlog → backpressure, priority lanes, DLQ monitoring.
- Cascade failure → bulkheads, independent failure domains.
- Data inconsistency on recovery → idempotent replay, reconciliation job.