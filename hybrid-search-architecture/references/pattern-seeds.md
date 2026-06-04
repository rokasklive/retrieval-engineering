# Pattern seeds — reusable design moves

Each gives the problem it solves, when to use it, when NOT to, and the specialized skill that extends it. These are the recurring shapes a good hybrid design reaches for.

## Seed 14 — RRF default with escape hatch
- **Problem.** Choosing between rank-level (RRF) and score-level fusion. RRF is robust but discards score magnitude; score-level preserves signal but is fragile to distribution shift.
- **Use when.** All new hybrid deployments. Any system where score distributions are unknown, variable, or unmonitored. When vendor defaults can't be trusted.
- **Don't use when.** Single-path retrieval (no fusion needed). When you have proven, monitored, stable calibration *and* the 2–5% score-level gain matters.
- **Extends to.** This skill owns fusion-method selection entirely. `[[Evaluation Operations]]` provides the infrastructure to compare methods.

## Seed 7 — Complementarity-aware path selection
- **Problem.** Adding paths can *degrade* quality if the new path is redundant or noisy. "More paths = better" is the most dangerous heuristic in hybrid search.
- **Use when.** Adding any second or third path. Quarterly architecture review. Evaluating whether to replace or augment an existing path.
- **Don't use when.** The first path (nothing to complement). When complementarity is known a priori — lexical + dense for a corpus with both identifiers and paraphrase.
- **Extends to.** This skill owns complementarity measurement and path selection. `[[Retrieval Pipeline Design]]` decides whether hybrid is justified at the architectural level.

## Seed 5 — Query-adaptive routing
- **Problem.** Not all queries benefit from all paths. Running full hybrid on every query wastes latency on simple ones (exact-ID queries that need only BM25).
- **Use when.** ≥2 paths and a measurable latency budget. Diverse query-type distribution.
- **Don't use when.** Single-path retrieval. When routing accuracy is low (<80%) and misrouting degrades quality below running full hybrid on everything.
- **Extends to.** `[[Query Understanding & Routing]]` (future) — formal intent classification, hardness prediction, uncertain-routing fallback.

## Seed 16 — Score-ceiling awareness
- **Problem.** Agent-facing systems need confidence signals, but raw scores from different paths are incomparable. Agents over-trust high scores from weak paths.
- **Use when.** Agent-facing retrieval APIs. Results feeding LLM decision-making. Multi-source retrieval where score meaning varies by source.
- **Don't use when.** Human-facing search UIs (different information needs). Single-retriever systems with internally consistent scores.
- **Extends to.** This skill owns the *fusion-level* confidence signal (top-gap, source tags). `[[Agent-Facing Retrieval Design]]` (future) owns *agent-level* interpretation.

## Seed — BM25 confidence cascade (AgentIR 2026)
- **Problem.** Running dense retrieval (~52ms) on every query wastes latency when BM25 alone suffices for confident queries.
- **Use when.** Multi-path systems with tight latency budgets, where BM25 produces discriminative score gaps between top results. The trigger is the normalized top-gap `c(q)=(s₀−s₁)/s₀` exceeding a threshold τ_c — classifier-free routing.
- **Don't use when.** Predominantly paraphrastic queries (dense always needed). When BM25 is poorly tuned (the gap isn't trustworthy).
- **Extends to.** This skill owns the cascade trigger design. `[[Precision & Storage Optimization]]` (future) may extend with quantization-aware cascading.
