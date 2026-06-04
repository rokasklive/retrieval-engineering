---
name: retrieval-pipeline-design
description: >
  Design staged, eval-gated retrieval pipelines for search and RAG — deciding how
  many stages, what each does (candidate generation vs reranking), which retrieval
  method (BM25, dense, sparse, hybrid) fits which query classes, and when
  reranking or hybrid is actually justified. Use whenever architecting a search
  system: choosing between BM25 and vector search, deciding if you need a
  reranker, setting up search over docs, reviewing a proposed pipeline, diagnosing
  why retrieval is bad, or picking top-K. Trigger on any question about whether to
  add a stage, use hybrid/dense/a reranker, or structure a retrieval system —
  even when "pipeline" isn't mentioned. Owns stage composition, retrieval method
  choice per query class, recall-vs-precision split, reranker justification gates,
  and production readiness. Does NOT implement fusion math (Hybrid Search
  Architecture), chunking algorithms (Chunking Strategy Design), or index tuning.
---

# Retrieval Pipeline Design

Design retrieval pipelines that start simple and earn their complexity. The output is a **design artifact** — a reasoned proposal for how a search/RAG system should retrieve and rank, grounded in the user's query classes, corpus, and constraints. You design the architecture; you do not write the final implementation.

## What this skill owns (and what it doesn't)

**Owns:** stage composition (how many stages, what each does, in what order), the recall-vs-precision split between candidate generation and reranking, initial retrieval method choice per query class, whether hybrid is justified, whether reranking is justified, retrieval window size (top-K), failure triage across stages, query-adaptive routing, and the design-time requirements for evaluation and production readiness.

**Does NOT own** (defer these, name the owner, state a placeholder assumption, flag for verification):
- Fusion math — RRF parameters, score-level fusion, weighting → `[[Hybrid Search Architecture]]`
- Chunk boundaries, size, overlap, strategy → `[[Chunking Strategy Design]]`
- Evaluation program construction — judgment lists, metric methodology, coverage monitoring → `[[Evaluation Operations]]`
- Drift monitoring, re-embedding triggers, migration runbooks → `[[Drift Management & Freshness]]`
- Index internals — HNSW M/efConstruction, IVFFlat nprobe, quantization → `[[Vector Index Operations]]` (future)
- The actual code.

When you hit a decision you don't own: (1) state the decision, (2) name the owning skill with a `[[wikilink]]`, (3) state the placeholder assumption you'll proceed with, (4) flag that it must be verified before implementation. Don't silently assume, and don't pretend to design what you don't own.

## The mindset (read this before designing anything)

These are the design habits that separate a robust pipeline from a fragile demo. Internalize the *reasoning*, not just the rules.

1. **Start simple; add stages only when a measured symptom demands it.** The base system is BM25 or dense-only — one stage. The "7-layer maturity model" is a ladder you climb one rung at a time in response to observed failures, never a menu you order from. Systems built complex-first hit the demo-to-production cliff (curated demo looks perfect; real queries get ~31% accuracy). Before proposing any layer, ask: *"What symptom would I see in evaluation if this layer were missing?"* No symptom → no layer.

2. **Separate the recall job from the precision job.** Candidate generation (Stage 1) is cheap, broad, recall-optimized: its job is to *get the right document into the pool*. Reranking (Stage 2) is expensive, narrow, precision-optimized: its job is to *put it first*. End-to-end quality ≈ retrieval recall × reranking precision — neither stage compensates for the other's failure. Designs that conflate them get systems that are neither fast nor accurate.

3. **Design by query class, not one-size-fits-all.** Exact-identifier queries (SKUs, error codes, function names) need 1 stage of BM25. Semantic/paraphrase queries need dense, often + reranking. Mixed corpora need hybrid. Routing simple queries through the full pipeline just buys latency. If you don't know the query classes yet, you are not ready to choose an architecture — establish them first.

4. **Retrieve generously.** First-stage similarity is cheap; the reranker/LLM call is what's expensive. Feed the reranker a wide pool (top-K ≈ 75–200, not 10). A reranker can only reorder what it's given — it cannot recover a document the first stage never retrieved. Inner LIMIT should exceed final result count × 3, and must stay ≤ the index's search depth (e.g. ef_search) or recall is silently capped.

5. **Evaluation is infrastructure, not decoration.** "No eval baseline" is the root cause behind essentially every production RAG failure. Require a baseline, a representative judged query set, and coverage *before* any quality claim. Offline metrics are filters ("probably worse"), not decision gates ("definitely better"). You cannot tune what you cannot measure — and you cannot measure a reranker's benefit without a first-stage recall number.

6. **Drift is inevitable; design for it up front.** Every retrieval system decays (~8–12% MRR loss/year) and *no standard alert fires* — not latency, not error rate. Production designs carry a `model_version` on every vector, a freshness policy, and canary queries from day one. Retrofitting these after silent decay has already cost you is the common, expensive path.

7. **Chunking is the binding upstream constraint.** You cannot out-embed or out-rerank bad chunks (structural vs. fixed-character chunking is a ~1.9× nDCG difference). You don't design chunking here — but if it's undefined, flag it as a structural quality unknown and cross-reference `[[Chunking Strategy Design]]`. Don't design a pipeline on top of an unexamined chunking assumption without saying so.

8. **Rank-based fusion over raw-score mixing.** BM25 scores, cosine similarities, and neural scores live in incomparable ranges; normalization breaks on every model upgrade. Default to RRF (k=60) — rank-based, zero-config, distribution-shift robust. (The math is `[[Hybrid Search Architecture]]`'s job; the *default choice* is yours.)

The full, cited versions of the laws behind these habits are in `references/laws.md` — read it when you need the evidence, the exact numbers, or the violation symptoms to cite.

## The design procedure

Work through these stages in order. Early stages gather the inputs that later stages depend on — skipping ahead to architecture before you understand the query classes and corpus is the most common way designs go wrong. You don't need heavy ceremony for each; for a simple problem several stages collapse to a sentence. But *consciously visit each one*.

`references/design-inputs.md` details what to extract for each input and why it changes the design — consult it when the user's brief is thin and you need to know what to ask for.

### A — Define the search problem
Establish: who searches and in what language; what task (navigational / informational / transactional); **the query classes** (exact identifiers, short keywords, natural-language/semantic, entity lookups, typo-prone, multi-hop); what makes a result "good" (binary or graded relevance → drives MRR vs. NDCG); whether answers live in one document or span several; latency budget (P50/P95/P99); freshness needs; access-control boundaries. The query-class taxonomy is the single most important output here — almost every later decision keys off it.

### B — Characterize the corpus
Document types (narrative / structured / code / tables), structure and metadata quality, size, update frequency, entity density, presence of exact identifiers, proportion of long documents (>1000 tokens → chunking matters), and access-control structure. This tells you whether lexical, dense, or hybrid is even appropriate, and whether chunking is a live risk.

### C — Choose the initial retrieval strategy
Pick the *starting* method from the query classes + corpus (see the table in `references/decision-rules.md` for the full matrix):
- Predominantly exact identifiers / rare terms / short keywords, or no GPU → **BM25**. Fastest, cheapest, strongest zero-shot baseline.
- Predominantly natural-language / paraphrase / conceptual, narrative corpus, GPU available → **dense**.
- Need semantic signal without a GPU → **sparse (SPLADE)**.
- Both exact identifiers *and* semantic queries present → **hybrid (lexical + dense)** — the production default, because the two fail in opposite directions.

Guardrails: exact identifiers in scope → a BM25 leg is non-negotiable (dense retrieval is blind to identifiers it never saw in training). Paraphrase in scope → lexical-only is insufficient. Query classes unknown → stop and establish them; don't pick an advanced architecture on a guess.

### D — Design the stage composition
Decide, per stage: which method(s) generate candidates and at what recall target; the candidate-pool size (≈50 fast / 200 general / 800 accuracy-critical); filtering order (default **post-filter** to preserve recall; pre-filter only with deterministic, highly selective constraints; filter *after* fusion, *before* the reranker); whether query rewriting/expansion is warranted (only for short/interrogative queries on answer-shaped corpora); whether to rerank (Stage F); and **fallback behavior** (empty first-stage → expand/broaden; reranker timeout → return first-stage order). Produce a stage diagram and per-stage responsibilities.

### E — Decide whether hybrid is justified
Hybrid earns its place when paths are *complementary* — measured by the unique relevant documents a path contributes that others miss, not by its standalone quality. Justified when query classes mix exact identifiers (need BM25) with paraphrase/conceptual (need dense), or when metadata/fuzzy needs add an orthogonal path. **Not** justified when only one query class exists, when the extra path's precision is below threshold (a weak path poisons the candidate pool — fused quality can drop below the best single path), or when latency can't absorb it. Default to two-path (lexical + dense); a third path needs measured complementarity, not a completeness argument.

### F — Decide whether reranking is justified
Recommend a reranker only when **all** hold: (1) first-stage recall is sufficient — the right docs are *in* the pool; (2) ordering is the actual problem — high recall, low precision; (3) latency budget absorbs it (~100–200ms cross-encoder, 500ms+ LLM); (4) you can measure the improvement against a baseline. It's premature when first-stage precision is already adequate, the budget is <50ms, the candidate set is tiny (<20), or no baseline exists. **The hard gate:** a reranker cannot recover documents absent from the candidate pool — so a reranker proposed without a first-stage recall number is an unverifiable quality claim; surface that before endorsing it. Conversely, *whenever results feed an LLM*, a reranker is almost always warranted — skipping it is the single largest common quality regression. Never propose a cross-encoder as the *first-stage* retriever (it's O(N) transformer passes per query — pathologically slow beyond ~10K docs).

### G — Plan evaluation before tuning
Require (don't merely suggest): a **baseline** (current NDCG/MRR/P@K before changes); a representative judged **query set** (≥50, drawn from real queries, not engineered demos); **coverage@K** reported beside every metric (<0.50 → metrics unreliable); metrics matched to query classes; **before/after per-stage marginal contribution**; **per-query-class breakdown** (so per-class failures aren't averaged into invisibility); a **retrieval-position triage** bucket; and explicit **falsification conditions** ("if the reranker adds <3% NDCG, it isn't justified"). The methodology itself is `[[Evaluation Operations]]`; here you specify *what must be measured and what would prove the design wrong.*

### H — Plan production readiness
Every production design includes: a non-nullable `model_version` on every vector table (vectors from different models live in different spaces — mixing them is geometrically meaningless and unrollbackable); a freshness policy (re-embedding triggers, reindex schedule, staleness SLA); canary queries (20–50 fixed queries with known answers, tracked weekly, >7% MRR drop → investigate); per-stage latency monitoring; a fallback strategy per stage; and a handoff to `[[Drift Management & Freshness]]` for the operational implementation. Name *who* owns the canary checks and drift response.

## Decision gates (apply continuously)

These fire whenever the relevant trigger appears in the user's proposal — they are gates, not gentle suggestions. Full rationale and the cited failure/law behind each is in `references/decision-rules.md`.

| Gate | When it fires | What you do |
|------|---------------|-------------|
| Unknown query classes | architecture chosen before classes are known | stop; establish query classes first (Stage A) |
| Exact identifiers, dense-only | identifiers in queries/corpus, no lexical leg | require a BM25 leg; dense is blind to unseen identifiers |
| Paraphrase, lexical-only | natural-language queries, BM25 only | add a dense leg; BM25 can't match synonyms |
| Undefined chunking | chunking strategy unspecified | flag as structural quality unknown; cross-ref `[[Chunking Strategy Design]]` |
| Reranker over low recall | low recall@inner_LIMIT | reranking can't fix it; fix recall first |
| Raw-score fusion | score-level combination of different retrievers | default to RRF (k=60); demand monitoring if they insist on scores |
| Noisy fusion path | a path adds noise without unique recall | remove/down-weight; require fused ≥ best single path |
| No judged query set | no eval baseline | treat all quality claims as unverified |
| Unjustified stage | stage added with no measured symptom | reject the complexity; demand the symptom |
| Coverage@K < 0.50 | low judged coverage | flag metrics as potentially unreliable |
| No model versioning | embeddings without `model_version` | flag migration/rollback risk; require the column |
| Cross-encoder first-stage | cross-encoder as retriever on >10K docs | reject; cross-encoder is final-stage only |
| Inner LIMIT < ef_search | inner LIMIT below ANN search depth | flag silently capped recall |
| Reranker, no recall baseline | reranker proposed without recall measurement | flag unverifiable quality claim |

## Failure modes to catch

When reviewing or designing, watch for these patterns. Full symptom/cause/prevention for each is in `references/failure-modes.md`.

- **Skipping the reranker when results feed an LLM** — high recall, low precision; the LLM gets badly ordered context and gets blamed.
- **Dense-only with exact-identifier queries** — the embedder treats SKUs/error codes as noise.
- **One pipeline for every query** — simple queries pay full-pipeline latency for nothing.
- **Misdiagnosing retrieval failure as reranking failure** — teams tune the reranker when the doc was never retrieved. Always run the retrieval-position check first: not in top-K → retrieval failure; in top-K but mis-ranked → reranking failure.
- **No evaluation baseline** — "what's our quality?" has no answer; user complaints are the only signal.
- **Fixed-character chunking** — chunks end mid-sentence; quality craters regardless of embedder.
- **Near-identical candidates** (adjacent SKUs/variants) — the reranker sees identical text; enrich the reranked text with differentiating metadata.
- **Architecture overbuilt before symptoms justify it** — the demo-to-production gap.

## Output: the design artifact

Produce these sections. Scale the depth to the problem — a 5K-record exact-match catalog needs a few tight sentences per section, not pages — but address every one, even if only to say "not applicable, because…". Saying *why* a section is N/A is itself a design decision worth recording.

1. **Problem and Query Classes** — who searches, query-class taxonomy, what success means
2. **Corpus Assumptions** — types, structure, size, update frequency, metadata quality
3. **Proposed Pipeline** — stage diagram with the flow of candidates
4. **Stage Responsibilities** — per stage: what it does, what it optimizes for, what it does *not* do
5. **Retrieval Method Rationale** — why this method for these classes; which decision gates applied
6. **Chunking Assumptions** — assumed strategy; verified or flagged as unknown
7. **Hybrid / Fusion Rationale** — is hybrid justified, by which classes, with what complementarity evidence
8. **Reranking Rationale** — is reranking justified; first-stage recall baseline; latency check
9. **Evaluation Plan** — baseline, query set, judged examples, metrics, before/after, per-class breakdown, falsification conditions
10. **Production Readiness Notes** — model versioning, freshness, canary queries, latency monitoring, fallback, drift handoff
11. **Known Risks** — likeliest failure modes and their mitigations
12. **Deferred Specialized Decisions** — what you don't own, cross-referenced with `[[wikilinks]]` and placeholder assumptions
13. **Minimal Next Experiment** — from a cold start, the single experiment that yields the most information (a falsifiable next step, never "deploy and see")

## Worked judgment calls

**Exact-match catalog (5K product records, queries like "find DQ4312-101").** One stage: BM25. No dense, no reranker, no hybrid. Chunking irrelevant (short structured records — say so). Exact identifiers dominate → BM25 is not just sufficient but *correct*; adding dense would be complexity with no symptom. Next experiment: confirm BM25 P@1 on a sample of real SKU queries.

**Someone proposes dense-only, but queries include "find error ERR_TIMEOUT in logs".** Flag the missing BM25 leg. The embedder has no special representation for `ERR_TIMEOUT` — it's exact-identifier-blind. Require hybrid (BM25 + dense), routing identifier-bearing queries to the lexical leg.

**Someone proposes combining BM25 and cosine scores by linear weighting.** Flag score incompatibility — different ranges, different distributions, breaks on the next model upgrade. Default to RRF (k=60). If they insist on score-level fusion, it's `[[Hybrid Search Architecture]]`'s call and requires monitored, calibrated distributions.

**Someone wants a cross-encoder reranker but has no first-stage recall number.** Hold. Without recall you can't tell whether the right docs are even in the pool — so you can't attribute any improvement to the reranker. Require: measure recall@inner_LIMIT first; *then* add the reranker and measure the delta against that baseline.
