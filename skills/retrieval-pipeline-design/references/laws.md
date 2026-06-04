# Laws of Retrieval Pipeline Design

The cited, evidenced versions of the design habits in SKILL.md. Each law has a statement, why it matters for design, the design consequence, and the violation symptom to look for and cite.

---

## Law 1 — Staged Retrieval (Filter-Then-Verify)
- **Statement**: Every retrieval problem decomposes into cheap broad recall + expensive precise ranking.
- **Why it matters**: Single-stage exact computation at scale fails geometrically. Conflating the stages produces systems that are neither fast nor accurate.
- **Design consequence**: Always separate candidate generation from final ranking. Retrieve generously (top-K ≈ 75+). Never use a cross-encoder as a first-stage retriever on >10K documents. Keep inner LIMIT > final count × 3 and ≤ ef_search.
- **Violation symptom**: Latency explosion (cross-encoder first-stage), silently capped recall (inner LIMIT < ef_search), or precision collapse (no reranker).

## Law 2 — Chunking Primacy (Binding Constraint)
- **Statement**: The chunk boundary is the irreducible binding constraint on all downstream retrieval quality.
- **Why it matters**: You cannot out-embed, out-rank, or out-rerank bad chunking. Fixed-character chunking nDCG@5 = 0.244 vs. structural paragraph-group nDCG@5 = 0.459 — a 1.88× swing from chunking alone.
- **Design consequence**: Flag undefined chunking as a structural quality unknown. Cross-reference `[[Chunking Strategy Design]]`. Design stages assuming chunking is adequate — but require verification.
- **Violation symptom**: Quality problems trace to chunk boundaries splitting key passages.

## Law 3 — Retrieval-Reranking Asymmetry (Ceiling-Floor)
- **Statement**: Retrieval recall sets the quality ceiling; reranking precision determines how close to it you get. Neither compensates for the other.
- **Why it matters**: Fixing the wrong subsystem wastes effort. Retrieval fixes (embeddings, indexing) require full reindex; reranking fixes (params, context) swap at query time. End-to-end quality ≈ recall × precision.
- **Design consequence**: Triage with a retrieval-position check before debugging. Invest in reranker agility (swappable without reindex). Retrieve generously — the ceiling is set at Stage 1.
- **Violation symptom**: Fixing the reranker when the doc was never retrieved, or tuning the retriever when the doc was present but mis-ranked.

## Law 5 — Weakest Link (Poisoned Candidate Pool)
- **Statement**: In multi-path retrieval, fused quality is bounded by the weakest included path.
- **Why it matters**: A low-quality retriever injects noise the reranker cannot fully recover. Hybrid accuracy can fall below the best single path.
- **Design consequence**: Measure each path's precision@K before including it. Remove or down-weight paths that add noise without unique recall. Validate: fused quality ≥ max(single-path quality).
- **Violation symptom**: Hybrid accuracy degrading below the best single path.

## Law 6 — Complementarity (Orthogonal Path)
- **Statement**: A retrieval path's value is the unique relevant documents it contributes that others miss — not its standalone quality.
- **Why it matters**: A weak-but-complementary retriever helps fusion more than a strong-but-redundant one. Training on residuals does not guarantee complementarity.
- **Design consequence**: Measure complementarity before adding a path. Lexical and dense are complementary by construction (vocabulary mismatch vs. identifier blindness). A third path must be justified by complementarity, not completeness.
- **Violation symptom**: Adding a path that produces noise without unique recall; fused quality < best single path.

## Law 8 — Drift (Silent Decay)
- **Statement**: Every retrieval system degrades continuously — embedding drift (8–12% annual MRR loss), index degradation, ground-truth staleness.
- **Why it matters**: MRR fell 0.85 → 0.62 over six months with zero alerts. Latency, error rate, and throughput don't trigger on quality decay.
- **Design consequence**: Every vector carries model-version metadata. Production designs include freshness policy, canary monitoring, and re-embedding triggers. Cross-reference `[[Drift Management & Freshness]]`.
- **Violation symptom**: Mixed-version indexes silently degrading; "nobody owns the embedding model."

## Law 10 — Score Incompatibility (Rank Over Score)
- **Statement**: Scores from different retrieval methods are inherently incomparable — BM25, cosine, and neural scores live in different ranges and distributions.
- **Why it matters**: Score normalization is fragile and shifts with every model upgrade and corpus change. Rank-based fusion (RRF) is distribution-shift robust.
- **Design consequence**: Default to RRF (k=60) — zero-config, sidesteps incompatibility. Use weighted RRF when one retriever is consistently stronger. Avoid score-level fusion unless distributions are calibrated and monitored.
- **Violation symptom**: Raw-score fusion corrupting ranking; unpinned fusion type silently degrading; normalization breaking after a model upgrade.

## Law 12 — Eval-Gated Complexity (Symptom-Driven Architecture)
- **Statement**: Every architectural layer must be justified by an observed, measured symptom — not theoretical completeness.
- **Why it matters**: Perfect-demo systems hit 8.4% confidence, 31.2% accuracy on real queries. The 7-layer maturity model is a symptom-driven ladder, not an ambition checklist.
- **Design consequence**: Never deploy all 7 layers from scratch. Start at Layer 1 (BM25 or dense-only). Add each layer only when its failure symptom appears in evaluation. Require an evaluation baseline before any stage addition.
- **Violation symptom**: All layers deployed without proving each; demo quality accepted as production evidence; no evaluation baseline.

## Law 14 — Model Versioning (Vector Provenance)
- **Statement**: Every vector must carry the identity of the embedding model that produced it — vectors from different models live in different spaces.
- **Why it matters**: Cosine similarity between text-embedding-3-small and bge-large-en vectors is geometrically meaningless. Without version stamping, migrations silently degrade quality.
- **Design consequence**: Require a non-nullable `model_version` column on every vector table. Design for blue-green index deployment. Cross-reference `[[Drift Management & Freshness]]`.
- **Violation symptom**: Mixed-version indexes; index lock-in from embedder changes; inconsistent embeddings across versions.

## Law 15 — Reranker Dominance (Final Arbiter)
- **Statement**: After reranking, sparse and dense first-stage retrievers converge to nearly identical quality (37.35 vs. 37.56 NDCG). The reranker dominates final quality.
- **Why it matters**: Allocate budget to reranker quality, not the last 2% of first-stage retrieval. The reranker is swappable without index rebuild. A cross-encoder adds ~+11% NDCG over BM25 alone.
- **Design consequence**: Always include a reranker when results feed an LLM (skipping it is the largest single common regression). Invest in reranker iteration velocity. First-stage choice matters for cost/infra, not final quality once a strong reranker is present.
- **Violation symptom**: Skipping the reranker; over-investing in first-stage retrieval past the convergence point.
- **Caveat**: The convergence figure comes from small models (22–30M params) on NanoNFCorpus — an empirical tendency, MEDIUM-HIGH confidence, not a guarantee for billion-doc corpora or large models.
