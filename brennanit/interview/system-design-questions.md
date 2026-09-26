# System Design Questions and Answers

These questions cover AI applications, integrations, reusable modules, and infrastructure. For each, use the structured design framework: clarify, define the flow, identify components, address non-functional requirements, discuss trade-offs, and close with risks.

## 1. Design an AI Ticket Summariser

**Clarify:** Ticket source systems, data classification, whether output is advisory or executable, latency target, and volume.

**Flow:** Validate ticket ID and authorization -> retrieve ticket via resilient client -> classify data -> retrieve related context if needed -> construct versioned prompt -> call approved model -> validate structured response -> apply human-review threshold -> return or act -> emit audit and telemetry.

**Components:** API gateway, ticket integration client, retrieval service, prompt manager, model endpoint, output validator, human review queue, cache, observability.

**Non-functional:** Timeouts and circuit breakers on external calls, idempotency for re-runs, encryption for restricted data, token and cost controls.

**Trade-offs:** Synchronous retrieval with asynchronous summarization for lower latency; advisory summary versus auto-update with approval.

**Risks:** Wrong summary, hallucinated fields, missing tickets. Mitigate with citations, faithfulness evaluation, and human review for high-impact updates.

## 2. Design a RAG Knowledge Assistant

**Clarify:** Document sources, classification, freshness requirement, permission model, and whether answers are public or personalized.

**Flow:** Ingest documents -> chunk and embed -> index with metadata -> apply permission filters -> retrieve relevant chunks -> rerank -> construct prompt -> generate answer with citations -> validate output -> audit.

**Components:** Ingestion pipeline, chunking service, embedding service, vector/search store, retriever, reranker, prompt manager, model, citation generator, observability.

**Non-functional:** Index freshness and sync cadence, tenant isolation, metadata filtering, token budgeting, latency targets for retrieval and generation.

**Trade-offs:** Hybrid (keyword plus vector) retrieval versus pure vector; real-time versus batch ingestion.

**Risks:** Stale answers, wrong citations, permission leakage. Mitigate with freshness checks, citation validation, and pre-retrieval permission filtering.

## 3. Design the Beacon Module Framework

**Clarify:** What a module is, how modules are packaged and discovered, execution isolation, and permission boundaries.

**Flow:** Module registered with manifest -> capability discovery -> secure load in sandbox or container -> input validation -> prompt/model selection -> output validation -> return to caller -> audit.

**Components:** Module registry, manifest store, execution runtime, identity and permission service, prompt and model catalog, validator, telemetry.

**Non-functional:** Module isolation, signed and versioned modules, least-privilege execution, resource limits, audit per module.

**Trade-offs:** Native execution for speed versus container sandboxes for isolation; shared versus dedicated runtimes.

**Risks:** Malicious or faulty modules, privilege escalation. Mitigate with sandboxing, allowlisted tools, and output validation.

## 4. Design a Cowork Skill

**Clarify:** Skill trigger, inputs, output contract, whether the skill acts or recommends, and data classification.

**Flow:** Trigger received -> input validated -> context retrieved -> prompt constructed -> model called -> output validated -> action or response -> telemetry.

**Components:** Trigger listener, validation layer, retrieval, prompt manager, model, action executor, human approval gate, observability.

**Non-functional:** Idempotency for retried triggers, rate limiting, secrets via managed identity, token budget.

**Trade-offs:** Fully autonomous skill versus require-approval mode; inline execution versus queued.

**Risks:** Duplicate actions, unauthorized execution. Mitigate with idempotency keys, approval gates, and least-privilege tool credentials.

## 5. Design a Microsoft 365 Integration

**Clarify:** Scope permissions, which Graph endpoints, polling versus webhooks, delta sync, and throttling limits.

**Flow:** OAuth2 authorization -> token cache and refresh -> Graph API calls with paging -> delta queries for incremental sync -> handle throttling and 429 -> normalize to internal model -> persist or return -> audit.

**Components:** Auth service, token cache, Graph client, pagination and delta handlers, throttling controller, data transformer, persistence, observability.

**Non-functional:** Least-privilege scopes, short-lived tokens, respect Retry-After, retry with jitter, idempotency for upserts.

**Trade-offs:** Real-time webhooks versus scheduled delta sync; broad versus narrow scopes.

**Risks:** Throttling causing lag, deleted items not reflected, consent drift. Mitigate with backoff, reconciliation jobs, and scope audits.

## 6. Design a CMDB/ITSM Integration

**Clarify:** Item types (CI, incident, problem, change), field mapping, sync direction, incremental sync, and conflict resolution.

**Flow:** Credential resolution -> API call with paging -> normalize and dedupe -> preserve source identifiers -> conflict detection -> upsert -> audit.

**Components:** Credential store, ITSM API client, field mapper, conflict resolver, persistence layer, sync scheduler, observability.

**Non-functional:** Pagination handling, incremental sync via updated timestamps, idempotency, error classification.

**Trade-offs:** Full refresh versus delta sync; real-time webhook versus scheduled batch.

**Risks:** Data drift, duplicate records, mapping errors. Mitigate with source ID preservation, conflict reporting, and reconciliation.

## 7. Design an Evaluation Harness

**Clarify:** Which AI capabilities are tested, success metrics, dataset sources, comparison criteria, and release gates.

**Flow:** Load dataset -> run candidate and baseline -> compute metrics -> compare results -> generate report -> evaluate significance -> gate release.

**Components:** Dataset manager, test runner, metric calculators, comparator, evaluator, reporting, gate policy.

**Non-functional:** Versioned datasets and golden answers, statistical significance checks, cost and latency measurement, regression thresholds.

**Trade-offs:** Accuracy versus cost; automated versus human-reviewed judgments.

**Risks:** Test data staleness, metric gaming. Mitigate with human review of golden answers and regular dataset refresh.

## 8. Design Human-in-the-Loop Review

**Clarify:** What decisions need review, confidence thresholds, reviewer assignment, and turnaround time.

**Flow:** Model output -> confidence score -> if below threshold, route to review -> reviewer approves, edits, or rejects -> result recorded -> feedback loop updates thresholds.

**Components:** Confidence scorer, review queue, reviewer assignment, audit log, feedback store, threshold manager.

**Non-functional:** SLA tracking for reviews, audit trail, least-privilege reviewer access.

**Trade-offs:** More review catches errors but slows delivery; confidence thresholds that are too high reduce automation value.

**Risks:** Review backlog, low reviewer precision. Mitigate with clear guidelines, sampling for training, and threshold calibration.

## 9. Design Data Residency and Sovereignty Controls

**Clarify:** Which regions data may use, customer constraints, and how routing decisions are made.

**Flow:** Request -> data classification -> customer region policy -> route to approved regional endpoint -> process -> enforce no cross-border transfer -> audit.

**Components:** Classification engine, policy store, regional routing, regional model endpoints, audit log, exception handler.

**Non-functional:** Policy stored per customer, fail closed on unknown region, encryption at rest and in transit.

**Trade-offs:** Fewer regional endpoints reduce cost but limit coverage; caching across borders risks leakage.

**Risks:** Data processed outside approved region, policy gaps. Mitigate with fail-closed routing and regular audits.

## 10. Design a Scalable AI Application Platform

**Clarify:** Expected request volume, latency targets, tenant model, and deployment cadence.

**Flow:** Request -> gateway -> authentication -> routing -> stateless processing -> shared caches and queues -> pooled model and storage -> response.

**Components:** API gateway, auth service, load balancer, stateless app instances, cache layer, message queue, shared vector store, model pool, observability.

**Non-functional:** Horizontal scaling, per-tenant isolation or resource quotas, circuit breakers, rate limiting.

**Trade-offs:** Multi-tenant sharing versus dedicated tenancy; caching for speed versus stale data risk.

**Risks:** Noisy neighbors, cache stampede, model concurrency limits. Mitigate with quotas, circuit breakers, and cache warming.

## 11. Design Failure Handling and Degraded Mode

**Clarify:** Which failures matter most, acceptable degraded behavior, and fallback content.

**Flow:** Detect failure -> classify transient versus permanent -> apply circuit breaker -> fall back to cached or static response -> retry background job -> report degraded status.

**Components:** Health checker, circuit breaker, fallback provider, retry queue, alerting, status dashboard.

**Non-functional:** Timeouts, bounded queues, idempotent operations, clear user messaging.

**Trade-offs:** Stale data now versus error; graceful degradation versus fail-fast.

**Risks:** Silent failures, stale fallback content. Mitigate with freshness timestamps and monitoring on degraded responses.

## 12. Design a Deployment and Release Pipeline

**Clarify:** Environments, deploy frequency, rollback expectation, and who can release.

**Flow:** Commit -> CI build and tests -> security scan -> container build and scan -> deploy to dev -> automated tests -> deploy to staging -> acceptance tests -> canary to prod -> full rollout.

**Components:** Source, CI runner, scanner, registry, deployment engine, environments, gate policies, observability.

**Non-functional:** Immutable artifacts, canary metrics gates, one-click rollback, drift detection.

**Trade-offs:** Fast releases versus stability gates; canary size affects blast radius.

**Risks:** Broken release, data migration lock-in. Mitigate with rollback idempotency and canary gating on business metrics.

## 13. Design Observability for an AI Application

**Clarify:** Which signals matter to users, operators, and auditors, and the alert recipients.

**Flow:** Emit correlation ID -> log events -> record metrics -> trace requests -> capture AI metrics -> set alerts -> build dashboards -> audit log.

**Components:** Logger, metrics collector, trace exporter, AI metrics store, dashboard, alert manager, audit store.

**Non-functional:** Correlation IDs across services, PII excluded from telemetry, retention policies.

**Trade-offs:** Rich telemetry versus privacy and cost.

**Risks:** Noisy alerts, missing AI-specific signals. Mitigate with incident-based alerting and quality gates.

## 14. Design a Reusable Integration Connector

**Clarify:** Supported targets, authentication types, pagination, and error mapping needs.

**Flow:** Receive request -> resolve credentials and endpoint -> authenticate -> call API with paging -> normalize response -> validate -> cache -> return -> observability.

**Components:** Config store, auth manager, API client, pagination handler, response normalizer, cache, telemetry.

**Non-functional:** Externalized configuration, retries with backoff, idempotency, standard error format.

**Trade-offs:** Generic interface versus provider-specific features; caching speed versus freshness.

**Risks:** Provider lock-in, stale cache. Mitigate with stable internal contracts and cache TTL policies.

## 15. Design a Feedback Loop to Improve an AI Feature

**Clarify:** Sources of feedback, signals of correctness, and how improvements are validated.

**Flow:** Collect feedback -> label examples -> aggregate signals -> identify failure patterns -> update prompt, data, or model -> evaluate against held-out set -> release if improved.

**Components:** Feedback collector, labeling store, analytics, trainer, evaluator, deployment pipeline.

**Non-functional:** Feedback traceability, human review of labels, statistical significance.

**Trade-offs:** Human labeling cost versus volume of feedback; fast iteration versus rigor.

**Risks:** Feedback bias, regression. Mitigate with golden datasets and A/B evaluation.

## 16. Design a system to identify and automate a manual process

**Clarify:** What the manual task is, who does it, how often, its inputs and outputs, data classification, and whether the outcome is advisory or executable.

**Flow:** Discovery interview -> document current steps and pain points -> classify data -> decide AI versus rules versus combination -> design minimal viable capability -> build against spec -> peer review -> test -> release -> measure effort reduction.

**Components:** Process profiler, data classifier, connector to source system, decision engine, approval gate if executable, telemetry, outcome dashboard.

**Non-functional:** Least privilege for source systems, audit trail, idempotent actions, fail-safe defaults.

**Trade-offs:** Full automation versus human-in-the-loop; one-off script versus reusable component.

**Risks:** Automating the wrong thing, low-quality output. Mitigate by starting small, measuring the baseline effort first, and requiring approval for executable actions.

## 17. Design telemetry and outcome measurement for a shipped capability

**Clarify:** Who consumes the signals, which business outcome matters, and the acceptable reporting cadence.

**Flow:** Define success metrics with stakeholders -> instrument flows with correlation IDs -> emit logs, metrics, traces, and audits -> build dashboards -> set alerts on customer impact -> review weekly and feed back to improvement.

**Components:** Instrumentation library, correlation ID generator, metrics store, logs store, trace backend, dashboard, alert manager, audit store, feedback loop.

**Non-functional:** PII excluded from telemetry, retention policies, sampling strategy, cost awareness.

**Trade-offs:** Rich telemetry versus privacy and cost; automated versus human-reported outcomes.

**Risks:** Vanity metrics, missing customer-impact signals. Mitigate by tying metrics to KPIs and reviewing them regularly.

## 18. Design an enablement and builder-support platform

**Clarify:** Who the builders are, what they need to reuse, and how adoption is tracked.

**Flow:** Publish reusable components with contracts and runbooks -> expose discovery and documentation -> provide worked examples and pairing -> collect feedback -> improve components and re-measure usage.

**Components:** Component registry, documentation site, example repository, support queue, usage analytics, feedback loop, versioning and deprecation policy.

**Non-functional:** Clear ownership and SLAs, least-privilege access, audit of component use, easy feedback mechanism.

**Trade-offs:** Standardized components versus customization; self-service versus supported hand-holding.

**Risks:** Low adoption, components drifting from use. Mitigate with owner-driven evolution and usage metrics.

## 19. Design a prompt and model regression testing system

**Clarify:** Which AI capabilities are covered, the evaluation dataset, success metrics, comparison criteria, and release gates.

**Flow:** Load versioned dataset and golden answers -> run candidate and baseline -> compute metrics -> compare with significance checks -> block release on regression -> store results for audit.

**Components:** Dataset manager, test runner, metric calculators, comparator, evaluator, reporting store, gate policy, notifications.

**Non-functional:** Deterministic test inputs, versioned prompts and models, reproducible environments, audit trail of results.

**Trade-offs:** Coverage versus execution cost; automated metrics versus human labels.

**Risks:** Test data staleness, metric gaming. Mitigate with regular golden-answer review and A/B evaluation.

## 20. Design a Beacon module that acts on user requests

**Clarify:** What the module does, whether it recommends or executes, data classification, latency target, and approval requirements.

**Flow:** Receive request -> validate input and authorization -> classify data -> resolve context -> construct versioned prompt -> call approved model -> validate structured output -> check approval threshold -> execute action or return recommendation -> audit and emit telemetry.

**Components:** API gateway, validation layer, data classifier, retrieval, prompt manager, model endpoint, output validator, approval gate, action executor, audit log, observability.

**Non-functional:** Idempotency for retried requests, rate limiting, least-privilege tool credentials, short token budgets.

**Trade-offs:** Fully autonomous versus require-approval mode; inline execution versus queued background work.

**Risks:** Unauthorized or duplicated actions. Mitigate with idempotency keys, approval gates, and least-privilege credentials.
