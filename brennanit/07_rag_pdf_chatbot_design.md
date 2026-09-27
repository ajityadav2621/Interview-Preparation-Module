# RAG PDF Chatbot — Full Design Flow ("Ask Anything From This PDF")

This is a complete system-design walkthrough you can reproduce on a whiteboard: a chatbot that lets a user upload a PDF and ask questions about its contents, grounded in the actual document (not the model's memory). This is the canonical RAG use case and a very likely interview design question given the JD's RAG/agents/MCP mention.

---

## 1. High-Level Flow

```
                 ┌─────────────────────────────────────────────┐
                 │              INGESTION (once per PDF)         │
                 └─────────────────────────────────────────────┘
[User uploads PDF]
        │
        ▼
[1. Extract text]  ──(scanned/image PDF?)──▶ [OCR fallback]
        │
        ▼
[2. Clean & split into pages/sections]
        │
        ▼
[3. Chunk text]  (fixed-size + overlap, or semantic)
        │
        ▼
[4. Embed each chunk]  (embedding model → vector)
        │
        ▼
[5. Store in vector index]  (vector + chunk text + metadata: page #, doc id)


                 ┌─────────────────────────────────────────────┐
                 │           QUERY TIME (every question)         │
                 └─────────────────────────────────────────────┘
[User asks a question]
        │
        ▼
[6. Embed the question]  (same embedding model as ingestion)
        │
        ▼
[7. Similarity search]  (top-k nearest chunks from the vector index)
        │
        ▼
[8. (optional) Rerank]  (cross-encoder re-scores top-k for relevance)
        │
        ▼
[9. Build prompt]  (system instructions + retrieved chunks + chat history + question)
        │
        ▼
[10. Call LLM]  → generate grounded answer
        │
        ▼
[11. Post-process]  (attach citations: which page/chunk each claim came from)
        │
        ▼
[Return answer + sources to user]
```

---

## 2. Step-by-Step Detail

### Step 1–2: Extract & Clean Text
- Use a PDF text-extraction library (e.g., `pdfplumber`, `PyMuPDF`/`fitz`) to pull text per page, preserving page numbers as metadata.
- If a page returns little/no extractable text, it's likely a scanned image — fall back to OCR (e.g., Tesseract) for that page.
- Clean: strip headers/footers/page numbers if they repeat identically across pages (they add noise to chunks and don't help answer questions), normalize whitespace.

### Step 3: Chunking Strategy
This is the part most likely to get probed in depth:
- **Fixed-size chunking with overlap** (simplest, usually good enough): split into ~300–800 token chunks with ~10–15% overlap between consecutive chunks, so an idea that spans a chunk boundary still has surrounding context in at least one chunk. (See the `chunk_text` function in `05_coding_questions.md`, question 11 — same logic.)
- **Chunk by structure** when possible: split on headings/sections/paragraphs first, then apply a size limit within each section — this keeps semantically related content together better than a blind fixed-size cut.
- **Token count vs character count**: chunk by token count (using the same tokenizer as your embedding/LLM model), not raw character count, because token limits (both for embeddings and for the LLM's context window) are what actually constrain you.
- **Trade-off**: smaller chunks → more precise retrieval (less irrelevant text mixed in) but risk losing context that spans chunks; larger chunks → more context per chunk but noisier retrieval and fewer chunks fit in the LLM's context window at query time.
- **Always store metadata per chunk**: source document ID, page number, chunk index — this is what lets you cite "page 4" back to the user and is non-negotiable for a trustworthy chatbot.

### Step 4–5: Embed & Store
- Each chunk is passed through an embedding model to get a fixed-length vector (e.g., 384–1536 dimensions depending on model).
- Store vectors in a vector index: for a small single-PDF chatbot, an in-memory library (e.g., FAISS) or a lightweight embedded DB (Chroma) is enough; for a production multi-document system, a managed vector store (pgvector on Postgres, Azure AI Search, Pinecone) that supports metadata filtering (e.g., "only search within this document") matters more.
- Index type matters at scale: brute-force cosine similarity (see `05_coding_questions.md` question 12) is fine for one PDF; an approximate-nearest-neighbor index (like HNSW) is what real vector DBs use to stay fast as the corpus grows into millions of chunks.

### Step 6–7: Query & Retrieve
- Embed the user's question with the *same* embedding model used at ingestion (mismatched models produce incomparable vector spaces).
- Retrieve top-k (commonly k=3–8) chunks by similarity score.
- Optionally filter by metadata first (e.g., only this document's chunks) before ranking by similarity, for efficiency and correctness.

### Step 8: Reranking (optional but common in production)
- The initial similarity search (a "bi-encoder" — question and chunks embedded independently) is fast but approximate. A reranker (a "cross-encoder" that scores the question and each candidate chunk *together*) re-sorts the top-k for better precision before they go into the prompt — worth it when retrieval quality noticeably affects answer quality.

### Step 9: Prompt Construction
A typical prompt structure:
```
System: You are a helpful assistant that answers questions ONLY using the provided
document excerpts. If the answer isn't in the excerpts, say you don't know — do not
guess. Cite the page number for each claim.

Context:
[Page 3]: <chunk text>
[Page 7]: <chunk text>
...

Conversation history:
<previous turns, if any>

Question: <user's question>
```
- The explicit "answer only from the excerpts, say you don't know otherwise" instruction is the main lever for reducing hallucination — it doesn't eliminate it, which is why citations matter (Step 11) so a user can verify.
- Chat history is included so follow-up questions ("what about the second point?") resolve correctly — but history itself takes up context window space, so older turns are often summarized or dropped as the conversation grows.

### Step 10–11: Generate & Cite
- Call the LLM with the constructed prompt.
- Post-process the response to map claims back to the specific chunks/pages used (either by asking the model to output citations inline, or by simple heuristic matching of which retrieved chunk(s) most overlap the answer) — this is what makes the chatbot trustworthy/verifiable rather than a black box.

---

## 3. Key Design Decisions to Discuss If Asked to Extend This

| Question | Consideration |
|---|---|
| Multiple PDFs / a knowledge base? | Add a `document_id` filter at retrieval time; consider per-document vs cross-document search modes. |
| PDF updates over time? | Re-chunk and re-embed only the changed sections if you can detect them, rather than the whole document, to save cost. |
| Very large PDFs (100s of pages)? | Consider a two-stage retrieval: first retrieve relevant *sections* (coarse), then chunks within those sections (fine) — avoids searching the entire chunk set for every query. |
| Tables/images in the PDF? | Tables often need specialized extraction (e.g., `pdfplumber`'s table extraction) since raw text extraction garbles column alignment; images may need a vision model or captioning step if their content matters. |
| Cost control? | Cache embeddings (never re-embed unchanged chunks), cache common questions' answers, and consider a cheaper/smaller model for simple factual lookups vs a larger model for synthesis-heavy questions. |
| Evaluating quality? | Build a golden set of Q&A pairs for this specific document type and run the eval-harness approach from `03_advanced.md` — check both retrieval quality (right chunks retrieved) and generation quality (answer is correct and properly grounded) separately, since a bug in either stage causes a bad final answer. |

---

## 4. One-Sentence Answer If Asked to Summarize This Whole Flow
> "Extract and chunk the PDF's text with overlap so context isn't lost at boundaries, embed each chunk and store it with page-level metadata in a vector index, then at query time embed the question, retrieve the most similar chunks, feed them into the LLM's prompt with an instruction to answer only from that context, and return the answer with page citations so the user can verify it."
