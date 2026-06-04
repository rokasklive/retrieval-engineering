# Laws — the evidence behind the mindset

The three laws hybrid-search design must obey. Each gives the statement, why it matters, what to do differently, and the symptom that says it's being violated. Cite these by number when you flag a problem.

---

## Law 5 — Weakest Link (the poisoned candidate pool)

**Statement.** In multi-path retrieval, the quality of the fused result is bounded by the *weakest* included path — a single low-quality retriever poisons the entire candidate pool.

**Why it matters.** Fusion (RRF or score-level) combines the *union* of each path's results. A path contributing mostly noise occupies pool slots that could hold high-quality candidates from strong paths. The consequence is counter-intuitive and routinely surprises teams: hybrid accuracy can drop *below* the best single path. "More paths = better" is false.

**Design consequence.** Measure each path's standalone precision@K before including it (Stage C). Remove or down-weight paths that add noise without unique recall. Validate the floor: fused quality ≥ max(single-path quality) (Stage H, DR12). Use query-adaptive routing to skip a weak path on the query classes where it underperforms rather than removing it everywhere (Stage F).

**Violation symptoms.** Hybrid NDCG below the best single path. Adding a path degrades rather than improves quality. A silently changed vendor fusionType (Failure V27) letting the weak link go undetected.

---

## Law 6 — Complementarity (the orthogonal path)

**Statement.** A path's value is the *unique relevant documents it contributes that other paths miss* — not its standalone quality.

**Why it matters.** A weak-but-complementary retriever can improve fused results more than a strong-but-redundant one. Complementarity does not emerge automatically — it must be measured and designed for. Lexical and dense are complementary *by construction*: BM25 fails on vocabulary mismatch (synonyms, paraphrase) exactly where dense succeeds, and dense is blind to unseen exact identifiers exactly where BM25 succeeds. They fail in opposite directions, which is why that pairing is the default.

**Design consequence.** Measure complementarity (RoC) before adding any path beyond the first (Stage D, DR4/DR13). For each pair: |A∩B|, |A−B|, |B−A|, and |B−A|/|A∪B|. If |A∩B| exceeds ~50% of |A∪B|, the paths are mostly redundant. Select paths for complementarity, not standalone NDCG.

**Violation symptoms.** A path added "for completeness" that produces no unique recall. Three-path NDCG identical to two-path, with the extra latency wasted. Removing a path doesn't move the metric.

---

## Law 10 — Score Incompatibility (rank over score)

**Statement.** Scores from different retrieval methods are inherently incomparable — BM25 scores (unbounded, long-tailed), cosine similarities ([-1,1], concentrated near 0), and neural logits (arbitrary) live in different ranges and distributions.

**Why it matters.** Normalization (min-max, z-score) can *partially* align scores but fails when distributions shift — a model upgrade, a corpus change, a different query type. The breakage is silent: no error, just degraded ranking. RRF sidesteps the problem entirely by operating on ranks, which are always comparable. Important nuances from the 2026-06 evidence:
- **RRF is parametric, not magic** (Bruch et al. 2023). Per-retriever k tuning yields ~2–5% NDCG but doesn't generalize out-of-distribution. Per-source *weights* matter more than global k (DR24).
- **CC (convex combination) consistently outperforms RRF when labeled data exists** — and is sample-efficient (~100 queries). RRF is the best *zero-shot* default, not the best method overall (DR21).
- **Fusion gains can shrink or vanish after reranking** (Medrano et al. 2026). Verify past the reranker, not just at retrieval (DR22).

**Design consequence.** Default to RRF (k=60) as the zero-shot default. With ~100+ labeled queries, evaluate CC. Tune per-source weights before global k. Avoid score-level fusion unless distributions are calibrated *and* monitored. For agent consumers, use the relative top-gap `c(q)=(s₀−s₁)/s₀` as confidence, not raw scores (DR23). Pin fusionType explicitly (DR10).

**Violation symptoms.** Raw-score fusion corrupting ranking after a model upgrade. Treating RRF as universally optimal when labeled data is available. Assuming fusion gains survive reranking without verifying.

---

## Invariants (operational, falsifiable)

- **Invariant 6 — Fusion ≥ Single Path.** Fused quality must be ≥ the best single path on a representative judged set. A falsifiable checkpoint (Stage H), not an aspiration.
- **Invariant 7 — Independent Measurability.** Every retrieval path must be independently measurable (Stage C). If it isn't, that's the first thing to fix — before any fusion tuning, complementarity claim, or floor check is even possible.
