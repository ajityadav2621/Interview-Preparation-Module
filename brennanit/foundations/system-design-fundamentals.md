# 2. System Design Fundamentals

## 2.1 What System Design Means

System design is the process of turning requirements into an architecture that is:

- Correct for the business problem
- Secure and compliant
- Reliable under failure
- Scalable for expected growth
- Observable in production
- Maintainable by other engineers
- Cost-conscious

For this role, system design usually focuses on AI-assisted applications, integrations, reusable modules, and deployment pipelines rather than internet-scale infrastructure alone.

## 2.2 A Repeatable System Design Method

Use this sequence in an interview or real design review:

```text
1. Clarify requirements
2. Define scope and non-goals
3. Identify users and journeys
4. Estimate scale and constraints
5. Define data and classification
6. Design high-level architecture
7. Define interfaces and contracts
8. Design failure handling
9. Design security and compliance
10. Design observability and operations
11. Discuss trade-offs and alternatives
12. Summarize the recommended design
```

## 2.3 Clarifying Questions

Before drawing an architecture, ask:

### Functional questions

- What problem does the system solve?
- Who uses it and how often?
- What are the inputs and outputs?
- Is the result real-time, batch, or both?
- Does the system only recommend, or does it execute actions?
- What are the acceptance criteria?

### Non-functional questions

- What latency is acceptable?
- What availability target is expected?
- How many users or requests per minute?
- What is the maximum data volume?
- What data classification applies?
- Where may data be processed and stored?
- What audit and reporting requirements exist?
- What happens when an external system is unavailable?

### AI-specific questions

- Is an LLM required, or would deterministic logic be better?
- What context does the model need?
- How will output quality be measured?
- What happens when the model is uncertain?
- Is human approval required?
- How will prompt and model changes be tested?

## 2.4 Common Architecture Patterns

### Layered application

```text
User / calling system
        |
API layer
        |
Business logic layer
        |
Integration layer
        |
External systems
```

Use when the application has clear responsibilities and moderate complexity.

### Event-driven architecture

```text
Producer -> Message queue / event bus -> Consumer(s)
```

Use when work is asynchronous, bursts are expected, or multiple consumers need the same event.

### Orchestration

A central controller decides the sequence of steps.

Use when the workflow is complex, ordered, and requires centralized visibility.

### Choreography

Each service reacts to events independently.

Use when services are loosely coupled and can make local decisions.

### Gateway pattern

```text
Clients -> API gateway -> Internal services
```

The gateway handles authentication, routing, rate limiting, request validation, and cross-cutting concerns.

### Sidecar pattern

A helper process runs beside the main application to provide logging, security, networking, or telemetry.

Use when infrastructure concerns should be separated from application logic.

## 2.5 Reliability Fundamentals

### Timeouts

Every external call needs a timeout. A timeout prevents a slow dependency from consuming all application resources.

### Retries

Retry only transient failures such as network interruptions, timeouts, and selected 5xx responses. Do not blindly retry authentication failures, validation errors, or business-rule failures.

Use:

- Limited attempts
- Exponential backoff
- Jitter
- Retry budgets
- Idempotency keys for operations that can be repeated

### Circuit breakers

A circuit breaker stops calls to a failing dependency after repeated failures. It protects the application from cascading failure.

States:

```text
Closed -> Open -> Half-open -> Closed
```

### Rate limiting

Rate limiting protects downstream systems and controls cost. It can be applied per user, tenant, service, or endpoint.

### Bulkheads

Bulkheads isolate resources so one failing integration cannot exhaust the entire application.

### Idempotency

An idempotent operation can be repeated without producing a different outcome. It is essential for retries, webhooks, payments, ticket updates, and asynchronous workflows.

## 2.6 Data and Storage Design

### Choose storage by access pattern

| Need | Typical choice |
|---|---|
| Structured transactional data | Relational database |
| Documents and large objects | Object storage |
| Short-lived cache | Redis or in-memory cache |
| Event stream | Message broker / event hub |
| Search | Search index |
| Vector retrieval | Vector database or vector index |

### Data lifecycle

Define:

- Retention period
- Archival rules
- Deletion behavior
- Backup and restore
- Legal hold requirements
- Data residency
- Encryption requirements

### Data classification

Every data item should have a classification:

- Public
- Internal
- Confidential
- Restricted

Classification determines storage, access, encryption, logging, retention, and whether external AI processing is allowed.

## 2.7 AI Application Design

An AI application is more than a model call. A production design includes:

```text
Input validation
  -> Data classification
  -> Context retrieval
  -> Prompt construction
  -> Model call
  -> Output validation
  -> Human review if needed
  -> Action or response
  -> Audit and telemetry
```

Important controls:

- Prompt versioning
- Model versioning
- Token budget
- Output schema
- Confidence or uncertainty handling
- Hallucination checks
- Human-in-the-loop thresholds
- Evaluation harness
- Secure data routing

## 2.8 Integration Design

For every external integration, document:

- Authentication method
- API contract
- Rate limits
- Pagination
- Timeout and retry behavior
- Error mapping
- Idempotency
- Data transformation
- Secrets location
- Monitoring signals
- Fallback behavior

## 2.9 Observability Design

Use three pillars:

| Pillar | Question answered |
|---|---|
| Logs | What happened? |
| Metrics | How is the system performing? |
| Traces | Where did time and failures occur? |

For AI applications, also measure:

- Token usage
- Cost per request
- Latency by model and prompt version
- Output quality
- Human escalation rate
- Retrieval quality
- Drift and regression signals

## 2.10 Deployment Design

### Blue-green deployment

Two identical environments exist. Traffic moves from the current version to the new version after validation.

Best for:

- Low-risk rollback
- High availability
- Clear release boundaries

### Canary deployment

A small percentage of traffic receives the new version first.

Best for:

- Detecting problems early
- Gradual confidence building
- Comparing old and new behavior

### Rolling deployment

Instances are replaced gradually.

Best for:

- Simple services
- Stateless workloads
- Avoiding duplicate infrastructure

### Rollback

A rollback plan must answer:

- What triggers rollback?
- Who can trigger it?
- How long does it take?
- What happens to data changes?
- How are users informed?

## 2.11 System Design Interview Checklist

Before finishing, ensure you have covered:

- Requirements and assumptions
- High-level diagram
- Data flow
- API or interface contracts
- Storage choices
- Scaling strategy
- Failure handling
- Security and privacy
- Observability
- Deployment and rollback
- Trade-offs
- Future improvements

## 2.12 Interview Answer: “How Would You Design an AI Ticket Summariser?”

> I would first clarify the ticket sources, data classification, latency target, and whether the output is advisory or executable. The application would validate the ticket ID, retrieve the ticket through a resilient API client, classify the data, retrieve relevant context if needed, construct a versioned prompt, call an approved model endpoint, validate the structured response, and apply a human-review threshold. Results would be cached, audited, and measured for quality, latency, cost, and human escalation. External calls would have timeouts, retries, rate limits, and circuit breakers, and the deployment would support rollback and telemetry.

## 2.13 Interview Answer: “How Do You Choose Between Synchronous and Asynchronous Processing?”

> I choose synchronous processing when the user needs an immediate response and the work is short and predictable. I choose asynchronous processing when work is long-running, bursty, retry-prone, or shared by multiple consumers. For AI workflows, I often use a synchronous lightweight API for validation and status, with an asynchronous queue for retrieval, model calls, and approval workflows.

## 2.14 Interview Answer: “How Do You Prevent Cascading Failures?”

> I combine timeouts, bounded connection pools, retries with backoff and jitter, circuit breakers, bulkheads, and rate limiting. I also provide fallback behavior and make operations idempotent. Most importantly, I monitor dependency health and test failure scenarios before release.
