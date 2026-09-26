# Testing, Debugging & Observability

## 1. Layered testing strategy for an AI application

| Layer | What it covers | Tools / approach |
|---|---|---|
| Unit | Pure functions, validators, prompt builders, mappers | pytest, mocks |
| Integration | DB, external APIs (contract tests), vector store | testcontainers, Pact, VCR cassettes |
| Contract | API spec compliance (OpenAPI) | schemathesis, dredd |
| Evaluation | RAG quality, hallucination, faithfulness, cost | custom harness, golden dataset |
| E2E / UAT | Full user journey against spec | Playwright, manual scripts |

**Run order:** Unit → Integration → Contract → Evaluation → E2E. Gate each stage in CI.

## 2. Debugging a production issue quickly

1. Start at telemetry: correlation ID → logs + trace for the failing request.
2. Identify the slow/failing span: validation, retrieval, model, output validation, action.
3. Reproduce locally with same inputs and version (feature flag or commit).
4. Narrow: isolate each stage, check inputs/outputs, compare with expected.
5. Fix → verify against failing case → run regression tests → deploy with canary.

**Key:** Correlation IDs everywhere, structured logs, traces across services, PII exclusion.

## 3. How do you test an AI feature's quality without a codebase?

- Maintain evaluation dataset (inputs + golden answers or rubrics).
- Run candidate vs baseline on every prompt/model change.
- Metrics: faithfulness, correctness, retrieval quality, latency, cost, safety.
- Compare distributions, not just means; check worst-case failures.
- Block release on regression; require explicit sign-off for intentional changes.

## 4. Observability signals for an AI application

**Three pillars:**
- Logs: request flow, decisions, errors (with correlation ID, no PII).
- Metrics: request rate, latency p50/p95/p99, error rate, token usage, cost, model/prompt version.
- Traces: end-to-end latency, span per stage (retrieval, generation, validation, action).

**AI-specific:**
- Retrieval quality (precision@k, recall@k, MRR).
- Output quality (faithfulness, citation coverage).
- Human escalation rate, refusal rate, safety events.
- Confidence score distribution.

**Alerts:** On customer impact (error rate, latency, quality drop), not component metrics alone.

## 5. Contract testing for external integrations

- Define provider contract (OpenAPI) and consumer expectations (Pact).
- Run provider verification in CI on every change.
- Consumer runs against mock; provider runs against real spec.
- Prevents breaking changes reaching staging.