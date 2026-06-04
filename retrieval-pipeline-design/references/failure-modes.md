# Failure Modes

Patterns to recognize and prevent when designing or reviewing a pipeline. Each has a symptom, likely cause, design prevention, and the relevant law.

---

### R1 — Skipping the reranker
- **Symptom**: High recall, low precision. The LLM receives poorly ordered context and generates poor responses; teams blame the LLM.
- **Cause**: First-stage retrievers optimize recall, not precision. With context limited to top-5–10, ordering matters enormously.
- **Prevention**: Include a cross-encoder reranker after first-stage retrieval whenever results feed an LLM. Measure Precision@K independently of Recall@K.
- **Law**: 15 (Reranker Dominance), 3 (Asymmetry).

### V15 — Exact-identifier blindness in dense retrieval
- **Symptom**: Vector search returns wrong results for queries with exact identifiers (SKUs, error codes, function names).
- **Cause**: Embedding models reduce text to continuous vectors; identifiers unseen in training have no special status.
- **Prevention**: Include a BM25/lexical leg; route identifier-bearing queries to it (DR2).
- **Law**: 6 (Complementarity).

### R19 — One-size-fits-all stage count
- **Symptom**: The full multi-stage pipeline runs for every query; simple queries incur needless latency.
- **Cause**: Pipeline designed as a fixed sequence with no query-adaptive routing.
- **Prevention**: Route by query type. Exact-ID → BM25 only. Short → BM25 + optional dense. Semantic/long → full pipeline.
- **Law**: 12 (Eval-Gated Complexity), 1 (Staged Retrieval).

### V12 — Inner LIMIT exceeds ANN search depth
- **Symptom**: Correct documents silently dropped; recall ceiling inexplicably low.
- **Cause**: Inner LIMIT set above the index's effective search radius (ef_search for HNSW).
- **Prevention**: Verify inner LIMIT ≤ ef_search. Values: 50 / 200 / 800 (DR14).
- **Law**: 1.

### V26 — Cross-encoder as first-stage retriever
- **Symptom**: Query latency >10s on modest corpora; linear CPU/memory scaling with document count.
- **Cause**: Cross-encoders process query+document jointly through self-attention — O(N) transformer passes per query.
- **Prevention**: Use bi-encoder/dense/BM25 for first-stage; cross-encoder only for top-K reranking (K ≤ 200) (DR13).
- **Law**: 1, 15.

### R16 — Misdiagnosing retrieval failure as reranking failure
- **Symptom**: Teams iterate on reranker parameters with no improvement; effort wasted on the wrong subsystem.
- **Cause**: The symptom (wrong final results) looks identical regardless of which stage failed.
- **Prevention**: Run a retrieval-position check first. Not in top-K → retrieval failure. In top-K but not first → reranking failure. Build position tracking into monitoring.
- **Law**: 3.

### R3 — No evaluation baseline
- **Symptom**: Can't answer "what's our quality?" User complaints are the only degradation signal.
- **Cause**: System built as a prototype without evaluation infrastructure.
- **Prevention**: Require a baseline before any architectural change. Minimum: canary MRR tracking, faithfulness scoring, weekly eval trends (DR8).
- **Law**: 12, 7 (Evaluation Gap).

### V21 — Fixed-character chunking destroys retrieval quality
- **Symptom**: nDCG@5 = 0.244 despite a good embedder; chunks end mid-sentence.
- **Cause**: Cutting at a fixed byte/character offset ignores semantic, syntactic, and structural boundaries.
- **Prevention**: Flag undefined chunking as a structural quality unknown; cross-ref `[[Chunking Strategy Design]]` (DR4).
- **Law**: 2.

### R20 — Reranker context insufficient for near-identical candidates
- **Symptom**: The reranker can't distinguish adjacent SKUs or product variants; flat score distribution.
- **Cause**: The reranker sees text content only; when documents differ only in structured fields, it sees identical text.
- **Prevention**: Concatenate differentiating metadata (SKU, brand, version) into the text passed to the reranker.
- **Law**: 15, 3.

### R15 — Demo-to-production gap
- **Symptom**: Perfect demo behavior, catastrophic real-user performance (8.4% confidence, 31.2% accuracy).
- **Cause**: Demo queries curated to match system strengths; real users ask unpredictable, edge-heavy questions.
- **Prevention**: Build a diverse labeled test set from real user queries — typos, ambiguity, edge cases. Evaluate on the production query distribution.
- **Law**: 12, 7.

### Drift / production (cross-ref `[[Drift Management & Freshness]]`)

| Failure | Symptom | Prevention |
|---------|---------|------------|
| No model versioning | Mixed-version indexes silently degrade; can't diagnose or roll back | Non-nullable `model_version` per vector table; blue-green deployment |
| No freshness policy | Stale embeddings; new content not searchable; old embeddings drift | Freshness SLA; re-embedding triggers; hot/cold tiering |
| Overbuilt before symptoms | Fragile, high-latency, unmaintainable; demo-to-production gap | Start simple; add each layer only when its symptom appears |
