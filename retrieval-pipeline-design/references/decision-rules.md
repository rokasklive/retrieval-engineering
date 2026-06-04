# Decision Rules

Operational gates with their rationale and the failure/law behind each. These are gates, not suggestions — when the trigger appears in a design or proposal, apply the rule.

## Initial retrieval strategy matrix

| Strategy | When appropriate |
|----------|-----------------|
| **BM25 (lexical)** | Queries predominantly exact identifiers, rare terms, short keywords. Structured fields. No GPU. Fastest, cheapest, strongest zero-shot baseline. |
| **Dense vector** | Queries predominantly natural-language, paraphrase, cross-lingual, conceptual. GPU available. Narrative/unstructured corpus. |
| **Sparse (SPLADE)** | Need semantic signal without a GPU. Inverted-index infrastructure exists. |
| **Hybrid (lexical + dense)** | Mixed query types — exact identifiers AND semantic queries. The production default; lexical and dense fail in opposite directions. |
| **Metadata-filtered** | Reliable structured metadata (dates, categories, permissions); filtering essential to quality. |
| **SQL-native** | Corpus in PostgreSQL with tsvector; exact + full-text queries; pgvector for the vector leg. |

## Architecture selection

- **DR1 — Unknown query classes → no advanced architecture yet.** Establish query classes first (Stage A). Choosing architecture on a guess is the most expensive early mistake.
- **DR2 — Exact identifiers matter → never dense-only.** Always include a BM25/lexical leg. Dense retrieval is blind to identifiers it never saw in training (Failure V15, Law 6).
- **DR3 — Paraphrase matters → lexical-only is insufficient.** BM25 can't match synonyms/paraphrases; include a dense leg.
- **DR4 — Chunking undefined → pipeline quality is structurally unknown.** Flag as an unresolved dependency. Five minutes reading chunks beats five hours of hyperparameter tuning (Law 2).
- **DR5 — Low first-stage recall → don't expect reranking to fix quality.** The reranker can't recover documents the first stage missed (Law 3).

## Fusion

- **DR6 — Raw-score fusion across retrievers → flag score incompatibility.** Default to RRF (k=60). Score-level fusion only with calibrated, monitored distributions (Law 10).
- **DR7 — A path adds noise without unique recall → remove or down-weight it.** Validate: fused quality ≥ max(single-path quality) (Law 5).

## Evaluation

- **DR8 — No judged query set → quality claims are unverified.** "No eval baseline" is the root cause across production failures (Failure R3).
- **DR9 — Stage added without a measured symptom → reject the complexity.** Every layer must be symptom-justified (Law 12).
- **DR10 — Coverage@K < 0.50 → flag metrics as potentially unreliable.** Unjudged results are invisible to measurement (Law 11).

## Production

- **DR11 — Embeddings lack model versioning → flag maintenance/migration risk.** Every vector table needs a non-nullable `model_version` (Law 14).
- **DR12 — High freshness needs → design re-embedding/update policy early.** Drift is silent and continuous (Law 8); cross-ref `[[Drift Management & Freshness]]`.

## Reranking

- **DR13 — Never propose a cross-encoder as first-stage retriever on >10K docs.** O(N) transformer passes per query — pathologically slow (Failure V26).
- **DR14 — Inner LIMIT below ef_search → flag silently capped recall.** Inner LIMIT must be ≤ ef_search for HNSW (Failure V12). Common values: 50 (fast), 200 (general), 800 (accuracy-critical).
- **DR15 — Reranker proposed without a first-stage recall baseline → flag unverifiable quality claim.** You can't measure reranker benefit without a recall baseline.

## Query-length routing heuristic (initial proxy)

A coarse starting point before a formal query taxonomy exists: <4 tokens → lexical-priority; 4–20 → balanced hybrid; >20 → semantic-priority. Treat as a proxy to be refined with real query data, not a final routing policy.
