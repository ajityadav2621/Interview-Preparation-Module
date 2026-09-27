# Level 3 — Advanced

Goal: show senior-adjacent judgment — architecture trade-offs, governance, and the operating model unique to a "Foundry"-style AI delivery team. Each topic includes internal mechanics so you can go a layer deeper if pushed.

---

## 1. Spec-Driven Delivery & the Foundry Pipeline

The JD describes: **submission → AI review → concept approval → spec approval → build (AI-assisted) → sprint board → peer review → test → release**, aimed at repeatable, reusable delivery.

**Why this structure exists (the mechanics of the risk it manages)**: AI-assisted build tools are very good at producing plausible-looking code from an ambiguous spec by silently filling gaps with assumptions. A traditional pipeline catches wrong *logic*; this pipeline adds an earlier gate (spec approval) specifically because the more dangerous failure mode is code that is syntactically fine and does the *wrong thing* confidently. That's the internal reasoning behind "raise gaps rather than assume them" being called out explicitly.

**Be ready to explain**:
- How you'd structure a build so the *component* is reusable: define a clear interface/contract (inputs, outputs, config) so the internals can be swapped or extended without other teams needing to know how it works internally — this is the same principle as an API contract, applied to internal code.
- What you do when you find a spec gap mid-build: pause, document the specific gap and your proposed default, escalate to the spec owner rather than silently deciding — because a wrong assumption here doesn't just affect one build, it affects every future consumer of the "reusable" component.

---

## 2. System Design for AI-Assisted Applications

**Worked example — "AI drafts a response, human approves before sending"**:

- **Data flow**: request comes in → sensitive fields get classified/redacted as needed *before* they're sent to any model (data classification has to happen upstream of the model call, not after — once data leaves the boundary via an API call, you can't take it back).
- **Where the LLM sits**: usually an async job, not inline in the request/response cycle, because generation can take seconds and you don't want to hold an HTTP connection open or block a web worker.
- **Human-in-the-loop checkpoint**: the draft is written to a queue/inbox state (e.g., `status = pending_review`), not sent directly — a human action transitions it to `approved`/`sent`. This is a state machine, not just a UI toggle.
- **Failure modes and fallbacks**: model timeout → retry once, then flag for manual drafting; malformed output (e.g., asked for JSON, got prose) → validate and retry with a stricter prompt or reject and log; hallucinated facts → this is why RAG grounding and human review exist as two separate safety nets, not one.
- **Observability**: log the prompt, model version, retrieved context (if RAG), and output for every generation — this is what lets you debug a bad output *after* the fact instead of only noticing it broke in production with no trail.

---

## 3. Evaluation Harnesses & Regression Testing for AI Features

**Why unit tests aren't enough, mechanically**: a unit test asserts an exact expected value; LLM output for the same input can vary between runs (temperature/sampling) and "correct" is often a matter of degree (a good summary vs a great one), not a binary match.

**How an eval harness actually works internally**:
1. A **golden set**: a fixed list of representative inputs, each with either an expected output or a rubric describing what a good output looks like.
2. A **scorer**: either exact/fuzzy match (for structured outputs), a rule-based check (e.g., "does it contain required fields"), or an "LLM-as-judge" — a separate model call graded against the rubric, producing a numeric or pass/fail score.
3. **Aggregation**: scores across the golden set roll up into a pass rate or average score.
4. **Threshold**: a release gate — e.g., "must score ≥ 90% on the golden set" — blocks a change from shipping if quality regressed.
5. **Regression testing** = re-running this exact harness whenever the prompt, model version, or retrieval source changes, and diffing the score against the previous baseline, so a silent quality drop is caught in CI rather than by a customer.

**Be ready to explain**: designing a lightweight harness from scratch for a single feature — pick ~20–50 representative real inputs, define pass/fail or scoring criteria per input, automate running them and comparing to baseline on every relevant change.

---

## 4. Telemetry, Usage Reporting & Outcome Measurement

**How this is typically implemented internally**: instrument key events (request received, model called, latency, success/failure, human-review outcome) as structured log events or metrics emitted to a collector (e.g., Application Insights, Prometheus, or a custom events table). These get aggregated into dashboards. The distinction that matters:
- **System telemetry** answers "is it running" (uptime, error rate, latency).
- **Outcome telemetry** answers "is it working" (e.g., average manual review time before vs after the feature launched, volume of tickets auto-resolved) — tied directly to the JD's "Improved Efficiency" KPI, which is explicitly measured *after* release, not just at launch.

**Be ready to explain**: what you'd capture at launch to have a *baseline* to compare against later — you can't prove a reduction in manual effort three months in if you didn't measure the "before" state.

---

## 5. Cloud & Deployment Architecture (Azure-leaning)

**How container deployment works internally, end to end**: your CI pipeline builds a Docker image, tags it (often with the commit SHA), and pushes it to a container registry (e.g., Azure Container Registry). A deployment step (via IaC — Bicep/Terraform, or a release pipeline) tells the hosting platform (App Service for Containers, Azure Container Apps, or AKS) to pull the new image and start replacing running instances — typically a **rolling update**: new instances start and pass a health check before old instances are terminated, so there's no hard downtime window.

**Rollback internals**: because images are immutable and tagged, rollback is usually just re-pointing the deployment at the previous image tag — fast, because you're not rebuilding, just redeploying a known-good artifact.

**Be ready to explain**: how you'd roll back a bad deployment quickly (redeploy previous image tag; if using blue/green, just flip traffic back).

---

## 6. Security & Governance

**How OAuth2/token-based access control works internally (relevant for M365/MCP/ITSM integrations)**: the app never sees the user's actual password — it's issued a short-lived access token (and sometimes a longer-lived refresh token) scoped to specific permissions after an authorization flow. Every API call presents the token; the receiving service validates it (checking signature, expiry, and scope) before granting access. Secrets/keys used for these integrations should live in a managed secret store (e.g., Key Vault) and be pulled at runtime — never embedded in code or in a prompt sent to a model.

**Why AI-generated code needs a different review lens**: a human author's mistakes tend to cluster around genuinely hard edge cases; AI-generated code's mistakes more often look like *confidently wrong* patterns — subtly incorrect logic that reads as correct, unnecessary but plausible-looking dependencies, or copying a pattern that doesn't fit this specific system's constraints (e.g., ignoring the data classification rule because the training data didn't include it). Review has to explicitly check "does this match the approved spec and our architectural rules" rather than just "does this look like good code."

---

## 7. Behavioral / Judgment Questions to Prepare For
- Tell me about a time you had to raise a gap in a spec instead of guessing.
- Tell me about a time you built something reusable instead of a one-off fix — what made you choose that path?
- Tell me about a production incident you helped resolve — what was the root cause and what changed afterward?
- Tell me about a time an AI coding assistant gave you a wrong or risky suggestion — how did you catch it?
- How do you decide what needs human review vs what can be automated end-to-end?

---

### Self-check before the interview
You should be able to, in a structured way (situation → approach → outcome):
- Walk through designing an AI-assisted feature end-to-end, including where the human-in-the-loop checkpoint sits as a state transition, not just a UI element.
- Explain how an eval harness with an LLM-as-judge actually scores an output, mechanically.
- Explain a rolling deployment and why it avoids downtime.
- Explain how you'd instrument a shipped feature to prove it reduced manual effort, including what baseline you needed *before* launch.
