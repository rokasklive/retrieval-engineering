---
name: evaluation-operations
description: >
  Design and operate evaluation for a search/retrieval/RAG system — the judged
  set, metrics, and gates that prove changes actually help. Use whenever setting
  up, auditing, or debating how retrieval quality is measured: building a judged
  set, choosing NDCG vs MRR vs MAP vs ERR, deciding if Coverage@K matters,
  trusting GPT-4 as a judge, interpreting offline metrics, picking canary
  queries, decomposing aggregate NDCG into per-signal breakdowns, or knowing if
  a retrieval change really improved things. Trigger on any proposal to compare
  two retrieval configurations, ship on an offline metric alone, treat unjudged
  docs as irrelevant, or trust an eval that looks healthy while users complain.
  Owns judged-set construction (multi-paradigm pooling), metric selection,
  Coverage@K enforcement, before/after methodology, canary query design,
  LLM-as-Judge calibration, and offline-to-online validation pipeline. Does NOT
  operate canaries in production (Drift Management), design pipelines or
  chunking, or define business metrics.
---

# Evaluation Operations

Design and operate the evaluation program for a retrieval system **so that a change is provably better before it ships — not merely better on a number nobody trusts.** The output is a **design artifact**: a reasoned evaluation program — which queries to judge, how to build the judged set without bias, which metric answers the question being asked, what coverage makes those metrics trustworthy, and the gate sequence (offline filter → online A/B → ship) that separates "probably not worse" from "actually better." You design and run the evaluation; you don't design the pipeline it measures.

The single most important idea: *offline metrics are filters, not verdicts. They can tell you a change is probably worse; they cannot tell you it's better — only an online test can.* Most evaluation disasters are a team treating a filter as a verdict, reporting a metric without the coverage that makes it mean anything, or building the judged set with the very system it's supposed to fairly evaluate.

And the first idea, before any of that: *no eval baseline = flying blind.* "No runnable evaluation" is the root cause underneath nearly every production retrieval failure in the corpus. Before tuning a metric or debating a fusion method, ask whether a baseline even exists. If it doesn't, that is the work.

## What this skill owns (and what it doesn't)

**Owns:** query-set construction (PPS sampling from production logs, query-type taxonomy for evaluation); judged-set / judgment-pool construction from ≥2 retrieval paradigms with graded relevance; Coverage@K measurement, reporting, and threshold enforcement; evaluation baseline establishment as the prerequisite to any change; metric selection per task type (MRR / NDCG / MAP / ERR); per-signal decomposition by query type, signal, and domain; before/after comparison methodology, falsification conditions, ablation design; eval-gated architecture progression (Law 12 — add a layer only for a measured symptom); canary query *design* and baseline MRR (20–50 fixed queries with known answers); LLM-as-Judge calibration protocol against human gold sets; annotation selection-bias detection and mitigation; the offline-filter → A/B-gate → ship pipeline and offline-online correlation tracking.

**Does NOT own** (defer, name the owner, state a placeholder, flag for verification):
- *Operating* canaries in production, drift alerting, re-embedding triggers, runtime quality response → `[[Drift Management & Freshness]]`. This skill owns canary *design and baseline*; that skill owns canary *operation*.
- Pipeline stage count, whether hybrid/reranking is justified architecturally → `[[Retrieval Pipeline Design]]` (this skill says *what the eval must prove*; that skill decides *what to build*).
- Fusion method, routing, score combination → `[[Hybrid Search Architecture]]` (this skill provides the per-signal decomposition that fusion evaluation consumes).
- Chunk boundaries, size, overlap → `[[Chunking Strategy Design]]` (chunk quality is *measured* here, *designed* there).
- Business-metric definition (conversion, retention, revenue) — domain-specific, outside search evaluation.
- A/B-testing platform implementation (interleaving engine, sequential-test statistics) — this skill defines the *gate criteria*, not the experimentation infrastructure.
- Monitoring dashboards → `[[Monitoring & Alerting]]` (future).

When you hit a decision you don't own: (1) state it, (2) name the owning skill with a `[[wikilink]]`, (3) state the placeholder assumption you'll proceed with, (4) flag that it must be verified before implementation. Don't silently assume, and don't pretend to design what you don't own.

## The mindset (read before designing anything)

These are the design habits that separate an evaluation program you can trust from a green dashboard above a system users are abandoning. Internalize the reasoning, not just the rule. The cited, full-detail versions of the underlying laws are in `references/laws.md` — read it when you need the numbers, evidence, or exact violation symptoms.

1. **No eval baseline = flying blind — establish it first, before any change.** Without a pre-established baseline, every change is indistinguishable from noise, and "the demo works" silently becomes 8.4% confidence / 31.2% accuracy on real queries (Failure R15). Every model swap, index rebuild, or "small tweak" without a baseline is a gamble nobody is scoring. The very first deliverable for a system that has no evaluation is a runnable evaluation command and a recorded baseline — not a better metric, not a fancier judge. Symptom of violation: a retrieval system being changed with no command that returns a single quality number. (Law 12, Failure R3.)

2. **Offline metrics are filters, never verdicts.** Offline can tell you a change is *directionally wrong*; it cannot tell you it's *right*. The AUC inversion case is definitive: a 0.92-AUC model saved $1,840/mo while a 0.78-AUC model saved $2,320/mo — offline ranked them backwards. Use offline to *block regressions*, never to *ship improvements*. The flow is always offline-filter → A/B-gate → ship. Shipping on offline alone cost a documented 21% of business value. Symptom of violation: "NDCG went up 8%, let's ship." (Law 7, Failure EV9.)

3. **Coverage is the meta-metric — report it beside every number, no exceptions.** Coverage@K (the fraction of top-K results that actually have a human judgment) determines whether your other metrics mean anything. Below 0.50, NDCG is structurally biased against any retriever that finds documents your pool never judged — and those unjudged docs may be the best results. A bare "NDCG = 0.71" is not a result; "NDCG = 0.71 at Coverage@10 = 0.83" is. Below 0.50 → flag every metric as unreliable and fix coverage before drawing conclusions. Symptom of violation: a metric reported without its coverage. (Law 11.)

3.5 **Unjudged ≠ irrelevant.** The silent default in almost every eval harness — treat any document without a judgment as non-relevant — is exactly the bias that punishes retrievers finding things your pool missed. An unjudged document is *unknown*, not *bad*. Treating unknown as bad is how a better retriever scores worse. Pair this always with the coverage rule above.

4. **The judged set is a self-fulfilling prophecy — pool from ≥2 paradigms.** Whatever system builds the pool defines what counts as "relevant." A BM25-built pool labels what BM25 finds; a dense retriever then finds *unlabeled* documents that score as "not relevant," so dense "looks worse," so the pool is never updated — the prophecy reinforces itself. BEIR re-annotation moved dense retrievers up 10–30 percentile points once the documents lexical pooling missed were actually judged. Always pool from ≥2 retrieval paradigms (BM25 + dense at minimum) at depth N≈100. Symptom of violation: evaluating a dense retriever on a pool built entirely by BM25. (Law 11, Failure EV1.)

5. **Aggregate metrics mask catastrophic failures — decompose by signal.** "Good NDCG" routinely means "excellent on semantic queries, 0% recall on exact-identifier queries." A single blended number averages a triumph and a disaster into something that looks fine. Decompose every comparison by query type (exact, keyword, semantic, entity), by signal (precision, recall, MRR, coverage), and by domain. Aggregate green + any per-signal red (>20% below aggregate) = **REJECT**, not ship. Symptom of violation: a single aggregate NDCG standing in for a multi-class system. (Failure EV5.)

6. **Canary queries are the continuous pulse — between formal evals they're the only quality signal.** Latency, error rate, and throughput catch none of retrieval's positional degradation: MRR drifted 0.85 → 0.62 over six months with zero alerts, while a weekly canary would have caught it in week 2. Design 20–50 fixed queries with known correct answers, spanning *all* query types (not just easy ones), with a baseline MRR and a >7% week-over-week drop trigger. (You design and baseline them here; `[[Drift Management & Freshness]]` runs them.) Symptom of violation: a production retrieval system with no canary set. (Pattern Seed 23.)

7. **LLM-as-Judge is convenient but blind — calibrate it weekly or don't trust it.** An LLM judge shares the generator's knowledge boundaries, so it can't catch domain errors it doesn't know are errors; it's systematically lenient on plausible-but-wrong answers, prefers the first result (position bias), and inflates its own model's output (self-enhancement). Calibrate weekly against a human gold set, use a *different* judge model than the generator, randomize document order, and multi-sample-vote. Correlation with humans below ~0.85 → recalibrate or suspend it for decisions. Symptom of violation: an uncalibrated "faithfulness = 0.92" cited as fact. (Failure mode EV-judge.)

8. **Eval-gate complexity — add a layer only for a measured symptom (Law 12).** Every architectural layer (reranker, second index, query rewriter) must be justified by an *observed, measured* failure in the current system — not theoretical completeness. The progression is: establish baseline → observe a specific failure pattern → add the layer that targets it → measure its marginal contribution → keep only if it earned its latency. Never deploy with all layers from scratch. This is the evaluation discipline behind `[[Retrieval Pipeline Design]]`'s architecture decisions. Symptom of violation: approving a new layer with no failure pattern that names it.

## The design procedure

Work these stages in order — early stages produce the inputs later ones depend on, and choosing a metric before you know the task type, or trusting a metric before you know its coverage, is how evaluations mislead. Scale the depth to the problem: a 60-query catalog eval collapses several stages to a sentence, but *consciously visit each one*. `references/design-inputs.md` details what to extract for each input and why it changes the design — consult it when the brief is thin. The first question, always: **does a baseline exist?** If not, Stage A *is* the project.

### A — Establish the evaluation baseline
If no runnable evaluation exists, this comes before everything (DR1). Sample queries from production logs with PPS (probability-proportional-to-size) sampling so the judged set mirrors the real distribution — never curated demo queries (Failure R15). Classify them by type: exact identifier, short keyword, natural-language/semantic, entity lookup, typo-afflicted. Build a judged set of ≥50 queries representative of production. Pick metrics by task type (Stage C). Record the baseline: "NDCG@10 = X, Coverage@10 = Y, on [date]." This is the reference point every future comparison is measured against. **Output → Evaluation Baseline** (query-set provenance, type distribution, metric choice + rationale, baseline values, Coverage@K).

### B — Construct the judgment pool (avoid annotation selection bias)
Identify every active retrieval paradigm (BM25, dense, hybrid, fuzzy, SPLADE). Run the query set through each, pool the top-N (N≈100), and take the *union* per query — never let one paradigm define the pool alone (DR5–DR6, Law 11). Judge on a graded scale (0 = not relevant, 1 = marginal, 2 = relevant, 3 = highly relevant), not binary — graded relevance is what lets NDCG distinguish the match-quality differences hybrid search turns on (DR7). Prefer author-as-judge or domain experts over crowd-sourcing for consistency. Compute **Coverage@K per retrieval method**, not just overall (DR8) — a healthy overall coverage can hide a dense path at 0.30. Coverage@10 < 0.50 → flag, plan re-annotation, do not use the metrics for decisions (DR3). **Output → Judgment Pool Design** (contributing paradigms, pool depth, relevance scale, Coverage@K per method, bias assessment).

### C — Select and configure metrics
Match the metric to the task the user actually performs (full matrix in `references/decision-rules.md`):
- Single correct answer (known-item, factoid, canary) → **MRR** primary, P@1 secondary (DR10).
- Graded relevance, general/hybrid/RAG search → **NDCG@10** primary, ERR secondary (DR11).
- Binary relevance, full recall needed (e-discovery, patent, comprehensive research) → **MAP** primary, Recall@K secondary (DR12).
- User scans and stops (navigational, autocomplete, typeahead) → **ERR** primary (DR13).

Then define the **per-signal decomposition** plan (NDCG_exact, NDCG_keyword, NDCG_semantic, NDCG_entity, plus per-signal precision/recall/MRR/coverage) and the **falsification conditions** — the metric pattern that would *disprove* a claimed improvement (e.g., "any per-signal NDCG >20% below aggregate"). Coverage@K rides alongside every metric. **Output → Metric Configuration** (primary/secondary per task type, per-signal plan, falsification conditions).

### D — Design the canary query set (design only)
Select 20–50 queries with *known* correct answers from the judged set, spanning all query types — including the hard ones (DR-canary). Record each canary's known-correct document ID or passage. Establish baseline MRR per canary and in aggregate. Define the degradation trigger: >7% week-over-week MRR drop → investigate. Note explicitly that *operation* (the weekly run, the alerting) is owned by `[[Drift Management & Freshness]]` — you produce the set, the baseline, and the threshold. **Output → Canary Query Design** (query list, known answers, baseline MRR per-query and aggregate, degradation threshold, handoff note).

### E — Define the LLM-as-Judge calibration protocol (if an LLM judge is used)
Build a human gold set of 50–100 query-result pairs with human relevance judgments. Weekly: run the LLM judge over the gold set and correlate against the humans. Use a *different* judge model than the generator (kills self-enhancement bias), randomize document order (kills position bias), and multi-sample-vote 3–5 times with majority. Track the human-correlation trend over time; below ~0.85 → recalibrate or suspend the judge for decisions (DR15). Never let an uncalibrated LLM judge be the sole arbiter. If no LLM judge is in use, say so and skip — don't invent one. **Output → LLM-as-Judge Protocol** (gold-set size, cadence, judge-model choice, randomization, correlation threshold, miscalibration escalation).

### F — Define the offline-to-online validation pipeline
Wire the gate sequence (Law 7): **offline filter** (run the eval; NDCG drop >3% or any per-signal red → REJECT and iterate; no change → don't ship; improvement → proceed) → **online A/B test** on production traffic with business metrics → **ship only on a statistically significant business-metric improvement** (DR18–DR19). Track offline-online correlation across tests; below ~0.5, offline metrics aren't predictive — increase coverage, diversify pooling, recalibrate (DR17). If A/B testing is impossible (healthcare, legal, low traffic), say so and compensate: maximize offline rigor — ≥2 pooling paradigms, Coverage ≥0.90, a human gold set — and flag explicitly that offline alone cannot *guarantee* a production improvement (DR20). **Output → Offline-to-Online Validation Pipeline** (offline thresholds, A/B criteria, shipping gate, correlation tracking plan).

### G — Set the continuous evaluation schedule
Define the cadence: **weekly** canary MRR + coverage check; **monthly** full eval run with per-signal decomposition and baseline comparison; **quarterly** judgment-pool refresh and annotation-quality audit; **on every model change or index rebuild** a pre/post eval with a coverage report. Store results as a time series for trend analysis. Alert on: canary MRR drop >7%, coverage drop >10%, any per-signal metric >20% below aggregate. Add production failures to the eval set monthly — the eval set should get *harder* over time, not easier, or it decays into a demo (Failure R15). **Output → Continuous Evaluation Schedule** (cadence, trigger-based evals, alert thresholds, eval-set evolution plan).

## Decision gates (apply continuously)

These fire the moment the trigger appears in the user's proposal — gates, not gentle suggestions. Full rationale, the cited failure/law behind each, and the rest of DR1–DR20 are in `references/decision-rules.md`.

| Gate | When it fires | What you do |
|------|---------------|-------------|
| No baseline | system being changed with no runnable eval | stop; establish the baseline first — that's the work (DR1, R3) |
| Metric without coverage | NDCG/MRR reported, no Coverage@K | demand Coverage@K beside it; a bare metric is not a result (DR2) |
| Low coverage | Coverage@10 < 0.50 | flag every metric UNRELIABLE; re-pool/re-annotate before deciding (DR3, EV4) |
| Ship on offline | "offline metric improved, let's ship" | offline is a filter, not a gate; route to A/B before shipping (DR4, DR18, Law 7) |
| Lexical-only pool, dense retriever | dense judged against a BM25-built pool | REJECT; re-pool from ≥2 paradigms; dense suffers 5–10× annotation holes (DR5–DR6, EV1) |
| Binary relevance, hybrid system | 0/1 judgments for multi-path retrieval | upgrade to graded 0–3; binary can't capture match-quality differences (DR7) |
| Unjudged = irrelevant | precision@K counting unjudged docs as wrong | stop; unjudged is unknown, not non-relevant (DR9, Law 11) |
| Aggregate green, per-signal red | any per-signal metric >20% below aggregate | REJECT the change; a class has collapsed (DR14, EV5) |
| Uncalibrated LLM judge | LLM judge as sole arbiter, no human calibration | require weekly human gold-set calibration, different judge model (DR15, EV-judge) |
| Demo queries as evidence | quality claimed from curated/demo queries | REJECT; PPS-sample from production logs (DR16, R15) |
| Weak offline-online correlation | <0.5 across 3+ A/B tests | offline isn't predictive; raise coverage, diversify pooling (DR17) |
| No A/B feasible | live experimentation prohibited | maximize offline rigor (≥2 paradigms, Cov ≥0.90, gold set); flag the residual risk (DR20) |
| New layer, no symptom | architectural layer proposed for completeness | block until a measured failure pattern names it (Law 12) |

## Failure modes to catch

Watch for these patterns when designing or reviewing. Full symptom / cause / prevention / law for each is in `references/failure-modes.md`.

- **No eval baseline (R3)** — team can't answer "what's our quality?"; user complaints are the only degradation signal. → establish a runnable baseline first, minimum a weekly canary MRR.
- **Annotation selection bias (EV1)** — dense looks worse than BM25; re-annotation jumps it 10–30 percentile points. The "dense is bad" conclusion is a pool artifact. → diverse pooling from ≥2 paradigms, coverage per method, re-annotate holes.
- **Low coverage hiding unreliability (EV4)** — Coverage@10 < 0.50; metrics look stable while users complain; new methods score "worse" by finding unjudged docs. → report coverage beside every metric, pool at N≈100.
- **Offline-online gap (EV9)** — offline-improved changes fail A/B; AUC inversion ($1,840 vs $2,320/mo). → offline as filter only, A/B-gate every ship, track correlation.
- **Single-signal masking (EV5)** — aggregate NDCG fine while exact-ID recall is 0%. → decompose by query type/signal/domain; any signal >20% below aggregate = REJECT.
- **Demo-to-production gap (R15)** — perfect demos, 8.4% confidence / 31.2% accuracy on real queries. → PPS-sample production logs; never cite demo performance as quality.
- **Uncalibrated LLM judge (EV-judge)** — leniency on plausible-but-wrong, position bias, self-enhancement inflate the score. → weekly human calibration, different judge model, randomized order, multi-sample voting.

## Output: the design artifact

Produce these sections. Scale depth to the problem — a 60-query catalog eval needs a few tight sentences per section, not pages — but address every one, even if only to say "not applicable, because…". Saying *why* a section is N/A is itself a recorded design decision.

1. **Evaluation Baseline** — query-set provenance (PPS from logs), query-type distribution, metric choice + rationale, baseline values, Coverage@K, establishment date (Stage A)
2. **Judgment Pool Design** — contributing paradigms (≥2), pool depth, graded relevance scale, Coverage@K per method, annotation-bias assessment (Stage B)
3. **Metric Configuration** — primary/secondary metric per task type, per-signal decomposition plan, falsification conditions (Stage C)
4. **Canary Query Design** — 20–50 queries spanning all types, known answers, baseline MRR per-query and aggregate, >7% degradation threshold, operation handoff to `[[Drift Management & Freshness]]` (Stage D)
5. **LLM-as-Judge Calibration Protocol** — gold-set size, weekly cadence, different judge model, randomization, ≥0.85 correlation threshold, miscalibration escalation (Stage E; only if a judge is used)
6. **Offline-to-Online Validation Pipeline** — offline filter thresholds, A/B criteria, shipping gate, offline-online correlation tracking (Stage F)
7. **Continuous Evaluation Schedule** — weekly/monthly/quarterly cadence, trigger-based evals, alert thresholds, eval-set evolution (Stage G)
8. **Failure Mode Prevention Checklist** — which of EV1/EV4/EV5/EV9/R15/R3/EV-judge this design guards against, and how
9. **Coverage Monitoring Plan** — Coverage@K reporting, threshold warnings, re-annotation triggers, coverage trend monitoring
10. **Deferred Specialized Decisions** — what you don't own, cross-referenced with `[[wikilinks]]` + placeholder assumptions (e.g. canary operation, pipeline/fusion/chunking, business metrics, A/B platform)
11. **Known Risks** — likeliest failure modes for *this* program (thin coverage, no A/B feasibility, judge drift), mitigations, residual risk
12. **Minimal Next Experiment** — the single most informative next step (e.g. "judge the top-100 pooled docs for 50 PPS-sampled queries across 5 types and report Coverage@10 per method") — falsifiable, never "deploy and monitor"

Note the gaps the corpus doesn't fill, when relevant: **agent-specific evaluation** (retrieval whose consumer is an agent — success is downstream task completion, not NDCG; future `[[Agent-Facing Retrieval Design]]`), **multi-hop evaluation** (standard metrics miss multi-hop success; >15% of RAG queries are multi-hop), and **evaluation drift** (when judged sets go stale). Name the gap; don't pretend to fill it.

## Worked judgment calls

**"Set up evaluation for our new search system — 50k docs, ~1k queries/day."** Start at Stage A: there is no baseline, so that's the whole job. PPS-sample ≥50 queries from the real query logs (not demo queries — Failure R15), classify them by type, pool from BM25 + dense (≥2 paradigms, Stage B), judge on a graded 0–3 scale, and record "NDCG@10 = X, Coverage@10 = Y on [date]." NDCG@10 primary, MRR secondary. Design a 20–50 query canary set and an LLM-as-Judge calibration protocol if a judge is in play. Define the offline-filter → A/B → ship pipeline. The deliverable is a *runnable* eval command and a recorded baseline, not a metric opinion.

**"Our dense retriever scores worse than BM25 on NDCG — should we drop it? (Pool is BM25-only, Coverage@10 = 0.45.)"** Do not accept the conclusion. A BM25-built pool at 0.45 coverage is structurally rigged against dense (Failure EV1, Law 11): dense finds documents the pool never judged, and unjudged counts as non-relevant, so dense *must* look worse. Re-pool from BM25 + dense, re-annotate the top unjudged dense results, get Coverage@10 above 0.50, and re-measure (DR3, DR6). Dense routinely jumps 10–30 percentile points after this. Only then is the comparison real.

**"NDCG@10 improved 0.65 → 0.71 after a new embedding model — ship it?"** Decompose first (Stage C). The per-signal breakdown shows NDCG_semantic 0.82 (up) but NDCG_exact 0.31 (down from 0.58) — exact-identifier queries collapsed. Aggregate green, per-signal red, the red signal >20% below aggregate → **REJECT** (DR14, Failure EV5). The new model trades exact matching for semantic gains; that's a regression for any identifier-bearing query. Don't ship; investigate the exact-ID collapse.

**"Offline NDCG improved 8% with our new reranker — ship?"** Offline is a *filter*, not a *verdict* (Law 7, DR4). An 8% offline gain earns a ticket to the A/B test, not a release. Ship only if business metrics improve with statistical significance online. Cite the AUC inversion (a better offline model lost $480/mo of value) so the team feels *why*. If A/B is infeasible, flag the residual risk explicitly and maximize offline rigor instead (DR20) — don't pretend offline closed the gap.

**"We use GPT-4 to judge faithfulness — it says 0.92, is that reliable?"** Not yet (Failure EV-judge, DR15). An uncalibrated LLM judge is lenient on plausible-but-wrong answers, biased toward the first result, and inflates its own model's output. The 0.92 is unverified until you calibrate weekly against a human gold set, using a *different* judge model than the generator, with randomized document order and multi-sample voting. If human correlation is below ~0.85, the number can't gate decisions. Set up the protocol (Stage E) before trusting it.

**"We added a reranker because it's best practice."** Eval-gate it (Law 12). "Best practice" is not a measured symptom. What failure pattern in the *current* system does the reranker target — and does the baseline show it? Establish baseline → name the observed failure → add the reranker → measure its marginal NDCG contribution → keep it only if it earned its latency. Architecture follows evidence; the architecture decision itself is `[[Retrieval Pipeline Design]]`'s, but the eval discipline gating it is yours.
