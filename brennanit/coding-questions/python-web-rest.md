# Python Web & REST APIs

## 1. Design a REST endpoint to create a ticket in an ITSM system

**Clarify:** Required fields, validation rules, authentication, idempotency, response format, rate limits from downstream.

**Approach:**
- Validate input schema at the edge (required fields, types, enums).
- Check authentication and authorization (scope/role).
- Generate or require an idempotency key; store with request hash to deduplicate retries.
- Call downstream ITSM API with timeout, retry (backoff + jitter) on transient errors, circuit breaker on repeated failures.
- Normalize downstream response to internal contract; return 201 with location or 202 for async.
- Emit structured logs, metrics (latency, success, throttle), and audit event.

**Trade-offs:** Synchronous vs async acknowledgment; strict validation vs permissive with downstream errors.

**Edge cases:** Duplicate idempotency key, downstream 429/5xx, missing required field, auth token expiry.

## 2. Compare Flask and Django for a new AI-backed microservice

| Aspect | Flask | Django |
|---|---|---|
| Boilerplate | Minimal, explicit | Batteries-included (admin, auth, ORM) |
| Learning curve | Lower initial | Higher initial, faster later |
| REST APIs | Manual routing, extensions | DRF or built-in views |
| ORM | SQLAlchemy or other | Django ORM |
| Admin/CRUD | Build yourself | Built-in |
| Migrations | Alembic (separate) | Built-in `makemigrations`/`migrate` |
| When to choose | Small services, explicit control | Larger apps, shared conventions |

**Answer:** Choose Flask for small, focused services where you want explicit control. Choose Django when you need shared scaffolding (auth, admin, migrations) across multiple services and team familiarity exists.

## 3. How do you version a REST API?

- URL versioning (`/api/v1/tickets`) — explicit, cacheable, easy to route.
- Header versioning (`Accept: application/vnd.company.v1+json`) — clean URLs, harder to test.
- Media-type versioning — most RESTful, most complex.
- Prefer URL versioning for external contracts; header for internal.
- Never break existing versions; add new version, run both, deprecate with notice.

## 4. Implement request validation and consistent error responses

**Validation:** Use a schema library (Pydantic, Marshmallow) at the edge. Reject early with 400, structured error body:
```json
{ "error": "VALIDATION_ERROR", "details": [{ "field": "email", "code": "INVALID_FORMAT" }] }
```

**Error mapping:** Map downstream 401→401, 403→403, 429→429 (with Retry-After), 5xx→502/503 with correlation ID. Never leak stack traces.

**Trade-offs:** Fail-fast vs collect-all-errors; strict schemas vs flexible parsing.

## 5. How do you secure a Python web API?

- Auth: OAuth2/JWT validation (verify sig, iss, aud, exp, scopes) at middleware layer.
- Authorization: Role/scope checks per endpoint or resource.
- Input: Validate, sanitize, limit payload size.
- Output: Validate model output against schema before acting.
- Secrets: Never in code; fetch from Key Vault at runtime via managed identity.
- TLS: Enforce HTTPS, modern ciphers, HSTS.
- Rate limit: Per-identity and per-IP.
- Audit: Log auth decisions and sensitive operations.