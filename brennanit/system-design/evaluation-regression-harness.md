# Evaluation & Regression Harness

## Clarify
- Which AI capabilities: RAG, summarization, classification, extraction, agents?
- Dataset: size, representativeness, edge cases, golden answers (human-reviewed)?
- Metrics: faithfulness, relevance, correctness, completeness, safety, consistency, cost, latency?
- Comparison: candidate vs baseline (previous prompt/model version)?
- Gates: what blocks release? (quality drop, safety increase, cost exceed)
- Frequency: on every PR, nightly, on-demand?

## Flow
```
Dataset Management
  Curate: representative + edge + ambiguous + failure + high-risk
  Golden answers: human-reviewed, versioned, with rubrics
  Store: versioned (dataset v1, v2...), immutable per version

Test Execution
  Load dataset version + candidate (prompt vX, model vY) + baseline (vX-1, vY-1)
  For each sample: run candidate → collect output, tokens, latency, cost
  → Run baseline (same inputs)
  → Compute metrics per sample + aggregate

Comparison & Gate
  Compare: mean, pass rate, per-category, worst-case, statistical significance
  Gate: if quality drop > threshold OR safety failures > 0 OR cost > budget → FAIL
  Report: HTML/JSON with drill-down, diff highlights, recommendations
  Store results for audit and trend analysis
```

## Components
| Component | Responsibility |
|---|---|
| Dataset Manager | Versioning, golden answers, rubrics, sampling |
| Test Runner | Parallel execution, timeout, retry, resource isolation |
| Metric Calculators | Faithfulness, citation check, correctness (LLM-judge or rule), safety |
| Comparator | Statistical comparison, per-category breakdown |
| Gate Policy | Thresholds, blocking rules, escalation |
| Reporting | Dashboard, PR comment, audit trail |
| Notification | Slack/email on gate failure, trend alerts |

## Non-functional
- Deterministic test inputs (fixed seed, frozen model for baseline).
- Reproducible environment (pinned dependencies, containerized runner).
- Cost control: max tokens per run, sampling for large datasets.
- Privacy: no PII in dataset; synthetic or redacted.
- Audit: every run logged with config, dataset version, results.

## Trade-offs
| Decision | Option A | Option B |
|---|---|---|
| Automated metrics only | Fast, cheap | May miss nuance |
| Human-in-loop for subset | Higher confidence | Slower, expensive |
| Full dataset every PR | High confidence | Slow, costly |
| Nightly full + PR subset | Balance | More complex |

## Risks & Mitigations
- Test data staleness → quarterly refresh, monitoring on production drift.
- Metric gaming → human-reviewed golden set, A/B with production traffic.
- False confidence → worst-case tracking, adversarial examples in dataset.
- Cost explosion → sampling, token budgets, tiered runs (smoke → full).