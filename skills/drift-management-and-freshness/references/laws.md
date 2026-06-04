# Laws — the evidence behind the mindset

The three laws a drift-management plan must obey. Each gives the statement, why it matters, what to do differently, and the symptom that says it's being violated. Cite these by number when you flag a problem.

---

## Law 8 — Drift (the silent decay law)

**Statement.** Every retrieval system degrades continuously over time — embedding drift (8–12% annual MRR loss), index structural degradation, and evaluation ground-truth staleness — and standard monitoring captures none of it.

**Why it matters.** MRR dropped from 0.85 to 0.62 over six months with zero alerts firing. No standard service metric (latency, error rate, throughput) captures "the system is getting slightly worse at answering questions" — that regression is invisible to everything an ops team normally watches. Drift is gradual erosion, not a loud crash: distributions shift as models get silently updated by providers, language evolves, and the corpus grows. There are three drift *types* (model, index, evaluation) across three *timescales*, and each type needs its own detector — one signal has blind spots the others cover.

**Design consequence.** Three complementary detection signals are mandatory: canary-query MRR (sensitive, needs ground truth, weekly), cosine-distribution monitoring (zero-ground-truth, continuous, an early warning), and parameter-inflation monitoring (ef_search/nprobe ratio, continuous). Establish baselines at deploy time — the delta from baseline is the only signal. Use blue-green index deployment for model migrations, hot/cold tiering for cost-effective reindexing, and periodic full REINDEX as the structural cure.

**Violation symptoms.** A production retrieval system with no canary queries and only standard service monitoring — drift is happening undetected. A team that can report latency and error rate but cannot answer "is retrieval quality better or worse than last month?"

---

## Law 4 — Recall-Latency Contract (the silent degradation law)

**Statement.** Every ANN index operates under a recall-latency contract — when the contract degrades, both recall and latency slide silently, and neither standard alerting captures it.

**Why it matters.** As the vector space degrades (centroid collapse, soft-delete accumulation, data growth), achieving the same recall requires progressively more search effort — or the same effort yields worse recall. The universal first response is to raise ef_search/nprobe, which restores recall while permanently increasing latency and leaving the root cause untouched. The escalation is not a fix; it *is* the diagnostic signal. A documented case: nprobe 5×'d over eight months to hold recall, latency permanently up, root cause never addressed.

**Design consequence.** Monitor the ef_search/nprobe ratio vs. baseline: >1.5× → WARN (investigate root cause); >2× → CRITICAL (plan immediate REINDEX). Never permanently inflate parameters — the ratio is a diagnostic signal, not a tuning target. Periodic REINDEX is the only known cure for centroid collapse and soft-delete bloat. Confirm structural degradation with recall@10 vs. exact k-NN (>5% gap) and index-type-specific signals (centroid entropy for IVFFlat, index size vs. live rows for HNSW). *Choosing* the correct baseline ef_search is `[[Retrieval Pipeline Design]]`'s job; monitoring *inflation away from it* is this skill's.

**Violation symptoms.** ef_search increased from 40 to 200 over eight months with no REINDEX — recall maintained but latency 5×'d and the root cause untouched. A team proposing to make an inflated parameter value the new default.

---

## Law 14 — Model Versioning (the vector provenance law)

**Statement.** Every vector in a retrieval index must carry the identity of the embedding model that produced it — vectors from different models live in different spaces, and mixing them silently degrades retrieval quality.

**Why it matters.** Two vectors from different models, even if both 768-dimensional, exist in different semantic geometries. ANN search returns geometry-nearest neighbors from *both* spaces, so half the results come from the wrong geometry — and cosine similarity between vectors from different models is meaningless, so the system can't even tell. Model-version metadata is the prerequisite for safe migration and for cohort-based quality analysis ("did Model B improve recall on cohort X vs. Model A?"). Retrofitting version metadata onto an existing index is expensive, which is why it must exist from day one.

**Design consequence.** Every vector table MUST have a `model_version` column as a non-nullable schema constraint — not a convention, a constraint. ANN search must be constrained to a single model-version cohort. All model migrations use blue-green index deployment with cohort isolation. The column also enables cross-version quality comparison during and after migration.

**Violation symptoms.** `SELECT model_version, COUNT(*) FROM vectors` returns >1 distinct version in the same index without cohort isolation — a mixed-version index silently producing garbage (Failure V25). A `model_version` column that is nullable, absent, or populated by convention rather than enforced.
