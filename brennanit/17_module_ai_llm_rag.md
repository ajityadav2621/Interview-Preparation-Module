# Module 7: AI / LLM & RAG (QuickRAG Project)

---

## PART 1: Fundamentals (must know cold)

- **Embedding**: a fixed-length vector of numbers representing a piece of text's meaning, produced by an embedding model — semantically similar text produces vectors that are close together in that vector space.
- **Chunking**: splitting a long document into smaller pieces before embedding, because (a) embedding models have input length limits and (b) smaller, focused chunks retrieve more precisely than one giant embedded blob.
- **Overlap**: consecutive chunks share a small amount of text so an idea spanning a chunk boundary isn't split with zero shared context between the two resulting chunks.
- **Cosine similarity**: the standard way to measure how "close" two embedding vectors are — measures the angle between vectors, not their magnitude, which suits embeddings where direction encodes meaning.
- **Vector store/index**: where embeddings are stored so you can efficiently find the nearest ones to a query vector (brute-force for small datasets, an ANN index like HNSW/FAISS for large ones).
- **RAG (Retrieval-Augmented Generation)**: retrieve relevant chunks for a query, inject them into the LLM's prompt as context, so the model answers grounded in that specific text instead of purely from what it memorized during training.
- **Hallucination**: an LLM generating plausible-sounding but false/unsupported information — RAG reduces but does not eliminate this; instructing the model to say "I don't know" when the context doesn't contain an answer, and citing sources, are the standard mitigations.
- **Agent**: an LLM that can call tools/functions in a loop (observe result → decide next step) rather than a single one-shot prompt-response.
- **MCP (Model Context Protocol)**: a standardized interface for exposing tools/data sources to an LLM/agent, so an integration written once as an MCP server can be reused by any compatible client instead of writing bespoke tool-calling glue per app.

---

## PART 2: Interview Questions & Answers

**Q1: Walk me through the QuickRAG pipeline end to end.**
> A PDF is ingested and its text extracted, then split into overlapping chunks so an idea near a chunk boundary isn't lost. Each chunk is passed through an embedding model to get a vector, and stored in an in-memory vector store alongside metadata like the source page. At query time, the user's question is embedded with the same model, compared against the stored vectors using cosine similarity, and the most similar chunks are retrieved. Those chunks are inserted into the LLM's prompt as grounding context, and the model generates an answer based on them — with the API designed to return a structured, predictable response so it's callable by an agent, not just a human-facing chat UI.

**Q2: Why chunk with overlap instead of just splitting text into fixed non-overlapping blocks?**
> Without overlap, an important sentence or idea that happens to fall right at a chunk boundary gets split in half, and neither resulting chunk has the full context — so a query relevant to that idea might retrieve a chunk missing half the relevant information. A small overlap (commonly 10–15% of the chunk size) means that boundary content appears intact in at least one of the two adjacent chunks.

**Q3: Why did you use in-memory vector retrieval instead of a dedicated vector database?**
> At QuickRAG's scale, brute-force cosine similarity over the stored embeddings is fast enough and avoids the operational overhead of standing up and maintaining a separate vector database for a project this size. I'm aware that a production system with a much larger corpus (millions of chunks) would need an approximate-nearest-neighbor index like FAISS's HNSW to stay fast, since brute-force comparison against every vector becomes too slow at that scale — that's a deliberate next step I'd take if this needed to scale up.

**Q4: What does "agent-consumable API design" actually mean, concretely?**
> It means the request/response schema is strict and predictable — structured JSON in, structured JSON out — rather than free-text responses an agent would have to parse heuristically. Error responses use consistent, specific codes rather than plain error strings, so an agent can programmatically decide what to do next (retry, ask for clarification, give up) based on the error type. It also generally means idempotent endpoints where safe, since an agent might retry a call it's unsure succeeded.

**Q5: How would you reduce hallucination in a RAG chatbot answering questions from a PDF?**
> First, the prompt explicitly instructs the model to answer only using the provided excerpts and to say it doesn't know rather than guess if the answer isn't there. Second, returning citations (which page/chunk the answer came from) lets the user verify the claim themselves rather than trusting the model blindly. Third, improving retrieval quality itself — better chunking, and optionally a reranking step — reduces the chance that irrelevant context gets fed to the model in the first place, since a model grounded in irrelevant context is more likely to fill the gap with a plausible-sounding guess.

**Q6: What's the difference between RAG and fine-tuning, and when would you pick one over the other?**
> RAG changes *what information* the model has access to at answer time without touching the model's weights — good when you need up-to-date or proprietary data, or when you want traceability back to a source. Fine-tuning changes the model's weights to consistently alter its *behavior/style/format* — good when you need the model to reliably behave a certain way (e.g., a specific output format or tone) regardless of what's retrieved, but it doesn't help the model "know" new facts unless those facts were in the fine-tuning data, and it's more expensive and slower to iterate on than adjusting a prompt or retrieval pipeline.

---

## PART 3: How It Works Internally

**How an embedding model actually produces a vector**: text is first tokenized, then passed through a transformer-based neural network; the model's internal representations (typically the final hidden state or a pooled combination of token representations) are projected into a fixed-length vector. The model is trained (often via contrastive learning) so that texts with similar meaning end up with vectors that are close together in that high-dimensional space — this is *learned* from training data, not a hand-designed rule, which is why embedding quality depends heavily on which embedding model you use.

**How cosine similarity works mechanically**: for two vectors A and B, cosine similarity = (A · B) / (‖A‖ × ‖B‖) — the dot product of the vectors divided by the product of their magnitudes. This measures the *angle* between the vectors, not their length, so it's insensitive to a vector's raw magnitude, which matters because embedding magnitude doesn't reliably encode meaning the way direction does. A cosine similarity of 1 means the vectors point in exactly the same direction (maximally similar); 0 means orthogonal (unrelated); -1 means opposite.

**How an approximate-nearest-neighbor (ANN) index like HNSW avoids brute-force comparison at scale**: HNSW (Hierarchical Navigable Small World) builds a multi-layer graph where each node (a vector) is connected to a small number of "nearby" nodes; upper layers have fewer, longer-range connections for fast coarse navigation, lower layers have denser, short-range connections for fine-grained search. A query traverses from the top layer down, greedily moving toward closer neighbors at each layer, landing near the true nearest neighbors in roughly logarithmic time relative to the dataset size — instead of comparing the query against every single stored vector (which is what brute-force cosine similarity does, and why it doesn't scale to millions of vectors).

**How the agent tool-calling loop works internally**: the LLM is given a list of available tools (name, description, expected parameters) as part of its context. On each turn, the model either outputs a final text answer or a structured "tool call" request (naming a tool and arguments). The calling application intercepts this, actually executes the real action (an API call, a DB query, a retrieval step), and appends the tool's result back into the conversation context as if it were another message. The model then continues, now with that result available, and decides its next step — repeating until it produces a final answer. The model itself never executes anything; it only ever predicts what action *should* be taken next, which is why the calling application's tool implementation is what actually determines what's safe/possible for the agent to do.

**How MCP standardizes this**: instead of every app writing custom code to describe its tools to a specific model provider's function-calling format, an MCP server exposes tools/resources through a common protocol (tool discovery, invocation, and result format are all standardized). Any MCP-compatible client/agent can then connect to that server and use its tools without bespoke integration code — the same principle as how a standard REST/OpenAPI contract lets any HTTP client talk to any compliant server, applied specifically to LLM tool-calling.
