# RAG Knowledge Assistant

## Clarify
- Document sources (SharePoint, Confluence, file shares), classification, freshness SLA.
- Permission model: per-doc ACL, inherited from source?
- Answer style: citations required, follow-up questions, "I don't know" allowed?
- Volume: queries/day, peak QPS, latency target (p95 < 3s?).
- Evaluation: how is quality measured today?

## Flow
```
Ingestion (batch/streaming)
  Source docs → Chunk (size/overlap) → Embed (model v pinned) 
  → Upsert to vector index + metadata (source_id, updated_at, classification, permissions)
  → Delta sync on source change (webhook or scheduled)

Query
  User question → Rewrite/expand (optional) → Embed
  → Hybrid search (vector + keyword) → Metadata filter (perms, classification, freshness)
  → Rerank (cross-encoder) → Top-K chunks
  → Construct prompt: system + retrieved chunks (with citations) + user question
  → Model call (temp low, max tokens, structured output)
  → Validate: citations map to chunks, faithfulness, confidence
  → Return answer + citations + confidence
  → Log for evaluation
```

## Components
| Component | Responsibility |
|---|---|
| Ingestion Pipeline | Chunk, embed, index, metadata, delta sync |
| Vector Index | HNSW/IVF, tenant isolation, metadata filtering |
| Hybrid Retriever | Vector + BM25, fusion (RRF), configurable weights |
| Reranker | Cross-encoder, top-N → top-K |
| Permission Filter | Pre-retrieval ACL check (source IDs user can see) |
| Prompt Manager | Versioned, templated, citation format enforced |
| Model Endpoint | Regional, approved, token/cost budget |
| Output Validator | Schema, citation verification, faithfulness scorer |
| Evaluation Harness | Dataset, metrics, regression gate |

## Non-functional
- Freshness: delta sync < 5 min for critical sources; batch nightly for archive.
- Tenant isolation: separate index namespaces or metadata filter.
- Latency: retrieval p95 < 500ms, generation p95 < 2s.
- Cost: token budget per query; cache frequent queries (TTL, invalidation on source change).
- Security: pre-retrieval permission filter (never rely on post-filter), classification gate.

## Trade-offs
| Decision | Option A | Option B |
|---|---|---|
| Pure vector vs hybrid | Simpler, semantic only | Better for exact matches, codes, IDs |
| Rerank always | Higher quality | Extra latency, cost |
| Real-time vs batch sync | Fresh | Simpler, cheaper |

## Risks & Mitigations
- Stale answer → freshness metadata + TTL + delta sync monitoring.
- Wrong citation → citation validator + faithfulness eval.
- Permission leak → pre-filter at index query time, audit logs.
- No relevant doc → "I don't know" + suggest human, not hallucinate.
- Cost spike → token budget, cache, query classification to route simple to cheaper model.