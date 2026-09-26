# 1. Role, Foundry Pipeline, and Delivery Flow

## 1.1 What the Role Is

The AI Application Development Engineer turns approved business ideas into reliable, reusable software. The role is not just prompt writing or one-off scripting. It combines:

- Application development
- API and system integration
- AI/LLM application design
- Security and governance
- Testing and quality engineering
- DevOps and release ownership
- Documentation and knowledge sharing

The engineer works from an approved specification, identifies missing information early, builds the smallest useful version, validates it with users and reviewers, and then turns the result into a reusable component.

## 1.2 The Foundry Delivery Pipeline

```text
Idea / manual process
        |
        v
Submission
        |
        v
AI review
        |
        v
Concept approval
        |
        v
Specification approval
        |
        v
Build
        |
        v
Sprint board / engineering delivery
        |
        v
Peer review
        |
        v
Automated testing
        |
        v
Security and compliance review
        |
        v
Release
        |
        v
Handover, telemetry, outcome measurement
```

## 1.3 What Happens at Each Stage

| Stage | Main question | Engineer's responsibility | Typical output |
|---|---|---|---|
| Submission | What problem are we solving? | Clarify the business pain, users, and expected outcome | Problem statement |
| AI review | Is AI the right solution? | Identify whether automation, rules, AI, or a combination is appropriate | Recommendation |
| Concept approval | Is the idea worth pursuing? | Explain value, feasibility, risks, and rough effort | Approved concept |
| Spec approval | What exactly must be built? | Convert requirements into measurable acceptance criteria and raise gaps | Approved specification |
| Build | How do we turn the spec into software? | Design, implement, reuse existing components, and document decisions | Working increment |
| Sprint board | What is the next deliverable? | Break work into small, testable tasks and communicate progress | Sprint items |
| Peer review | Is the solution maintainable and safe? | Review code, design, security, tests, and documentation | Review feedback |
| Testing | Does it work as specified? | Run unit, integration, contract, quality, and user acceptance tests | Test evidence |
| Release | Can it operate safely in production? | Deploy gradually, monitor, and prepare rollback | Production release |
| Handover | Can others operate and improve it? | Document runbooks, ownership, metrics, and known limitations | Operational handover |

## 1.4 Spec-Driven Development

A good specification should answer:

- Who is the user?
- What problem is being solved?
- What are the inputs and outputs?
- What is the expected success measure?
- What data is processed?
- What is the data classification?
- Where may the data be processed and stored?
- When is human approval required?
- What are the failure and fallback behaviors?
- What are the acceptance criteria?
- What must be logged and measured?

A strong engineer does not silently fill gaps. They record assumptions, ask targeted questions, and obtain approval before building.

## 1.5 Reusability Principles

A capability should be built once and consumed many times.

| Principle | Meaning |
|---|---|
| Configuration over duplication | Put service names, endpoints, models, thresholds, and prompts in configuration |
| Components over copies | Extract shared authentication, caching, validation, and logging |
| Contracts over assumptions | Use OpenAPI, schemas, and explicit error formats |
| Versioning | Version prompts, APIs, schemas, and deployment templates |
| Documentation | Record how to configure, deploy, operate, and retire the component |
| Observability | Every reusable component exposes health, usage, quality, and error signals |

## 1.6 Definition of Done

A feature is not done when it merely runs locally. It is done when:

- It matches the approved specification
- Acceptance criteria are met
- Security and data-classification rules are satisfied
- Human-in-the-loop controls are implemented where required
- Unit, integration, contract, and quality tests pass
- Errors, retries, timeouts, and fallback behavior are defined
- Telemetry and audit events are emitted
- Documentation and runbook are updated
- Peer review is complete
- Deployment and rollback are understood
- The capability is reusable or a clear exception is documented

## 1.7 Interview Answer: “Tell Me About This Role”

A practical answer:

> This role sits between business requirements and production software. I would take an approved specification, validate assumptions, design a secure and reusable solution, build it with AI assistance and normal engineering practices, and take it through peer review, testing, deployment, and measurement. The important part is not just producing a working prototype, but making the result reliable, governed, observable, and reusable across service lines.

## 1.8 Interview Answer: “How Do You Handle an Ambiguous Specification?”

> I first separate facts from assumptions. I identify missing inputs, unclear success measures, data-classification gaps, integration dependencies, and human-review requirements. I then propose options with trade-offs, document the recommended assumption, and obtain approval before implementation. If the ambiguity affects security, cost, or user outcomes, I treat it as a blocker rather than a minor detail.

## 1.9 Interview Answer: “How Do You Use AI Coding Assistants Safely?”

> I use AI assistants for boilerplate, alternative designs, test ideas, and explanations, but I remain responsible for correctness. I verify generated code against the specification, check security and data handling, run tests, review dependencies, and never paste restricted data or secrets into an external tool. AI output is treated as a draft that must pass the same review and governance as human-written code.

## 1.10 Engineer Responsibilities in Practice

The Foundry engineer operates across five connected responsibilities. Each maps to material earlier in this guide.

| Responsibility | What it means in practice | Key prep areas |
|---|---|---|
| Build AI applications and modules | Turn approved specs into Beacon modules and Cowork skills that are reliable, auditable, and reusable | AI fundamentals, architectures, system design |
| Build reusable components | Deliver capability once and consume it across service lines via stable contracts and configuration | Reusability, contracts, observability design |
| Develop integrations | Connect apps to REST APIs, OpenAPI contracts, Microsoft 365, CMDBs, ITSM platforms, and MCP connectors | Integration fundamentals, auth, resilience |
| Operate the delivery pipeline | Move ideas through submission, AI review, concept and spec approval, build, peer review, testing, and release | DevOps fundamentals, CI/CD, definitions of done |
| Measure and improve | Instrument telemetry, usage, quality, and outcome metrics; build evaluation harnesses; enable other builders | Observability, evaluation, continuous improvement |

## 1.11 Technical Stack to Master

The role blends modern Python web development with AI engineering and cloud-native delivery. Build familiarity with each area conceptually; implementation can be developed on the job.

### Python web application development
- Build web applications with a framework such as **Django** or **Flask**.
- Design RESTful APIs, validate request/response contracts, and handle errors consistently.
- Use a relational database with an ORM, migrations, and transactions.
- Apply authentication, authorization, and input validation at the edge.

### AI and integration tooling
- **RAG, prompting, agents, and MCP** concepts for assembling AI capabilities.
- **REST APIs, OpenAPI/Swagger contracts, OAuth2, Microsoft Graph/365, CMDB/ITSM** integrations.
- **Evaluation harnesses** and **prompt and model regression testing** to protect quality.

### Delivery and operations
- **Git** version control with feature branches and pull requests.
- **CI/CD pipelines**, automated testing, and containerised deployment with **Docker**.
- **Infrastructure as Code** and cloud platforms, preferably **Azure**.
- **Linux** environments and **Bash** scripting for automation.
- **Telemetry, usage reporting, and outcome measurement** for shipped capability.

## 1.12 Linking This Guide to the Job

Use this map to study efficiently based on the role priorities.

1. Start with **1. Role, Foundry Pipeline, and Delivery Flow** (this file) to ground every concept in where it is used.
2. Cover **2. System Design Fundamentals** and **4. Integration Fundamentals** for architecture and connections to M365, CMDB/ITSM, and MCP.
3. Study **3. AI Application Fundamentals** and **5. DevOps, Security, and Observability** for RAG, evaluation, security, and release safety.
4. Review **6. Coding Fundamentals** for Python web, databases, and problem-solving, and **7. Required Basics** for the daily toolset.
5. Practice with the **interview** files: system design, technical, and behavioral questions.

This keeps preparation concept-first and tied to real Foundry delivery.

## 1.13 Engineering Quality Targets

The KPIs for this role are concrete. Measure yourself against them in preparation and in practice.

| KPI | Engineering signal |
|---|---|
| Application delivery | Definition of Done met; acceptance criteria passed |
| Quality and reuse | Peer review feedback incorporated; components published once |
| Improved efficiency | Manual effort replaced; outcome measured after release |
| Compliance | Data classification, audit, and human-in-the-loop applied |
| Continuous improvement | Tooling, tests, quality, and docs improved iteratively |

Good answers in the interview should link technical choices back to these outcomes.
