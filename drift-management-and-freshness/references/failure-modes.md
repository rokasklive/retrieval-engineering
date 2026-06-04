# Failure modes — symptom, cause, prevention

The drift failures the corpus characterizes well. Each gives the symptom you'll observe, the underlying cause, the design prevention, and the governing law. Cite these by code when you flag a problem.

---

## V7 — Silent Embedding Drift (8–12% annual MRR loss)

**Symptom.** MRR drops from 0.85 to 0.62 over six months. Zero alerts fire. Users notice wrong answers before monitoring does. Business metrics decline with no obvious explanation.

**Cause.** Embedding distributions shift as models get silently updated by providers, language evolves, and content grows. Drift is gradual erosion, not a loud crash. Standard service monitoring (latency, error rate, throughput) captures no quality regression — there is no standard metric for "the system is getting slightly worse at answering questions."

**Prevention.** Three detection signals — canary MRR (weekly, >7% w/w drop trigger), cosine-distribution monitoring (continuous, zero-ground-truth, KL >0.1 WARN), parameter-inflation monitoring (ef_search/nprobe ratio). Establish baselines at deploy time. Continuous, not episodic (DR1–DR6).

**Law.** 8 (Drift).

---

## V25 — Mixed-Version Indexes Silently Degrade

**Symptom.** After a model upgrade, search quality degrades with no errors. Results appear random or subtly wrong. Old and new vectors coexist in the same ANN index — roughly half the neighbors come from the wrong geometry space.

**Cause.** Vectors from different embedding models live in different geometry spaces. ANN search returns geometry-nearest neighbors from *both* spaces, and cosine similarity between vectors from different models is meaningless, so the system can't detect the problem itself.

**Prevention.** `model_version` as a non-nullable schema constraint (DR7). ANN search constrained to a single model-version cohort. Blue-green index deployment with cohort isolation for all migrations (DR8, DR15). A quick audit: `SELECT model_version, COUNT(*) FROM vectors`.

**Law.** 14 (Model Versioning).

---

## V6 — Recall-Latency Contract Violation with Zero Alerts

**Symptom.** The system maintains recall but latency creeps up over months — or latency stays flat while recall silently drops. Neither standard monitor triggers. Eventually it breaches latency SLOs and someone notices.

**Cause.** As the vector space degrades (centroid collapse, soft-delete accumulation, data growth), maintaining recall requires progressively more search effort. Parameter escalation masks the root cause, so the contract degrades invisibly.

**Prevention.** Monitor the recall-latency ratio. Track ef_search/nprobe inflation vs. baseline (>1.5× WARN, >2× CRITICAL). Treat inflation as a diagnostic signal, never a permanent fix (DR5–DR6, DR11). Confirm with recall@10 vs. exact k-NN (DR13).

**Law.** 4 (Recall-Latency Contract).

---

## V5 — Centroid Collapse Masked by nprobe Inflation

**Symptom.** Teams raise nprobe to hold recall as centroids collapse. Latency permanently increases. The index structure keeps degrading underneath. Eight months later nprobe has 5×'d and the root cause is untouched.

**Cause.** Parameter inflation is easier than reindexing — it "fixes" the symptom (recall) while hiding the disease (stale centroids). Engineers don't realize the parameter increase is masking a structural problem.

**Prevention.** Monitor centroid entropy (the strongest early signal) and per-centroid hit-rate skew (>40% → warning). Treat inflation as diagnostic, not a fix. REINDEX rather than permanently inflating; confirm recall returns at the baseline parameter value afterward (DR11–DR12).

**Law.** 4 (Recall-Latency Contract), 8 (Drift).

---

## V31 — Embedding Cache Cold-Start

**Symptom.** After a model upgrade, latency spikes dramatically. Queries hit the full index-computation path. Latency gradually improves over hours as the cache warms. Users experience degraded performance during the warm-up window.

**Cause.** Embedding result caches are per-version — a version change invalidates ALL cached computation, so the new cache starts empty and every query triggers full computation until it populates.

**Prevention.** Pre-warm caches before the alias swap by running representative queries against the new index during blue-green deployment. Load-test cold-start behavior. Budget 1–4 hours for warm-up depending on query diversity (DR16).

**Law.** 8 (Drift).
