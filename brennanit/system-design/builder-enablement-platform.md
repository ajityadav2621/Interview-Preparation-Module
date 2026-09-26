# Builder Enablement Platform

## Clarify
- Builders: SPECIal Coders, service line engineers, citizen developers?
- What they reuse: connectors, Beacon modules, Cowork skills, prompts, eval harness?
- Discovery: how do they find what exists?
- Onboarding: docs, examples, pairing, sandbox?
- Support: SLA, office hours, issue queue?
- Adoption metrics: what signals usage and success?

## Flow
```
Publish
  Component ready → Manifest (schema, config, version, owner, SLA)
  → Validation (tests, security, docs) → Registry → Discoverable

Discover & Onboard
  Builder searches registry → Sees: description, version, config schema, 
  runbook, examples, metrics, owner, SLA
  → Tries in sandbox (pre-provisioned, isolated) → Works → Adopts

Adopt & Operate
  Builder adds dependency (config only) → Deploys
  → Runtime: component emits standard metrics, logs, traces
  → Builder sees: health, usage, quality, cost in shared dashboard

Support & Evolve
  Issue queue (tagged by component) → Owner triages → Fix → Version bump
  → Deprecation policy: 2 versions notice, migration guide, support window
  → Usage analytics → Invest in high-usage, deprecate low-usage
```

## Components
| Component | Responsibility |
|---|---|
| Component Registry | Manifest store, search, versioning, deprecation, metrics |
| Manifest Schema | Name, version, description, config schema, contract, SLA, owner |
| Sandbox Environment | Pre-provisioned, isolated, realistic data, auto-cleanup |
| Example Repository | Copy-paste starters, tested, versioned with component |
| Documentation Site | Auto-generated from manifest + markdown, searchable |
| Support Queue | Tagged by component, SLA, owner rotation, escalation |
| Usage Analytics | Adoption, health, quality, cost per component |
| Deprecation Manager | Notice, migration guide, timeline, forced migration |

## Non-functional
- Self-service: builder never waits for platform team to use a component.
- Contracts: stable internal model, versioned, backward-compatible.
- Observability: every component emits standard signals automatically.
- Security: component runs with its own least-privilege identity.
- Cost allocation: per-component, per-tenant visible.

## Trade-offs
| Decision | Option A | Option B |
|---|---|---|
| Strict standards | Consistent, safe | Slower to publish |
| Loose standards | Fast innovation | Fragmentation, risk |
| Central ownership | Quality, support | Bottleneck |
| Federated ownership | Scale | Inconsistent quality |

## Risks & Mitigations
- Low adoption → worked examples, pairing, office hours, usage metrics.
- Component drift → versioning, contract tests, deprecation policy.
- Support overload → self-service docs, sandbox, community forum.
- Shadow IT (builders bypass platform) → make platform easier than DIY.
- Version hell → semver, compatibility matrix, automated upgrade PRs.