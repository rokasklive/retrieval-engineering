---
name: drift-management-and-freshness
description: >
  Detect, diagnose, and remediate production retrieval quality degradation over time: 
  embedding drift, ANN index decay, evaluation staleness, and index freshness as 
  data/models change. Use when an existing search/RAG system may be worsening: sliding quality, 
  MRR drops, creeping latency, stale results, cosine similarity shifts, model upgrades, 
  re-embedding decisions, blue-green index swaps, REINDEX cadence, cold-tier freshness, or drift 
  monitoring. Trigger even without “drift”: proposals to raise ef_search/nprobe, swap embedding models, 
  mix vectors from different models, schedule reindexing, or rely only on latency/error dashboards while 
  users report worse answers are in scope. Owns detection, diagnosis, freshness strategy, model-version 
  enforcement, blue-green migration, and REINDEX scheduling. Does not design eval methodology, 
  canary queries, index hyperparameter tuning, chunking, fusion, or monitoring dashboards.
---

# Drift Management & Freshness

Keep a production retrieval system from silently rotting. The output is a **design artifact**: an operational plan for detecting that quality is decaying, diagnosing *which* kind of decay, and remediating it — plus the freshness and migration machinery that keeps the index trustworthy as data grows and models change. You operate and maintain the system; you don't design the eval that scores it or the pipeline it runs on.

The single most important idea: *every retrieval system degrades, and standard monitoring sees none of it.* Latency, error rate, and throughput are all green while MRR slides 0.85 → 0.62 over six months and users quietly start getting worse answers. Drift is gradual erosion, not a crash — so the only question that matters is whether you detect it. Detection is a deliberate act; decay is automatic.

And the prerequisite, before any detection: *you cannot detect drift without a baseline.* The delta between a recorded baseline and today is the entire signal. If no baseline exists — no canary MRR, no cosine distribution, no parameter baseline — establishing one is the first job, not a better alert.

## What this skill owns (and what it doesn't)

**Owns:** drift baseline establishment at deploy time; the three complementary detection signals (canary-query MRR weekly, cosine-similarity distribution monitoring continuously, ef_search/nprobe parameter-inflation monitoring); the drift-type diagnosis workflow (model vs. index vs. evaluation drift); model-version enforcement (non-nullable `model_version` on every vector, cohort isolation in ANN search); blue-green index deployment for all model migrations; hot/cold freshness tiering; explicit freshness-SLA declaration and monitoring; periodic REINDEX scheduling; embedding-cache cold-start pre-warming; the recall-latency contract as an operational signal; canary-query *operation* (running the set, alerting on it).

**Does NOT own** (defer, name the owner, state a placeholder, flag for verification):
- Canary-query *design*, judged-set / metric methodology, baseline eval construction → `[[Evaluation Operations]]`. This skill *runs* the canary set and alerts on it; that skill *designs* it and sets the threshold.
- Index hyperparameter tuning (M, efConstruction, list count, the *correct* ef_search value) → `[[Retrieval Pipeline Design]]`. This skill monitors ef_search *inflation* as a symptom; it does not pick the right baseline value.
- Chunk boundaries, size, overlap → `[[Chunking Strategy Design]]`.
- Fusion method, routing, score combination → `[[Hybrid Search Architecture]]`.
- Monitoring dashboards, alert routing infrastructure, per-tenant drift isolation → `[[Monitoring & Alerting]]` (future).
- Detailed migration runbooks for 1B+ vectors, cost modeling → `[[Migration & Deployment Operations]]` (future). This skill provides the blue-green *pattern*; that skill extends it with large-scale runbooks.

When you hit a decision you don't own: (1) state it, (2) name the owning skill with a `[[wikilink]]`, (3) state the placeholder assumption you'll proceed with, (4) flag that it must be verified before implementation. Don't silently assume canary queries exist, and don't pretend to design what you don't own.

## The mindset (read before designing anything)

These are the operational habits that separate a system you can trust over years from one that demos well and quietly dies. Internalize the reasoning, not just the rule. The cited, full-detail versions of the underlying laws are in `references/laws.md` — read it when you need the numbers, evidence, or exact violation symptoms.

1. **Drift is inevitable; detection is optional.** Embedding drift (8–12% annual MRR loss), index structural degradation, and evaluation ground-truth staleness are universal — three drift *types* across three timescales. Systems that monitor drift survive; systems that don't degrade to uselessness in 6–12 months. The canonical failure is MRR 0.85 → 0.62 over six months with zero alerts, because standard service monitoring (latency, error rate, throughput) captures none of "the system is getting slightly worse at answering questions." Three drift types ⇒ three detection signals are mandatory, not optional. Symptom of violation: a live retrieval system with only standard service monitoring — no canary MRR, no cosine distribution, no parameter-inflation tracking. (Law 8.)

2. **Parameter inflation is a symptom, not a solution.** When ef_search climbs 40 → 80 → 120 → 200 to "keep recall up," the index is dying — it doesn't need more search effort, it needs repair. Raising parameters maintains recall *temporarily* while permanently inflating latency, and the root cause (stale centroids, soft-delete bloat, graph degradation) worsens underneath. The escalation *is* the diagnostic signal: track the ratio vs. baseline — >1.5× → WARN and investigate; >2× → CRITICAL, plan an immediate REINDEX. Never adopt the inflated value as the "new normal." Symptom of violation: a permanently elevated ef_search/nprobe with no investigation into why. (Law 4.)

3. **Every vector must know its parent.** A vector without `model_version` metadata is an orphan — you can't diagnose its quality or migrate it safely. Vectors from different embedding models live in different geometry spaces; cosine similarity between a `text-embedding-3-small` vector and a `bge-large-en` vector is geometrically meaningless even if both are 768-dim. A mixed-version ANN index returns neighbors from *both* spaces — half the results are garbage, silently. `model_version` must be a non-nullable schema constraint from day one; retrofitting is expensive. ANN search must be constrained to a single model-version cohort. Symptom of violation: `SELECT model_version, COUNT(*) FROM vectors` returns >1 version in the same index without cohort isolation. (Law 14, Failure V25.)

4. **Blue-green is the only safe migration pattern.** "Just rebuild the index" means downtime, mixed versions, or no rollback. Blue-green: build the new index in parallel → validate against canary queries and the full eval set → pre-warm caches → atomic alias swap → monitor 48h → decommission the old index. If quality degrades, swap the alias back — instant rollback. A model upgrade with no rollback step, or one that decommissions the old index before the validation window closes, is not a migration plan; it's a gamble. Symptom of violation: a model-upgrade procedure without a rollback step. (Pattern Seed 3.)

5. **Not all data ages equally.** Reindexing everything on every freshness cycle is wasteful — for a Pareto-skewed corpus, the top ~20% most-accessed, time-sensitive data needs weekly (or CDC-driven) reindexing while the bottom ~80% archival data can be reindexed quarterly or only on major model upgrades. That hot/cold split is the single highest-leverage cost optimization (≈82% reindexing-cost reduction). But cold ≠ immortal: sample-audit the cold tier's recall quarterly, because data can drift even without writes when the model or query patterns change. Symptom of violation: a flat "reindex everything weekly" schedule for a corpus with skewed access. (Pattern Seed 11.)

6. **Freshness is a choice, not an accident.** Every index has a consistency level; if you didn't choose it, you accidentally chose the default. PostgreSQL: ACID-immediate. Elasticsearch: ~1s refresh. CDC-based: at-least-once with propagation delay. Materialized views: batch, silently stale. Declare the freshness SLA explicitly — "new documents are searchable within X seconds" — then monitor actual freshness and alert when it's violated. The tell that you've drifted into an accidental default: you can't answer "how stale can our search results be?" Symptom of violation: an undeclared, unmonitored freshness level. (Pattern Seed 18.)

7. **Cold caches kill migrations.** Embedding result caches are per-version — a model change invalidates *all* cached computation, so the new index starts cold and every query hits the full path until it warms. The result is a latency spike right after the alias swap that gradually heals over hours. Pre-warm during blue-green by running representative queries against the new index *before* the swap; test cold-start behavior under load; budget 1–4 hours of warm-up depending on query diversity. Symptom of violation: latency spikes immediately after migration with gradual recovery — no pre-warming was done. (Failure V31.)

## The operational procedure

Work these stages in order — you can't detect drift without baselines (Stage A), can't diagnose without the three signals (Stage B), and can't remediate without a diagnosis (Stage C). Scale the depth to the problem, but *consciously visit each stage*. `references/design-inputs.md` details what to extract for each input and why it changes the plan — consult it when the brief is thin. The first question, always: **do baselines exist?** If not, Stage A *is* the project (DR1).

### A — Establish drift baselines
You cannot detect drift without them. Record, at deploy time (or now, if the system is already live and has none): **canary MRR** — per-canary and aggregate, using the set designed by `[[Evaluation Operations]]`; **cosine distribution** — mean and variance of query-to-top-10 cosine similarity over a representative window; **parameter baseline** — current ef_search / nprobe; **model version** — verify the `model_version` column exists, is non-nullable, and is populated; **freshness** — current freshness SLA (last-indexed timestamp vs. now). Write it down: "Baseline: canary MRR = X, cosine mean = Y / var = Z, ef_search = W, nprobe = V, model = M, freshness = T, on [date]." **Output → Drift Baselines.**

### B — Deploy the three detection signals
One signal has blind spots; three triangulate. **Canary MRR (weekly):** run the canary set, track per-canary and aggregate trend as a time series, alert on >7% week-over-week drop (DR2). **Cosine distribution (continuous, zero-ground-truth):** track mean and variance; a falling mean with rising variance is distribution shift, often *before* users notice; KL divergence from baseline >0.1 → WARN, >0.2 → CRITICAL (DR3–DR4). **Parameter inflation (continuous):** track the ef_search/nprobe ratio vs. baseline — >1.5× WARN, >2× CRITICAL (DR5–DR6) — plus index-type-specific structural signals: centroid entropy and per-centroid hit-rate skew for IVFFlat, index size vs. live-row count for HNSW. **Output → Three-Signal Detection Deployment.**

### C — Diagnose when a signal fires
Don't remediate blind — classify the drift type first, because the cure differs. Use the signal *combination*: canary MRR dropping + cosine stable → evaluation drift (stale canary set) or query-distribution shift; canary MRR dropping + cosine shifting → model/embedding drift; parameter inflation rising + recall stable → index structural degradation; index size ≫ live rows → soft-delete accumulation (HNSW); centroid entropy → 0 with hit-rate skew >40% → centroid collapse (IVFFlat). Confirm the root cause (e.g., recall@10 vs. exact k-NN; >5% gap confirms an index problem; `model_version` cohort analysis confirms model drift), then map to a remediation: model drift → blue-green migration (Stage E); index degradation → REINDEX (Stage F); query shift → no urgent action, refresh the eval set at next cadence (flag to `[[Evaluation Operations]]`); centroid collapse → REINDEX sooner or migrate IVFFlat→HNSW. Execute → verify recovery against baseline → document. **Output → Drift Diagnosis Workflow.**

### D — Design the freshness strategy
Match freshness to data velocity, then tier for cost. Classify by requirement: real-time (<1s, CDC / transactional index), near-real-time (1–5s, ES-style refresh), batch (1–60 min), delayed (>1h, materialized views). Apply hot/cold tiering when access is skewed: hot (top ~20%, time-sensitive) reindexed weekly or on CDC events; cold (bottom ~80%, archival) quarterly or on major model upgrades (≈82% cost reduction). Declare the SLA explicitly — "new documents searchable within X seconds" — monitor last-indexed-vs-now and stale-document count, and alert on SLA violation. Plan cold-tier staleness sampling (Stage F / DR21). **Output → Freshness Strategy.**

### E — Define the blue-green model migration
Zero-downtime, instant-rollback model swaps. **Pre-upgrade:** record the pre-upgrade canary MRR baseline. **Build:** construct the new index with the new model in parallel — no user impact. **Validate:** run canary + full eval set against it, compare to the pre-upgrade baseline. **Pre-warm:** run representative queries to warm the new index's embedding cache (Failure V31). **Swap:** atomic alias swap. **Monitor:** track canary MRR for ≥48h vs. baseline. **Rollback decision:** >7% MRR drop or user complaints → swap the alias back (instant). **Decommission:** only after the 48h window. Populate `model_version` distinctly for old and new cohorts. **Document** the quality delta and lessons. **Output → Blue-Green Migration Procedure.**

### F — Schedule periodic REINDEX
REINDEX is the only known cure for centroid collapse and soft-delete bloat — schedule it; don't wait for a crisis. Set cadence from insert velocity, query-shift rate, observed parameter-inflation ratio, and index type (IVFFlat degrades faster than HNSW under writes): weekly for the hot tier, quarterly for the cold tier, and on every model upgrade. Automate into maintenance windows where possible. Measure effectiveness: compare pre- vs. post-REINDEX canary MRR and confirm parameters return to baseline. Spot-check cold-tier recall quarterly (DR21) — sample ~50 queries; recall >95% of baseline = healthy, >10% drop = reindex sooner. Don't skip REINDEX because "things seem fine" — degradation is silent. **Output → REINDEX Schedule.**

## Decision gates (apply continuously)

These fire the moment the trigger appears — gates, not gentle suggestions. Full rationale and the rest of DR1–DR21 are in `references/decision-rules.md`.

| Gate | When it fires | What you do |
|------|---------------|-------------|
| No baseline | drift work requested, no canary/cosine/parameter baseline | stop; establish baselines first — that's the work (DR1) |
| Canary drop | canary MRR >7% week-over-week | investigate immediately; go to diagnosis (DR2) |
| Cosine shift | mean falling + variance rising, or KL >0.1 | WARN; correlate with canary MRR; >0.2 → CRITICAL (DR3–DR4) |
| Parameter inflation | ef_search/nprobe >1.5× baseline | WARN, diagnose root cause — never adopt as new normal (DR5, DR11) |
| Severe inflation | ef_search/nprobe >2× baseline | CRITICAL; plan immediate REINDEX (DR6) |
| Nullable version | `model_version` nullable or absent | reject; make it a non-nullable schema constraint (DR7) |
| Mixed-version index | >1 `model_version` in one ANN index, no cohort isolation | STOP; isolate cohorts, plan blue-green (DR8, V25) |
| Migration, no baseline | model upgrade with no pre-upgrade quality baseline | reject the migration until a baseline exists (DR9) |
| Migration, no rollback | model upgrade with no rollback step | reject; require blue-green with alias swap-back (DR10, DR15) |
| Inflation as fix | "let's just make ef_search=120 the default" | reject; diagnose + REINDEX, return to baseline (DR11) |
| Centroid collapse | IVFFlat entropy → 0, hit-rate skew >40% | REINDEX within a week (DR12) |
| Recall gap | recall@10 vs. exact k-NN >5% | index degradation confirmed; schedule REINDEX (DR13) |
| Soft-delete bloat | HNSW index size >2× live rows | REINDEX (DR14) |
| No pre-warm | alias swap with no cache pre-warming | pre-warm + load-test cold start first (DR16, V31) |
| Early decommission | old index dropped before 48h validation | block; monitor ≥48h before decommission (DR17) |
| Undeclared freshness | can't answer "how stale can results be?" | declare + monitor a freshness SLA (DR19) |
| Flat reindex | "reindex everything weekly" on skewed access | apply hot/cold tiering (DR20) |
| Out of scope | asked to design canaries / tune M / pick metrics | defer to the owning skill, state placeholder, flag |

## Failure modes to catch

Watch for these. Full symptom / cause / prevention / law for each is in `references/failure-modes.md`.

- **Silent embedding drift (V7)** — MRR 0.85 → 0.62 over six months, zero alerts; users notice before monitoring does. → three detection signals, baselines at deploy time, continuous not episodic.
- **Mixed-version index (V25)** — post-upgrade results go random; old and new vectors share one ANN index, half the neighbors from the wrong geometry. → non-nullable `model_version`, cohort-isolated search, blue-green.
- **Recall-latency contract violation (V6)** — recall held but latency crept up for months (or vice-versa), no standard alert fired. → monitor the recall-latency ratio; treat parameter inflation as a signal.
- **Centroid collapse masked by nprobe inflation (V5)** — nprobe 5×'d over eight months to hold recall; latency permanently up; root cause untouched. → centroid-entropy monitoring, REINDEX not inflation.
- **Embedding cache cold-start (V31)** — latency spikes right after a model swap, heals over hours. → pre-warm caches before the alias swap, load-test cold start.

## Output: the design artifact

Produce these sections. Scale depth to the problem — a small index needs a few tight sentences per section, not pages — but address every one, even if only to say "not applicable, because…". Saying *why* a section is N/A is itself a recorded decision.

1. **Drift Baselines** — canary MRR (per-query + aggregate), cosine distribution (mean/var), ef_search/nprobe, model version, freshness SLA, date (Stage A)
2. **Three-Signal Detection Deployment** — canary MRR schedule, cosine monitoring setup, parameter-inflation thresholds, structural signals per index type, alert escalation (Stage B)
3. **Drift Diagnosis Workflow** — drift-type classification table, root-cause checklist, remediation decision tree (Stage C)
4. **Freshness Strategy** — velocity classification, hot/cold tiering plan, freshness SLA, monitoring + alert thresholds (Stage D)
5. **Blue-Green Migration Procedure** — pre-upgrade checklist, validation criteria, pre-warm step, rollback trigger, 48h monitoring, decommission (Stage E)
6. **REINDEX Schedule** — hot/cold cadence, trigger-based conditions, pre/post validation, cold-tier sampling, automation (Stage F)
7. **Failure-Mode Prevention Checklist** — which of V7/V25/V6/V5/V31 this plan guards against, and how
8. **Model-Version Schema Design** — `model_version` column spec (non-nullable), cohort-isolation query pattern, migration-state tracking
9. **Cache Cold-Start Mitigation Plan** — pre-warming procedure, load-test parameters, warm-up budget
10. **Deferred Specialized Decisions** — what you don't own, cross-referenced with `[[wikilinks]]` + placeholder assumptions (canary *design*, hyperparameter tuning, dashboards, 1B+ runbooks)
11. **Known Risks** — likeliest failure modes for *this* system, mitigations, residual risk
12. **Minimal Next Action** — the single most informative next step (e.g. "add a non-nullable `model_version` column and run `SELECT model_version, COUNT(*)` to confirm no mixed cohort exists"), falsifiable, never "deploy and monitor"

Note the gaps the corpus doesn't fill, when relevant: **automated remediation** (self-healing indexes that detect and reindex automatically — not yet harvested), **per-tenant drift** in multi-tenant systems (→ future `[[Monitoring & Alerting]]`), and **evaluation drift detection** (when ground truth goes stale — bridge concept; name it, don't pretend to solve it). Name the gap; don't invent a method.

## Worked judgment calls

**"Set up drift monitoring for our production search — 5M vectors, HNSW, ~10k queries/day."** Start at Stage A: there are almost certainly no baselines, so that's the spine of the job. Confirm the canary set exists (if not, flag the dependency to `[[Evaluation Operations]]` — design is theirs) and record its baseline MRR. Record cosine mean/var and current ef_search. Add a non-nullable `model_version` column now and verify a single cohort. Then Stage B: weekly canary run with a >7% trigger, continuous cosine monitoring (KL >0.1 WARN), parameter-inflation tracking vs. the recorded ef_search, plus HNSW index-size-vs-live-rows. Declare a freshness SLA ("searchable within 60s"), set REINDEX cadence (monthly hot / quarterly cold if access is skewed), and document the blue-green procedure for the next model upgrade. The deliverable is a *running* detection plan with recorded baselines — not "we'll add monitoring later," which is exactly how V7 happens.

**"ef_search has crept from 40 to 120 over six months; recall's fine but P99 went 50ms → 180ms. Just make 120 the default?"** No (DR11). A 3× inflation is a *diagnostic signal*, not a new baseline — the index is degrading and you're paying latency to hide it (Failure V5/V6). Diagnose: HNSW soft-delete accumulation (index size vs. live rows), recall@10 vs. exact k-NN (>5% gap = confirmed), graph degradation from continuous writes. Root cause is almost certainly writes without REINDEX. Remediate: REINDEX, confirm recall returns at ef_search=40, return ef_search to 40. Adopting 120 freezes the latency hit *and* lets the root cause keep degrading.

**"Upgrading from BGE-large-en to BGE-M3, 10M vectors — migrate without downtime?"** Blue-green (DR15, Stage E): build the BGE-M3 index in parallel, populate `model_version` distinctly ('bge-large-en-v1.5' vs 'bge-m3-v1'), validate against canaries + full eval vs. the pre-upgrade baseline, pre-warm caches (Failure V31), atomic alias swap, monitor 48h, decommission only after. Rollback trigger: >7% MRR drop → swap back. Two flags: canary *design* must be confirmed with `[[Evaluation Operations]]` (placeholder: assume the existing set is valid); detailed 10M-vector runbooks are deferred to `[[Migration & Deployment Operations]]` (future). The thing to never do: run both models in one ANN index — that's Failure V25.

**"Cosine mean dropped 0.82 → 0.71, variance 0.02 → 0.08, but canary MRR is steady. Problem?"** Investigate, don't panic. Cosine shift without an MRR drop has three live hypotheses: the embedding provider pushed a silent update (geometry moved, quality didn't), the query distribution shifted, or this is the *leading indicator* and MRR degradation follows in 2–3 weeks. Action: tighten the canary cadence and watch. If MRR drops next week, the cosine shift was the early warning — exactly why the zero-ground-truth signal exists. If MRR holds for 3+ weeks, recalibrate the cosine baseline to the new distribution. The one wrong move is ignoring it.

**"Hot/cold tiering's deployed — 200k hot (weekly), 800k cold (quarterly). How do we know the cold tier hasn't gone stale?"** Cold-tier staleness sampling (DR21): sample ~50 queries that hit cold documents, measure recall vs. the baseline recorded at tiering time. >95% of baseline = healthy; >10% drop = the cold data drifted and needs reindexing sooner. Monitor cosine distribution on the cold tier separately from hot. The trap is assuming "archival = frozen quality" — data drifts even without writes when the model or query patterns move, so cold needs its own baseline and its own quarterly audit.

**"Can you design our canary query set and tune our HNSW M/efConstruction while you're at it?"** Out of scope — and saying so cleanly is part of the job (DR-scope). Canary *design* (which queries, known answers, threshold) is `[[Evaluation Operations]]`; index hyperparameter tuning (M, efConstruction, the *right* ef_search value) is `[[Retrieval Pipeline Design]]`. State the placeholder you'll proceed on ("assume a 30-query canary set with baseline MRR exists; assume ef_search=40 is the tuned baseline"), flag that both must be verified with the owning skills before implementation, and then do what *is* yours: operate the canary set, monitor ef_search *inflation* against that assumed baseline, and enforce `model_version`.
