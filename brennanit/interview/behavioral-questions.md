# Behavioral and Scenario Questions

Behavioral questions assess how you collaborate, solve problems, and handle ambiguity. Use the Situation - Action - Result - Learning structure. Keep answers concrete with specific examples and, where possible, a quantified outcome.

## 1. Tell me about a time you worked from an ambiguous specification.

Situation: I received a request to "summarise customer tickets" with no agreed fields, sources, or success measure.

Action: I separated facts from assumptions and listed the gaps: ticket sources, data classification, output fields, latency, success criteria, and human-review thresholds. I proposed two options with trade-offs, documented a recommended assumption, and obtained approval before building.

Result: We avoided rework and delivered a summariser that matched the agreed criteria on the first review.

Learning: Clarifying and documenting assumptions early is faster than correcting a wrong direction later.

## 2. How do you use AI coding assistants safely?

Situation: I wanted to use an AI assistant to speed up boilerplate and tests without compromising quality or data safety.

Action: I treated AI output as a draft, never pasted restricted data or secrets, and verified generated code against the specification. I reviewed dependencies, ran tests, and put every AI-generated change through the same peer review and security gates as human-written code.

Result: We accelerated routine work while keeping review and governance intact.

Learning: Responsible use means staying in the loop and applying the same standards.

## 3. Tell me about a time you gave a peer helpful review feedback.

Situation: A teammate submitted a change to an integration client that handled retries but ignored idempotency.

Action: I pointed out the duplication risk under retries and suggested adding an idempotency key plus a unit test using a repeated request. I also noted the missing throttle metrics.

Result: They updated the code and added the test, and the metrics later helped us catch an upstream rate-limit issue.

Learning: Frame feedback around the user outcome and a concrete, testable improvement.

## 4. Describe a conflict you had with a teammate and how you resolved it.

Situation: A designer and I disagreed on whether a feature should be real-time or batched because the spec was unclear on latency.

Action: I brought the business impact into the discussion, proposing we build the real-time path only for the user-facing confirmation and queue the heavy work. I documented the decision and shared it with the product owner for confirmation.

Result: We delivered a responsive experience without overbuilding, and the owner appreciated the trade-off clarity.

Learning: Conflicts are best resolved around impact and agreed decision records, not opinions.

## 5. Tell me about a production incident you handled.

Situation: An integration started returning 429 responses, causing a backlog and rising error rates for users.

Action: I paused traffic via the circuit breaker to protect downstream services, respected Retry-After, and reduced concurrency. I traced the issue to a change in the upstream's rate limit and coordinated with their team while we queued work for replay.

Result: Error rates returned to normal within minutes, and no data was lost.

Learning: Circuit breakers and idempotent retries are only effective when paired with clear ownership and fast communication.

## 6. Give an example of when you identified a manual process that could be automated.

Situation: Weekly status reports were being typed manually by two analysts, taking about four hours combined.

Action: I clarified the inputs, stakeholders, and format with the team, then proposed an automated report pulling from the same sources with a quality check. I built and validated the smallest useful version first.

Result: The report now runs automatically and frees about four hours per week for analysis.

Learning: Validate the smallest version with stakeholders before scaling the solution.

## 7. Tell me about a time you had to explain a technical decision to a non-technical audience.

Situation: Leadership was deciding whether to adopt a new model provider that lowered cost but raised latency.

Action: I framed the decision around three measurable outcomes: customer experience, risk, and cost, and I showed a small comparison table rather than technical detail.

Result: They chose the provider that best balanced user experience and cost for the use case.

Learning: Translate technical trade-offs into business outcomes and keep the comparison simple.

## 8. Describe how you approach documentation for a feature you built.

Situation: I delivered a new reusable connector and needed others to adopt it safely.

Action: I documented the setup, configuration, operating limits, and known failure modes in a runbook, and I added the key metrics and alerts to the shared dashboard.

Result: Two other teams adopted the connector within a month with minimal support requests.

Learning: Living documentation with runbooks and dashboards is more useful than a one-time write-up.

## 9. Tell me about a time you made a component reusable across teams.

Situation: Several teams needed access to the same CMDB data, each building their own integration.

Action: I extracted a shared connector with externalized configuration, a stable internal contract, and standard metrics, then worked with the first team to validate it.

Result: Three teams now use the connector, and maintenance effort dropped accordingly.

Learning: Reuse succeeds when ownership, contracts, and observability are designed in from the start.

## 10. Give an example of learning a new technology quickly for a project.

Situation: A project required evaluating a graph-based retriever, but I had not used the platform before.

Action: I read the official concepts, built a small prototype scoped to the exact use case, and compared results against our existing baseline using the same evaluation harness.

Result: We made an evidence-based decision on whether to adopt it, and I could explain the trade-offs to the team.

Learning: Scope learning to the immediate problem and validate against a baseline.

## 11. Tell me about a time you had to prioritize work under competing deadlines.

Situation: I had a security review, a release preparation, and a support issue all due the same week.

Action: I assessed impact and dependencies, escalated the support issue's priority, moved the review forward by preparing evidence in advance, and protected release time by deferring a non-critical task.

Result: All three were completed without a late release, and the support issue turned out to be lower severity than first thought.

Learning: Clarity on impact and early escalation prevents last-minute surprises.

## 12. Describe a time you enabled another team to succeed.

Situation: A downstream team was blocked because our API's error format was inconsistent and poorly documented.

Action: I published a clear error contract with status codes and examples, added validation tests, and walked the team through it in a short session.

Result: Their integration time dropped, and we received no more ambiguity-related bugs.

Learning: Clear contracts and a brief handoff accelerate other teams more than lengthy documentation alone.

## 13. Tell me about a time your first solution did not work and how you recovered.

Situation: My initial chunking strategy for a RAG index produced low recall on long documents.

Action: I analyzed failure cases, tried a smaller chunk size with hybrid retrieval, and validated against the held-out test set.

Result: Recall improved and latency stayed within budget after a small reranking addition.

Learning: Iterate against a test set, and measure before optimizing.

## 14. How do you handle feedback that challenges your design?

Situation: A security review challenged my choice to call an external API from a user-facing service.

Action: I gathered the exact concern, asked about the accepted alternatives, and proposed a scoped backend-to-backend call with least-privilege credentials and audit logging.

Result: The design was approved with the additional controls, and the reviewer became a collaborator.

Learning: Treat challenging feedback as a chance to strengthen the design, not defend it.

## 15. Tell me about a time you measured the outcome of something you built.

Situation: After launching an AI summariser, I needed to confirm it delivered value.

Action: I tracked resolution time and human-edited percentage as leading indicators, plus user satisfaction and cost per request as lagging indicators, and reviewed them weekly.

Result: Resolution time dropped 15 percent and human edits dropped, confirming the feature's value.

Learning: Measuring both leading and lagging indicators confirms whether a capability actually helps.

## 16. Tell me about a continuous improvement you led.

Situation: Our integration client had flaky retries and no shared metrics, and several teams were rebuilding similar logic.

Action: I standardized timeouts, backoff with jitter, and circuit breakers in a reusable connector, added throttle and latency metrics, and documented the pattern for other teams.

Result: Retry failures dropped, and three teams adopted the connector instead of writing their own clients.

Learning: Reusable improvements compound faster than one-off fixes.

## 17. Describe enabling another team to use a component you built.

Situation: Another service line wanted to automate ticket updates, but had no prior experience with our integrations.

Action: I paired with a representative, walked through the reusable connector, and created a worked example scoped to their use case. I also added a short runbook and a feedback channel for questions.

Result: They adopted the connector for two workflows within a week and asked for no bespoke code.

Learning: Pair and example beats documentation when enabling new builders.

## 18. Tell me about a defect you owned from detection to resolution.

Situation: A Beacon module was returning inconsistent results for some requests, and users reported delays.

Action: I traced the issue using the correlation ID and found that a slow upstream API was being retried without a circuit breaker, exhausting connection pools. I added a timeout, bounded pool, and fallback, then verified with a load test.

Result: Latency returned to normal and the failure mode was removed.

Learning: Resilience controls and observability must be designed together, not added after failure.

## 19. Give an example of documenting a decision others could follow.

Situation: We chose blue-green deployment over canary for a release, but others later questioned the choice.

Action: I recorded the decision in a short internal note covering context, options considered, the decision, and consequences, and linked it from the runbook.

Result: A later team referenced it when choosing a strategy for a similar service, saving debate time.

Learning: Lightweight decision records prevent the same discussion from recurring.

## 20. Tell me about raising a gap you found in a specification.

Situation: A spec for a CMDB sync did not define how deleted items would be handled or how conflicts would resolve.

Action: I flagged these as open questions, proposed options with trade-offs, and documented the recommended assumption with data-classification and audit implications.

Result: We agreed on safe defaults before implementation, avoiding rework and a potential data-quality bug.

Learning: Treating a security or correctness gap as a blocker rather than an assumption pays off downstream.
