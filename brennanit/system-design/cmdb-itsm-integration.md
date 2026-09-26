# CMDB / ITSM Integration

## Clarify
- Item types: CIs, incidents, problems, changes, service requests?
- Sync direction: one-way (pull) or bidirectional?
- Field mapping: external → internal normalized model?
- Incremental sync: `updated_since` or webhook?
- Conflict resolution: last-write-wins, manual, source-of-truth?
- Volume: total items, daily changes, page sizes?
- Permissions: who can read/write which items?

## Flow
```
Credential Resolution
  → API client (auth, base URL, version) per environment

Sync (scheduled or webhook-triggered)
  Fetch page → Normalize fields → Preserve source IDs
  → Detect conflicts (hash comparison) → Resolve per policy
  → Idempotent upsert (source_id + content_hash)
  → Update sync cursor / delta token
  → Emit metrics: items processed, conflicts, errors, lag

Webhook (if supported)
  Verify signature → Check idempotency (event ID) → Persist event
  → ACK fast → Process async → Normalize → Upsert → Metrics
```

## Components
| Component | Responsibility |
|---|---|
| Credential Store | Per-environment, rotated, least-privilege |
| API Client | Paging, auth, timeout, retry, CB, rate limit |
| Field Mapper | External → internal, configurable, versioned |
| Conflict Resolver | Detect, log, resolve per policy (auto/manual) |
| Sync Scheduler | Incremental (cursor), full refresh, monitoring |
| Webhook Handler | Signature verify, dedupe, queue, process |
| Persistence | Idempotent upsert, audit trail, source ID index |
| Observability | Sync lag, conflict rate, error rate, volume |

## Non-functional
- Pagination: cursor preferred; handle max page size, resume token.
- Idempotency: unique index on `(source_system, source_id, content_hash)`.
- Rate limits: token bucket per integration, respect server `Retry-After`.
- Schema changes: versioned mapper, backward-compatible, contract tests.
- Data classification: tag items, enforce access controls on internal model.

## Trade-offs
| Decision | Option A | Option B |
|---|---|---|
| Pull vs push | Simpler, controlled rate | Real-time, more complex |
| Full refresh nightly | Catches drift | Heavy, longer window |
| Auto-resolve conflicts | Faster | Risk of data loss |

## Risks & Mitigations
- Data drift → nightly reconciliation job + drift alerts.
- Duplicate records → idempotent upsert on source ID + hash.
- Mapping errors → versioned mapper, unit tests, contract tests.
- Throttling → client-side rate limit, circuit breaker, alert on 429 rate.
- Permission changes → sync ACLs, audit access mismatches.