# 5. DevOps, Security, and Observability

## 5.1 The Software Delivery Lifecycle

A production AI application moves from idea to value through repeatable stages. Each stage has a clear owner, a quality gate, and a measured outcome.

```text
Source
  -> Build
  -> Security scan and dependency check
  -> Unit and integration tests
  -> Package and scan image
  -> Deploy to environment
  -> Acceptance and quality tests
  -> Release (gradual)
  -> Operate and monitor
  -> Outcome measurement
```

Key principles:

- Automation replaces manual steps to reduce variance and human error.
- Every change is traceable from commit to production.
- Gates protect quality, security, and compliance before a release proceeds.
- Releases are gradual and reversible.

## 5.2 Git Workflow and Branch Strategy

A simple, safe workflow reduces risk:

- A protected `main` branch represents production-ready code.
- Feature work happens in short-lived feature branches.
- Changes are submitted as pull or merge requests with peer review.
- Reviews check correctness, security, tests, and documentation.
- CI runs automatically on every change and must pass before merge.
- Releases are tagged so any production state can be identified.

Keep branches short-lived to minimize merge conflicts and make changes easier to review.

## 5.3 Containers and Images

Containers package an application and its dependencies consistently.

Important practices:

- Base images are chosen for security and support, then pinned to a digest.
- Images are scanned for vulnerabilities and sensitive content before release.
- A software bill of materials (SBOM) records every dependency.
- Images contain no secrets and expose only required ports.
- Containers run as a non-root user with minimal capabilities.

A container image is a deployment unit, not a debugging tool. Debugging happens in staging with the same image used in production.

## 5.4 Infrastructure as Code

Infrastructure defined as code is versioned, reviewed, and tested like application code.

- Templates describe the desired state declaratively.
- Changes are reviewed and applied automatically.
- Idempotent operations mean re-running a template is safe.
- Drift detection alerts when reality diverges from the declared state.
- Destruction and recreation are planned, not performed by hand.

This makes environments reproducible and auditable.

## 5.5 Environments and Isolation

Separate environments isolate risk and prevent mistakes reaching production.

- Development is for experimentation and local testing.
- Staging mirrors production for validation.
- Production has the strictest controls and observability.
- Data is masked or synthetic outside production where possible.
- Access between environments is restricted and audited.

## 5.6 Deployment Strategies

Each strategy trades off speed, risk, and complexity.

### Blue-green

Two identical environments exist side by side. Traffic switches from the current version to the new version in a single step.

Best for:

- Zero-downtime releases
- Fast and safe rollback
- Clear release boundaries

### Canary

A small percentage of traffic receives the new version first, then the share grows as confidence builds.

Best for:

- Detecting problems early
- Comparing old and new behavior
- Reducing blast radius

### Rolling

Instances are replaced gradually across the pool.

Best for:

- Stateless, simple services
- Avoiding duplicate infrastructure cost
- Routine, low-risk updates

## 5.7 Rollback

A rollback plan must be decided before release. Important questions:

- What condition triggers a rollback?
- Who is authorized to trigger it?
- How long does it take to complete?
- How are in-flight requests handled?
- What happens to data created by the new version?
- How are users or downstream systems informed?

Automated, observable rollbacks are safer than manual intervention under pressure.

## 5.8 Data Classification and Sovereignty

Not all data is treated equally. Classification drives every protection decision.

| Level | Treatment |
|---|---|
| Public | Open access, minimal controls |
| Internal | Organization access, basic access control |
| Confidential | Need-to-know, encryption, limited logging |
| Restricted | Strict controls, limited processing, audit required |

Sovereignty rules where data may be processed and stored. These must be confirmed during design, not assumed during build.

## 5.9 Secrets Management

Secrets must never live in source code, images, or logs.

- Secrets are stored in a managed secret store such as Azure Key Vault.
- Applications fetch secrets at runtime using a managed identity or short-lived credential.
- Access is least-privilege and auditable.
- Secrets are rotated automatically on a defined schedule.
- Secret values are excluded from logs and traces.

## 5.10 Access Control and Least Privilege

Every identity should have the minimum access required.

- Identities are roles or service principals, not shared accounts.
- Permissions are granted per operation and per data classification.
- Short-lived tokens are preferred over long-lived credentials.
- Access is granted just-in-time and reviewed regularly.
- A denied request is the safe default.

## 5.11 Encryption and Data Protection

Protect data at rest and in transit.

- Transit uses TLS with modern, version-pinned protocols.
- Rest uses managed encryption, with customer-managed keys where required.
- Encryption is applied before external AI processing of restricted data.
- Keys are rotated and accessed only by approved services.

## 5.12 Supply Chain and Dependency Security

Modern applications depend on many external components.

- Dependencies are pinned to exact versions.
- An SBOM is generated for every build.
- Vulnerabilities are scanned and tracked to resolution.
- Builds run in isolated environments with controlled inputs.
- Provenance is recorded so a deployed artifact can be traced to its source.

## 5.13 AI-Specific Security

AI applications add new risk surfaces.

- Prompts are treated as untrusted input and separated from instructions.
- Model output is validated against a schema before any action.
- Tools available to the model are explicitly allowlisted.
- External tool calls are scoped to least-privilege identities.
- High-risk actions require human approval.
- All model requests and decisions are audited.

## 5.14 Observability Three Pillars

Observability is built in, not bolted on.

| Signal | Purpose |
|---|---|
| Logs | Explain what happened for a specific request |
| Metrics | Show trends, rates, latency, errors, and cost |
| Traces | Follow one request across services |
| Audits | Prove who did what, when, and why |

A shared correlation ID links every signal for one logical operation so a failure can be investigated end to end.

## 5.15 AI Observability

AI applications need the three pillars plus AI-specific signals.

- Token usage and cost per request
- Model and prompt version in use
- Retrieval quality and context relevance
- Output quality and faithfulness
- Confidence or uncertainty of responses
- Human escalation and override rate
- Refusal and safety event count
- Latency broken down by retrieval, generation, and validation

These feed automated quality guards that can block a release or raise an alert.

## 5.16 Alerting and Error Budgets

Metrics become action when they trigger the right response.

- Alerts are based on customer impact, not just component metrics.
- Alerts have clear owners, runbooks, and escalation paths.
- Error budgets allow risk-taking while protecting reliability targets.
- Pages wake people for urgent issues; tickets track improvement work.
- On-call rotations and incident playbooks are rehearsed.

## 5.17 Outcome Measurement

Reliability means nothing if the application does not deliver value.

- Business metrics show whether the capability achieves its goal.
- Leading indicators predict problems before users notice them.
- Lagging indicators confirm the outcome after delivery.
- Quality and cost are tracked together to avoid trading one away.
- Regular reviews compare expectations against reality.

## 5.18 Interview Questions and Answers

### Q: How do you release safely to production?

> I prefer a gradual strategy such as a canary or blue-green deployment behind an API gateway. Traffic is routed explicitly, health checks and key metrics are watched continuously, and a documented rollback is ready. I also gate the release on automated security and quality checks and keep the rollback idempotent so re-running it is safe.

### Q: What is the difference between monitoring and observability?

> Monitoring tracks known signals and alerts when predefined thresholds are crossed. Observability is the ability to ask new questions of a system when something unexpected happens. A well-designed system is observable through logs, metrics, and traces with correlation IDs, so problems can be diagnosed rather than just detected.

### Q: How do you protect secrets in source code?

> Secrets never go in code. They are stored in a managed secret store and retrieved at runtime using a managed identity or short-lived credential. Access is audited and least-privilege, logs never contain secret values, and rotation is automated. Scans in CI also block any secret accidentally committed.

### Q: How do you prevent prompt injection?

> I treat all retrieved and user content as untrusted data and keep instructions separate. I validate model output against a schema before acting on it, allowlist the tools the model can call, scope tool actions to least privilege, and require human approval for high-risk commands. Every prompt and decision is logged for audit.

### Q: How do you detect that an AI application is degrading?

> I instrument retrieval quality, output faithfulness, token and cost, latency, and human escalation rate. I run an evaluation dataset against the current and previous versions and compare average score, pass rate, and worst-case failures. I would block the release if quality drops, safety failures rise, or cost exceeds budget.

### Q: How do you design an error budget?

> I set an availability target, such as 99.5 percent, and subtract the achieved reliability to get the error budget. While budget remains, I can take safe risks like a new feature or a larger canary. When the budget is nearly exhausted, I freeze risky changes and focus on stability.
