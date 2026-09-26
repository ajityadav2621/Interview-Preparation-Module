# Data Residency & Sovereignty Controls

## Clarify
- Customer requirements: which regions allowed for processing/storage?
- Data classifications that trigger residency rules?
- Model endpoints: which regions have approved models?
- Cross-border transfer: never allowed, or with safeguards?
- Audit: how to prove compliance per request?

## Flow
```
Request Received
  → Classify data (public/internal/confidential/restricted)
  → Lookup customer policy: allowed regions per classification
  → Route to regional endpoint (model, vector index, storage)
  → Enforce: no cross-border calls, no logging of restricted content
  → Process → Return result
  → Emit audit: classification, region, model version, data movement (none)
```

## Components
| Component | Responsibility |
|---|---|
| Classification Engine | Rule-based + ML, runs at ingress, tags request |
| Policy Store | Per-customer: classification → allowed regions, model endpoints |
| Regional Router | Directs to correct region; fail-closed on unknown |
| Regional Model Endpoints | Approved models per region, same version where possible |
| Regional Storage/Index | Vector DB, cache, logs per region |
| Audit Logger | Classification, region, model, no cross-border proof |
| Exception Handler | Rare approved transfers: logged, approved, time-limited |

## Non-functional
- Fail-closed: if region unknown or policy missing → reject, don't guess.
- Encryption: at rest (CMEK per region), in transit (TLS 1.3).
- Model parity: same prompt/model version across regions; drift detection.
- Latency: regional endpoints add latency; cache warm, capacity plan.
- Cost: multiple regions = more infra; right-size per region.

## Trade-offs
| Decision | Option A | Option B |
|---|---|---|
| Few regions, route all | Lower cost, simpler | May violate sovereignty |
| Per-customer region | Compliant | Higher cost, complexity |
| Cache in region only | Compliant | Cold start on new region |

## Risks & Mitigations
- Data processed outside region → fail-closed router, integration tests per region.
- Model version drift across regions → automated parity check, single deploy pipeline.
- Cache leakage → region-scoped cache keys, TTL, no cross-region replication.
- Audit gaps → structured audit events, automated compliance reports.
- New regulation → policy engine, rapid region onboarding playbook.