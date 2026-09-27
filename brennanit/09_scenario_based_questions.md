# Scenario-Based Questions — AI Application Development Engineer

These are realistic "what would you do" scenarios built around this specific JD (spec-driven delivery, reuse, AI-assisted build, integrations, governance). Each has a suggested approach and a model spoken answer. Use the STAR structure (Situation → Task → Action → Result) when you adapt these into real stories from your own experience.

---

### Scenario 1 — Ambiguous Spec Mid-Build
**Situation**: You're building a Cowork skill from an approved spec. Halfway through, you realize the spec doesn't say what should happen if the AI model can't find a confident answer — it just says "return the answer to the user."

**How to approach it**: Don't silently pick a behavior. Pause, write down the specific gap, propose a safe default (e.g., "if confidence is low, respond with 'I'm not certain, here's what I found' rather than guessing"), and raise it with the spec owner through the Foundry process rather than deciding unilaterally — because this component may be reused elsewhere.

**Model answer**:
> I'd stop and flag it rather than guess, because if I bake in an assumption here, it doesn't just affect this one build — it affects everyone who reuses this component later. I'd document the gap precisely, propose a default that leans safe (surface uncertainty to the user instead of a confident wrong answer), and get sign-off before continuing. If the timeline's tight, I'd build with that safe default so the feature is still shippable, and update it once the real answer comes back.

---

### Scenario 2 — AI-Generated Code Looks Right But Isn't
**Situation**: You used an AI coding assistant to scaffold an integration with a CMDB API. The code compiles, tests pass on the happy path, but in review someone notices it doesn't handle the case where the CMDB returns a partial/paginated result set — it silently only processes page 1.

**How to approach it**: This is exactly the "confidently wrong" failure mode AI-assisted code introduces — it looks complete because it handles the common case fluently. Walk through how you'd have caught it: reviewing the API docs against the code, not just the code against itself; writing a test case for a multi-page response, not just single-page.

**Model answer**:
> This is a good example of why AI-assisted code needs a different review lens than human-written code — it tends to look complete even when it's silently incomplete. I'd have caught this by checking the actual API documentation for pagination behavior rather than trusting that the generated code covered it, and by writing a test with a mocked multi-page response specifically, since the assistant only had the happy-path example in its context. Going forward, I'd add that as a standard test case in our integration test template so it's not missed next time — which also makes it reusable, not just a one-off fix.

---

### Scenario 3 — Build Once, Reused Everywhere
**Situation**: A service line asks you to build a small tool that summarizes support tickets. Three weeks later, a different service line asks for something almost identical for a different ticketing system.

**How to approach it**: Talk about designing the first build with reuse in mind from the start — separating the core "summarize this text" logic from the "fetch from this specific ticketing system" adapter — so the second request becomes "write a new adapter," not "rebuild the whole thing."

**Model answer**:
> When I built the first one, I'd separate the summarization logic (which is generic — it just takes ticket text and returns a summary) from the integration layer that's specific to the first ticketing system's API. So when the second service line's request comes in, most of the work is writing a new thin adapter for their system's API shape, and the summarization core, prompt, and evaluation harness are reused as-is. That's the difference between the "Quality and Reuse" KPI being met versus rebuilding this from scratch every time a new service line asks.

---

### Scenario 4 — Production Incident: AI Feature Misbehaving
**Situation**: A shipped AI feature that drafts customer responses starts producing oddly formatted output overnight — some responses are missing required fields. No code was deployed recently.

**How to approach it**: Walk through a debugging process for an AI system specifically — check if the underlying model version changed (even without your code changing), check recent input patterns for anything unusual, check logs of the actual prompts/outputs (this is why logging prompt + output + model version matters, per the telemetry section).

**Model answer**:
> Since nothing was deployed on our side, I'd first check whether the underlying model version changed upstream — model providers do update models, and behavior can shift even with an identical prompt. I'd pull recent logs of the actual prompts and outputs to see if there's a pattern — maybe a new type of input is triggering it, or the output format instruction is being followed less reliably by the new model version. If it is a model-side change, the fix might be tightening the prompt's formatting instruction or adding stricter output validation with a retry, and I'd add this exact case to our evaluation harness so a similar regression is caught automatically before it reaches production next time.

---

### Scenario 5 — Human-in-the-Loop Design Decision
**Situation**: You're designing a workflow where an AI agent can update records in a CMDB automatically based on incoming requests. Leadership asks: "does this need human approval, or can it just run automatically?"

**How to approach it**: Frame the answer around impact/reversibility and confidence, not a blanket yes/no.

**Model answer**:
> It depends on the specific action, not the feature as a whole. If the update is low-risk and easily reversible — like updating a description field — and the model's confidence is high, I'd let it run automatically but log every change for audit. If it's touching something high-impact or hard to reverse — like decommissioning an asset or changing ownership — I'd require human approval regardless of confidence, because the cost of a wrong automatic action there is much higher than the time saved. I'd also make sure anything touching classified or sensitive data gets a checkpoint by default, since that's a governance requirement, not just a risk-tolerance choice.

---

### Scenario 6 — Conflicting Priorities: Speed vs Reusability
**Situation**: You're two days from a sprint deadline. Building the feature the "quick and dirty" way would hit the deadline; building it properly as a reusable component would take an extra week.

**How to approach it**: Don't present this as a purely binary choice — talk about scoping down what "properly" means, communicating the trade-off transparently, and possibly shipping the quick version with a follow-up ticket to refactor for reuse.

**Model answer**:
> I'd surface the trade-off explicitly rather than silently picking one — deadlines matter, but so does the reuse KPI this team is measured on. I'd look for a middle ground: ship a version that meets the deadline but keep the core logic reasonably separated from the one-off glue code, so it's not thrown away, just not fully generalized yet. I'd flag clearly to the team that it's a scoped-down version and file a follow-up to generalize it, rather than quietly letting "temporary" become permanent technical debt that gets rebuilt from scratch next time someone needs something similar.

---

### Scenario 7 — Evaluating a New AI Feature Before Release
**Situation**: A teammate wants to ship an AI feature that classifies incoming tickets by urgency, but there's no formal way to know if it's actually accurate yet — it "seems to work" in manual spot-checks.

**How to approach it**: Push back constructively on "seems to work," and propose a lightweight evaluation harness instead of blocking the release entirely.

**Model answer**:
> "Seems to work" from spot-checks isn't something we can catch a regression against later, so I'd propose we spend a small amount of time building a lightweight eval harness first — pull maybe 30 real historical tickets with known correct urgency labels, run the classifier against them, and get an actual accuracy number with a release threshold. It doesn't need to delay the release much, but it gives us a baseline so if a future prompt or model change quietly makes it worse, we catch it in the pipeline instead of from a customer complaint.

---

### Scenario 8 — Integration Failure Across Systems
**Situation**: A workflow that pulls data from an M365 source, processes it with an LLM, and writes results to an ITSM platform starts failing intermittently — about 1 in 20 runs.

**How to approach it**: Walk through isolating which of the three systems is the actual failure point rather than guessing, and designing for partial failure (retries, idempotency) rather than assuming everything will succeed every time.

**Model answer**:
> With an intermittent failure across a multi-system pipeline, I'd first add or check logging at each boundary — the M365 call, the LLM call, and the ITSM write — so I can see which step is actually failing 1 in 20 times rather than guessing. If it's a transient network/timeout issue on one of the external calls, I'd add retry with backoff there specifically, and make sure the ITSM write is idempotent so a retry after a partial failure doesn't create a duplicate record. I'd also add an alert if the failure rate crosses a threshold, so we catch a worsening trend before it becomes a bigger incident.

---

### Scenario 9 — Explaining a Technical Trade-off to a Non-Technical Stakeholder
**Situation**: A business stakeholder wants the AI chatbot to "just know everything about every document instantly" with no delay, and doesn't understand why chunking/retrieval takes any noticeable time.

**How to approach it**: Translate the technical constraint (why RAG retrieval + LLM generation takes time) into business terms (accuracy and trust trade-off) without over-explaining internals they don't need.

**Model answer**:
> I'd frame it around the trade-off they actually care about: speed versus accuracy and trust. To answer a question correctly and point back to the right page, the system first has to look through the document for the most relevant parts — like a person quickly flipping to the right page before answering, instead of guessing from memory. That lookup step is what takes a small amount of time, but it's also what stops the chatbot from confidently making things up. I'd show them the delay is on the order of a second or two, not minutes, and that skipping it would trade a small delay for a real risk of wrong answers.

---

### How to Use This File
Pick 3–4 of these that best match your actual experience, and rewrite the "model answer" in your own voice using a real example from your background instead of the generic one given here — interviewers can tell a memorized script from a real story.
