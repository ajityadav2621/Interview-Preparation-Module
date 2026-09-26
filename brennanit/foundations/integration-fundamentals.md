# 4. Integration Fundamentals

## 4.1 What an Integration Does

An integration connects the AI application to an external system such as Microsoft 365, ServiceNow, a CMDB, an ITSM platform, a file store, or an internal data service.

A good integration is not just an HTTP call. It must handle:

- Authentication and authorization
- API contracts and versioning
- Pagination and large result sets
- Rate limits and throttling
- Timeouts and retries
- Idempotency
- Data transformation and validation
- Error handling and fallback behavior
- Secrets management
- Monitoring and auditability

## 4.2 Standard Integration Flow

```text
Application request
        |
        v
Validate input and authorization
        |
        v
Resolve credentials and endpoint
        |
        v
Call external API with timeout
        |
        v
Handle success, retryable failure, or permanent failure
        |
        v
Transform and validate response
        |
        v
Cache or persist result if appropriate
        |
        v
Return result to application workflow
        |
        v
Emit metrics, logs, traces, and audit events
```

## 4.3 REST API Fundamentals

### Resources and methods

| Method | Meaning | Typical idempotency |
|---|---|---|
| GET | Read a resource | Yes |
| POST | Create or invoke an action | Usually no |
| PUT | Replace a resource | Yes |
| PATCH | Partially update a resource | Usually yes, but confirm |
| DELETE | Remove a resource | Usually yes |

### Important HTTP status codes

| Status | Meaning | Engineer response |
|---|---|---|
| 200 | Successful read or action | Process response |
| 201 | Resource created | Store or return identifier |
| 202 | Accepted for asynchronous work | Track job/status |
| 204 | Successful with no body | Continue workflow |
| 400 | Invalid request | Fix request; do not retry unchanged |
| 401 | Authentication failed | Refresh or repair credentials |
| 403 | Authorized identity lacks permission | Stop and alert if unexpected |
| 404 | Resource not found | Decide whether this is expected |
| 409 | Conflict | Resolve based on business rule |
| 429 | Rate limited | Respect Retry-After and back off |
| 500/502/503/504 | Temporary service failure | Retry with limits, then fallback |

## 4.4 OpenAPI and Contract-First Design

OpenAPI/Swagger describes:

- Endpoints and methods
- Request and response schemas
- Authentication requirements
- Error formats
- Versioning
- Examples

Contract-first design means the API contract is agreed before implementation. This reduces ambiguity and allows independent teams to build and test against the same expectations.

A good contract defines:

- Required and optional fields
- Data types and allowed values
- Pagination behavior
- Error structure
- Deprecation and version policy
- Rate limits and SLA expectations

## 4.5 Authentication Patterns

| Pattern | Best for | Key considerations |
|---|---|---|
| API key | Internal or low-risk service access | Store in secret manager; rotate regularly |
| OAuth2 client credentials | Service-to-service access | Use least-privilege scopes and short-lived tokens |
| OAuth2 authorization code | User-delegated access | Handle refresh tokens and consent carefully |
| JWT validation | Microservice authentication | Verify signature, issuer, audience, expiry, and claims |
| Mutual TLS | High-trust environments | Manage certificates and rotation |

## 4.6 Pagination

Large APIs rarely return everything in one response.

| Pattern | Description | Watch-outs |
|---|---|---|
| Offset/limit | Skip N records and return M | Can become slow or inconsistent as data changes |
| Cursor | Use an opaque continuation token | Preferred for large or changing datasets |
| Page number | Request a numbered page | Simple but can shift when data changes |
| Link headers | Follow a server-provided next link | Flexible but requires link parsing |

Always define:

- Maximum page size
- Continuation behavior
- What happens when data changes during pagination
- How failures are resumed

## 4.7 Rate Limiting and Throttling

Rate limits protect external systems and control cost.

Common controls:

- Token bucket or sliding-window limiter
- Per-tenant and per-user limits
- Respect for `Retry-After`
- Backoff with jitter
- Queueing or graceful degradation when limits are reached
- Metrics for throttled requests

A good integration does not treat a 429 as a normal success or a permanent failure. It pauses, retries safely, and reports the pressure.

## 4.8 Retry, Timeout, and Circuit Breaker

### Timeout

A timeout prevents an application from waiting forever. Every external call should have:

- Connection timeout
- Read timeout
- Total request timeout

### Retry

Retry only transient failures:

- Network interruption
- Timeout
- 429 rate limit
- Selected 5xx responses

Do not automatically retry:

- Authentication failures
- Validation errors
- Business-rule failures
- Non-idempotent actions without an idempotency key

Use:

- Maximum attempts
- Exponential backoff
- Jitter
- Retry budget
- Idempotency key where applicable

### Circuit breaker

A circuit breaker stops calls to a failing dependency after repeated failures.

```text
Closed -> Open -> Half-open -> Closed
```

- Closed: normal traffic
- Open: fail fast and protect the application
- Half-open: test a small number of calls to see whether recovery has occurred

## 4.9 Idempotency

An operation is idempotent when repeating it produces the same outcome as doing it once.

Idempotency is essential for:

- Retried requests
- Webhooks
- Ticket updates
- Payments or approvals
- Asynchronous job execution

Use an idempotency key when a repeated request must not create duplicate records or duplicate side effects.

## 4.10 Webhooks and Asynchronous Events

Webhooks let an external system notify the application when something changes.

A reliable webhook flow includes:

```text
External system -> Signed webhook -> Validate signature
        -> Check idempotency -> Persist event
        -> Acknowledge quickly -> Process asynchronously
```

Important controls:

- Verify signatures
- Reject replayed events
- Store event IDs
- Acknowledge quickly
- Process asynchronously
- Retry failed processing
- Use a dead-letter queue for poison events

## 4.11 Microsoft 365 Integration Fundamentals

Microsoft 365 integrations commonly use Microsoft Graph.

Typical capabilities:

- Mail, calendar, and contacts
- Files and SharePoint
- Teams messages and channels
- User and group lookup
- Delta queries for incremental synchronization
- Webhook subscriptions for change notifications

Key design points:

- Use OAuth2 and least-privilege scopes
- Store tokens and secrets securely
- Respect Graph throttling
- Use delta queries where possible
- Handle deleted and moved items
- Avoid logging message bodies or sensitive content unnecessarily

## 4.12 CMDB and ITSM Integration Fundamentals

CMDB and ITSM integrations usually involve:

- Configuration items
- Incidents
- Problems
- Changes
- Service requests
- Relationships and dependencies

Important design points:

- Map external fields to a normalized internal model
- Preserve source identifiers
- Handle different status and priority values
- Support pagination and incremental synchronization
- Detect and report data conflicts
- Respect service ownership and access controls

## 4.13 MCP Connector Fundamentals

MCP (Model Context Protocol) standardizes how AI applications discover and use external tools, resources, and prompts.

Core concepts:

| Concept | Meaning |
|---|---|
| MCP client | The AI application requesting capabilities |
| MCP server | The system exposing tools or data |
| Tool | An action the model can request |
| Resource | Data the model can read |
| Prompt | A reusable prompt template |
| Transport | Communication method, such as stdio or HTTP |

MCP is useful when connectors need to be reusable across multiple AI applications. It still requires authentication, authorization, validation, rate limiting, and audit logging.

## 4.14 Integration Observability

Every integration should expose:

- Request count and success rate
- Latency percentiles
- Error rate by status code
- Throttle count
- Retry count
- Circuit-breaker state
- Data volume transferred
- Cost or quota consumption
- Trace ID for correlation

## 4.15 Interview Questions and Answers

### Q: How would you integrate with an external ITSM API that rate-limits requests?

> I would first read the API contract and identify authentication, pagination, rate limits, and idempotency requirements. I would implement a token-bucket or sliding-window limiter, respect `Retry-After`, and use bounded retries with exponential backoff and jitter for transient failures. Long-running work would be asynchronous, and repeated requests would carry an idempotency key. I would expose throttle, retry, latency, and error metrics and provide a fallback such as cached data or a clear degraded response.

### Q: What is the difference between authentication and authorization?

> Authentication proves who the caller is. Authorization determines what that caller is allowed to do. A system can authenticate a user or service successfully but still deny access because the identity lacks the required role, scope, tenant, or data classification permission.

### Q: How do you handle a 429 response?

> I treat 429 as a temporary capacity signal. I stop sending traffic for the requested period, respect `Retry-After`, and use backoff with jitter before retrying. I also measure throttling and may reduce concurrency or route work to a queue. I do not retry a 429 in a tight loop because that worsens the downstream problem.

### Q: How do you design a reusable connector?

> I separate the connector interface from the provider-specific implementation. The connector owns authentication, request construction, pagination, retries, and response normalization. The application receives a stable internal model rather than provider-specific fields. Configuration such as endpoint, scopes, timeouts, and rate limits is externalized, and the connector emits standard metrics and logs.

### Q: Why is idempotency important?

> Retries and duplicate webhook deliveries are normal in distributed systems. Without idempotency, the same request can create duplicate tickets, duplicate approvals, or repeated side effects. An idempotency key lets the receiving system recognize a repeated operation and return the original result safely.

### Q: How do you handle API version changes?

> I prefer explicit API versioning through the URL, header, or contract version. I document compatibility rules, test old and new versions during migration, and avoid depending on undocumented fields. If a breaking change is required, I introduce a new version, run both versions temporarily if needed, and retire the old version only after consumers have migrated.

### Q: What would you do if an external API is slow?

> I would set a strict timeout, use a bounded connection pool, and avoid blocking all application threads. If the dependency is repeatedly slow, a circuit breaker would open and the application would use a fallback or return a degraded response. I would trace the call, alert on latency, and work with the owning team to identify the bottleneck.

### Q: How do you protect secrets used by integrations?

> Secrets are stored in a managed secret store such as Azure Key Vault, not in source code or configuration files. Applications retrieve them at runtime using a managed identity or short-lived credential. Access is least-privilege, secrets are rotated, and secret usage is audited. Logs and traces never contain secret values.
