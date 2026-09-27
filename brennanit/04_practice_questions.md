# Practice Questions — With Full Answers

Try answering each yourself first (out loud or written), then compare against the model answer. Model answers are written the way you'd actually say them in an interview — complete, but concise.

---

## Fundamentals

**1. What's the difference between a list and a tuple in Python, and when would you choose one over the other?**

> A list is mutable and a tuple is immutable. Internally, a list is a dynamic array that over-allocates space so appends are usually O(1); a tuple is a fixed-size, fixed-content array. Because tuples are immutable, they're hashable (if their contents are), so I can use a tuple as a dictionary key or put it in a set — I can't do that with a list. I reach for a tuple when I want a fixed, unchangeable record (like a coordinate pair or a return value with a fixed shape), and a list when the collection needs to grow, shrink, or be reordered.

**2. Explain the difference between `INNER JOIN` and `LEFT JOIN`.**

> An `INNER JOIN` returns only rows where there's a match in both tables. A `LEFT JOIN` returns every row from the left table, and if there's no matching row in the right table, the right table's columns come back as NULL. For example, joining `customers` to `orders`: an inner join gives me only customers who've placed an order; a left join gives me every customer, with NULLs for those who haven't ordered yet — which is what I'd use if I need to report on customers with zero orders.

**3. Walk me through what happens when you run `git pull` on a branch that has diverged from remote.**

> `git pull` is really `git fetch` followed by `git merge` (or rebase, depending on config). The fetch downloads the remote's new commits without touching my working branch. The merge step then tries to combine my local commits and the new remote commits. If both changed the same lines, Git can't auto-merge and marks the file as conflicted — I'd open the file, see the `<<<<<<<` / `=======` / `>>>>>>>` markers, manually choose the correct content, then `git add` the resolved file and complete the merge with `git commit`.

**4. What's the difference between a 401 and a 403 HTTP status code?**

> 401 Unauthorized actually means "not authenticated" — the server doesn't know who you are, or your credentials are missing/invalid. 403 Forbidden means the server does know who you are, but you don't have permission to do this specific thing. A quick way I remember it: 401 is "please log in," 403 is "you're logged in, but no."

**5. How would you find and kill a process using a specific port on Linux?**

> I'd run `lsof -i :PORT` (or `ss -tulpn | grep PORT` if `lsof` isn't available) to get the PID of whatever's bound to that port, then `kill <PID>` to send it a graceful termination signal (SIGTERM), or `kill -9 <PID>` if it's not responding and I need to force-kill it (SIGKILL).

---

## Intermediate

**6. How would you structure a Django or Flask app so a module you build can be reused in a different project without copy-pasting code?**

> I'd keep the core business logic in plain Python classes/functions that don't depend on Django/Flask request objects directly — the framework layer (views, blueprints) becomes a thin adapter that translates an HTTP request into a call to that logic and translates the result back into a response. Configuration (API keys, feature flags) goes through environment variables or a settings object passed in, not hardcoded values. That way the logic itself can be pip-installed or imported into a completely different project, and only the thin adapter layer needs to be rewritten per framework.

**7. You need to integrate with an external ITSM system's REST API that occasionally times out. How do you handle it?**

> First, I'd wrap the call with a timeout and retry with exponential backoff plus jitter, so I'm not hammering a struggling system with instant retries. Second, since retries risk duplicate side effects (like creating the same ticket twice), I'd send an idempotency key with the request if the API supports it, or check for an existing record before creating a new one. Third, if failures persist past a threshold, I'd trip a circuit breaker — stop calling for a cooldown period and surface an alert — rather than letting every request in the system queue up waiting on a dead dependency. I'd also log every attempt so a failure is debuggable after the fact.

**8. What's the point of a multi-stage Docker build?**

> It keeps the final production image small and secure by separating the build environment from the runtime environment. In the first stage, I use a full image with compilers and build tools to compile or bundle the app. In the final stage, I start from a minimal base image and `COPY --from=<build stage>` only the compiled artifact — so none of the build tools, source code, or intermediate files end up in the image that actually runs in production, which also reduces the attack surface.

**9. Explain Retrieval-Augmented Generation (RAG) to someone non-technical.**

> Think of it like an open-book exam instead of a closed-book one. Normally an AI model answers purely from what it memorized during training, which can be outdated or just wrong for your specific company's data. With RAG, before answering, the system first looks up the most relevant pages from your actual documents — like flipping to the right page in a manual — and hands those pages to the model along with the question, so it answers based on what's actually written there instead of guessing from memory.

**10. What's the difference between an LLM agent and a single prompt-response call?**

> A single call is one round trip: you send a prompt, you get text back, done. An agent runs a loop: the model can decide it needs more information or needs to take an action, so instead of just answering it requests a tool call — like "search the database" or "send this email" — the application actually executes that action, feeds the result back to the model, and the model decides its next step. This repeats until the model has enough to give a final answer. So an agent can complete multi-step tasks that require real actions and intermediate information, not just generate text.

---

## Advanced

**11. A business team submits a spec for an AI feature, but partway through the build you realize the spec is ambiguous about what happens when the model isn't confident. What do you do?**

> I wouldn't guess and bake an assumption into the build, because if this becomes a reusable component, that silent assumption affects every future consumer of it. I'd pause that part of the build, write down the specific gap and a proposed sensible default — for example, "route low-confidence outputs to human review rather than auto-sending" — and raise it with the spec owner through the Foundry's process rather than deciding unilaterally. If the timeline is tight, I'd build with the safest default (human review) so the feature is shippable while the actual answer gets confirmed, and document the decision once it's resolved.

**12. How would you design a lightweight evaluation harness for an AI feature that drafts email responses?**

> I'd start with a golden set of maybe 20–30 real, representative example inputs — a range of easy and hard cases. For each, I'd define what "good" looks like, either an expected answer for simple cases or a rubric for open-ended ones (e.g., "correct tone, addresses all questions asked, no fabricated facts"). For scoring, I'd use a mix: rule-based checks for structural requirements (like "must include a greeting" or valid JSON), and an LLM-as-judge call graded against the rubric for quality. I'd aggregate scores into a pass rate and set a release threshold — say 90%. Then, critically, I'd re-run this exact harness automatically whenever the prompt or model changes, and compare against the last baseline, so a quality regression is caught in the pipeline instead of by an unhappy user.

**13. How do you decide where a human-in-the-loop checkpoint belongs in an AI-assisted workflow?**

> I look at two things: impact and reversibility, and model confidence. If the action is high-impact or hard to undo — sending an external communication, changing a customer record, approving a financial transaction — I put a mandatory human checkpoint before it happens, regardless of confidence. If the action is low-risk and easily reversible — like suggesting an internal draft that a human will read anyway — I can let it be more automated. I also factor in data sensitivity: anything touching classified or regulated data gets an extra checkpoint even if the action itself seems low-risk, because the governance requirement isn't really about the action, it's about the data.

**14. What would you specifically check when reviewing a pull request that was largely AI-generated, versus one written entirely by a human teammate?**

> With a human-written PR, I'm mostly checking for logic errors and edge cases a person might've missed under time pressure. With an AI-generated PR, I additionally check for things that "look" right but aren't: does the logic actually match the approved spec, or does it match a plausible-sounding but wrong assumption the tool filled in? Are there unnecessary dependencies or patterns pulled in that don't fit our architecture? Is error handling actually present, or does it just look present? I also specifically check it against our architectural rules — data classification handling, human-in-the-loop requirements — because those are exactly the kind of context an AI assistant doesn't know unless it's explicitly told, and it's very good at writing code that reads as complete while quietly skipping them.

**15. How would you prove, three months after release, that an AI feature actually reduced manual effort for the business?**

> I'd need a baseline captured before launch — how long the manual process took, or how many manual steps it involved, measured the same way I'll measure it after. After launch, I'd track outcome telemetry alongside system telemetry: not just "is the feature running and how many people used it," but "how much time or how many manual steps did it actually remove," ideally tied to the same metric used in the baseline. I'd present the before/after comparison, not just usage counts, since high usage alone doesn't prove efficiency gained — it's possible to be used a lot and still not save meaningful time if the workflow around it is clunky.

---

## Behavioral — Structure Every Answer This Way (STAR)
**Situation → Task → Action → Result.** Keep each to under 90 seconds. Prepare one real story for each of these, from your own experience:
- A time you worked from an incomplete or ambiguous specification.
- A time you built something reusable rather than a one-off fix.
- A time you caught a mistake in AI-generated code or output.
- A production issue you diagnosed and resolved.
- A time you had to explain a technical trade-off to a non-technical stakeholder.
