# Decision rules — operational gates (DR1–DR21)

These are gates, not suggestions. Each has a rule and the trigger that fires it. Cite them by number in design artifacts. Grouped by concern.

## Drift detection

| # | Rule | Trigger |
|---|------|---------|
| DR1 | **No drift baselines → establish them before ANY other drift work.** Canary MRR, cosine distribution, and parameter baselines are the first deliverable; the delta from baseline is the only drift signal there is (Law 8). | Any production system without canary MRR + cosine + parameter baselines |
| DR2 | **Canary MRR >7% week-over-week drop → investigate immediately.** Route to the diagnosis workflow (Stage C). | Weekly canary-query run |
| DR3 | **Cosine distribution mean falling + variance rising → model-drift warning; correlate with canary MRR.** It's often the leading indicator — MRR may follow in 2–3 weeks. | Continuous cosine monitoring detects a shift |
| DR4 | **KL divergence of cosine distribution from baseline >0.1 → WARN; >0.2 → CRITICAL.** | Continuous monitoring |
| DR5 | **ef_search/nprobe ratio >1.5× baseline → WARN, investigate root cause.** The inflation is a diagnostic signal, not a tuning target (Law 4). | Continuous parameter monitoring |
| DR6 | **ef_search/nprobe ratio >2× baseline → CRITICAL, plan immediate REINDEX.** | Continuous parameter monitoring |

## Model versioning

| # | Rule | Trigger |
|---|------|---------|
| DR7 | **`model_version` must be a non-nullable schema constraint — not a convention.** From day one; retrofitting is expensive (Law 14). | Schema design, or any vector table lacking it |
| DR8 | **>1 `model_version` in one ANN index without cohort isolation → STOP; isolate cohorts and plan blue-green migration.** Mixed-version search returns neighbors from incompatible geometries (Failure V25). | Any query revealing mixed versions |
| DR9 | **Model upgrade with no pre-established quality baseline → REJECT the migration.** You can't tell if the new model is better without the old model's number. | Any proposed model upgrade |
| DR10 | **Model upgrade with no rollback procedure → REJECT the migration.** | Migration planning |

## Remediation

| # | Rule | Trigger |
|---|------|---------|
| DR11 | **Parameter inflation detected → diagnose root cause first; never permanently increase parameters to mask it.** Adopting the inflated value freezes the latency hit and lets the root cause keep degrading (Failure V5/V6). | DR5 or DR6 fires |
| DR12 | **IVFFlat centroid entropy approaching 0 (or per-centroid hit-rate skew >40%) → plan REINDEX within 1 week.** Centroid collapse is the strongest early structural signal. | IVFFlat centroid monitoring |
| DR13 | **recall@10 vs. exact k-NN gap >5% → index degradation confirmed; schedule REINDEX.** | Periodic recall comparison |
| DR14 | **HNSW index size >2× live rows → soft-delete accumulation; REINDEX.** | HNSW index-size monitoring |

## Migration

| # | Rule | Trigger |
|---|------|---------|
| DR15 | **All production model upgrades use blue-green index deployment — no exceptions.** Build → validate → pre-warm → atomic alias swap → monitor → decommission (Pattern Seed 3). | Any model upgrade |
| DR16 | **Pre-warm caches before the alias swap; load-test cold-start behavior.** A version change invalidates all cached computation (Failure V31). | Blue-green migration |
| DR17 | **Monitor the new index ≥48h before decommissioning the old one.** Keep instant rollback available throughout. | Post-swap monitoring period |
| DR18 | **Post-migration canary MRR >7% below pre-migration → instant rollback via alias swap.** | DR17 monitoring period |

## Freshness

| # | Rule | Trigger |
|---|------|---------|
| DR19 | **The index freshness SLA must be explicitly declared and monitored.** If you can't answer "how stale can results be?", you've accidentally chosen the engine's default (Pattern Seed 18). | Any production index |
| DR20 | **Skewed access → hot/cold tiering: hot tier (top ~20%) weekly or CDC-based, cold tier (bottom ~80%) quarterly.** A flat "reindex everything" schedule wastes ≈82% of reindexing cost (Pattern Seed 11). | Large index with Pareto-skewed access |
| DR21 | **Cold tier must be staleness-sampled quarterly.** Sample ~50 queries against cold docs; recall <95% of baseline → reindex sooner. Cold ≠ frozen quality — data drifts even without writes when the model or query patterns move. | Quarterly audit of any cold tier |

## Ownership boundary

| # | Rule | Trigger |
|---|------|---------|
| DR-scope | **A decision you don't own → state it, name the owning skill, state a placeholder assumption, flag it for verification.** Canary *design* and metric choice → `[[Evaluation Operations]]`; index hyperparameter tuning (M, efConstruction, the *correct* ef_search) → `[[Retrieval Pipeline Design]]`; chunking → `[[Chunking Strategy Design]]`; fusion → `[[Hybrid Search Architecture]]`; dashboards / per-tenant drift → `[[Monitoring & Alerting]]` (future); 1B+ runbooks → `[[Migration & Deployment Operations]]` (future). | Any out-of-scope decision surfaces during drift work |
