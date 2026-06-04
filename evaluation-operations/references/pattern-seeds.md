# Pattern seeds — reusable evaluation patterns

Patterns the skill should reach for. Each gives the problem it solves, when to use it, when not to, and the ownership boundary with downstream skills.

---

## Pattern Seed 9 — Per-Signal Evaluation Decomposition

**Problem it solves.** Aggregate NDCG@10 can hide that one query type is catastrophically broken while another is excellent. A system with "good NDCG" may have 0% recall on exact-identifier queries — the average launders the disaster (Failure EV5).

**Method.** For every comparison, break the metric down by query type (exact, keyword, semantic, entity), by signal (precision, recall, MRR, coverage), and by domain. Compute each segment separately. Flag any per-signal metric >20% below the aggregate. Aggregate green + per-signal red = REJECT (DR14).

**When to use.** All production evaluation. Before any architecture change claiming "improvement."

**When NOT to use.** Trivial eval sets with <30 queries (segments become too small to be reliable). Genuinely single-query-type systems.

**Ownership.** Owned by this skill end-to-end. The decomposition output *feeds* `[[Hybrid Search Architecture]]` (per-signal numbers are how fusion quality is judged per class) — this skill produces them, that skill consumes them.

---

## Pattern Seed 6 — Eval-Gated Progression

**Problem it solves.** Teams add architectural complexity based on ambition, not evidence — producing fragile systems that demo well and fail in production (Failure R15).

**Method.** Establish baseline → observe a specific, measured failure pattern → add the single layer that targets it → measure its marginal contribution → keep only if it earned its latency → repeat. Never deploy all layers from scratch. Every layer traces to an evaluation finding (Law 12).

**When to use.** All new retrieval system deployments. Before any architecture addition.

**When NOT to use.** Well-understood domains with pre-validated architecture patterns (standard e-commerce, classic IR) where the symptom is already known.

**Ownership.** This skill owns eval-gated progression as *evaluation methodology* — the discipline of requiring a measured symptom and measuring marginal contribution. `[[Retrieval Pipeline Design]]` owns it as the *architectural principle* — what to actually build. Cross-reference required.

---

## Pattern Seed 23 — Continuous Evaluation Pipeline

**Problem it solves.** One-time evaluation at deploy time goes stale. Query distributions shift, documents change, nobody re-runs the eval, and the green number on the wall stops meaning anything.

**Method.** Weekly canary MRR + coverage check; monthly full eval with per-signal decomposition and baseline comparison; quarterly judgment-pool refresh and annotation audit; pre/post eval on every model change or index rebuild. Store results as a time series. Add production failures to the eval set monthly so it gets harder, not easier (Stage G).

**When to use.** All production retrieval systems. Multi-team environments especially.

**When NOT to use.** Development systems with no stable query set yet.

**Ownership.** This skill owns the evaluation pipeline *schedule* and the baselines it produces. `[[Drift Management & Freshness]]` owns *drift detection and alerting* that consumes these evaluation results, and the *operation* of the canary set this skill designs.

---

## Pattern Seed 10 — Retrieval-Reranking Triage

**Problem it solves.** When retrieval fails, teams misdiagnose — tuning the reranker when the document was never retrieved, or tuning the retriever when the document was in the candidate set but ranked low. Effort goes to the wrong stage.

**Method.** For a failing query with known ground truth, check whether the relevant document was in the candidate set at all. If it was never retrieved → a candidate-generation (retriever/recall) problem. If it was retrieved but ranked below the cutoff → a reranking/ordering problem. This requires `get_retrieval_position()` — visibility into where the gold doc landed at each stage.

**When to use.** Debugging any retrieval quality issue. Production monitoring of failure root causes.

**When NOT to use.** When you lack ground truth for the failing queries (you can't triage without knowing the right answer).

**Ownership.** This skill owns the *diagnostic methodology* (how to localize the failure). `[[Retrieval Pipeline Design]]` owns the *pipeline stage* that exposes `get_retrieval_position()` — this skill says the instrumentation must exist; that skill builds it.
