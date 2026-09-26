# Reusable Integration Connector

## Clarify
- Target systems: REST, GraphQL, SOAP, database, file?
- Auth patterns: API key, OAuth2 (client creds, auth code), JWT, mTLS?
- Pagination: cursor, offset, link headers?
- Rate limits: per-target, per-tenant, dynamic?
- Error mapping: external codes → internal standard?
- Idempotency: supported by target? key format?
- Config: endpoint, timeouts, scopes, retry policy, rate limit — externalized?

## Flow
```
Request Received (internal model)
  → Resolve config (endpoint, auth, timeouts, rate limit, version)
  → Authenticate (token cache, refresh, mTLS cert)
  → Map internal → external request (versioned)
  → Call with pagination (auto-follow pages, resume token)
  → Handle responses:
      2xx → Map external → internal (versioned, validated)
      401 → Refresh token, retry once
      429 → Retry-After + backoff + jitter
      5xx → Retry with CB, then fallback
  → Cache if safe (GET, TTL, tenant key)
  → Return internal model + metadata (source, fetched_at, staleness)
  → Emit metrics: latency, success, throttle, retry, CB state
```

## Components
| Component | Responsibility |
|---|---|
| Config Store | Externalized, versioned, per-environment, validated |
| Auth Manager | Token cache, refresh, mTLS, least-privilege scopes |
| HTTP Client | Timeouts, pooling, retry, CB, rate limiter, idempotency |
| Pagination Handler | Cursor/offset/link, resume token, max pages |
| Response Mapper | External → internal schema, versioned, validation |
| Cache | TTL, tenant isolation, invalidation on write |
| Error Mapper | External codes → internal error taxonomy |
| Observability | Standard metrics, logs, traces per call |

## Non-functional
- Externalized config: no code change for endpoint/timeout/scope updates.
- Retry: only transient, max attempts, budget, idempotency key.
- Circuit breaker: per target, independent, metrics exposed.
- Rate limit: token bucket, respect server limits, per-tenant + global.
- Versioning: connector version + target API version in config.
- Security: secrets in Key Vault, never logged, scanned.

## Trade-offs
| Decision | Option A | Option B |
|---|---|---|
| Generic interface | Reusable across targets | Least common denominator |
| Target-specific features | Full power | More maintenance |
| Cache aggressive | Fast | Staleness risk |
| Cache conservative | Fresh | Higher latency |

## Risks & Mitigations
- Target API change → contract tests, versioned mapper, deprecation policy.
- Throttling storms → client-side rate limit, queue, alert on 429 rate.
- Stale cache → TTL, invalidation webhook, versioned cache keys.
- Secret rotation → Key Vault integration, zero-downtime refresh.
- Connector bloat → split by target, shared core library.