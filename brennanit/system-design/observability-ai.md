# Observability for AI Applications

## Clarify
- Audiences: developers, SREs, product, auditors, compliance?
- Signals: logs, metrics, traces, audits, business outcomes?
- Retention: hot (days), warm (months), cold (years)?
- PII: what must never appear in telemetry?
- Cost budget: % of infra spend on observability?

## Flow
```
Instrumentation (auto + manual)
  Every request: correlation ID (W3C traceparent)
  Logs: structured JSON, level, timestamp, correlation_id, 
        stage, decision, tokens, cost, confidence, NO PII
  Metrics: counters, histograms, gauges (Prometheus exposition)
  Traces: spans per stage (auth, retrieval, generation, validation, action)
  Audits: immutable, tamper-proof, classification, region, actor

Collection & Storage
  Logs → Loki/Elastic (retention by level)
  Metrics → Prometheus/Thanos (downsample: 1m→5m→1h)
  Traces → Tempo/Jaeger (tail-based sampling: keep errors+slow)
  Audits → Append-only store (immutable, queryable)

Analysis & Alerting
  Dashboards: RED (rate, errors, duration) per capability
  AI dashboards: tokens, cost, quality, confidence, escalation
  Alerts: on customer impact (error rate, latency, quality drop)
          NOT on component metrics alone
  SLOs: availability, latency p95, quality pass rate, cost/query
  Error budgets: burn rate alerts, page on fast burn
```

## Components
| Component | Responsibility |
|---|---|
| Instrumentation Lib | Auto-inject correlation ID, standard fields, sampling |
| Log Pipeline | Structured parsing, PII redaction, retention |
| Metrics Pipeline | Prometheus scrape, recording rules, downsampling |
| Trace Backend | Tail sampling, span linking, service graph |
| Audit Store | Immutable, signed, queryable, compliance-ready |
| Dashboards | Per-capability, per-tenant, executive |
| Alert Manager | Route, deduplicate, escalate, silence |
| SLO Tracker | Error budget, burn rate, reporting |

## Non-functional
- Correlation ID propagated everywhere (HTTP headers, queue metadata, DB comments).
- PII redaction at source (instrumentation lib), not downstream.
- Sampling: head (1% baseline) + tail (100% errors, slow, high-cost).
- Cost awareness: metric cardinality limits, log volume caps.
- Retention: logs 30d hot/1y cold; metrics 13m; traces 7d; audits 7y.

## Trade-offs
| Decision | Option A | Option B |
|---|---|---|
| Full traces vs sampled | Complete picture | Cost, storage |
| Rich logs vs metrics | Debuggable | Query cost |
| Auto-instrument vs manual | Consistent | May miss domain context |

## Risks & Mitigations
- PII leak → redaction in lib, automated scans, audit.
- Alert fatigue → SLO-based alerts, burn rate, owner rotation.
- Missing AI signals → dedicated AI metrics (tokens, quality, confidence).
- Cardinality explosion → label allowlist, relabel rules.
- Vendor lock-in → OpenTelemetry native, standard exporters.