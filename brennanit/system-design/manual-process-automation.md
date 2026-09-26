# Manual Process → AI Automation

## Clarify
- Current process: steps, actors, tools, pain points, volume, error rate?
- Data classification of inputs/outputs?
- Desired outcome: full automation, assist, or triage?
- Success metrics: time saved, error reduction, user satisfaction?
- Rollout: pilot team, gradual expansion?
- Fallback: manual process remains available?

## Flow
```
Discovery
  Interview SMEs → Document current flow (steps, decisions, tools, data)
  → Measure baseline: time, errors, handoffs, volume, classification
  → Identify automation candidates (rules vs AI vs hybrid)

Design
  For each candidate:
    Define input/output contract, acceptance criteria
    Choose approach: deterministic rule, RAG, agent, classifier
    Design human-in-loop points (thresholds, escalation)
    Define evaluation dataset (real samples from current process)

Build (smallest useful version)
  Implement → Peer review → Test (unit, integration, eval)
  → Deploy to pilot → Measure against baseline
  → Iterate based on feedback

Rollout
  Expand to more teams → Monitor metrics → Document runbook
  → Deprecate manual steps → Handover to ops
```

## Components
| Component | Responsibility |
|---|---|
| Process Profiler | Interview guide, flow diagram, baseline metrics |
| Decision Engine | Rules + AI, versioned, auditable, explainable |
| Human-in-Loop | Review queue, SLA, feedback capture |
| Evaluation Harness | Baseline comparison, quality gate |
| Telemetry | Time saved, error rate, adoption, user satisfaction |
| Runbook | Operate, troubleshoot, escalate, update |

## Non-functional
- Start with rules for deterministic steps; AI only where needed.
- Human-in-loop from day one for high-risk decisions.
- Baseline measurement before any code — no "feel" metrics.
- Data classification gates what AI can process.
- Reusable components: decision engine, review queue, eval harness.

## Trade-offs
| Decision | Option A | Option B |
|---|---|---|
| Automate end-to-end | Max efficiency | Higher risk |
| Assist + human decide | Safer, builds trust | Less time saved |
| Pilot one team | Fast feedback | May not generalize |
| Pilot multiple | Broader validation | Slower |

## Risks & Mitigations
- Automating the wrong thing → baseline first, pilot, measure.
- Low-quality AI output → eval harness, human review, iterative improvement.
- User rejection → involve SMEs from start, co-design, visible feedback channel.
- Data leakage → classification gate, regional model, no external AI for restricted.
- Maintenance burden → reusable components, clear ownership, docs.