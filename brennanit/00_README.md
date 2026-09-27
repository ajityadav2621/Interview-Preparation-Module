# Interview Preparation — AI Application Development Engineer (The Foundry)

This folder is your structured prep kit for the **AI Application Development Engineer** role at Brennan's "Foundry." It's organized by difficulty so you can build up progressively rather than jumping around.

## How this role reads (in one paragraph)
This is a **spec-driven, AI-assisted software engineering** role, not a pure ML/data-science role. You'll take approved specs → build with AI coding assistants → peer review → test → release, and turn one-off builds into reusable components (Beacon modules, Cowork skills). Core stack signals from the JD: Python web frameworks (Django/Flask), relational databases, REST/OpenAPI integrations, Git, CI/CD, Docker/containerization, Azure exposure, Linux/Bash, and LLM-adjacent concepts (prompting, RAG, agents, MCP) that you're expected to be able to *pick up on the job* rather than already master.

## Folder contents

| File | Level | Covers |
|---|---|---|
| `01_fundamentals.md` | Foundations | Python, Git, SQL/relational DBs, REST basics, Linux/Bash, HTTP/API fundamentals |
| `02_intermediate.md` | Intermediate | Django/Flask in depth, API design & OpenAPI, Docker, CI/CD, testing, LLM app concepts (prompting, RAG, agents, MCP) |
| `03_advanced.md` | Advanced | System design for AI-assisted apps, evaluation harnesses & regression testing, telemetry, Azure architecture, security/governance, spec-driven delivery, human-in-the-loop design |
| `04_practice_questions.md` | Mixed | Full model answers (not just pointers) to sample technical + behavioral questions, organized by level |
| `05_coding_questions.md` | Foundations → Advanced | 15 Python coding questions with full solutions, explanations, and complexity — fundamentals through RAG/API-flavored problems |
| `06_n8n_automation.md` | Intermediate → Advanced | n8n workflow automation: internals, node types, a worked human-in-the-loop flow, and interview Q&A |
| `07_rag_pdf_chatbot_design.md` | Advanced | Full "ask anything from this PDF" system design: extraction → chunking → embedding → retrieval → generation → citations, with trade-offs |
| `08_python_concepts.md` | Basic → Intermediate → Advanced | Python the *language* itself: mutability, comprehensions/generators, decorators, context managers, OOP, GIL/concurrency, memory management, descriptors, metaclasses |

## What's new in this version
Every topic file now includes a **"how it works internally"** section (e.g., how Docker isolation actually works via namespaces/cgroups, how the RAG pipeline works step by step, how a Django request flows through WSGI/middleware/ORM, how Git stores objects, how a JWT is verified) — not just definitions. `04_practice_questions.md` now has full written-out model answers, not just bullet pointers, so you can read them as if hearing yourself answer in the interview.

## Suggested prep order
1. Skim the JD mapping above so you know *why* each topic matters.
2. Work through `01 → 02 → 03` in order — each assumes the previous one.
3. Do `05_coding_questions.md` alongside `01`/`02` — the fundamentals problems pair with Level 1, the advanced/RAG-flavored ones pair with Level 3.
4. Read `06_n8n_automation.md` and `07_rag_pdf_chatbot_design.md` after `03_advanced.md` — they apply the same concepts (human-in-the-loop, RAG, evaluation) to two concrete, likely-to-come-up scenarios.
5. Use `04_practice_questions.md` to self-test after each stage, not before.
6. Prepare 2–3 concrete stories from your own experience for: (a) working from an ambiguous/incomplete spec, (b) building something reusable instead of one-off, (c) using an AI coding assistant effectively, (d) a production incident you debugged.

## What to emphasize given this specific JD
- **Spec discipline**: "Ability to work from a written specification, and to raise gaps rather than assume them" is called out explicitly — have an example ready.
- **Reuse over one-off builds**: Quality & Reuse is a named KPI. Talk about componentization, templates, shared libraries.
- **AI-assisted, not AI-replaced**: They want people who use AI coding assistants and agent tooling daily, but still own code quality, review, and production standards.
- **Governance awareness**: Data classification, data sovereignty, human-in-the-loop — even lightweight familiarity signals maturity for this role.
