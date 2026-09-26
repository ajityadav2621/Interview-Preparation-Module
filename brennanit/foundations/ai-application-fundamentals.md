# 3. AI Application Fundamentals

## 3.1 What Makes an AI Application Different?

A traditional application follows deterministic logic. An AI application uses a model that produces probabilistic outputs. That means the engineer must design for:

- Variable output
- Hallucination risk
- Token and latency cost
- Prompt and model changes
- Quality regression
- Data leakage and prompt injection
- Human review and auditability

The engineering goal is to make the surrounding system reliable even though the model output is not fully deterministic.

## 3.2 LLM Basics

| Concept | Meaning |
|---|---|
| Token | A piece of text processed by the model |
| Context window | Maximum input and output tokens the model can process |
| Prompt | Instructions and context given to the model |
| Temperature | Controls randomness; lower is more deterministic |
| System message | High-level behavior and policy instructions |
| User message | The current request or task |
| Assistant message | The model's generated response |
| Completion | The generated output |
| Embedding | A numerical representation of text used for semantic search |
| Function/tool calling | A structured way for the model to request an action |

## 3.3 Prompt Engineering Fundamentals

A strong prompt includes:

1. **Role** — what the model should act as
2. **Task** — the exact action to perform
3. **Context** — relevant facts and source material
4. **Constraints** — what the model must not do
5. **Output format** — JSON, bullets, table, or fixed schema
6. **Examples** — one or more demonstrations when useful
7. **Quality checks** — ask the model to verify completeness or cite sources

### Good Prompt Characteristics

- Specific rather than vague
- Explicit about source of truth
- Clear about uncertainty
- Designed for the intended output schema
- Tested with representative and edge-case inputs
- Versioned and reviewed like code

## 3.4 Retrieval-Augmented Generation (RAG)

RAG combines retrieval with generation.

```text
Source documents
    |
    v
Ingestion
    |
    v
Chunking
    |
    v
Embedding
    |
    v
Vector or search index
    |
    v
User question
    |
    v
Retrieve relevant chunks
    |
    v
Optional reranking and permission filtering
    |
    v
Prompt with retrieved context
    |
    v
LLM generates answer with citations
```

### RAG Quality Factors

| Factor | Why it matters |
|---|---|
| Chunk size | Too small loses context; too large adds noise |
| Retrieval method | Keyword, vector, hybrid, or graph retrieval |
| Reranking | Improves relevance of top results |
| Metadata filtering | Enforces permissions, date, source, and classification |
| Freshness | Stale documents produce stale answers |
| Citations | Let users verify the answer |
| Evaluation | Detects retrieval and generation regressions |

### RAG Failure Modes

- Relevant document is not retrieved
- Wrong document is retrieved
- Retrieved context is outdated
- Model ignores the context
- Model invents facts
- Permissions are not enforced before retrieval
- Answer is correct but lacks citations

## 3.5 Agents and Tool Use

An agent combines:

- A model
- A plan or workflow
- Tools or connectors
- Memory or state
- Guardrails
- Human approval points

Agents are useful when the task requires multiple steps, conditional decisions, or external actions. They are more complex and riskier than a single prompt.

### When to Use an Agent

Use an agent when:

- The task has multiple dependent steps
- The system must choose between tools
- The workflow changes based on retrieved data
- The user needs an interactive assistant
- Human review is required at specific points

### When Not to Use an Agent

Prefer a deterministic workflow when:

- The task is simple and repeatable
- The output must be highly predictable
- The action is high risk and should not be delegated to autonomous decisions
- A rules engine or normal API integration is sufficient

## 3.6 Model Context Protocol (MCP)

MCP is a standard way for AI applications to discover and use external tools, resources, and prompts.

### Why MCP Matters

- Standardizes tool interfaces
- Separates the AI application from specific integrations
- Makes connectors reusable
- Supports capability discovery
- Reduces custom glue code

### MCP Concepts

| Concept | Meaning |
|---|---|
| MCP client | The AI application requesting capabilities |
| MCP server | The system exposing tools, resources, or prompts |
| Tool | An action the model can request |
| Resource | Data the model can read |
| Prompt | Reusable prompt template exposed by a server |
| Transport | How client and server communicate, such as stdio or HTTP |

## 3.7 Evaluation and Regression Testing

AI features need evaluation in addition to normal software tests.

### Evaluation Dimensions

| Dimension | Question |
|---|---|
| Faithfulness | Is the answer supported by the retrieved context? |
| Relevance | Does the answer address the user's request? |
| Correctness | Is the answer factually accurate? |
| Completeness | Are all required parts present? |
| Safety | Does the output follow policy and avoid harmful content? |
| Consistency | Do similar inputs produce similar outputs? |
| Cost | Is the token and latency cost acceptable? |

### Evaluation Dataset

A good dataset includes:

- Representative real requests
- Edge cases
- Ambiguous requests
- Known failure cases
- High-risk and sensitive examples
- Expected outputs or scoring rubrics
- Golden answers reviewed by humans

### Regression Detection

Compare a candidate version against a baseline using:

- Average score
- Pass rate
- Per-category performance
- Worst-case failures
- Statistical significance where appropriate
- Cost and latency changes

A release should be blocked when quality drops, safety failures increase, or cost exceeds the approved budget.

## 3.8 Safety and Security Fundamentals

### Prompt Injection

A prompt injection tries to override the application's instructions through user or retrieved content.

Mitigations:

- Treat retrieved content as untrusted data
- Separate instructions from data
- Use allowlists for actions
- Validate model output before execution
- Apply least privilege to tools
- Keep high-risk actions behind human approval

### Data Leakage

Prevent leakage by:

- Classifying data before processing
- Redacting or masking sensitive fields
- Using approved regional LLM endpoints
- Avoiding external tools for restricted data
- Logging only metadata, not raw sensitive content
- Encrypting data in transit and at rest

### Output Validation

Never execute model output directly. Validate:

- Schema
- Allowed values
- Data types
- Ranges
- Permissions
- Policy rules
- Confidence threshold

## 3.9 Interview Answer: “Explain RAG to a Non-Technical Stakeholder”

> RAG lets an AI application answer using the organization's own approved information. First, documents are broken into smaller pieces and indexed. When a user asks a question, the system finds the most relevant pieces, checks permissions, and gives them to the model as context. The model then generates an answer based on that context. This improves accuracy and allows the application to cite sources, but it still needs evaluation because retrieval can return the wrong document or the model can still misinterpret the context.

## 3.10 Interview Answer: “How Would You Reduce Hallucinations?”

> I would reduce hallucinations by improving retrieval quality, filtering results by permissions and freshness, giving the model clear instructions to use only the supplied context, and requiring citations. I would validate the output against the source material and use a faithfulness evaluator. For high-risk answers, I would add a confidence threshold and route uncertain responses to human review. I would also maintain a test set of known questions and run it before every release to detect regressions.

## 3.11 Interview Answer: “When Would You Choose an Agent Instead of a Normal Workflow?”

> I would choose an agent when the task requires multiple dependent steps, tool selection, or conditional branching that is difficult to encode as a fixed workflow. For a predictable, repeatable process, I would prefer a deterministic workflow because it is easier to test, audit, and operate. I would also ensure the agent has explicit tool permissions, output validation, and human approval for any consequential action.
