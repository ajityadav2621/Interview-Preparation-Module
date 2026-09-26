# RAG, Prompting & AI Evaluation

## 1. Walk through a RAG pipeline from ingestion to answer

**Ingestion:** Source docs → chunk (size/overlap tuned for domain) → embed (model version pinned) → upsert to vector index with metadata (source, classification, updated_at, permissions).

**Query time:** User question → rewrite/expand if needed → embed → vector search (+ keyword hybrid) → metadata filter (permissions, freshness, classification) → rerank (cross-encoder) → top-K → construct prompt with citations.

**Generation:** System prompt (role, constraints, citation format) + user prompt + retrieved chunks → model call (temp low, max tokens) → parse structured output.

**Validation:** Faithfulness check (answer supported by chunks?), citation verification, confidence score.

**Failure modes:** No relevant chunks, wrong chunks retrieved, stale data, model ignores context, hallucination, permission leak, missing citations.

## 2. How do you reduce hallucinations?

- Improve retrieval: better chunking, hybrid search, rerank, metadata filters.
- Constrain prompt: "Use ONLY the provided context. If unsure, say you don't know."
- Require citations inline; validate each citation maps to a retrieved chunk.
- Faithfulness evaluator (separate model or rule-based) on every response.
- Confidence threshold → route low-confidence to human review.
- Evaluation dataset with golden answers; block release on regression.

## 3. When would you use an agent vs a fixed workflow?

**Agent when:** Multi-step, tool selection depends on data, conditional branching, interactive, human approval at points.

**Fixed workflow when:** Deterministic steps, repeatable, auditable, testable, low risk, simpler to operate.

**Always:** Explicit tool allowlist, output schema validation, least-privilege credentials, human approval for consequential actions.

## 4. What is MCP and why does it matter?

MCP (Model Context Protocol) standardizes how AI apps discover and use tools, resources, and prompts from external servers.

**Benefits:** Reusable connectors, capability discovery, less custom glue, consistent auth/validation/rate-limit/audit across servers.

**Still need:** AuthN/AuthZ, validation, rate limiting, audit logging — MCP doesn't replace security controls.

## 5. Design an evaluation harness for AI features

**Dataset:** Representative + edge + ambiguous + failure cases + golden answers (human-reviewed) + scoring rubrics.

**Metrics:** Faithfulness, relevance, correctness, completeness, safety, consistency, cost, latency.

**Process:** Version prompts + models. Run candidate and baseline. Compare mean, pass rate, per-category, worst-case. Statistical significance where appropriate.

**Gate:** Block release if quality drops, safety failures rise, or cost exceeds budget.

## 6. Prompt engineering best practices

- Role, task, context, constraints, output format, examples, quality checks.
- Specific, explicit source-of-truth, clear on uncertainty, schema-aligned.
- Version prompts like code; test with representative + edge inputs.
- Separate instructions from data to prevent injection.