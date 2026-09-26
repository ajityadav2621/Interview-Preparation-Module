# Prompt & Model Regression Testing

## Clarify
- Scope: which prompts, which models, which capabilities?
- Dataset: versioned, golden answers, rubrics, adversarial?
- Metrics: quality (faithfulness, correctness), safety, cost, latency?
- Comparison: candidate vs baseline (previous prompt/model version)?
- Gate: what blocks release?
- Frequency: every PR, nightly, on model vendor update?

## Flow
```
Version Control
  Prompts: versioned in Git (prompt_v1.yaml, prompt_v2.yaml)
  Models: pinned vendor version (gpt-4o-2024-08-06) + regional endpoint
  Config: prompt_version + model_version per capability

Test Execution (on every change to prompt or model config)
  Load dataset version + candidate (prompt vX, model vY) + baseline (vX-1, vY-1)
  For each sample:
    Run candidate → collect output, tokens, latency, cost
    Run baseline (same inputs, same runtime)
  Compute metrics per sample + aggregate

Comparison
  Quality: mean, pass rate, per-category, worst-case, statistical significance
  Safety: refusal rate, harmful content rate, PII leakage
  Cost: mean tokens, p95 tokens, estimated cost
  Latency: p50, p95, p99
  Drift: embedding distance (candidate output vs baseline output)

Gate Policy
  BLOCK if: quality drop > threshold OR safety failures > 0 OR cost > budget
  WARN if: latency regression, quality flat but cost up
  PASS: quality ↑ or ↔, safety ↔, cost ↔ or ↓

Report & Audit
  PR comment with summary + link to full report
  Store: config, dataset version, results, gate decision, approver
  Trend dashboard: quality/cost/latency over versions
```

## Components
| Component | Responsibility |
|---|---|
| Prompt/Model Config | Versioned, per-capability, Git-tracked |
| Dataset Manager | Versioned, golden answers, rubrics, sampling |
| Test Runner | Parallel, deterministic, resource-isolated, timeout |
| Metric Calculators | Quality (LLM-judge + rules), safety, cost, latency |
| Comparator | Statistical, per-category, worst-case, drift |
| Gate Evaluator | Thresholds, blocking rules, escalation |
| Report Generator | PR comment, HTML, JSON, audit trail |
| Trend Dashboard | Version-over-version quality, cost, latency |

## Non-functional
- Deterministic: fixed seed, frozen baseline model, pinned dependencies.
- Reproducible: containerized runner, same environment every run.
- Cost control: max tokens/sample, sampling for large datasets.
- Privacy: no PII in dataset; synthetic/redacted.
- Audit: immutable record of every test run + gate decision.

## Trade-offs
| Decision | Option A | Option B |
|---|---|---|
| Full dataset every PR | High confidence | Slow, costly |
| Smoke (subset) on PR + full nightly | Balance | More complex |
| Automated metrics only | Fast | May miss nuance |
| Human-in-loop for subset | Confidence | Slower |

## Risks & Mitigations
- Test data staleness → quarterly refresh, production drift monitoring.
- Metric gaming → human-reviewed golden set, A/B with production.
- False confidence → worst-case tracking, adversarial examples.
- Vendor model change without notice → pin versions, monitor vendor changelog.
- Cost explosion → tiered runs (smoke → full), token budgets.