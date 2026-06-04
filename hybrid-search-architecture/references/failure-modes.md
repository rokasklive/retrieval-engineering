# Failure modes — symptom / cause / prevention

The recurring ways hybrid systems break. Each gives the symptom to recognize, the likely cause, the design prevention, and the law it violates. Use these to *diagnose* a sick system and to *anticipate* risks in the Known Risks section.

---

## V15 — Exact-identifier blindness in vector search

- **Symptom.** Vector search returns wrong results for queries with exact identifiers (SKUs, error codes, function names, CLI flags). Adjacent identifiers (ERR_404 vs ERR_405, neighboring SKUs) collapse together in embedding space.
- **Cause.** Embedding models reduce text to continuous vectors. Identifiers never seen during training have no special status — the model treats them as noise and can't distinguish them.
- **Prevention.** Always include a BM25/lexical leg. Route identifier-bearing queries to BM25 only (DR1, DR14). Never ship dense-only retrieval when exact identifiers appear in the query classes.
- **Law.** Law 6 (complementarity — lexical and dense are complementary by construction; this is the most common complementarity opportunity).

## V27 — Weaviate unpinned fusionType (vendor-default drift)

- **Symptom.** After a vendor version upgrade, search quality silently degrades. No errors. Hybrid accuracy drops below the best single path.
- **Cause.** Weaviate v1.24 changed the default fusionType from rankedFusion (RRF) to relativeScoreFusion (score-based, more fragile). Teams trusting defaults inherited the change with no deprecation warning.
- **Prevention.** Pin fusionType, k, ef, and similarity metric explicitly. Audit fusion parameters after any vendor version upgrade (DR10).
- **Law.** Law 5 (weakest link — the silent change hides the regression) + Law 10 (the new default was score-based, which is more fragile).

## V13 — Pre-filter overshrinking

- **Symptom.** Metadata-filtered results are sparse or empty. Relevant documents exist but aren't retrieved — they were filtered out before the retriever saw them.
- **Cause.** Pre-filtering shrinks the candidate pool before retrieval. Documents matching the metadata constraint but ranked poorly by the retriever are never considered. Low-selectivity filters are especially dangerous.
- **Prevention.** Default to post-filtering (after retrieval/fusion, before reranking). Switch to pre-filter only when post-filter recall degradation is measured and unacceptable and selectivity is >90% (DR18, DR19).
- **Law.** Law 5 (weakest link — pre-filter narrows the recall ceiling for every later stage).

## Hybrid accuracy below best single path

- **Symptom.** Adding a path degrades overall quality. Fused NDCG < best single-path NDCG. Team assumed "more paths = better" without measuring.
- **Cause.** The added path contributes noise without unique recall; its low-precision results occupy pool slots that could hold the strong path's good candidates.
- **Prevention.** Measure each path independently before fusing. Verify fused ≥ best single (DR12); if violated, identify and remove the noisy path. Measure complementarity — a path with no unique recall is worse than no path.
- **Law.** Law 5 (weakest link) + Law 6 (the path lacked complementarity).

## Score-level fusion without calibration

- **Symptom.** A linear combination of BM25 scores and cosine similarities ranks erratically; quality varies unpredictably by query type; after a model upgrade the ranking breaks completely.
- **Cause.** BM25 (0–20, long-tailed) and cosine ([-1,1], concentrated near 0) were combined without normalization; when the embedding model was upgraded the cosine distribution shifted and broke the tuned weights.
- **Prevention.** Default to RRF (k=60) (DR6). Use score-level fusion only with calibrated, stable, monitored distributions (DR9). Never blend raw scores from different methods.
- **Law.** Law 10 (score incompatibility).

## Third path without complementarity

- **Symptom.** Three-path hybrid (lexical + dense + SPLADE) scores identically to two-path on NDCG@10; latency up ~33% for zero gain.
- **Cause.** The third path retrieves substantially the same documents as an existing path (>50% overlap of the union). Added for "completeness," not measured complementarity.
- **Prevention.** Measure RoC before adding a third path (DR4). If |R(new) − R(existing)| is small, it's redundant. Only add low-overlap, high-unique-contribution paths.
- **Law.** Law 6 (complementarity).

## Metadata filtering before fusion

- **Symptom.** A filter applied to one path's output before fusion silently discards documents the *other* path would have surfaced.
- **Cause.** The filter ran on one retriever's results before combining with the other's; the complementary path's relevant docs were dropped because they didn't survive the first path's pre-filter.
- **Prevention.** Fuse all paths first, then post-filter the combined pool (DR18). Ensures no path's unique results are lost to early filtering.
- **Law.** Law 5 (weakest link — pre-fusion filtering removes the complementary path's contribution).

## Fusion gains neutralized after reranking (Medrano et al. 2026)

- **Symptom.** Fusion improves upstream recall but end-to-end quality doesn't move. Hit@10 drops (e.g. 0.51 → 0.48) after reranking. No significant improvement over the single-query baseline; latency up for nothing.
- **Cause.** Extra fusion candidates fail to survive cross-encoder reranking and context truncation. Reranker capacity (K=10) caps how many new candidates reach the final context. Reformulation near-duplicates add contextual redundancy.
- **Prevention.** Verify fused quality *after* reranking and truncation, not just at retrieval (DR22). If after-reranking NDCG ≤ single-query baseline, fusion isn't delivering end-to-end value — default to the simpler path until production eval proves the benefit survives.
- **Law.** Eval-gated complexity — don't add complexity without measured benefit through the *full* pipeline.

## Agent over-trusting incomparable scores

- **Symptom.** An agent treats a BM25 score of 18.3 as more confident than a cosine of 0.92 and proceeds with the wrong context, because the raw scores look comparable.
- **Cause.** The retrieval API exposes raw scores from multiple fusion paths with no transformation or context that they're incomparable.
- **Prevention.** Never expose raw cross-retriever scores to agents. Use relative confidence: the top-gap `c(q)=(s₀−s₁)/s₀`, source-tagged results ("found via exact match"), per-result uncertainty flags (DR23).
- **Law.** Law 10 (score incompatibility, at the agent-interface layer).
