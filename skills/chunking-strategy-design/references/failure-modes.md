# Failure modes — symptom, cause, prevention

The chunking failures the corpus characterizes well. Each gives the symptom you'll observe, the underlying cause, the design prevention, and the governing law. Cite these by code when you flag a problem.

---

## V21 — Fixed-Character Chunking Destroys Retrieval Quality

**Symptom.** nDCG@5 = 0.244 — the worst performer across all 36 strategies in the benchmark. Chunks end mid-sentence or mid-word. Retrieval quality is poor regardless of which embedding model is used.

**Cause.** Cutting text at a fixed byte/character offset ignores all semantic, syntactic, and structural boundaries. Every chunk has random content edges, so most embeddings represent fragments rather than complete ideas.

**Prevention.** DR5 — REJECT any fixed-character chunking proposal outright. Use structural boundaries first. For genuinely unstructured text, use recursive text splitting with paragraph → sentence boundary awareness (DR1–DR3), never a raw offset cut.

**Law.** 2 (Chunking Primacy).

---

## V22 — Chunking and Quantization Losses Compound Multiplicatively

**Symptom.** Quantization quality measured on well-chunked benchmarks overestimates production performance. Real-world degradation is far worse than expected — e.g. 94% recall retention on academic benchmarks drops below 80% on poorly chunked production data.

**Cause.** Information loss from bad chunk boundaries compounds *multiplicatively* with quantization precision loss — not additively. The bits quantization discards may have been the only signal separating a relevant passage from an irrelevant one; if chunking already blurred the passage, there is no margin left to spend.

**Prevention.** DR15–DR17 — never evaluate quantization on academic chunks; always measure tolerance on production chunking quality. Optimize chunking before any compression strategy. Measure degradation on poorly-chunked vs well-chunked portions of the corpus separately so the compound effect is visible.

**Law.** 2 (Chunking Primacy), and Law 9 (Compound Optimization, owned elsewhere).

---

## V23 — Chunks That Don't Make Sense Standalone

**Symptom.** Retrieval returns chunks, but the LLM generates wrong or confused answers. The chunks turn out to be partial sentences, fragments, or bullet points lacking context. LLM confidence is high while accuracy is low.

**Cause.** Chunking strategies that don't guarantee each chunk is a self-contained semantic unit produce embeddings of partial/incomplete content. The embedding model faithfully encodes the fragment; the LLM faithfully tries to use it. A chunk that references "as discussed above" without the antecedent, or is one bullet from a longer list, has no standalone meaning to encode.

**Prevention.** DR13 — human spot-check for standalone interpretability (the acid test: can a person understand what this chunk is about without reading the surrounding text?). Use semantic boundaries (paragraphs, sections, topic transitions), not arbitrary offsets.

**Law.** 2 (Chunking Primacy).

---

## R6 — Multi-Hop Fragmentation

**Symptom.** > 15% of queries that require information spanning multiple documents fail. Single-hop retrieval can't assemble the complete answer. The correct information exists in the corpus but is split across chunks that aren't linked.

**Cause.** The chunking strategy doesn't preserve cross-document or cross-chunk connections. Information needed to answer a query spans multiple chunks with no parent-child or cross-reference links to reassemble them.

**Prevention.** Design the metadata schema with parent-child retrieval support (Stage E). Enable adjacent-chunk retrieval for context expansion. Track multi-hop query volume separately. Note the boundary: the *multi-hop retrieval pipeline* (retrieve → read → retrieve more) is owned by `[[Retrieval Pipeline Design]]`; chunking *enables* it through metadata and links but does not design the pipeline.

**Law.** 2 (Chunking Primacy), and Law 3 (Retrieval-Reranking Asymmetry, owned elsewhere).

---

## V24 — Domain-Generic Chunking on Specialized Corpora

**Symptom.** A single chunking strategy applied globally produces wildly different nDCG across domains. Legal retrieval is strong but product-catalog retrieval is broken — or the reverse.

**Cause.** A "default" chunking strategy deployed without per-domain validation. What works for legal documents (PGC) fails for product catalogs (attribute-aware); what works for biology (DFC) fails for mathematics. There is no universal best strategy — the domain determines the strategy.

**Prevention.** DR1–DR4 — domain-to-strategy mapping. Stage A content-type inventory categorizes the corpus before any strategy is chosen. Stage F validates *per content type*, not in aggregate. Never deploy one global chunking strategy across heterogeneous content.

**Law.** 2 (Chunking Primacy).

---

## R16 — Misdiagnosing Chunking Failure as Embedding Failure

**Symptom.** A team spends hours tuning embedding models, rerankers, or ANN parameters while the real root cause is chunk boundaries splitting key passages. Quality doesn't improve because the wrong lever is being pulled.

**Cause.** Chunk-boundary failures are invisible without chunk-to-answer traceability instrumentation. The symptom — bad retrieval results — looks identical whether the cause is chunking, embedding, or indexing, so teams reach for the most familiar lever (the model) first.

**Prevention.** Stage G — chunk-to-answer traceability instrumentation makes boundary failures visible. Stage F — chunk-boundary failure-rate measurement. DR12 — > 5% boundary failure rate means fix chunking first. Follow the diagnostic sequence: chunking → embedding → indexing → reranking, in that order.

**Law.** 2 (Chunking Primacy), and Law 3 (Retrieval-Reranking Asymmetry, owned elsewhere).
