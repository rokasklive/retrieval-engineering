# Decision rules — operational gates (DR1–DR24)

These are gates, not suggestions. Each has a rule and the trigger that fires it. Cite them by number in design artifacts. Grouped by concern.

## Path selection

| # | Rule | Trigger |
|---|------|---------|
| DR1 | **Exact identifiers present → always include a BM25/lexical leg, and route identifier queries to it only.** Dense is structurally blind to identifiers unseen in training (prevents V15). | Exact identifiers in query classes |
| DR2 | **Paraphrase / natural-language queries present → lexical-only is insufficient; add a dense leg.** BM25 can't match synonyms or paraphrase. | Paraphrastic queries present |
| DR3 | **Typos expected → add a fuzzy/trigram leg; route typo queries to fuzzy + BM25.** | Typo-prone users/queries |
| DR4 | **Adding a third path → require complementarity evidence (RoC).** What does it uniquely contribute? If RoC < ~0.10 (overlap >~50% of union), it's redundant — reject. | Third path proposed |
| DR5 | **A path whose standalone precision@K is below threshold must not enter fusion.** A noisy path poisons the pool (Law 5). | Low per-path precision |

## Fusion method

| # | Rule | Trigger |
|---|------|---------|
| DR6 | **Default to RRF (k=60).** Zero-config, distribution-shift robust, sidesteps score incompatibility. | New hybrid deployment or fusion change |
| DR7 | **One retriever consistently stronger (>5% NDCG across classes) → weighted RRF, weighting it higher.** | Per-path quality asymmetry |
| DR8 | **TRF proposed → require NDCG-gain evidence + storage-budget assessment.** ~128× storage overhead; only justified if the ~+8.1% nDCG gain is real and the budget exists. | TRF proposed |
| DR9 | **Score-level fusion proposed → require evidence of stable, calibrated distributions + ongoing monitoring.** It breaks silently on distribution shift (Law 10). Without calibration, reject. | Score-level fusion proposed |
| DR10 | **Always pin fusionType explicitly** (also k, ef, similarity). Never rely on vendor version defaults — Weaviate v1.24 changed silently (V27). | Any vendor hybrid feature |
| DR21 | **Labeled data (~100+ queries) available → evaluate CC as a superior alternative to RRF.** CC consistently outperforms RRF and is sample-efficient (Bruch et al. 2023). RRF stays the zero-shot default. | Labeled eval data exists |
| DR24 | **Tune per-source weights (W_lex, W_dense) before global k.** Per-source weights matter more; only touch per-retriever k once weights are exhausted. | Tuning fusion parameters |

## Quality verification

| # | Rule | Trigger |
|---|------|---------|
| DR11 | **Before enabling fusion, measure each path independently** — precision@K, recall@K, latency. Satisfies Invariant 7. | New path / fusion enabled |
| DR12 | **After enabling fusion, verify fused quality ≥ max(single-path quality).** The fusion floor checkpoint. If violated, find and remove the noisy path (prevents Law 5 violation). | After any fusion-config change |
| DR13 | **Measure complementarity (RoC) before adding any path beyond the first two.** Lexical+dense are complementary by construction; a third must prove unique contribution (prevents Law 6 violation). | Third+ path proposed |
| DR22 | **Verify fused quality after reranking and truncation, not just at retrieval.** Fusion gains shrink or disappear downstream (Medrano et al. 2026). If after-reranking NDCG ≤ baseline, fusion isn't delivering. | After any fusion-config change |

## Routing

| # | Rule | Trigger |
|---|------|---------|
| DR14 | **Route exact-identifier queries to BM25 only.** Skip dense — avoids V15 and unnecessary latency. | Query contains exact identifier |
| DR15 | **Route typo queries to fuzzy + BM25 only.** Skip dense unless semantic intent is also present. | Apparent typo |
| DR16 | **Route paraphrastic queries to dense (+ reranker).** Skip BM25 if no exact identifiers present. | Semantic query, no identifiers |
| DR17 | **Route mixed/ambiguous queries to full hybrid.** The catch-all. | Ambiguous intent |

## Metadata filtering

| # | Rule | Trigger |
|---|------|---------|
| DR18 | **Default: post-filter after fusion, before reranking.** Preserves recall from all paths; reduces reranker cost. | Metadata filtering required |
| DR19 | **Switch to pre-filter only when post-filter recall degradation is measured and unacceptable** — and the filter is deterministic and >90% selective. Pre-filter risks overshrinking (V13). | Post-filter recall degrades |
| DR20 | **Filter before the reranker, not after.** Filtering after reranking wastes reranker compute on docs that get discarded. | Filters exist in pipeline |

## Agent-facing

| # | Rule | Trigger |
|---|------|---------|
| DR23 | **For agent consumers, communicate confidence via relative score gaps, not absolute scores.** Use `c(q)=(s₀−s₁)/s₀` or source-tagged results. Never expose raw cross-retriever scores as confidence. | Agent-facing retrieval API |

---

## Path selection matrix

| Path | Required when | Optional / skip when |
|------|---------------|----------------------|
| **Lexical (BM25)** | Exact identifiers; short keyword queries; rare terms | No identifiers, all queries paraphrastic |
| **Dense (bi-encoder)** | Paraphrastic / NL / cross-lingual / conceptual | All queries are exact identifiers; latency budget <50ms |
| **Fuzzy (trigram)** | Typo-prone queries; entity lookups with misspellings | Always well-formed queries (API consumers) |
| **Learned sparse (SPLADE)** | Semantic signal needed but no GPU; inverted-index infra exists | GPU available for dense; two paths already cover the classes |
