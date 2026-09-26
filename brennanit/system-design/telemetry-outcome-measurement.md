# Telemetry & Outcome Measurement

## Clarify
- Business outcome: what does "success" look like? (time saved, defects reduced, revenue?)
- Leading indicators: early signals (adoption, quality, latency)?
- Lagging indicators: outcome confirmation (ROI, retention, satisfaction)?
- Audiences: engineers (debug), PMs (product), leadership (ROI)?
- Cadence: real-time dashboards, weekly reviews, monthly reports?

## Flow
```
Define Success Metrics (with stakeholders)
  Leading: adoption rate, task success rate, quality score, latency, cost
  Lagging: manual hours saved, error reduction, user satisfaction, ROI

Instrumentation
  Every capability emits:
    - Correlation ID (end-to-end)
    - Capability ID + version
    - Tenant, user (pseudonymized), classification
    - Stage timings, tokens, cost, quality, confidence
    - Outcome: success/failure, human edit, escalation
  → Structured logs + metrics + traces + audit

Collection & Aggregation
  Pipeline: logs→Loki, metrics→Prometheus, traces→Tempo, audit→immutable store
  Dashboards: per-capability RED + AI quality + business outcome
  Alerts: on leading indicator degradation (quality drop, latency spike)
  Reports: weekly (team), monthly (leadership), ad-hoc (investigation)

Feedback Loop
  Review metrics weekly → Identify gaps → Hypothesize → Experiment → Measure
  Connect telemetry to eval harness (quality) and feedback loop (user voice)
  Quarterly: outcome report vs business case → Decide continue/pivot/stop
```

## Components
| Component | Responsibility |
|---|---|
| Metric Definitions | Standardized: name, type, labels, SLO, owner |
| Instrumentation Lib | Auto + manual, correlation ID, PII redaction, sampling |
| Pipeline | Reliable, ordered, buffered, replayable |
| Dashboards | Capability, tenant, executive; drill-down to trace |
| Alert Manager | SLO-based, burn rate, route to owner, runbook link |
| Reporting | Automated weekly/monthly, narrative + data |
| Outcome Attribution | Link capability → business metric (causality via experiment) |

## Non-functional
- PII: never in telemetry; pseudonymize user/tenant IDs.
- Cardinality: label allowlist, relabel high-cardinality.
- Cost: sampling (1% traces, 10% logs), downsample metrics.
- Retention: aligned to audit/compliance needs.
- Attribution: use A/B or quasi-experimental design for causality.

## Trade-offs
| Decision | Option A | Option B |
|---|---|---|
| Rich telemetry | Deep insight | Cost, privacy risk |
| Minimal telemetry | Cheap, safe | Blind spots |
| Automated reports | Consistent | May miss nuance |
| Narrative reports | Context | Manual effort |

## Risks & Mitigations
- Vanity metrics (adoption without outcome) → tie every metric to KPI.
- Missing customer impact signals → SLOs on user-facing latency/error/quality.
- Attribution fallacy → run experiments, not just correlations.
- Data overload → curated dashboards, alert on SLO breach only.
- Privacy breach → redaction at source, automated scans, audit.