# Pattern seeds — reusable design moves

Patterns the skill should reach for. Each gives the problem it solves, when to use it, when not to, and the ownership boundary with downstream skills.

---

## Pattern Seed 24 — Chunk-to-Answer Traceability

**Problem it solves.** When a RAG system produces a wrong answer, you cannot tell whether retrieval found the right chunks, whether the chunks contained the answer, or whether the LLM misinterpreted them. Without that, debugging is guesswork and chunk-boundary failures stay invisible (Failure R16).

**Method.** Put source_document_id + chunk_position metadata on every chunk (Stage E). For every RAG answer, log which chunks were retrieved and at what rank, whether the expected correct chunk was in the retrieved set and where, whether the chunk contained enough information to answer, and whether the LLM used it correctly. This enables root-cause triage: retrieval failure vs chunk-boundary failure vs generation failure (Stage G).

**When to use.** All RAG systems in production. Debugging RAG quality issues. Continuous chunking improvement.

**When NOT to use.** Non-RAG retrieval systems. When full logging is cost-prohibitive — sample instead.

**Ownership.** This skill owns the *traceability instrumentation design* (what to log, on which chunk fields). `[[Evaluation Operations]]` owns the *evaluation methodology* that consumes the traceability data. This skill makes failures visible; that skill judges them.

---

## Pattern Seed 9 — Per-Signal Evaluation Decomposition (applied to chunking)

**Problem it solves.** An aggregate metric can hide that one chunking strategy works well for legal documents and catastrophically for product descriptions. "Good average nDCG" masks per-domain failure (Failure V24).

**Method.** Evaluate chunking quality *per content type*, never only in aggregate. Compute nDCG (and boundary-failure rate, interpretability rate, size distribution) separately for each content type identified in Stage A. A single global number that looks healthy can average a triumph and a disaster.

**When to use.** All production evaluation where content types are diverse. Before any chunking change claiming "improvement."

**When NOT to use.** Trivial evaluation sets with homogeneous content (one content type).

**Ownership.** This skill *defines* the per-content-type decomposition (which segments to break out, why). `[[Evaluation Operations]]` owns the *evaluation framework* that executes the decomposition. Cross-reference required.

---

## Pattern Seed 2 — Filter-Then-Verify (applied to chunking)

**Problem it solves.** You cannot exhaustively evaluate every possible chunking configuration across every content type — the combinatorics explode. You need a cheap pre-filter to eliminate obviously bad strategies before expensive full evaluation.

**Method.** Use structural-first as the cheap filter: it eliminates fixed-character and naive-semantic candidates immediately (DR5–DR6). Then run full nDCG evaluation only on the top 3–5 surviving strategies per domain. The 36-strategy benchmark exemplifies this — most strategies are dominated and can be cut before the expensive comparison.

**When to use.** When testing multiple chunking strategies across multiple content types.

**When NOT to use.** A single content type with a known-optimal strategy already chosen.

**Ownership.** Owned by this skill, adapted from the universal Filter-Then-Verify pattern to chunking strategy selection.

---

## Pattern Seed 13 — Context-First Retrieval Design (applied to chunking)

**Problem it solves.** Teams pick chunking based on what's conventional ("512 tokens is standard") rather than what the downstream consumer actually needs. The result is chunks that don't serve the retrieval goal — too big for precise Q&A, too small for coherent RAG context.

**Method.** Start from the consumer (Stage D, input 3): an LLM doing RAG needs self-contained chunks with enough context to be usable; a human reader needs citation-friendly readable segments; a Q&A system needs small precisely-targeted chunks; summarization tolerates larger chunks. Choose size and overlap from that need, then check it fits the embedding model (Stage B), rather than starting from a default token count.

**When to use.** Designing chunking for any retrieval system where the consumer is an LLM or a human reader.

**When NOT to use.** When the consumer's context needs are already fully characterized and fixed.

**Ownership.** This skill applies the pattern to *chunking decisions* (size, overlap, coherence). `[[Retrieval Pipeline Design]]` applies the same pattern to *stage design*. Cross-reference required.
