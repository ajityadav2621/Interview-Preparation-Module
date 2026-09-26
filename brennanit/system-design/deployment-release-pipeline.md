# Deployment & Release Pipeline

## Clarify
- Environments: dev, staging, prod (prod-like staging)?
- Deploy frequency: on-demand, daily, weekly?
- Rollback target: < 5 min? Who can trigger?
- Gates: what blocks promotion? (tests, security scan, approval)
- Database migrations: backward-compatible, rollback plan?
- Canary: what % traffic, what metrics, how long?

## Flow
```
Commit → CI
  Build → Lint → Unit Tests → Security Scan (SAST, SCA, secrets)
  → Integration Tests → Contract Tests → Container Build
  → Image Scan (vuln, SBOM) → Push to Registry (immutable tag)

Deploy to Dev
  Auto-deploy on merge → Smoke tests → Integration tests

Promote to Staging
  Manual or auto (if dev passes) → Acceptance Tests → Eval Harness
  → Security Review (if config changed) → Performance Baseline

Release to Prod
  Canary (5% → 25% → 50% → 100%) over 30-60 min
  Gates per step: error rate, latency, business metrics, eval quality
  Rollback: one-click, < 2 min, idempotent

Post-Release
  Monitor: error rate, latency, quality, cost (15-30 min)
  Incident: auto-rollback on gate breach
  Retrospective: within 48h for any rollback/incident
```

## Components
| Component | Responsibility |
|---|---|
| CI Pipeline | Build, test, scan, package, sign, SBOM |
| Registry | Immutable images, vulnerability scan, promotion gates |
| Deployment Engine | K8s/ACA, Helm/Bicep, blue-green/canary/rolling |
| Gate Evaluator | Automated metrics + manual approvals |
| Rollback Controller | One-click, preserves data, idempotent |
| Database Migration | Backward-compatible, transactional, rollback script |
| Feature Flags | Capability toggle, percentage rollout, kill switch |
| Observability | Deployment markers in traces/metrics, dashboards |

## Non-functional
- Immutable artifacts: same image dev→staging→prod.
- Secrets: never in image; injected at runtime from Key Vault.
- Drift detection: desired vs actual state, alert on divergence.
- Audit: who deployed what, when, gate results, rollback reason.
- Cost: staging auto-scaled down off-hours.

## Trade-offs
| Decision | Option A | Option B |
|---|---|---|
| Blue-green | Fast rollback, zero-downtime | 2x prod capacity |
| Canary | Early detection, lower blast | Slower full rollout |
| Rolling | No extra capacity | Slower rollback |

## Risks & Mitigations
- Broken release → canary gates on business metrics, auto-rollback.
- Data migration lock-in → backward-compatible schema, dual-write, verify before cutover.
- Config drift → GitOps, drift detection, automated remediation.
- Secret leakage → runtime injection, no secrets in CI logs, scan.
- Rollback fails → idempotent rollback script, tested in staging monthly.