# Microsoft 365 / Graph Integration

## Clarify
- Capabilities needed: mail, calendar, files, Teams, users/groups?
- Auth: app-only (client credentials) or delegated (auth code)?
- Sync: real-time (webhooks) or scheduled (delta queries)?
- Data classification of M365 content?
- Throttling limits and retry strategy?
- Tenant isolation (multi-tenant app or single tenant)?

## Flow
```
Auth
  Client credentials → token cache (short-lived, auto-refresh)
  OR Auth code → user consent → token + refresh token (encrypted store)

API Calls
  Resolve token → Graph client (per-tenant if multi-tenant)
  → Request with pagination (cursor/nextLink) + timeout
  → Handle 429: Retry-After + backoff + jitter
  → Delta query for incremental sync (track deltaLink)
  → Normalize to internal model (preserve source IDs)
  → Upsert to local store (idempotent on source ID + hash)

Webhooks (if real-time)
  Subscription → validation → signed notification → verify signature
  → Check idempotency (notification ID) → Persist → ACK fast
  → Process async (queue) → Handle delete/move via delta
```

## Components
| Component | Responsibility |
|---|---|
| Auth Service | Token acquisition, cache, refresh, multi-tenant routing |
| Graph Client | Paging, delta, throttling, retry, CB, normalization |
| Delta Sync Scheduler | Incremental sync, full refresh fallback, monitoring |
| Webhook Manager | Subscription lifecycle, validation, signature verification |
| Data Transformer | Graph schema → internal model, classification tagging |
| Persistence | Idempotent upsert, source ID preservation, conflict detection |
| Observability | Latency, throttle, sync lag, error rates, token usage |

## Non-functional
- Least-privilege scopes (e.g., `Mail.Read`, not `Mail.ReadWrite`).
- Token cache: encrypted, short TTL, auto-refresh before expiry.
- Throttling: respect `Retry-After`, client-side token bucket, circuit breaker.
- Delta queries: store `deltaLink`, handle `410 Gone` (full resync).
- Data sensitivity: never log message bodies, redact in traces.
- Multi-tenant: isolate token caches, route per tenant.

## Trade-offs
| Decision | Option A | Option B |
|---|---|---|
| Webhooks vs delta polling | Near real-time | Simpler, more resilient |
| App-only vs delegated | No user interaction | User context, consent |
| Full normalize vs pass-through | Consistent internal model | Less transformation |

## Risks & Mitigations
- Throttling causes sync lag → client-side rate limit, monitoring, alert on lag.
- Deleted items missed → delta query handles deletes; reconciliation job weekly.
- Consent drift → audit granted scopes, alert on changes.
- Token expiry → proactive refresh, monitoring on auth failures.
- Schema changes → contract tests on Graph version, versioned client.