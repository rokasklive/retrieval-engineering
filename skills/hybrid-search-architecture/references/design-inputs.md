# Design inputs — what to extract and why it changes the design

Before designing fusion or routing, gather these inputs. For each: what to extract, and why it moves a design decision. When the user's brief is thin, this is your checklist of what to ask for. Don't design on silent assumptions — if an input is unknown, name it as an assumption and flag it for verification.

## 1. Query classes present
**Extract.** Categorize queries: exact identifiers (SKUs, error codes, CLI flags, function names); short keywords (1–3 words, ambiguous); natural-language / paraphrastic / cross-lingual; entity lookups; typo-prone; mixed-intent.
**Why.** Query classes determine which paths are necessary and how to route. Exact-ID → lexical; paraphrastic → dense; mixed → hybrid. Without this you can't justify path count or routing. This is the single most important input.

## 2. Existing retrieval infrastructure
**Extract.** Which paths exist today (BM25? dense? both?). Are paths independently measurable? Is there an eval baseline with per-path metrics? What fusion (if any) is used now?
**Why.** Sets the starting point. If no path is independently measurable (Invariant 7 violation), enabling per-path measurement comes *before* tuning fusion. If hybrid exists but fused < best single, diagnose the noisy path first.

## 3. Corpus identifier density
**Extract.** Proportion of queries containing exact identifiers; proportion of corpus fields needing exact match; entity density (names, citations).
**Why.** High identifier density → the lexical leg is non-negotiable (dense is identifier-blind, V15). Drives the "always include BM25" rule (DR1).

## 4. Latency budget
**Extract.** Per-query budget (P50/P95/P99). Per-path latency contribution. Can paths run in parallel?
**Why.** Each path adds latency; TRF adds ~128× storage. Query-adaptive routing skips unneeded paths within budget. Determines whether two- or three-path is feasible, and whether a BM25-confidence cascade is worth it.

## 5. Retrieval path count and types
**Extract.** How many paths proposed/existing, and which types (BM25, dense bi-encoder, SPLADE, fuzzy trigram)? What does each optimize for?
**Why.** Two-path (lexical + dense) is the default. Three-path needs complementarity justification (DR4). Four-path is almost certainly overbuilt. Types determine complementarity potential.

## 6. Score distribution characteristics
**Extract.** Per path: score range, distribution shape, sensitivity to query type. Are scores normalized to a common scale? Stable or shifting?
**Why.** Incomparable/shifting distributions → RRF mandatory (Law 10). Well-calibrated, stable, monitored distributions → score-level fusion *may* be acceptable.

## 7. Metadata filtering requirements
**Extract.** Metadata constraints (dates, categories, permissions, multi-tenancy). Filter selectivity — what fraction of candidates survive?
**Why.** Drives placement: post-filter default (preserves recall), pre-filter only when selectivity >90% and deterministic (else V13), in-algorithm when the DB supports it. Placement relative to fusion and reranking affects quality and cost.

## 8. Evaluation baseline status
**Extract.** Is there a judged query set? Can each path be evaluated independently? Is complementarity measurable? Are per-signal metrics available (NDCG_exact, NDCG_fuzzy, NDCG_semantic)?
**Why.** Without per-path measurement, complementarity is faith and path add/remove decisions are unverifiable. "No eval baseline" blocks all fusion tuning — say so. Methodology itself is `[[Evaluation Operations]]`.

## 9. Vendor / infrastructure constraints
**Extract.** Which infra (Elasticsearch, pgvector, Weaviate, Qdrant, Azure AI Search)? Native RRF? Are fusion params pinnable? Multi-service hybrid (e.g. pgvector + Elasticsearch)?
**Why.** Elasticsearch 8.0+, OpenSearch 2.12+, Qdrant offer native RRF. Weaviate has an unpinned fusionType default that has changed across versions (V27). PostgreSQL needs SQL-native RRF. Multi-service hybrid adds operational complexity (and is thinly evidenced — flag it).
