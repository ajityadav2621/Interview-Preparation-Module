# Module 8: Cloud & DevOps — AWS, Kubernetes, CI/CD

---

## PART 1: Fundamentals (must know cold)

- **Lambda**: a serverless function that runs your code in response to a trigger (API Gateway request, S3 event, schedule) without you managing a server — you pay per invocation/duration, and it scales automatically.
- **DynamoDB**: a managed NoSQL key-value/document database, designed for fast, predictable-latency access by primary key at scale, not for complex multi-table joins.
- **IAM & least privilege**: every AWS identity (user, role, service) should have only the specific permissions it needs — not broad access "just in case."
- **Container vs VM**: a container shares the host OS kernel (fast to start, lightweight); a VM includes a full separate guest OS (slower, heavier, stronger isolation).
- **Kubernetes core objects**: Pod (smallest deployable unit, one or more containers), Deployment (manages replica pods and rolling updates), Service (stable network endpoint for a set of pods), ConfigMap/Secret (configuration and sensitive values).
- **Rolling deployment**: new instances/pods start and pass a health check before old ones are terminated, so there's no hard downtime window during a deploy.
- **CI/CD pipeline stages**: lint → unit test → build → integration test → deploy — each stage is a gate; failing fast at an early cheap stage (lint) saves time versus failing at an expensive late stage (deploy).
- **Secrets management**: credentials/API keys should live in a secrets manager (AWS Secrets Manager, Kubernetes Secrets) and be pulled at runtime, never hardcoded or committed to Git.

---

## PART 2: Interview Questions & Answers

**Q1: How would you design a simple serverless endpoint using API Gateway, Lambda, and DynamoDB?**
> API Gateway receives the HTTP request and triggers a Lambda function, which contains the business logic and reads/writes to a DynamoDB table for storage. The Lambda's IAM execution role would be scoped narrowly — only the specific DynamoDB actions and table it actually needs (e.g., `GetItem`/`PutItem` on one table), following least privilege rather than broad DynamoDB access. Any sensitive config (like a third-party API key the Lambda needs) would be pulled from Secrets Manager at runtime rather than stored in plaintext environment variables.

**Q2: What's the difference between a container and a VM, and why does that matter for deployment speed?**
> A VM virtualizes hardware and runs a full separate guest operating system on top of a hypervisor — booting it means booting an entire OS, which takes real time and overhead. A container shares the host machine's kernel and is isolated using namespaces and cgroups, so starting one is just starting a regular process with an isolated view of the system — this is why containers start in milliseconds and are far more lightweight, which matters directly for how fast a rolling deployment or auto-scaling event can bring new capacity online.

**Q3: How does a rolling deployment avoid downtime?**
> Kubernetes (or any similar orchestrator) starts new pods running the new version alongside the existing old-version pods, and only routes traffic to a new pod once it passes its readiness probe (confirming it's actually able to serve requests). Old pods are terminated gradually as new ones become ready, so at every point during the rollout there's always a sufficient number of healthy pods serving traffic — no window where the service is fully down.

**Q4: How would you roll back a bad deployment quickly?**
> Since container images are immutable and tagged (often with the commit SHA), rollback is usually just redeploying the previous known-good image tag rather than rebuilding anything — Kubernetes performs the same rolling-update mechanism in reverse, bringing back the old-version pods and phasing out the bad ones. This is why keeping deployments tied to immutable, versioned artifacts (not "latest") matters — you need a specific previous version to roll back to.

**Q5: Why should secrets never be hardcoded in a Docker image or committed to a repo, even a private one?**
> A hardcoded secret ends up baked into every layer of the image (and in Git history even if later removed), which means anyone with access to the image or repo history has it — including CI logs, other engineers, or backup copies of old images. Pulling secrets at runtime from a dedicated secrets manager means the secret is never present in an artifact that gets copied/shared/versioned, and can be rotated without rebuilding or redeploying the application itself.

---

## PART 3: How It Works Internally

**How Lambda actually executes your code**: AWS maintains a pool of pre-warmed execution environments; when a trigger fires, it either reuses a warm environment (fast — a "warm start") or provisions a new one (slower — a "cold start," which includes downloading your code package and initializing the runtime). Your function runs inside that isolated environment for the duration of the invocation and is then either kept warm briefly for reuse or torn down — this is why Lambda scales near-instantly to many concurrent invocations (AWS just spins up more parallel execution environments) but individual cold-start latency can matter for latency-sensitive endpoints.

**How container isolation actually works (the mechanism, not just the concept)**: a container is a normal Linux process, isolated using **namespaces** (giving it its own view of process IDs, network interfaces, mounted filesystems, hostname — so it "sees" only itself) and **cgroups** (limiting how much CPU/memory/IO it can consume). The container's filesystem is built from **layered images** stacked with a union filesystem (overlayfs) — read-only image layers plus a writable layer on top — which is why unchanged layers are cached and reused across builds/containers.

**How Kubernetes' control loop actually manages state**: Kubernetes is built around a **reconciliation loop** — you declare a desired state (e.g., "3 replicas of this pod should be running"), and a controller continuously compares the actual cluster state against that desired state, taking action (starting/stopping pods) whenever they diverge. This is why Kubernetes self-heals: if a pod crashes, the actual state (2 running) no longer matches the desired state (3), and the controller starts a replacement pod automatically, without anyone manually intervening.

**How a CI/CD pipeline runner actually works**: a webhook fires on a git push/PR event; the CI system spins up a fresh, ephemeral runner (frequently itself a container, so each run starts from a known-clean state with no leftover state from a previous run) that checks out the code at that exact commit and executes the defined pipeline steps in sequence, failing the whole pipeline (and typically blocking merge) if any step returns a non-zero exit code. Running in an ephemeral, isolated environment per run is what makes CI results reproducible — a flaky pass/fail caused by leftover state from a previous run is a bug you want the pipeline design to prevent, not something to work around manually.
