# Human-in-the-Loop Review System

## Clarify
- Which decisions need review? (High-risk actions, low confidence, sensitive data)
- Confidence threshold: fixed or per-capability?
- Reviewer pool: dedicated, rotation, SME on-call?
- SLA: max time to review? Escalation if missed?
- Review actions: approve, edit, reject, request more info?
- Feedback loop: reviewer corrections → improve prompts/models?

## Flow
```
Model Output + Confidence Score
  → If confidence ≥ threshold → Auto-approve (with audit)
  → Else → Route to review queue (priority by risk/urgency)
    Reviewer sees: input, context, model output, citations, confidence, risk flags
    → Action: Approve / Edit & Approve / Reject / Escalate
    → Record: decision, reviewer, time, rationale
  → If edited → Re-validate output schema → Execute/Return
  → Store review record for feedback loop
  → Periodic: analyze rejections/edits → identify patterns → retrain/prompt-tune
```

## Components
| Component | Responsibility |
|---|---|
| Confidence Scorer | Calibrated probability (temperature scaling, conformal) |
| Review Queue | Priority, assignment, SLA tracking, escalation |
| Reviewer UI | Context, citations, diff (edit vs original), action buttons |
| Audit Log | Immutable record: input, output, confidence, decision, reviewer |
| Feedback Store | Reviewer corrections → training/eval dataset |
| Threshold Manager | Per-capability thresholds, versioned, A/B testable |
| Notifications | Slack/email on SLA breach, high-risk items |

## Non-functional
- Least privilege: reviewers see only data they're cleared for (classification).
- SLA: configurable per risk level; breach → alert + escalation.
- Audit: tamper-proof log, retention per policy.
- Feedback: corrections fed to evaluation harness for regression testing.
- Calibration: confidence scores must be reliable (reliability diagrams).

## Trade-offs
| Decision | Option A | Option B |
|---|---|---|
| Fixed threshold | Simple | May over/under-review |
| Per-capability threshold | Tuned | More complex |
| Review all low-confidence | Safer | Reviewer fatigue |
| Sample low-confidence | Scales | Misses some errors |

## Risks & Mitigations
- Review backlog → SLA alerts, auto-escalation, capacity planning.
- Low reviewer precision → training, calibration exercises, sampling audit.
- Confidence miscalibration → regular reliability checks, recalibration.
- Feedback bias → diverse reviewer pool, blind review for calibration set.
- Stale thresholds → periodic review, A/B test threshold changes.