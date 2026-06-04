# Pattern Seeds

Reusable design moves. For each: the problem it solves, when to use, when not to, and the downstream specialized skill that extends it.

---

### PS1 — Staged retrieval (two-step reranking)
- **Solves**: Single-stage retrieval is either too slow (exact KNN) or too imprecise (ANN without refinement).
- **Use when**: >10K vectors; any quantization below fp32; latency-constrained but recall target high.
- **Don't use when**: <10K vectors (exact KNN is fast enough); single-stage is precise enough; 100% recall non-negotiable.
- **Note**: This is the foundational pattern — the core of pipeline design itself.

### PS5 — Query-adaptive routing
- **Solves**: Not all queries need all stages; the full pipeline wastes latency on simple queries.
- **Use when**: Multi-path retrieval with ≥3 paths and a measurable latency budget; diverse query distribution.
- **Don't use when**: Single-path retrieval; routing accuracy is low.
- **Downstream**: `[[Query Understanding & Routing]]` (future) for intent classification and routing accuracy.

### PS6 — Eval-gated progression
- **Solves**: Teams add complexity from ambition, not evidence → fragile systems that demo well and fail in production.
- **Use when**: All new deployments; before any architecture addition.
- **Don't use when**: Well-understood domains with pre-validated architecture patterns.
- **Downstream**: `[[Evaluation Operations]]` for methodology and infrastructure.

### PS10 — Retrieval-reranking triage
- **Solves**: Teams misdiagnose retrieval failures as reranking failures (and vice versa).
- **Use when**: Debugging any retrieval quality issue; production root-cause monitoring.
- **Don't use when**: Ground truth (known correct documents) is unavailable.
- **Note**: Owned here; `[[Evaluation Operations]]` may extend with production triage automation.

### PS14 — RRF default with escape hatch
- **Solves**: Score incompatibility across heterogeneous retrievers; need a default that works plus an escape hatch.
- **Use when**: All new hybrid deployments; score distributions unknown or variable.
- **Don't use when**: Single-path retrieval; proven, monitored, stable score calibration exists and marginal gain matters.
- **Downstream**: `[[Hybrid Search Architecture]]` for fusion math, TRF, weighted RRF.

### PS13 — Context-first retrieval design
- **Solves**: Teams select techniques by what's available, not what the LLM consumer needs.
- **Use when**: Designing any retrieval system for LLM consumption.
- **Don't use when**: Human-facing search (different UX requirements).
- **Downstream**: `[[Agent-Facing Retrieval Design]]` (future).

### PS7 — Complementarity-aware path selection
- **Solves**: Adding paths can degrade quality if the new path is redundant or noisy.
- **Use when**: Adding any second/third path; evaluating whether to replace or augment a path.
- **Don't use when**: First retrieval path; complementarity known a priori (lexical + dense).
- **Downstream**: `[[Hybrid Search Architecture]]` for complementarity measurement.

### PS22 — Metadata-enriched reranking
- **Solves**: Near-identical documents are indistinguishable to rerankers that see only text.
- **Use when**: Product search with variants; entity resolution with near-duplicates; documents differing mainly in structured attributes.
- **Don't use when**: Documents are textually distinct (metadata adds noise); reranker context window too small.
- **Downstream**: `[[Hybrid Search Architecture]]` may extend with structured-data integration.
