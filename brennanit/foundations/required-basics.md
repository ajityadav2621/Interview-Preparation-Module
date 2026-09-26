# 7. Required Basics and Study Plan

## 7.1 What You Must Know Before the Interview

This role blends application engineering, AI, integrations, security, and operations. You are expected to speak confidently across these areas, not deeply on every detail, but with enough foundation to reason and learn quickly.

The checklist below covers the minimum baseline. Anything you cannot explain clearly is worth a review pass.

### Programming and Application Fundamentals
- [ ] Write clean, readable code in at least one language (Python preferred for readability)
- [ ] Data structures: arrays, hash maps, sets, stacks, queues, trees, graphs
- [ ] Algorithms: sorting, searching, two pointers, sliding window, DFS/BFS, basic recursion
- [ ] Complexity analysis: time and space, best/average/worst case, amortization
- [ ] Problem-solving framework: clarify, plan, brute force, optimize, verify, reflect

### System Design and Architecture
- [ ] Design a layered application and explain component boundaries
- [ ] Explain the API gateway pattern and its responsibilities
- [ ] Distinguish orchestration from choreography and when each fits
- [ ] Apply timeouts, retries, circuit breakers, bulkheads, and idempotency
- [ ] Choose storage by access pattern and define a data lifecycle

### AI and LLM Fundamentals
- [ ] Explain RAG and contrast it with fine-tuning or plain prompting
- [ ] Describe chunking, embedding, retrieval, reranking, and permission filtering
- [ ] List common hallucination and failure modes and how to mitigate them
- [ ] Explain when to use an agent versus a fixed workflow
- [ ] Describe MCP and why standardized tool interfaces matter

### Integration Fundamentals
- [ ] REST methods, status codes, and idempotency
- [ ] OpenAPI as a contract-first approach
- [ ] OAuth2 flows, JWT validation, and API key management
- [ ] Pagination strategies and their trade-offs
- [ ] Rate limiting, retries, backoff with jitter, and circuit breakers

### DevOps and Delivery
- [ ] CI/CD stages and quality gates
- [ ] Git workflow with protected branches and pull requests
- [ ] Containerization basics and image scanning
- [ ] Infrastructure as code and environment isolation
- [ ] Deployment strategies: blue-green, canary, rolling, and rollback

### Security and Compliance
- [ ] Data classification levels and their meanings
- [ ] Data sovereignty and regional processing requirements
- [ ] Secrets management and least-privilege access
- [ ] Encryption at rest and in transit
- [ ] Prompt injection and output validation
- [ ] Supply chain risk and SBOM

### Observability
- [ ] Three pillars: logs, metrics, traces with correlation IDs
- [ ] AI-specific metrics: tokens, cost, quality, model and prompt version
- [ ] Alerting on customer impact and error budgets
- [ ] Outcome measurement with leading and lagging indicators

### Communication and Collaboration
- [ ] Use the Situation-Action-Result-Lesson structure for behavioral answers
- [ ] Explain technical decisions to non-technical stakeholders
- [ ] Give and receive peer review constructively
- [ ] Document decisions, runbooks, and operating procedures

### Python Web and Cloud Basics
- [ ] Write web applications with Django or Flask and design REST APIs
- [ ] Use a relational database with an ORM, migrations, and transactions
- [ ] Validate requests and return consistent error responses
- [ ] Version control with Git and feature-branch workflows
- [ ] Build and run containerised deployments with Docker
- [ ] Write Infrastructure as Code and deploy to Azure
- [ ] Operate Linux environments and write Bash scripts for automation
- [ ] Build evaluation harnesses and run prompt and model regression tests
- [ ] Collect telemetry, usage, and outcome metrics for shipped capability

## 7.2 Universal Answer Frameworks

### Technical answers
1. Clarify constraints and requirements.
2. Define the end-to-end flow.
3. Identify components and responsibilities.
4. Address non-functional requirements.
5. Discuss trade-offs explicitly.
6. Close with main risks and how to monitor them.

### Behavioral answers
- **Situation** — set the context briefly.
- **Action** — what did you personally do?
- **Result** — the outcome, ideally quantified.
- **Learning** — what would you do differently next time?

## 7.3 Suggested 30-Day Study Plan

| Week | Focus | Goals |
|---|---|---|
| 1 | Role, architecture, and AI fundamentals | Understand the Foundry pipeline, standard architecture flows, RAG, and evaluation |
| 2 | Integration, DevOps, security, and observability | Master REST, OAuth2, CI/CD, secrets, and the three pillars plus AI metrics |
| 3 | Coding and system design practice | Daily coding problems, two full system designs, review trade-offs |
| 4 | Interview simulation and polish | Mock interviews, behavioral stories, final review of weak areas |

Daily rhythm:

- 1 to 2 hours of focused study.
- Practice explaining concepts aloud, not just reading silently.
- End each session by writing one summary note from memory.

## 7.4 Quick Reference Tables

### HTTP status and action

| Code | Action |
|---|---|
| 2xx | Process response |
| 202 | Track async job |
| 401 | Refresh or repair credentials |
| 403 | Stop, check permissions |
| 409 | Resolve conflict |
| 429 | Back off and respect Retry-After |
| 5xx | Retry with limits, then fallback |

### Deployment strategy comparison

| Strategy | Rollback speed | Cost | Risk profile |
|---|---|---|---|
| Blue-green | Fast | Higher (duplicate capacity) | Low |
| Canary | Medium | Same | Detect early, lower blast radius |
| Rolling | Slow (incremental) | Lowest | Medium |

### Data classification impact

| Level | Storage | Processing | Logging | Retention |
|---|---|---|---|---|
| Public | Any | Any | Free | Standard |
| Internal | Controlled | Controlled | Metadata only | Standard |
| Confidential | Approved | Approved providers | Redacted | Defined |
| Restricted | Regional, encrypted | No external AI | Metadata only | Defined, audit |

## 7.5 Interview Answer: “Where Should I Start Studying?”

> I would start with the Foundry pipeline and the standard architecture flows so every concept connects to where it is used in the role. Then I would master RAG and retrieval quality, because data quality is the biggest lever for AI accuracy. Next, I would focus on integration resilience and security, since external APIs are the most common source of failures and data leaks. Finally, I would practice explaining designs and trade-offs aloud rather than memorizing code.

## 7.6 Final Readiness Self-Check

Before the interview, confirm you can explain:

- The Foundry delivery pipeline and your role in each stage.
- Why RAG improves accuracy and how to evaluate it.
- When to use an agent versus a deterministic workflow.
- How to handle a failing external API, including retries and circuit breakers.
- Why idempotency protects against duplicate requests.
- How to store and rotate a secret safely.
- The difference between monitoring and observability.
- One system design, start to finish, using the answer framework.

If you can explain these clearly without reading, you are ready to begin deeper preparation.
