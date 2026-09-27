# n8n Automation — Concepts, Internals & Interview Questions

*(Note: assuming "n8n" — the open-source workflow automation tool. If you meant something else, let me know and I'll redo this section.)*

n8n is relevant to this JD because the role explicitly touches integrations (REST APIs, M365, MCP connectors, CMDBs, ITSM platforms) and turning manual processes into automated, AI-assisted workflows — n8n is a common tool for wiring exactly that kind of automation without writing a full backend service for every integration.

---

## 1. What n8n Is
An open-source, node-based workflow automation tool (similar in spirit to Zapier/Make, but self-hostable and more developer-oriented — you can write custom JavaScript/Python inside a node, call any REST API, and connect it to your own infrastructure).

## 2. How It Works Internally
- A **workflow** is a directed graph stored as JSON: a list of **nodes** (each with a type, parameters, and position) and **connections** describing which node's output feeds which node's input.
- **Trigger nodes** start a workflow: a Webhook node (fires when an HTTP request hits a generated URL), a Cron/Schedule node (fires on a time interval), a manual trigger, or an app-specific trigger (e.g., "new email received").
- Data flows between nodes as an **array of items**, where each item is a JSON object (`{ json: {...}, binary: {...} }`). A node processes the incoming array and outputs a (possibly transformed/filtered) array for the next node — this is why one workflow run can naturally handle a batch of records, not just one.
- The **workflow engine** executes nodes in topological order following the connections graph, waiting for each node to finish before passing data downstream, and supports branching (IF/Switch nodes route items down different paths) and merging (Merge node combines multiple branches back together).
- **Credentials** (API keys, OAuth tokens) are stored encrypted separately from the workflow definition and referenced by ID — this is what lets you share/version a workflow without leaking secrets.
- Self-hosted n8n typically runs as a Docker container backed by a database (SQLite for small setups, Postgres for production) that stores workflow definitions and execution history/logs.

## 3. Common Node Types Relevant to This Role
- **HTTP Request** — calls any REST API (e.g., an ITSM system, a CMDB) with configurable method, headers, auth, and body — this is how n8n integrates with systems that don't have a dedicated pre-built node.
- **Webhook** — exposes a URL that external systems (or your own app) can POST to, to kick off a workflow (e.g., "on new ticket created, notify Slack").
- **Code node** — run custom JavaScript (or Python via a plugin/self-hosted setup) when built-in nodes aren't enough.
- **AI/LLM nodes** (in modern n8n) — call an LLM directly, or use an "AI Agent" node that can be given tools (other nodes/workflows) to call in a loop, similar in concept to the agent loop described in `02_intermediate.md`.
- **IF / Switch** — conditional branching based on data in the item.
- **Merge** — combine data from multiple branches (e.g., join API response with the original trigger data).

## 4. A Worked Example Flow (mirrors the JD's "manual process → AI-assisted automation")
**Goal**: when a new support request email arrives, have an AI draft a response, then require human approval before it's sent, and log the outcome for reporting.

```
[Email Trigger] 
     → [Code node: extract subject/body] 
     → [AI node: draft a response using the request text] 
     → [Set node: status = "pending_review", store draft] 
     → [Webhook/DB write: push to a review queue/inbox] 
     ... (human approves via a separate UI or Slack action, which fires a second webhook back into n8n) ...
     → [HTTP Request: send approved response via email/API] 
     → [HTTP Request: log outcome to telemetry/reporting store]
```
Notice this is the same human-in-the-loop state-machine pattern from `03_advanced.md` — n8n is just the orchestration layer wiring the steps together instead of hand-written backend code, which is exactly why it's a fast way to prototype/ship this kind of AI-assisted workflow.

## 5. Interview Questions & Model Answers

**Q: How is n8n different from something like Zapier or Make?**
> They solve the same class of problem — connecting apps and automating workflows without writing a full backend — but n8n is open-source and self-hostable, which matters for data sovereignty (you control where data flows and is stored), and it lets you drop into custom code (a Code node) or call any arbitrary REST API when a pre-built integration doesn't exist, which Zapier is more restrictive about.

**Q: How would you handle a step in an n8n workflow that calls an unreliable external API?**
> I'd configure retry-on-fail with a backoff on the HTTP Request node (n8n supports this natively), and design the downstream logic to be idempotent where possible — e.g., checking if a record already exists before creating a new one — so a retried step doesn't duplicate a side effect, the same idempotency principle as any other API integration.

**Q: How do you add a human approval step into an otherwise automated n8n workflow?**
> I'd split the workflow at the point requiring approval: the first part does the AI-assisted drafting and writes the result to a "pending" state (a database row, a Slack message with approve/reject buttons, or a small review UI), then stops. A second trigger (a webhook fired by the approval action) resumes the workflow from that state and performs the final action — so the "wait for a human" isn't the workflow blocking in memory, it's split into two independently triggered executions linked by shared state.

**Q: How would you keep credentials for an ITSM/CMDB integration secure in n8n?**
> Use n8n's built-in credentials store rather than hardcoding keys into a Code node or HTTP Request node's parameters — credentials are encrypted at rest and referenced by ID, and access can be scoped per workflow, so a workflow JSON can be exported/version-controlled without leaking secrets.

**Q: When would you *not* use n8n and instead write a custom service?**
> When the logic needs to be heavily tested and version-controlled as real code with unit tests and CI (n8n workflows are harder to unit test rigorously), when performance/throughput needs are high enough that the node-per-item execution model becomes a bottleneck, or when the logic is core, long-lived business logic that should live in the same codebase/review process as the rest of the application rather than a separately-managed visual workflow.
