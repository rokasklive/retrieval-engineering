---
name: hybrid-search-architecture
description: >
  Design, tune, and diagnose multi-path retrieval that fuses lexical (BM25),
  dense, sparse, and fuzzy retrievers — choosing the fusion method (RRF vs
  weighted RRF vs score-level vs CC), routing queries to paths, placing metadata
  filters, and exposing confidence to agents. Use whenever combining retrievers:
  fusing BM25 and vector results, choosing RRF or weighted scores, diagnosing
  hybrid worse than a single path, adding SPLADE as a third path, handling
  Weaviate/Elasticsearch hybrid breaking after upgrades, or surfacing confidence
  to an LLM downstream. Trigger on any proposal to weight, blend, normalize, or
  rank-combine results from multiple retrievers, or diagnose a hybrid system
  underperforming a single path. Owns fusion method selection, query-adaptive
  routing, complementarity measurement, metadata filter placement, per-path
  health monitoring, and agent-facing confidence signals. Does NOT decide if
  hybrid is needed (Retrieval Pipeline Design), pick rerankers, design chunks,
  or build the eval program.
---

# Hybrid Search Architecture

Design multi-path retrieval that fuses lexical, dense, sparse, and fuzzy retrievers **so the combination is provably better than its best single path** — never worse. The output is a **design artifact**: a reasoned proposal for which paths to run, how to route queries to them, how to fuse their results, where to filter, and how to verify the whole thing didn't make retrieval worse. You design the fusion and routing; you don't write the retrievers or the index.

The single most important idea: *more paths is not better — more **complementary** paths, fused on ranks not raw scores, verified against the best single path, is better.* Most hybrid-search failures are a violation of one of those three clauses.

## What this skill owns (and what it doesn't)

**Owns:** fusion method selection (RRF → weighted RRF → TRF → score-level → CC) and the evidence required to escalate past the RRF default; query-adaptive routing across paths; how many paths are justified and the complementarity evidence required to add one; per-path independent measurement and the fused-≥-best-single-path checkpoint; score-incompatibility diagnosis; metadata-filter placement relative to fusion and reranking; pinning fusion/search parameters against silent vendor-default drift; SQL-native RRF; and agent-facing confidence signals derived from fusion (score gaps, source tagging).

**Does NOT own** (defer, name the owner, state a placeholder, flag for verification):
- *Whether hybrid is justified at all*, stage count, and the reranker's role in the pipeline → `[[Retrieval Pipeline Design]]`. This skill owns the fusion that **feeds** the reranker, not the reranker.
- Chunk boundaries, size, overlap, strategy → `[[Chunking Strategy Design]]`
- Judged-set construction, metric methodology, offline-online correlation → `[[Evaluation Operations]]` (this skill says *what to measure*; that skill provides the *how*)
- Drift monitoring, re-embedding triggers, runtime path-health response → `[[Drift Management & Freshness]]`
- Individual retriever implementation — BM25 k1/b tuning, dense model choice, SPLADE training, index internals
- Agent-level interpretation of confidence (reasoning over it) → `[[Agent-Facing Retrieval Design]]` (future); this skill produces the *signal*, not its interpretation.

When you hit a decision you don't own: (1) state it, (2) name the owning skill with a `[[wikilink]]`, (3) state the placeholder assumption you'll proceed with, (4) flag that it must be verified before implementation. Don't silently assume, and don't pretend to design what you don't own.

## The mindset (read before designing anything)

These are the design habits that separate a robust hybrid system from one that's slower *and* worse than a single retriever. Internalize the reasoning, not just the rule. The cited, full-detail versions of the underlying laws are in `references/laws.md` — read it when you need the numbers, evidence, or exact violation symptoms.

1. **Ranks over scores — default to RRF (k=60).** BM25 scores (unbounded, long-tailed), cosine similarities ([-1,1], bunched near 0), and neural logits (arbitrary) are *inherently incomparable*. Normalization can align them today and break silently tomorrow when a model upgrade or corpus shift moves a distribution. RRF operates on ranks, which are always comparable — that's why it's the zero-config default and why vendors converged on k=60. Reach for score-level fusion only when distributions are proven stable, calibrated, *and* monitored. (Law 10.)

2. **Measure before you fuse — no blind path addition.** Never add a path without first measuring its standalone precision@K, its complementarity with what's already there, and its latency cost. "More paths = better" is the most dangerous heuristic in hybrid search: a path that contributes noise without unique recall takes pool slots from a strong path's good candidates and drags fused quality *below* the best single path.

3. **Complementarity over standalone quality — weak-but-orthogonal beats strong-but-redundant.** A path's value is the *unique relevant documents it contributes that others miss*, not its solo NDCG. Lexical and dense are complementary by construction (vocabulary mismatch vs. exact-identifier blindness — they fail in opposite directions), which is why that pairing is the default. A third path must *earn* its place with measured complementarity, never a completeness argument. (Law 6.)

4. **Verify the fusion quality floor — fused ≥ best single path.** This is the first diagnostic for *any* hybrid quality complaint. If fused quality isn't ≥ max(single-path quality) on a representative judged set, a path is injecting noise (Law 5) — find it, disable it, re-measure. Measured, never assumed. Break it down per query class: a path can help one class and poison another, which is a routing fix, not a removal.

5. **Route by query type — not every query needs every path.** Running the full pipeline on every query buys latency for nothing. Exact identifiers → BM25 only (and *never* dense-only — Failure V15). Typos → fuzzy + BM25. Paraphrastic → dense (+ reranker). Mixed/ambiguous → full hybrid (the catch-all). Route *before* fusion.

6. **Pin everything — never trust vendor defaults.** fusionType, k, weights, ef, similarity metric — all of these change silently across vendor versions (Weaviate v1.24 flipped its default from rankedFusion to relativeScoreFusion with no warning, silently degrading results — Failure V27). Pin every parameter explicitly and re-audit after any upgrade.

7. **Filter after fusion, before the reranker.** Fuse all paths first so no single retriever's results are pre-filtered away, then filter to shrink the reranker's bill. Post-filter is the default; switch to pre-filter only when post-filter recall degradation is measured and unacceptable and the filter is deterministic and highly selective (>90%). Filtering one path's output *before* fusion silently drops the complementary path's unique finds (Failure V13).

8. **Two paths default, three only with evidence.** Lexical + dense is the production default. A third path (SPLADE, fuzzy, cross-lingual) must show measured unique recall beyond the first two; if removing it doesn't move NDCG, it's pure latency and operational debt.

9. **Score-ceiling aware — don't hand raw scores to agents.** A cosine of 0.92 and a BM25 of 18.3 share no meaning, yet an agent will happily "trust" the bigger number. Expose *relative* confidence instead: the normalized top-gap `c(q)=(s₀−s₁)/s₀`, per-result source tags ("found via exact match" vs. "via semantic similarity"), and uncertainty flags. (Pattern Seed 16; AgentIR 2026.)

## The design procedure

Work these stages in order — early stages produce the inputs later ones depend on, and jumping to a fusion method before you know the query classes and per-path numbers is the most common way hybrid designs go wrong. Scale the depth to the problem: a two-path catalog search collapses several stages to a sentence, but *consciously visit each one*. `references/design-inputs.md` details what to extract for each input and why it changes the design — consult it when the brief is thin.

### A — Characterize query classes
Categorize the expected queries and estimate each one's production frequency and criticality: exact identifiers (SKUs, error codes, CLI flags, function names); short keywords (1–3 words, ambiguous); natural-language / paraphrastic / cross-lingual; entity lookups; typo-prone; mixed-intent; multi-hop. Map each class to its optimal path. This taxonomy drives path selection *and* routing — almost every later decision keys off it. **Output → Query Class Taxonomy.**

### B — Select retrieval paths
Choose paths from the query classes (full matrix in `references/decision-rules.md`):
- Exact identifiers / short keywords / rare terms present → a **lexical (BM25)** leg is non-negotiable (DR1); dense is structurally blind to identifiers it never saw in training.
- Paraphrastic / natural-language / cross-lingual present → lexical-only is insufficient; add a **dense** leg (DR2).
- Typos expected → add a **fuzzy/trigram** leg (DR3).
- Semantic signal needed but no GPU, inverted-index infra exists → consider **learned sparse (SPLADE)**.

Default to two-path (lexical + dense). A third path requires measured complementarity (Stage D, DR4), not a completeness argument. **Output → Retrieval Path Selection Rationale** (paths chosen, rules applied, paths rejected and why).

### C — Measure each path independently
Before *any* fusion, satisfy Invariant 7 (every path independently measurable): disable all other paths, query each retriever alone, and record standalone precision@K and recall@K **per query class**, plus per-path latency (P50/P95/P99). Identify which path fails on which class. Without this, complementarity and the fusion floor are acts of faith. **Output → Per-Path Quality Baseline** (precision@K × query-class matrix). *If paths aren't independently measurable yet, that is the first thing to fix — not fusion tuning.*

### D — Measure complementarity (RoC)
For ~100 representative queries spanning all classes, run each path alone and for each path pair compute overlap |A∩B|, A's unique |A−B|, B's unique |B−A|, and complementarity |B−A|/|A∪B|. If |A∩B| exceeds ~50% of |A∪B|, the paths are mostly redundant. Weight path value by complementarity, not standalone NDCG. This stage is what lets you *justify or kill* a proposed third path. **Output → Complementarity Analysis** (RoC matrix, per-class breakdown).

### E — Choose the fusion method
Walk the escalation ladder; do **not** start above rung 1, and never blend raw BM25 with cosine (full rationale + rules DR6–DR9, DR21 in `references/decision-rules.md`):
1. **RRF (k=60)** — zero-config, distribution-shift robust. The default for every new deployment.
2. **Weighted RRF** — when one path is measurably and consistently stronger (>5% NDCG across classes), weight it higher. Per-source weights matter more than tuning k (DR24).
3. **TRF (token-level RRF)** — only when accuracy is the binding constraint *and* the ~+8.1% nDCG justifies its ~128× storage cost (DR8).
4. **Score-level fusion** — only with calibrated, stable, monitored distributions and a use case where the 2–5% gain over RRF matters (DR9).
- **CC (convex combination)** — when ~100+ labeled queries exist, evaluate CC: it consistently beats RRF and is sample-efficient (Bruch et al. 2023, DR21). RRF stays the zero-shot default; CC is the labeled-data upgrade.

**Output → Fusion Method Rationale** (ladder rung taken, evidence for any escalation).

### F — Design query-adaptive routing
Route *before* fusion to skip paths a query doesn't need (DR14–DR17): exact identifier → BM25 only; apparent typos → fuzzy + BM25; paraphrastic, no identifiers → dense (+ reranker); mixed/ambiguous → full hybrid. Fallback when intent classification is unavailable: query length (<4 tokens → lexical; 4–20 → hybrid; >20 → semantic-priority). Routing must itself be evaluated — what fraction lands on the optimal path combo, and how is misrouting detected? **Output → Routing Strategy** (per-class rules, misrouting detection plan).

### G — Place metadata filtering
Default to **post-filter after fusion, before the reranker** (DR18, DR20) — preserves every path's recall and trims the reranker's cost. Switch to **pre-filter** only when post-filter recall degradation is measured and unacceptable, for deterministic filters with >90% selectivity (DR19). Upgrade to **in-algorithm** filtered ANN when the DB supports it (pgvector HNSW WHERE, Elasticsearch filter context). Never filter one path's output before fusion. **Output → Metadata Filtering Design** (placement, selectivity thresholds, rationale for any deviation).

### H — Verify the fusion quality floor
The checkpoint that gates the whole design: **fused quality ≥ max(single-path quality)** on the representative judged set (DR12). If violated, identify the noisy path, disable it, re-measure; if quality recovers, that path lacked complementarity — remove it or route around it. Check per query class — a path harmful to one class but helpful to another is a routing fix, not a removal. After-reranking check (DR22): verify the fusion gain *survives reranking and truncation*, not just the retrieval layer — gains routinely shrink or vanish downstream (Medrano et al. 2026); if after-reranking NDCG ≤ single-query baseline, fusion isn't earning its latency. **Output → Fusion Quality Verification** (fused vs. best-single NDCG, per-class, paths removed/down-weighted).

### I — Plan production hardening
Every design includes: **pinned** fusion params (fusionType, k, weights, similarity, ef) with a post-upgrade audit step (Failure V27); per-path latency budget and routing overhead; per-path weekly health monitoring (NDCG, recall@K, latency) to catch path degradation; weekly fused-vs-best-single monitoring with an alert if fused drops below single; `model_version` on every vector (handoff → `[[Drift Management & Freshness]]`); and, if PostgreSQL, the SQL-native RRF pattern (`UNION ALL` of per-path `ROW_NUMBER()` ranks → `GROUP BY doc_id` → `SUM(1.0/(60+rank))` → order by that). **Output → Production Hardening Notes.**

### J — Design the agent-facing confidence interface (if consumers are agents)
When results feed an LLM/agent rather than a human UI, never expose raw cross-retriever scores (DR23). Provide the normalized top-gap `c(q)=(s₀−s₁)/s₀` as a confidence signal, source-tag each result by the path that found it, and add per-result uncertainty flags. This also enables a classifier-free cascade: a confident BM25 top-gap can short-circuit the dense path to save latency (AgentIR 2026). **Output → Score-Ceiling-Aware Interface.**

## Decision gates (apply continuously)

These fire the moment the trigger appears in the user's proposal — gates, not gentle suggestions. Full rationale, the cited failure/law behind each, and the rest of DR1–DR24 are in `references/decision-rules.md`.

| Gate | When it fires | What you do |
|------|---------------|-------------|
| Exact identifiers, dense-only | identifiers in queries, no lexical leg | require a BM25 leg + route identifiers to it; dense is identifier-blind (DR1, V15) |
| Paraphrase, lexical-only | natural-language queries, BM25 only | add a dense leg (DR2) |
| Raw-score fusion | linear/weighted blend of BM25 + cosine | default to RRF (k=60); demand calibration+monitoring if they insist (DR6, DR9, Law 10) |
| Third path, no evidence | 3rd path proposed on a "more is better" argument | block until complementarity (RoC) is measured; redundant if RoC < ~0.10 (DR4, Law 6) |
| Fused < best single | hybrid NDCG below a single path | diagnose the noisy path; disable, re-measure, remove if quality recovers (DR12, Law 5) |
| Vendor defaults | fusionType/k/ef unpinned | pin everything explicitly; re-audit after upgrades (DR10, V27) |
| Pre-fusion filtering | filter applied to one path before fusing | move filter to after fusion; pre-fusion drops the complementary path's finds (DR18, V13) |
| Gain unverified past reranking | fusion judged at retrieval layer only | verify fused ≥ baseline *after* reranking + truncation (DR22) |
| Raw scores to an agent | agent consumes raw cross-path scores | expose relative confidence (top-gap, source tags), not raw scores (DR23) |
| TRF / score-level proposed | escalation past RRF without evidence | require the NDCG-gain + storage/calibration evidence the ladder demands (DR8, DR9) |

## Failure modes to catch

Watch for these patterns when designing or reviewing. Full symptom / cause / prevention / law for each is in `references/failure-modes.md`.

- **Exact-identifier blindness (V15)** — dense-only returns wrong docs for SKUs/error codes/flags; the embedder treats unseen identifiers as noise. → always a BM25 leg.
- **Unpinned vendor fusionType (V27)** — quality silently drops after an upgrade, no error. → pin everything.
- **Hybrid below best single path** — an added path injects noise without unique recall (Law 5). → fusion-floor checkpoint; find and remove the culprit.
- **Score-level fusion without calibration** — raw BM25 + cosine blend ranks erratically and breaks on the next model upgrade (Law 10). → RRF default.
- **Third path without complementarity** — three paths score identically to two, +33% latency for nothing (Law 6). → measure RoC first.
- **Pre-fusion metadata filtering (V13)** — one path's filter silently discards the complementary path's relevant docs. → filter after fusion.
- **Fusion gains neutralized after reranking** — upstream recall improves but end-to-end doesn't; new candidates don't survive the cross-encoder + truncation (Medrano et al. 2026). → verify past reranking.
- **Agent over-trusting incomparable scores** — agent treats a BM25 18.3 as more confident than a cosine 0.92. → relative confidence signals only.

## Output: the design artifact

Produce these sections. Scale depth to the problem — a two-path catalog needs a few tight sentences per section, not pages — but address every one, even if only to say "not applicable, because…". Saying *why* a section is N/A is itself a recorded design decision.

1. **Query Class Taxonomy** — classes present, est. frequency/criticality, per-class optimal path (Stage A)
2. **Retrieval Path Selection Rationale** — paths chosen, decision rules applied, paths rejected and why (Stage B)
3. **Per-Path Quality Baseline** — standalone precision@K × query-class matrix, per-path latency, per-class weaknesses (Stage C)
4. **Complementarity Analysis** — RoC matrix per path pair, overlap vs. unique contribution, per-class (Stage D)
5. **Fusion Method Rationale** — escalation rung taken (RRF → weighted → TRF → score-level; CC if labeled data), evidence for any escalation (Stage E)
6. **Routing Strategy** — per-class routing rules, routing-accuracy estimate, misrouting detection/fallback (Stage F)
7. **Metadata Filtering Design** — placement vs. fusion and reranker, selectivity thresholds, deviation rationale (Stage G)
8. **Fusion Quality Verification** — fused vs. best-single NDCG, per-class, noisy-path decisions, after-reranking check (Stage H)
9. **Production Hardening Notes** — pinned params, per-path + fusion monitoring, model versioning, SQL-native RRF if PostgreSQL (Stage I)
10. **Score-Ceiling-Aware Interface** — relative confidence signals, cascade trigger, result envelope (Stage J; agent consumers only)
11. **Known Risks** — likeliest failure modes for *this* design, mitigations, vendor-specific exposure, after-reranking shrinkage risk
12. **Deferred Specialized Decisions** — what you don't own, cross-referenced with `[[wikilinks]]` + placeholder assumptions
13. **Minimal Next Experiment** — the single experiment yielding the most information (e.g. "measure per-path precision@5 for 100 queries across 3 classes before building any fusion") — falsifiable, never "deploy and monitor"

## Worked judgment calls

**Dense-only catalog, queries like "find SKU DQ4312-101" / "error ERR_TIMEOUT".** Flag the missing BM25 leg immediately. The embedder has no special representation for `ERR_TIMEOUT` or adjacent SKUs (they collapse in embedding space) — exact-identifier blindness, Failure V15. Require a lexical leg and route identifier-bearing queries to BM25 only (DR1, DR14). Dense-only is not shippable when identifiers are in the query classes.

**"We'll fuse with `0.6*bm25 + 0.4*cosine` and sort by that."** Score incompatibility (Law 10): BM25 (0–20, long-tailed) and cosine ([-1,1], bunched near 0) live in different worlds, and the tuned weights break the next time the embedding model changes. Default to RRF (k=60). If they insist on score-level, it requires calibrated, stable, *monitored* distributions and only pays for a 2–5% gain that matters (DR6, DR9).

**"Three paths beat two — let's add SPLADE to BM25 + dense."** Block until complementarity is measured (DR4, Law 6). Run SPLADE alone on ~100 queries: what does it retrieve that BM25+dense miss? If overlap with dense exceeds ~50% of the union (RoC < ~0.10), it's redundant — pure latency and ops debt. Only add it if it shows real unique recall.

**"Our hybrid (BM25 + dense) NDCG@10 is 0.42 but dense-only is 0.47 — hybrid should be better."** Fusion floor violated → Law 5. The BM25 path is likely poisoning the pool. Disable it, re-measure; if dense-only recovers to 0.47, BM25 lacks complementarity for this corpus/query mix — remove it or route it only to the classes where it helps (DR12). Don't blame the reranker reflexively.

**"We just use Weaviate's default hybrid, didn't pin fusionType."** Failure V27 risk. Weaviate v1.24 silently switched the default from rankedFusion to relativeScoreFusion — a score-based method that's more fragile. Pin fusionType, k, and ef explicitly and audit after every version upgrade (DR10).

**"We pre-filter by category on the dense results, then fuse with BM25 — filtering first is efficient."** This silently discards BM25's category-matching documents that the dense path ranked poorly (Failure V13). Fuse BM25 + dense *first*, then post-filter the combined pool (DR18). Filtering before fusion removes the complementary path's whole contribution.
