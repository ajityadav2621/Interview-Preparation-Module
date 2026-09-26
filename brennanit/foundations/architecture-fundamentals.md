# Architecture Fundamentals

## 1. What Architecture Means

Architecture is the set of decisions that determine how a system is organized, how components communicate, how data moves, and how the system behaves under failure, growth, and change.

For this role, architecture is usually about:

- Turning a specification into a working application
- Integrating with APIs and enterprise systems
- Building reusable AI modules and skills
- Protecting sensitive data
- Making the solution observable and operable
- Supporting safe release and rollback

## 2. Core Architecture Goals

| Goal | Meaning |
|---|---|
| Correctness | The system does what the approved specification says |
| Reliability | The system continues to work when dependencies fail |
| Security | Access, data handling, and audit requirements are enforced |
| Scalability | The system can handle growth in users, requests, and data |
| Maintainability | Other engineers can understand, change, and operate it |
| Reusability | Common capabilities are built once and consumed many times |
| Observability | Engineers can see health, performance, errors, and business outcomes |
| Cost control | Token, compute, storage, and integration costs are measured and bounded |

## 3. Common Architecture Patterns

### Layered application

```text
Client
  |
API / Gateway
  |
Application or business logic
  |
Integration layer
  |
External systems
```

Use when responsibilities are clear and the application is not highly distributed.

### Gateway pattern

An API gateway provides one controlled entry point for clients. It can handle:

- Authentication and authorization
- Routing
- Rate limiting
- Request validation
- TLS termination
- Correlation IDs
- Basic abuse protection

### Event-driven architecture

```text
Producer -> Event bus or queue -> Consumer
```

Use when work is asynchronous, bursty, retry-prone, or needs to be consumed by multiple services.

### Orchestration

A central controller coordinates a sequence of steps.

Use when:

- The workflow has a defined order
- Steps depend on previous results
- Central visibility and control are important
- Human approval points are required

### Choreography

Services react to events independently.

Use when:

- Components should be loosely coupled
- Each service can make local decisions
- There is no need for a central workflow controller

### Sidecar pattern

A helper process runs beside the main application to provide cross-cutting capabilities such as logging, security, networking, or telemetry.

## 4. Designing the Data Flow

A typical AI application flow is:

```text
Request
  -> Authentication and authorization
  -> Input validation
  -> Data classification
  -> Context retrieval, if required
  -> Prompt construction
  -> Model call
  -> Output validation
  -> Human review, if required
  -> Response or action
  -> Audit and telemetry
```

Every stage should have:

- A clear input and output
- A defined owner
- A failure behavior
- A security control
- A measurable signal

## 5. Component Boundaries

Good boundaries separate concerns:

| Component | Responsibility |
|---|---|
| API layer | Receive requests, validate contracts, return responses |
| Application service | Coordinate business logic and workflow |
| Integration client | Communicate with external APIs and connectors |
| AI service | Manage prompts, model calls, and output validation |
| Data layer | Store durable state and configuration |
| Cache | Store short-lived or frequently reused results |
| Event layer | Decouple asynchronous work |
| Observability layer | Emit logs, metrics, traces, and audit events |

Avoid putting external API details, model prompts, and business rules in the same place. This makes the system easier to test and reuse.

## 6. Contracts and Interfaces

A contract defines what one component promises to another.

Important contract types:

- OpenAPI or Swagger for REST APIs
- JSON Schema or equivalent for payloads
- Event schemas for asynchronous messages
- Authentication and authorization rules
- Error formats and status codes
- Versioning and compatibility rules

A contract-first approach reduces assumptions and makes integrations easier to test.

## 7. Scaling Fundamentals

### Scale out

Add more instances behind a load balancer. This is the normal approach for stateless services.

### Scale up

Increase the capacity of an individual instance. This is useful temporarily but has limits.

### Cache

Reduce repeated work by storing safe, short-lived results. Caching must consider:

- Expiry
- Invalidation
- Tenant isolation
- Data classification
- Stale data risk

### Asynchronous processing

Move long-running or retry-prone work to a queue or event bus. This improves responsiveness and isolates failures.

### Partitioning

Split data or traffic by tenant, service line, or another stable key to reduce contention and improve isolation.

## 8. Failure Handling

A production design must answer:

- What happens if an external API is unavailable?
- What happens if the model times out?
- What happens if a request is duplicated?
- What happens if a queue consumer fails?
- What happens if a dependency is slow?
- What fallback is acceptable?
- When should the user see an error?
- When should work be retried?

Common controls:

- Timeouts
- Retries with backoff and jitter
- Circuit breakers
- Rate limiting
- Bulkheads
- Idempotency keys
- Dead-letter queues
- Fallback responses
- Graceful degradation

## 9. Security by Design

Security should be part of the architecture, not added later.

Key decisions:

- Which data classifications are allowed?
- Where may data be processed and stored?
- Which identities and roles can access each function?
- Which tools can the AI application call?
- What data may be logged?
- What requires human approval?
- How are secrets stored and rotated?
- How is prompt injection prevented?
- How are audit events protected?

## 10. Observability by Design

A system should be observable from the beginning.

| Signal | Purpose |
|---|---|
| Logs | Explain what happened for a specific request |
| Metrics | Show trends, rates, latency, errors, and cost |
| Traces | Follow one request across components |
| Audits | Prove who did what, when, and why |
| Business metrics | Show whether the capability created value |

For AI systems, also track:

- Token usage
- Cost per request
- Model and prompt version
- Retrieval quality
- Output quality
- Human escalation rate
- Refusal and safety events

## 11. Deployment and Rollback

A good release design includes:

- Separate development, staging, and production environments
- Automated deployment
- Health checks
- Gradual rollout
- Rollback procedure
- Database migration safety
- Feature flags where appropriate
- Monitoring before, during, and after release

Common strategies:

- **Blue-green**: safest rollback, requires duplicate capacity
- **Canary**: detects problems early with a small traffic share
- **Rolling**: simple and efficient for stateless services

## 12. Trade-offs to Discuss

There is rarely one perfect architecture. Be ready to discuss:

- Synchronous versus asynchronous processing
- Monolith versus modular service
- Centralized orchestration versus event choreography
- Cache speed versus stale data risk
- More model capability versus higher cost and latency
- Automation versus human control
- Reuse versus customization
- Strong consistency versus availability
- Rich telemetry versus privacy risk

## 13. Interview Answers

### Q: How do you start designing a new AI application?

> I start with the specification and clarify the user, inputs, outputs, data classification, latency, availability, and success measures. I then map the end-to-end flow, identify external dependencies, define component boundaries and contracts, and design security, failure handling, observability, and release controls. I prefer the simplest architecture that satisfies the requirements and can evolve safely.

### Q: How do you choose between synchronous and asynchronous processing?

> I use synchronous processing when the user needs an immediate response and the work is short and predictable. I use asynchronous processing for long-running, bursty, retry-prone, or multi-step work. In AI applications, I often validate and acknowledge synchronously, then process retrieval, model calls, and approvals asynchronously with status updates.

### Q: How do you make a system reusable?

> I separate configuration from behavior, define stable contracts, extract shared authentication and validation, version prompts and APIs, and document operating procedures. I also measure usage so that a component can be improved once and benefit multiple consumers.

### Q: How do you prevent cascading failures?

> I use timeouts, bounded connection pools, retries with backoff and jitter, circuit breakers, bulkheads, and rate limits. I also design fallback behavior and idempotent operations, and I monitor dependency health continuously.

### Q: What is the most important non-functional requirement for an AI application?

> It depends on the use case, but security and controllability are always foundational. An AI application must not leak data, execute unsafe actions, or produce unreviewed high-risk decisions. After that, reliability, latency, cost, and quality must be balanced against the business outcome.
