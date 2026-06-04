# Pattern seeds — reusable drift & freshness patterns

Patterns the skill should reach for. Each gives the problem it solves, when to use it, when not to, and the ownership boundary with downstream skills.

---

## Pattern Seed 3 — Blue-Green Index Deployment

**Problem it solves.** Embedding-model upgrades require full reindexing. Naive replacement causes downtime, or old and new vectors coexist and corrupt results (Failure V25).

**Method.** Build the new index with the new model in parallel (no user impact) → validate against canary queries and the full eval set vs. a pre-upgrade baseline → pre-warm caches → atomic alias swap → monitor canary MRR ≥48h → decommission the old index. If quality drops >7%, swap the alias back — instant rollback. Populate `model_version` distinctly per cohort.

**When to use.** Embedding-model change. Major index-structure change. Any time rollback must be instant.

**When NOT to use.** <100K vectors (a full rebuild is fast enough). Reranker-only changes (no reindex needed). Minor parameter tuning.

**Ownership.** Owned by this skill end-to-end for model migration. Future `[[Migration & Deployment Operations]]` extends it with detailed runbooks for 1B+ vectors and cost modeling.

---

## Pattern Seed 4 — Canary Query Drift Detection

**Problem it solves.** Embedding drift and quality degradation are silent — latency and error rate don't capture quality regression (MRR 0.85 → 0.62 with zero alerts).

**Method.** Run a fixed set of 20–50 queries with known correct answers weekly. Track per-canary and aggregate MRR as a time series. A >7% week-over-week drop triggers investigation. Store results for trend analysis.

**When to use.** All production retrieval systems. Any system where the corpus or query distribution evolves.

**When NOT to use.** Static, immutable corpora. Development / prototype systems.

**Ownership.** This skill owns canary query OPERATION (running it, alerting). `[[Evaluation Operations]]` owns canary query DESIGN (which queries, known answers, threshold). The handoff: Evaluation Ops designs the set; this skill runs it weekly and alerts on degradation. If no set exists, flag the dependency and proceed on a placeholder.

---

## Pattern Seed 11 — Hot/Cold Freshness Tiering

**Problem it solves.** Reindexing all documents every cycle is cost-prohibitive. The top ~20% of data is accessed ~80% of the time.

**Method.** Hot tier (top ~20% most-accessed, time-sensitive) reindexed weekly or on CDC events; cold tier (bottom ~80%, archival/stable) reindexed quarterly or only on major model upgrades. Achieves ≈82% reindexing-cost reduction. Sample-audit cold-tier recall quarterly — cold ≠ frozen quality.

**When to use.** Large indexes with skewed (Pareto) access patterns. Cost-sensitive reindexing. Frequent model updates.

**When NOT to use.** Small indexes (<1M vectors). Uniform access patterns. Real-time systems where all freshness is critical.

**Ownership.** Owned by this skill. Future `[[Precision & Storage Optimization]]` may consume tiering decisions to optimize quantization per tier.

---

## Pattern Seed 15 — Model Version Cohort Isolation

**Problem it solves.** Model upgrades must not create mixed-version indexes where vectors from different models are geometrically incomparable (Failure V25).

**Method.** Every vector carries a non-nullable `model_version`. ANN search is constrained to a single model-version cohort. Migrations move whole cohorts via blue-green; no query ever crosses geometries. The column also enables cross-version quality comparison ("did v2 improve recall on cohort X vs. v1?").

**When to use.** All vector indexes with any possibility of a model upgrade — from day one.

**When NOT to use.** Systems that will never change embedding models. Extremely short-lived indexes.

**Ownership.** Owned by this skill. Future `[[Monitoring & Alerting]]` extends it with per-version cohort quality dashboards.

---

## Pattern Seed 18 — Golden Signals with Search Semantics

**Problem it solves.** Standard service monitoring (latency, error rate, throughput) does not capture search-quality degradation (Law 8).

**Method.** Add search-specific signals alongside the standard golden signals: canary MRR (quality), cosine-distribution mean/variance (drift early warning), ef_search/nprobe inflation ratio (structural degradation), and freshness (last-indexed vs. now against a declared SLA). Each gets a baseline and an alert threshold.

**When to use.** All production search systems, before going live.

**When NOT to use.** Development / prototype systems.

**Ownership.** This skill owns the search-specific signals and their thresholds. Future `[[Monitoring & Alerting]]` integrates them into broader dashboards and alert routing.
