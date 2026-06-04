# Laws — the evidence behind the mindset

The three laws an evaluation program must obey. Each gives the statement, why it matters, what to do differently, and the symptom that says it's being violated. Cite these by number when you flag a problem.

---

## Law 7 — Evaluation Gap (the offline-online truth law)

**Statement.** Offline evaluation metrics do not predict online business impact — they are candidate *filters* at best; online A/B testing is the sole arbiter of whether a change actually improves the system.

**Why it matters.** The AUC inversion case is definitive: a model with AUC 0.92 saved $1,840/mo while a model with AUC 0.78 saved $2,320/mo — the *better* offline model produced the *worse* business outcome. Three structural gaps drive the divergence: (1) annotation selection bias — pools miss what newer methods find; (2) coverage scarcity — unjudged documents are invisible to the metric; (3) business-vs-relevance mismatch — a click is not a relevance judgment, and relevance is not revenue. Offline metrics measure relevance against a fixed pool; the business measures user behavior in the wild. Offline can tell you a change is directionally wrong; it cannot tell you it's right. Shipping on offline alone cost a documented 21% of business value.

**Design consequence.** Encode offline metrics as FILTERS, never DECISION GATES. Every evaluation design includes the offline-filter → A/B-gate → ship workflow (Stage F). Pair every offline metric with coverage (Law 11). Use canary-query MRR as the continuous production signal between formal evaluations. Track offline-online correlation across tests; below ~0.5, offline is not predictive — raise coverage, diversify pooling, recalibrate (DR17). When A/B is impossible, maximize offline rigor and flag explicitly that the gap cannot be *closed* offline (DR20).

**Violation symptoms.** A design that recommends shipping on an offline NDCG improvement with no A/B test. A team citing offline metrics as proof a change "works." Offline-improved changes repeatedly failing in production.

---

## Law 11 — Coverage Dependency (the unjudged-truth law)

**Statement.** Evaluation metrics become structurally unreliable when the fraction of top-K results carrying a human judgment (Coverage@K) drops below 0.50 — what you cannot see, you cannot measure, and the unseen may contain your best results.

**Why it matters.** Annotation selection bias is self-reinforcing. Dense retrievers suffer 5–10× higher annotation hole rates than lexical, because lexical-built pools never judged the documents dense finds. When Coverage@K < 0.50: NDCG is structurally biased against any retriever that surfaces documents outside the pool, and precision@K becomes a *lower bound* because unjudged docs are counted as non-relevant when they may be the best hits. BEIR re-annotation moved dense retrievers up 10–30 percentile points once the holes were judged. Coverage is the meta-metric: it decides whether every other metric is meaningful.

**Design consequence.** Report Coverage@K alongside every evaluation metric — non-negotiable (DR2). Coverage@10 < 0.50 → flag all metrics UNRELIABLE and fix coverage before deciding (DR3). Build judgment pools from ≥2 retrieval paradigms (DR5–DR6), pooled at depth N≈100 per system. Report coverage *per retrieval method*, not just overall (DR8) — a healthy aggregate can hide one starved path. Never treat unjudged documents as non-relevant in precision@K (DR9). Track the coverage trend: falling coverage means a stable-looking metric is silently decaying.

**Violation symptoms.** A report showing NDCG improvement with no coverage value, or with coverage <0.50 and no reliability warning. A precision@K that counts unjudged docs as wrong. A dense retriever "underperforming" on a lexical-only pool.

---

## Law 12 — Eval-Gated Complexity (the symptom-driven architecture law)

**Statement.** Every architectural layer added to a retrieval system must be justified by an observed, measured symptom in the current system — theoretical completeness is insufficient.

**Why it matters.** Systems with perfect demo behavior achieve 8.4% confidence and 31.2% accuracy on real queries (Failure R15) — complexity added for ambition rather than evidence produces fragile systems that look good in demos and fail in production. The maturity model is symptom-driven: start simple, measure, add a layer only when the eval proves the need, because each added layer spends latency budget and operational complexity. Without a baseline, you can't even see the symptom a layer is supposed to fix, so the layer is faith, not engineering.

**Design consequence.** Encode the eval-gated progression: establish baseline → observe a specific failure pattern → add the layer that targets that pattern → measure its marginal contribution → keep it only if it earned its cost → repeat. Never deploy with all layers from scratch. Every architecture decision must trace to a specific evaluation finding. This is the evaluation discipline underneath `[[Retrieval Pipeline Design]]`'s architecture choices — that skill decides *what* to build; this law governs *whether the evidence justifies it*. The prerequisite for all of it is Stage A: no baseline, no symptom visibility, no gate.

**Violation symptoms.** A new layer (reranker, second index, query rewriter) approved "because it's best practice" with no measured failure pattern naming it. Deploying a full multi-layer stack before any single-layer baseline exists. Architecture decisions with no traceable evaluation finding behind them.
