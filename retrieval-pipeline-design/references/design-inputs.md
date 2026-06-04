# Design Inputs

What to extract from the user (or their brief) before designing, and why each input changes the design. When a brief is thin, this is your checklist of what to ask for. You rarely get all of it — but name the ones you're assuming, because an unstated assumption here propagates into every downstream decision.

---

### Problem statement
**Extract**: Who uses the search? What task are they performing? What is a good answer? What happens when search fails?
**Why**: Sets the cost of false negatives (compliance/customer-matching → exact KNN, 100% recall; e-commerce → ANN acceptable) and the metric (single correct answer → MRR; graded relevance → NDCG).

### Expected query classes
**Extract**: Exact identifiers (SKUs, error codes, CLI flags, function names); short keyword (1–3 words, ambiguous); natural-language/semantic (paraphrase, cross-lingual, conceptual); entity lookups (names); typo-prone; multi-hop/compound.
**Why**: Query classes determine stage count. Exact-ID → 1 stage (BM25); semantic → 2–3. Unknown classes → don't pick an advanced architecture yet.

### Corpus description
**Extract**: Document types (narrative/structured/code/tables), size, update frequency, metadata quality, entity density, exact identifiers present, long-document proportion, multilingual content.
**Why**: Drives chunking risk, lexical-vs-dense-vs-hybrid suitability, and freshness policy.

### Data format
**Extract**: Structured (DB rows) vs. unstructured (docs/emails) vs. semi-structured (catalogs, sectioned legal docs). Field availability for filtering and metadata-enriched reranking.
**Why**: Structured + exact IDs → BM25 sufficient. Unstructured narrative → dense may be needed. Semi-structured → metadata filtering design.

### Latency budget
**Extract**: Total acceptable latency (P50/P95/P99), per-stage allocation. Real-time (<50ms), interactive (<500ms), batch (>1s).
**Why**: Cross-encoder reranking adds ~100–200ms — unsuitable for <50ms. Determines whether generous retrieval (top-K 75+) is feasible. LLM reranking only for high-value, latency-tolerant queries.

### Quality target
**Extract**: Required recall ceiling, precision@top-K, acceptable false-negative rate. Is 100% recall non-negotiable?
**Why**: 100% recall → exact KNN (no ANN). >95% → HNSW with high ef_search. <90% → aggressive quantization possible.

### Current search implementation
**Extract**: What exists now (BM25 only? dense only?), what eval baseline exists, current failure symptoms.
**Why**: Sets the starting rung on the maturity ladder. "No eval baseline" → start there before any architectural change.

### Evaluation baseline
**Extract**: Judged examples? Coverage@K? Query-set diversity? Per-signal breakdown? Canary queries?
**Why**: Without judged examples, quality claims are unverified. Coverage <0.50 → metrics unreliable. No query-class breakdown → per-class failures invisible.

### Operational constraints
**Extract**: Freshness (real-time/hourly/daily), security/access-control, multi-tenancy, compliance, team expertise, infrastructure (GPU? PostgreSQL-only?).
**Why**: GPU → dense/hybrid viable. PostgreSQL-only → SPLADE or BM25+dense via pgvector. Multi-tenancy → access-control design applies.

### Infrastructure constraints
**Extract**: PostgreSQL-only? Elasticsearch? Dedicated vector DB? Cloud lock-in? RAM/storage budget?
**Why**: PostgreSQL-only → pgvector + tsvector. Elasticsearch → native RRF. Dedicated vector DB (Weaviate/Qdrant) → native hybrid.
