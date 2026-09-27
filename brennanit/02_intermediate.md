# Level 2 — Intermediate

Goal: demonstrate you can actually build and ship the kind of applications the JD describes. Each topic includes **how it works internally**, not just what it's for.

---

## 1. Django / Flask in Depth

**How a Django request works internally**:
1. WSGI/ASGI server (e.g., Gunicorn/Uvicorn) receives the HTTP request and hands it to Django.
2. Django's middleware stack runs (each middleware can inspect/modify the request, e.g., auth, CSRF, session loading) before it reaches the URL resolver.
3. The URL resolver matches the path against `urls.py` patterns (compiled regex/route trees) and calls the matching view.
4. The view queries the ORM — which translates Python queryset code into SQL **lazily** (a queryset isn't executed until you iterate it, call `.count()`, etc.) — then renders a template or serializer.
5. The response passes back out through the middleware stack (in reverse) before being returned to the client.

**Django ORM internals**: `select_related` performs a SQL JOIN and fetches related objects in the same query (good for single-valued relations, e.g., ForeignKey). `prefetch_related` issues a *second* query for the related set and joins the results in Python (good for many-valued relations, e.g., reverse FK or ManyToMany) — this is the fix for the N+1 problem.

**Flask internals**: Flask is built on Werkzeug (WSGI toolkit) and Jinja2 (templating). An "application factory" pattern (`create_app()`) exists so you can instantiate the app differently per environment (test/dev/prod) — this matters for reusability, since a hardcoded global app object can't easily be reconfigured or reused across projects.

**Be ready to explain**: how you'd structure a module (Django app or Flask blueprint) so it can be dropped into a different project — keep business logic decoupled from framework-specific request/response handling, so the core logic is just plain Python functions/classes the framework calls into.

---

## 2. API Design & Contracts

**How OpenAPI/Swagger works internally**: it's a YAML/JSON document describing every endpoint, its parameters, request/response schemas, and auth requirements, following the OpenAPI Specification. Tooling reads this document to auto-generate interactive docs (Swagger UI), client SDKs, and server stubs — the spec is the single source of truth both sides code against, which is why treating it as a contract (not just documentation written after the fact) prevents drift between frontend/backend or between integrated systems.

**Retry/backoff internals**: exponential backoff means each retry waits longer than the last (e.g., 1s, 2s, 4s, 8s), often with jitter (small randomization) added so many clients retrying at once don't all hit the server in the same instant (a "thundering herd"). A circuit breaker tracks recent failure rate and stops sending requests for a cooldown period once a threshold is crossed, to avoid hammering a system that's already struggling.

**Be ready to explain**: designing an API that integrates with a CMDB/ITSM platform — use an idempotency key (a client-generated unique ID sent with the request) so if a retry occurs after a timeout, the server can recognize "I already processed this" and avoid creating a duplicate ticket/record.

---

## 3. Docker & Containerization

**How a container actually works internally** (this is a common interview probe): a container is *not* a lightweight VM. It's a normal Linux process that the kernel isolates using:
- **Namespaces** — give the process its own isolated view of PID space, network interfaces, mount points, hostname, etc., so it "sees" only its own processes/files even though it's running on the same kernel as the host.
- **cgroups (control groups)** — limit and account for how much CPU, memory, and I/O the process can use.
- A **union/layered filesystem** (e.g., overlayfs) stacks read-only image layers with a writable layer on top — this is why Docker images build in layers and why unchanged layers are cached between builds.

Because containers share the host kernel (no separate guest OS), they start in milliseconds and are far lighter than a VM, which boots a full separate OS.

**Multi-stage build internals**: a Dockerfile can have multiple `FROM` stages; you build/compile in an early stage (with compilers, build tools) and `COPY --from=<stage>` only the compiled artifact into a slim final stage — so build tools never ship in the production image.

**Be ready to explain**: why a container that "works on my machine" can fail elsewhere despite Docker (e.g., architecture mismatch, missing environment variables/secrets not baked into the image on purpose, host resource limits via cgroups).

---

## 4. CI/CD & Testing

**How a CI/CD pipeline works internally**: a webhook fires when you push/open a PR; the CI system spins up a fresh, ephemeral runner (often itself a container) that checks out your code and executes pipeline steps in order (lint → unit test → build → integration test → deploy), failing fast if any gate fails. Each run is isolated so results are reproducible and not polluted by a previous run's state.

**Mocking internals**: a mock replaces a real dependency (e.g., an external API client) with a fake object that returns pre-programmed responses, so a unit test exercises your logic without making a real network call — this is what keeps unit tests fast and deterministic (no flakiness from a third-party service being slow or down).

**Be ready to explain**: adding a regression test after a bug fix — write a test that reproduces the exact bug (fails on old code, passes on fixed code) so the pipeline blocks any future change that reintroduces it.

---

## 5. LLM Application Concepts

**How prompting actually works internally**: an LLM predicts the next token repeatedly given everything before it (the prompt + tokens generated so far). A "system prompt" is just text placed at the start of the context that the model was trained/tuned to weight heavily as instructions. Structured output ("respond only in JSON") works because the model is predicting tokens most likely to follow a JSON-shaped context — it's not a hard guarantee, which is why production systems often validate/parse the output and retry on failure.

**How RAG works internally, step by step**:
1. Documents are split into chunks (e.g., a few hundred tokens each).
2. Each chunk is converted into a vector (an **embedding**) by an embedding model — a list of numbers capturing the chunk's meaning such that semantically similar text produces nearby vectors.
3. Vectors are stored in a vector database/index.
4. At query time, the user's question is embedded the same way, and the index returns the chunks whose vectors are closest (via cosine similarity or similar) to the question's vector.
5. Those chunks are inserted into the prompt as context, and the LLM generates an answer grounded in them — reducing hallucination because the model is answering "using this text" rather than purely from training data.

**How an agent loop works internally**: the model is given a list of available tools (name, description, parameters). On each turn, the model either produces a final answer or requests a tool call; the calling application actually executes that tool (an API call, a DB query, etc.) and feeds the result back into the model's context; this repeats until the model produces a final answer. The "intelligence" of an agent is really just this observe → decide → act loop wrapped around repeated LLM calls.

**How MCP (Model Context Protocol) fits in**: it standardizes the *interface* between an LLM/agent application and external tools/data sources (a common schema for "here are my tools, here's how to call them"), so a new integration can be written once as an MCP server and reused by any MCP-compatible client/agent, instead of writing bespoke tool-calling glue code per model or per app.

**Be ready to explain**: when you'd reach for RAG (need up-to-date or proprietary knowledge, want traceability to a source) vs fine-tuning (need to change the model's behavior/style/format consistently) vs just improving the prompt (the model already "knows" the answer, it's a phrasing/formatting problem).

---

## 6. AI Coding Assistants in Daily Workflow

**How they work internally, at a level worth knowing**: a coding assistant sends your surrounding code (and sometimes repo context/retrieved snippets) as a prompt to an LLM, which predicts likely completions/edits token by token — it has no actual understanding of whether the code is correct, only what's statistically plausible given the training data and context, which is exactly why review and testing responsibility stays with you.

**Be ready to explain**: a time an AI-assisted suggestion was wrong or insecure (e.g., missed input validation, wrong edge case, an outdated API usage) and how you caught it (tests, code review, manual reasoning about the spec) — this maps directly to the JD's requirement of taking AI-generated output "through to production standard."

---

### Self-check before moving on
You should be able to, without notes:
- Explain what actually isolates a Docker container from the host (namespaces + cgroups), not just "it's like a lightweight VM."
- Explain the five-step RAG pipeline end to end.
- Explain the Django request lifecycle from WSGI server to response.
- Explain why exponential backoff with jitter is used instead of fixed-interval retries.
