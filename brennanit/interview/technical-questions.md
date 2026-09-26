# Technical Questions and Answers

These are focused technical questions that may appear in a screen or design discussion. Answers emphasize concepts, decisions, and trade-offs rather than code snippets.

## 1. What is the difference between authentication and authorization?

Authentication proves who the caller is, for example by verifying a token or certificate. Authorization determines what that identity is allowed to do. A caller can authenticate successfully but still be denied because the identity lacks the required role, scope, tenant, or data-permission.

## 2. What REST methods are idempotent, and why does it matter?

Safe and idempotent: GET (read only), PUT (replace), DELETE (remove). Usually idempotent: PATCH. Not idempotent: POST (create or arbitrary action). Idempotency matters because retries and duplicate deliveries must not cause unintended side effects, such as duplicate tickets or payments. An idempotency key makes non-idempotent operations safe to repeat.

## 3. When should you use an idempotency key?

Use an idempotency key whenever a request may be submitted more than once and producing a duplicate side effect is harmful. The most common cases are retries, webhooks, ticket updates, approvals, and payments. The receiving system stores the key and the result, and returns the same response for a repeated key without re-executing the action.

## 4. How do you handle a 429 Too Many Requests response?

Treat 429 as a temporary capacity signal, not a success or a permanent failure. Stop sending traffic for the requested period, honor the Retry-After header, and apply exponential backoff with jitter before retrying. Reduce concurrency, consider queueing, and expose throttling metrics. Never retry a 429 in a tight loop because it worsens the downstream problem.

## 5. How do you decide between OAuth2 client credentials and authorization code?

Client credentials is for service-to-service access with no user context; use least-privilege scopes and short-lived tokens. Authorization code is for user-delegated access and requires careful handling of refresh tokens and consent. Choose client credentials when the integration acts on behalf of the application, and authorization code when it must act on behalf of a signed-in user.

## 6. What is the difference between offset and cursor pagination?

Offset pagination (page and size) is simple but becomes slow and inconsistent on large or changing datasets. Cursor pagination uses an opaque continuation token and is preferred for large or real-time data because it avoids drift and duplicate or skipped rows. Always define a maximum page size, continuation behavior, and resume-on-failure behavior.

## 7. When should you use synchronous versus asynchronous processing?

Use synchronous processing when the user needs an immediate response and the work is short, predictable, and reliable. Use asynchronous processing for long-running, bursty, retry-prone, or multi-step work, or when multiple consumers need the same event. In AI applications, validate and acknowledge synchronously, then process retrieval, model calls, and approvals asynchronously with status updates.

## 8. Explain the circuit breaker pattern.

A circuit breaker stops calls to a failing dependency after repeated failures. It has three states: closed (normal traffic), open (fail fast and protect the application), and half-open (test a small number of calls to see if recovery occurred, then close). Combined with timeouts and bounded pools, it prevents cascading failures.

## 9. What is the difference between horizontal and vertical scaling?

Horizontal scaling adds more instances behind a load balancer; it is the normal approach for stateless services. Vertical scaling increases the capacity of an individual instance; it helps temporarily but hits hardware limits. Prefer horizontal scaling for stateless work and use vertical scaling only as a stopgap for stateful or single-threaded components.

## 10. What are the key differences between a relational database and a document database?

Relational databases store structured rows with a fixed schema, enforce relationships and constraints, and support ACID transactions. Document databases store semi-structured documents, offer flexible schemas, and scale horizontally. Choose a relational database for strongly consistent transactions and complex relationships, and a document database for evolving schemas and horizontal scale.

## 11. What is a database index, and when might it hurt performance?

An index is a data structure that speeds up lookups, often at the cost of write performance because every index must be updated on insert, update, or delete. Indexes hurt when there are too many of them, when they are on low-selectivity columns, or when they are not maintained. Always index by access pattern and measure the write overhead.

## 12. Explain the difference between a cache and a buffer.

A cache stores results of expensive operations so repeated reads are faster; it must handle invalidation and tenant isolation. A buffer decouples a producer from a consumer by holding items temporarily when the consumer cannot keep up. Use a queue as a buffer and a cache for read performance.

## 13. What is the CAP theorem, and how does it guide database choice?

CAP states that in a distributed system you can only guarantee two of consistency, availability, and partition tolerance. Because partitions happen, the real choice is between consistency and availability during a partition. Choose consistency when correctness is critical and availability when the system must stay responsive.

## 14. What is a container image, and how is it different from a container?

An image is a read-only layered template with the application and its dependencies. A container is a running instance of an image with a writable layer on top. Images should be scanned for vulnerabilities and never contain secrets; containers run as a non-root user with the minimum required capabilities.

## 15. What is Infrastructure as Code, and why use it?

Infrastructure as Code defines infrastructure declaratively in version-controlled files that are reviewed and applied automatically. It makes infrastructure reproducible, auditable, and testable, and it detects drift when reality diverges from the declared state. Templates should be idempotent so re-running them is safe.

## 16. How do you make a release safe and reversible?

Use a gradual strategy such as blue-green or canary, gate on automated security and quality checks, and keep a documented, tested rollback ready. Artifacts should be immutable, tagged, and traceable from commit to production. Monitor health and key metrics continuously during and after release.

## 17. How do you prevent prompt injection?

Treat all retrieved and user-supplied content as untrusted data and keep instructions separate from data. Validate model output against a schema before acting on it, allowlist the tools the model can call, scope tool actions to least-privilege identities, and require human approval for high-risk commands. Log every prompt and decision for audit.

## 18. How do you reduce hallucinations in an LLM application?

Improve retrieval quality with better chunking, hybrid retrieval, and reranking. Filter results by permissions and freshness. Instruct the model to use only the supplied context and to cite sources. Validate the output against source material using a faithfulness evaluator, apply a confidence threshold, and route uncertain or high-risk answers to human review. Maintain a test set of known questions and run it before every release to detect regressions.

## 19. What is the difference between RAG and fine-tuning?

RAG grounds a model in your organization's data at inference time through retrieval, so it adapts without changing model weights and can be updated by refreshing the index. Fine-tuning updates model weights on your data, which can improve style and domain adherence but requires retraining and can leak data. RAG is usually cheaper, safer, and faster to iterate for most enterprise use cases.

## 20. When would you choose an agent over a fixed workflow?

Choose an agent when a task has multiple dependent steps, requires choosing between tools, or changes based on retrieved data, and benefits from interactivity or human review. Prefer a deterministic workflow when the process is simple, repeatable, predictable, and easy to test, audit, and operate.

## 21. What does MCP do, and why does it matter?

MCP, the Model Context Protocol, standardizes how AI applications discover and use external tools, resources, and prompts. It separates the AI application from specific integrations, makes connectors reusable, and reduces custom glue code. Even with MCP, standard security controls still apply: authentication, authorization, validation, rate limiting, and audit.

## 22. What is the difference between precision and recall in AI evaluation?

Precision is the fraction of retrieved items that are relevant; recall is the fraction of relevant items that are retrieved. High precision means fewer false positives, high recall means fewer false negatives. The right balance depends on the use case: user-facing answers favor precision, discovery tasks favor recall.

## 23. What log levels should you use, and when should you log at each?

DEBUG for detailed troubleshooting (typically off in production), INFO for normal milestones and request summaries, WARN for unexpected but recoverable conditions, ERROR for failed requests that need attention, and FATAL for system-wide failures. Logs should include correlation IDs and exclude secrets.

## 24. What is the difference between a metric and an alert?

A metric is a measured value over time, such as latency, error rate, or token cost. An alert is a rule that fires when a metric crosses a threshold and triggers a response. Alerts should be based on customer impact rather than component metrics, and every alert should have an owner and a runbook.

## 25. What is the difference between a software bill of materials and a vulnerability scan?

A software bill of materials, or SBOM, is a complete inventory of components and dependencies in a build. A vulnerability scan checks those components against known vulnerability databases. Together they give visibility and protection; the SBOM enables traceability and the scan drives remediation.

## 26. What is the difference between encryption at rest and encryption in transit?

Encryption at rest protects data on disk or in a database, usually with provider-managed or customer-managed keys. Encryption in transit protects data moving between services using TLS. Both are required for sensitive data, and restricted data should be encrypted before external AI processing.

## 27. What is the difference between Django and Flask?

Django is batteries-included: it provides an ORM, admin, authentication, and conventions for larger applications. Flask is lightweight and explicit, leaving choices to the developer for smaller services. Choose Flask for small or learning projects and Django for larger applications where common scaffolding saves repeated work. Either is acceptable as long as you understand how you route requests, validate input, and handle errors.

## 28. How do you design a REST API backed by a relational database?

I define resources around the business nouns, map HTTP methods to actions, and keep input validation at the edge. I use the ORM for most access and add indexes for query patterns, and I rely on transactions when a request touches related records. I document the contract with OpenAPI, return consistent status codes and error formats, validate output before responding, and version intentionally when behavior changes.

## 29. What is a database transaction, and when should you use one?

A transaction groups statements so they commit or roll back together, preserving consistency. Use a transaction whenever a single business operation updates multiple rows or tables, such as creating a ticket and its first audit record. Choose the isolation level carefully: higher isolation prevents anomalies but reduces concurrency. Keep transactions short to avoid locking problems.

## 30. How do you handle schema changes safely?

I make schema changes backward-compatible first, such as adding a nullable column or new table, and deploy that before switching application code. I write idempotent migrations, back up before risky changes, and run them in a transaction or with a rollback plan. Destructive changes happen only after confirming no live dependency remains, and I measure the migration time against the maintenance window.

## 31. What is the difference between authentication and authorization in a web app?

Authentication confirms who the caller is, typically by validating a session or token. Authorization decides what that identity may do, usually by checking roles, scopes, or permissions on the requested resource. A user can authenticate but still be denied because the role lacks the required permission; authorization must therefore be checked on every protected operation.

## 32. How do you structure a Git workflow for safe collaboration?

I keep a protected main branch that reflects production-readiness, develop features on short-lived branches, and merge via pull requests that pass CI and at least one peer review. I keep branches short to reduce conflicts, rebase carefully if needed, and tag releases. I never commit secrets, and I use clear commit messages that explain why a change was made.

## 33. What are the stages of a CI/CD pipeline?

Build the artifact -> run lint and unit tests -> run security and dependency scans -> run integration and contract tests -> package and scan the container image -> deploy to a development environment -> run automated acceptance tests -> promote to staging -> run acceptance and quality tests -> gradually release to production with a rollback plan.

## 34. How do you make a container image safe for production?

I start from a minimal, patched base image pinned to a digest, install only required packages, create a non-root user, expose only needed ports, and run as non-root. I scan for vulnerabilities and secrets, generate an SBOM, never bake secrets into the image, and keep the image and its dependencies version-controlled and immutable.

## 35. How do you run infrastructure as code in Azure?

I define resources declaratively in version-controlled templates or Bicep, apply changes through a pipeline that previews and requires approval, and verify results with automated tests. I use managed identities and least-privilege roles, store state with locking and encryption, detect drift, and plan destroy and recreate as controlled operations.

## 36. Why would you script operational work in Bash on Linux?

Bash lets me automate repetitive tasks such as rotating logs, inspecting services, and moving data between systems. I navigate the filesystem, chain commands with pipes, use variables and loops, and check exit codes so that a failure stops the script. I keep scripts small, reviewed, and idempotent, and I prefer managed services over hand-rolled scripts when one exists.

## 37. How do you build an evaluation harness for AI features?

I start with a dataset of representative and edge-case inputs paired with golden answers or scoring rubrics. The harness runs the candidate and baseline versions, computes metrics such as faithfulness, correctness, retrieval quality, latency, and cost, and compares them with statistical awareness. A release is blocked when quality drops, safety failures rise, or cost exceeds budget.

## 38. How do you practice prompt and model regression testing?

I version both prompts and models and run the evaluation dataset on every change to either. I compare average scores, pass rates, per-category performance, worst-case failures, and cost and latency against the baseline. I treat a regression in quality, safety, or cost as a failed test and require a deliberate decision before proceeding.

## 39. How do you enable other builders rather than rebuilding for every request?

I deliver stable, well-documented components with externalized configuration, clear contracts, standard metrics, and a runbook. I pair on adoption, review their use of the component, and keep the component versioned so improvements benefit everyone. I measure usage to justify further investment and avoid bespoke rewrites.
