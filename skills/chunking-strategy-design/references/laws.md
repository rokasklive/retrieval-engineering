# Laws — the evidence behind the mindset

The one law a chunking strategy must obey, stated in full. The SKILL.md mindset is the working version; this is the cited evidence, the numbers, and the exact violation symptom. Cite it by number when you flag a problem.

---

## Law 2 — Chunking Primacy (the binding-constraint law)

**Statement.** The chunk boundary is the irreducible binding constraint on all downstream retrieval quality — you cannot out-embed, out-rank, or out-rerank a bad chunking strategy.

**Why it matters.** A chunk that cuts a key passage in half produces two embeddings, neither of which captures the complete information. No embedding-model quality and no reranker sophistication can reconstruct what was severed at the boundary — the information is simply not present in either vector to be recovered. The 36-strategy benchmark (Shaukat et al.) is definitive: fixed-character chunking achieves nDCG@5 = 0.244 (worst of all 36) while structural paragraph-group chunking achieves nDCG@5 = 0.459 — a **1.88× difference from chunking alone**, larger than the gap between strong and weak embedding models or the lift from any reranker. This is why "five minutes reading your chunks beats five hours of hyperparameter tuning": the chunk boundary determines what content is even visible to the embedding model, and everything downstream operates on whatever the boundary left behind.

**Design consequence.** Debug chunking *before* touching any other retrieval parameter. The strategy-selection decision tree (Stage C) must run before any MRL dimension reduction, quantization, or index configuration. Chunk-quality validation (Stage F) must precede any claim about retrieval quality. When retrieval is poor, eliminate chunking as a cause first — check for mid-sentence boundaries and non-interpretable chunks — before investigating models, indexes, or rerankers (Failure R16). And because chunking sits upstream of every compression step, its losses *compound* with quantization and dimension reduction (Failure V22): optimize the boundary before you optimize the bits.

**Violation symptoms.** A design artifact that specifies the embedding model, reranker, and index configuration in detail but treats chunking as "default 512-token with 10% overlap" with no per-content-type justification. A team tuning MRL dimensions or quantization precision while chunks still end mid-sentence. A recommendation to swap embedding models to fix retrieval quality without first inspecting chunk boundaries.

---

### Related laws owned by other skills (cited, not encoded here)

Chunking interacts with two laws this skill does not own. Reference them; don't re-derive them.

- **Law 9 — Compound Optimization.** Quantization, MRL reduction, and chunking quality compound *multiplicatively*, not additively (Failure V22). This skill's consequence: optimize chunking first, and never validate quantization tolerance on academic chunks. The full law lives with `[[Retrieval Pipeline Design]]` / compression briefs.
- **Law 3 — Retrieval-Reranking Asymmetry.** A reranker can only reorder what retrieval surfaced; it cannot recover a passage that chunking split and retrieval therefore missed (relevant to Failures R6, R16). Owned by `[[Retrieval Pipeline Design]]`.
