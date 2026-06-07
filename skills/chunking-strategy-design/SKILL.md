---
name: chunking-strategy-design
description: >
  Select, configure, and validate document chunking for search/retrieval/RAG systems: boundary placement, 
  chunk size, overlap, metadata, and proof that chunking works before downstream tuning. Use when 
  designing document splitting, diagnosing retrieval quality, or evaluating questions like chunk size, 
  fixed vs semantic vs structural splitting, overlap, legal/product/CJK corpora, multi-hop RAG, product 
  search, poor chunks causing wrong answers, or whether bad retrieval is really an embedding-model issue. 
  Trigger even when “chunking” is not mentioned: any proposal involving split strategy, chunk size, overlap, 
  or retrieval failures is in scope. Owns boundary placement, size/overlap, structure- and script-aware splitting, 
  chunk metadata, and chunk-quality validation. Does not implement splitter code, choose embeddings/rerankers, 
  design fusion, or own the broader evaluation program.
---

# Chunking Strategy Design

Select and validate the chunking strategy for a retrieval system **so that the boundaries are right before anyone tunes a model, an index, or a reranker — because nothing downstream can recover what the boundary destroyed.** The output is a **design artifact**: a reasoned chunking strategy — which boundaries to cut on for each content type, what size and overlap, what metadata every chunk carries, and how to prove the chunks are good. You select and evaluate strategies; you do not write the splitter code, pick the embedding model, or design the rest of the pipeline.

The single most important idea: *the chunk boundary is the binding constraint on all downstream retrieval quality.* You cannot out-embed, out-rank, or out-rerank a bad chunking strategy — a passage cut in half produces two embeddings, neither complete, and no model recovers what was severed. The 36-strategy benchmark is blunt about it: fixed-character chunking scores nDCG@5 = 0.244 while structural paragraph-group chunking scores 0.459 — a **1.88× gap from chunking alone**, larger than the gap between strong and weak embedding models. Hence the operating principle: *five minutes reading your actual chunks beats five hours of hyperparameter tuning.*

And the reflex that follows from it: when retrieval is bad, *check the chunks first.* The most expensive failure in the corpus is a team swapping embedding models and rerankers for hours while chunks quietly end mid-sentence (Failure R16). Before you touch any other lever, look at the boundaries.

## What this skill owns (and what it doesn't)

**Owns:** chunk boundary placement (structural vs semantic vs fixed-size, and the strategy-selection decision tree); chunk size decisions (optimal range by domain, ~128–2048 tokens; size-distribution measurement and validation); overlap configuration (when redundant coverage helps vs when overlap is masking bad boundaries); structure-aware chunking (parse headings/sections/paragraphs before applying size limits; heading-path metadata on every chunk); domain-to-strategy mapping (PGC for legal/structured, DFC for narrative/technical, attribute-aware for product catalogs, function/class boundaries for code, script-aware for CJK); chunk-to-answer traceability instrumentation design (source ID + position metadata so a retrieval failure can be localized to a boundary); chunk-quality validation (standalone-interpretability check, semantic-coherence check, size-distribution check, boundary-failure-rate measurement); CJK-aware chunking (script-aware splitting, character-level n-grams as fallback, NFC normalization); and chunking-failure diagnosis (distinguishing a boundary failure from an embedding, index, or reranker failure).

**Does NOT own** (defer, name the owner, state a placeholder, flag for verification):
- Writing the actual splitter/tokenizer code — this skill *selects* strategies; the agent does not implement chunking algorithms.
- Embedding-model selection → `[[Retrieval Pipeline Design]]` (this skill reads the model's max/optimal length and MRL support as a *constraint*; it doesn't choose the model).
- Reranker selection and pipeline stage composition → `[[Retrieval Pipeline Design]]`.
- Retrieval fusion, routing, score combination → `[[Hybrid Search Architecture]]` (fusion quality is *bounded* by chunk quality but designed there).
- The evaluation program — judged set, metric choice, coverage, gates → `[[Evaluation Operations]]` (chunk quality is *measured* here in isolation; the eval methodology that consumes it is designed there).
- Production drift monitoring, re-chunking on document updates, freshness operations → `[[Drift Management & Freshness]]`.
- Multimodal chunking (images, tables, diagrams alongside text) → `[[Multimodal Retrieval]]` (future, not yet briefed — name the gap, don't fill it).
- Full RAG answer-generation behavior → `[[RAG System Design]]` (future).

When you hit a decision you don't own: (1) state it, (2) name the owning skill with a `[[wikilink]]`, (3) state the placeholder assumption you'll proceed with, (4) flag that it must be verified before implementation. Don't silently assume, and don't pretend to design what you don't own.

## The mindset (read before designing anything)

These are the design habits that separate a chunking strategy you can trust from "512 tokens with 10% overlap" applied blindly to everything. Internalize the reasoning, not just the rule. The cited, full-detail version of the underlying law (Law 2, Chunking Primacy) is in `references/laws.md` — read it when you need the numbers, evidence, or exact violation symptom.

1. **Chunk first, tune later — chunking is the binding constraint, so debug it before any other retrieval parameter.** No embedding quality, ANN recall, or reranker precision can reconstruct information severed at a chunk boundary. The 36-strategy benchmark shows a 1.88× nDCG gap from chunking alone — larger than any model or reranker improvement. If retrieval quality is poor, eliminate chunking as a cause *first*: inspect boundaries for mid-sentence cuts and fragments before investigating models, indexes, or rerankers. Symptom of violation: tuning MRL dimensions or quantization while chunks still end mid-sentence. (Law 2, Failure R16.)

2. **Structural boundaries are your best friend — exhaust them before inventing your own.** Headings, paragraphs, sections, and table boundaries exist because the author already segmented the content semantically. Cutting at arbitrary offsets throws that away. Start with structural parsing; escalate to semantic chunking only when structural markers are absent or evaluation proves structural insufficient. Symptom of violation: deploying semantic (or, worse, fixed-character) chunking on documents that have clean heading hierarchies and paragraph breaks. (DR1, DR5–DR6.)

3. **Every chunk must stand alone — a fragment can't be retrieved or used.** The embedding model encodes whatever text it receives; if that's a sentence fragment, the vector represents a fragment, and an LLM that retrieves it can't use it. The acid test: *can a human understand what this chunk is about without reading the surrounding text?* If not, no retrieval system can. Symptom of violation: chunks that begin or end mid-sentence, reference "as discussed above" with no antecedent, or are a single bullet ripped from a list. (Failure V23, DR13.)

4. **Domain is destiny — there is no universal best strategy.** The best strategy for legal briefs fails for product catalogs. Legal/structured → PGC (paragraph-group). Narrative/technical → DFC (dynamic token). Product catalogs → attribute-aware structural (the signal is in the fields). Code → function/class boundaries. CJK → script-aware splitting. The domain picks the strategy; a single global strategy across heterogeneous content produces wildly uneven nDCG. Symptom of violation: one chunking strategy applied corpus-wide with no per-content-type justification. (DR1–DR4, Failure V24.)

5. **Overlap is insurance, not a cure — it masks boundary problems, it doesn't fix them.** If chunks end mid-sentence, overlapping 20% just puts the fragment in two chunks instead of one — both still fragments. Overlap helps only when boundaries are already good and retrieval benefits from redundant coverage. Treat it as a tuning parameter to optimize, never as a fix for bad boundaries. Symptom of violation: increasing overlap to address retrieval failures without first examining boundary quality. (DR9.)

6. **Chunk size distribution must be measured, not assumed.** "512-token chunks" is a target, not a reality — actual sizes vary, some 200, some 800. If P95 exceeds the embedding model's max input length, those chunks are silently truncated and lose information; if P5 is tiny, those chunks lack context. Measure the distribution (mean, median, P95, P99) and verify it fits the model's optimal range. The distribution shape tells you whether the strategy is working. Symptom of violation: knowing only the target size, with no measurement of the actual spread. (DR7–DR8.)

7. **Compound losses must be anticipated — optimize chunking before any compression.** Chunking quality and quantization precision compound *multiplicatively*. Quantization measured at 94–98% recall on clean academic chunks can drop below 80% on poorly chunked production text — the discarded bits may have been the only signal separating relevant from irrelevant. You cannot tune your way out of bad boundaries. Optimize chunking, validate it, *then* compress. Symptom of violation: deploying int8 quantization or MRL reduction without first validating production chunking quality. (Failure V22, DR15–DR17.)

## The design procedure

Work these stages in order — early stages produce the inputs later ones depend on, and choosing a strategy before you've inventoried content types, or a size before you know the model's input length, is how chunking goes wrong. Scale depth to the problem: a single-content-type catalog collapses several stages to a sentence, but *consciously visit each one*. `references/design-inputs.md` details what to extract for each input and why it changes the design — consult it when the brief is thin.

### A — Content-type inventory
Identify and categorize every content type in the corpus before choosing any strategy (skipping this is how Failure V24 happens). List the distinct document types (legal briefs, product listings, code files, narrative articles, tables, CJK text). For each: is structural markup present? heading hierarchy? paragraph breaks? tables? code blocks? Estimate the proportion of each type. Flag CJK / non-Latin script content and its proportion. Note any multimodal content (defer it — see `[[Multimodal Retrieval]]`). **Output → Content-Type Inventory** (a table: content type, structural markup present, proportion of corpus, chunking implication).

### B — Embedding model constraints check
Read the embedding model's limits as a *constraint* (you don't choose the model — that's `[[Retrieval Pipeline Design]]`). Look up max input token length and the optimal input length (usually 80–90% of max, not the max itself). Check whether the model supports MRL. If non-MRL: note that truncation causes sharp quality cliffs (>10% nDCG loss below ~256 dim — Failure V3) and that sub-256 reduction is a hard reject (DR11). If MRL: note the dimension sweep is available for later storage optimization. **Output → Model Constraints** (max tokens, optimal tokens, MRL yes/no, truncation risk level).

### C — Strategy selection per content type
Run the decision tree for *each* content type from Stage A (the full rules are in `references/decision-rules.md`):
1. Parseable structural boundaries (headings, paragraphs)? → **structural-first** (parse headings, subdivide paragraphs within sections) (DR1). Else →
2. Narrative/technical prose? → **DFC** (Dynamic Token Chunking) with semantic boundary detection (DR2). Else →
3. Highly structured content (legal, academic, specs)? → **PGC** (Paragraph Group Chunking) (DR3). Else →
4. Code? → **function/class/method boundaries**, imports carried as context. Else →
5. CJK / non-Latin script? → **script-aware splitter or character-level n-grams**, never token-count sizing (DR4). Else →
6. **Semantic chunking as the final fallback** only (DR6).

Two hard rejections ride alongside the tree: **fixed-character chunking → REJECT always** (DR5, Failure V21), and **semantic-as-default → REJECT** (DR6) — semantic is a fallback, not the starting point. Product catalogs are a named special case: **attribute-aware structural** (concatenate SKU + name + description as chunk text, keep specification fields as structured metadata). **Output → Strategy Selection per Content Type** (the strategy chosen for each type, justification, risks, evaluation plan).

### D — Size and overlap configuration
Configure per content type, driven by the downstream consumer (Pattern Seed 13), not by convention. Set the target size within the model's optimal range (typically 256–1024 tokens). Set overlap as a percentage of target (5–20%; >20% → flag, DR9). Short content below the target → keep as a single chunk, don't pad. Long content → ensure P95 fits within max input length (DR7). CJK → character-level sizing, not token count (DR10). Code → logical boundaries (function length), not token count. **Output → Chunk Size and Overlap Configuration** (per-content-type target size, overlap %, rationale, embedding-model fit check).

### E — Metadata schema design
Define what every chunk carries — this is what makes failures diagnosable later (Pattern Seed 24). Required: source_document_id, chunk_position (ordinal), heading_path (if structural), content_type tag. Recommended: language tag (multilingual corpora), creation_timestamp. For multi-hop / context expansion: parent-child links and adjacent-chunk retrieval support (this prevents Failure R6; note that the multi-hop *pipeline* is `[[Retrieval Pipeline Design]]`'s). For failure diagnosis: log retrieved chunk IDs and positions. **Output → Metadata Schema** (required + recommended fields, each tied to traceability or failure diagnosis).

### F — Quality validation (in isolation)
Define how to prove the chunks are good *before* production, holding everything else fixed — same embedding model, same reranker, vary only chunking (DR15). Human spot-check: 20 random chunks across content types — independently interpretable? (DR13; >10% failing = strategy is failing). Coherence check (automated): do chunks end on a sentence terminator, or mid-sentence? (DR14). Size-distribution measurement: P5/P25/P50/P75/P95/P99 — within ±50% of target? (DR8). Boundary-failure rate: for a known query set, what fraction of retrieval failures trace to boundary position? (>5% = chunking is the bottleneck, DR12). If a structural strategy was chosen, run the **PGC-vs-FCC baseline**: compare nDCG@5 against a fixed-character baseline on the *same* model + reranker — expect >1.5× improvement. Note: this isolated comparison is owned here; the broader eval program is `[[Evaluation Operations]]`. **Output → Quality Validation Plan** (validation methods, acceptance thresholds, remediation for failures).

### G — Chunk-to-answer traceability instrumentation
Design the instrumentation that makes a boundary failure visible — without it, chunking failures masquerade as embedding failures (Failure R16). For every RAG answer log: which chunks were retrieved and at what rank; whether the expected correct chunk was in the retrieved set and where; whether the chunk contained enough information to answer; whether the LLM used it correctly. This enables root-cause triage — retrieval failure vs boundary failure vs generation failure. The instrumentation is owned here; the evaluation that consumes it is `[[Evaluation Operations]]`'s. **Output → Chunk-to-Answer Traceability Design** (logged fields, query interface, diagnostic workflow).

## Decision gates (apply continuously)

These fire the moment the trigger appears in the user's proposal — gates, not gentle suggestions. Full rationale and the cited failure/law behind each (DR1–DR17) are in `references/decision-rules.md`.

| Gate | When it fires | What you do |
|------|---------------|-------------|
| Fixed-character chunking | any proposal to cut at a fixed byte/character offset | REJECT outright — worst performer in the benchmark, mid-sentence boundaries (DR5, V21) |
| Semantic-as-default | semantic chunking chosen without trying structural | REJECT as default; structural-first, semantic only as fallback (DR6) |
| No content-type inventory | one global strategy proposed for a mixed corpus | stop; inventory content types and justify a strategy per type (DR1–DR4, V24) |
| Chunk-first violation | embedding-model/reranker swap proposed to fix bad retrieval | check chunk boundaries first; don't change the model until chunking is cleared (Law 2, R16) |
| P95 over model max | size distribution P95 exceeds max input length | longest chunks are truncated; lower target or fix the long tail (DR7) |
| Size skew | P5 or P95 outside ±50% of target | document and diagnose the skew before trusting the strategy (DR8) |
| Overlap over 20% | configuration with >20% overlap | flag — overlap is masking boundaries, not fixing them; inspect boundary quality (DR9) |
| CJK token sizing | token-count sizing applied to CJK content | switch to script-aware / character-level sizing (DR4, DR10) |
| Non-MRL sub-256 | non-MRL model truncated below 256 dim | REJECT — sharp quality cliff (DR11, V3) |
| Non-interpretable chunks | >10% of chunks fail the standalone test | strategy is failing; move to structural/semantic boundaries (DR13, V23) |
| Mid-sentence boundaries | chunks don't end on a sentence terminator | chunking quality is low; fix the strategy (DR14, V21) |
| Quantize before chunk validation | quantization/MRL proposed before chunk quality proven | validate chunking first; measure quantization tolerance on production chunks (DR15–DR17, V22) |

## Failure modes to catch

Watch for these patterns when designing or reviewing. Full symptom / cause / prevention / law for each is in `references/failure-modes.md`.

- **V21 — Fixed-character chunking destroys quality.** nDCG@5 = 0.244 (worst of 36), chunks end mid-sentence, no model fixes it. → REJECT FCC; structural-first, or recursive paragraph→sentence splitting for unstructured text.
- **V22 — Chunking × quantization compound multiplicatively.** Benchmark quantization recall overstates production; 94% → <80% on poorly chunked text. → optimize chunking first; measure quantization tolerance on production chunks, not academic ones.
- **V23 — Chunks that don't stand alone.** Retrieval returns fragments; LLM is confidently wrong. → human standalone-interpretability spot-check; semantic boundaries; the acid test.
- **R6 — Multi-hop fragmentation.** >15% of cross-document queries fail; the answer spans unlinked chunks. → parent-child + adjacent-chunk metadata; track multi-hop volume; defer the pipeline to `[[Retrieval Pipeline Design]]`.
- **V24 — Domain-generic chunking on specialized corpora.** One global strategy → wildly uneven nDCG across domains. → content-type inventory; per-content-type strategy and validation.
- **R16 — Misdiagnosing chunking as embedding failure.** Hours spent tuning models when boundaries are the cause. → traceability instrumentation; boundary-failure-rate measurement; diagnostic order chunking → embedding → indexing → reranking.

## Output: the design artifact

Produce these sections. Scale depth to the problem — a single-content-type catalog needs a few tight sentences per section, not pages — but address every one, even if only to say "not applicable, because…". Saying *why* a section is N/A is itself a recorded design decision.

1. **Content-Type Inventory** — all content types, structural-markup catalog, CJK/non-Latin noted, proportions estimated (Stage A)
2. **Model Constraints** — max tokens, optimal tokens, MRL support, truncation risk (Stage B)
3. **Strategy Selection per Content Type** — the strategy for each content type, with justification and risks; fixed-character explicitly rejected; semantic only if structural/DFC/PGC are exhausted (Stage C)
4. **Chunk Size and Overlap Configuration** — per-content-type target size, overlap %, embedding-model fit check (Stage D)
5. **Metadata Schema** — required and recommended fields, traceability + parent-child links (Stage E)
6. **Quality Validation Plan** — spot-check, coherence, size distribution, boundary-failure rate, PGC-vs-FCC baseline; thresholds and remediation (Stage F)
7. **Chunk-to-Answer Traceability Design** — logged fields, query interface, diagnostic workflow (Stage G)
8. **Failure Mode Prevention Checklist** — which of V21/V22/V23/R6/V24/R16 this design guards against, and how
9. **Evaluation Isolation Plan** — how to evaluate chunking independently: same model, same reranker, vary only chunking; hand the methodology to `[[Evaluation Operations]]`
10. **Deferred Specialized Decisions & Known Risks** — what you don't own, cross-referenced with `[[wikilinks]]` + placeholder assumptions (embedding model, reranker, fusion, eval program, multi-hop pipeline, multimodal); plus the likeliest risks for *this* corpus and their mitigations

Note the gaps the corpus doesn't fill, when relevant: **CJK chunking** is supported by a single mechanism — give the "script-aware splitter, not token-count sizing" rule but flag deeper CJK guidance as lower-confidence; **late chunking / contextual document embeddings** is a promising technique not yet harvested — mention it as future, don't design on it; **multimodal chunking** is deferred to `[[Multimodal Retrieval]]`. Name the gap; don't pretend to fill it.

## Worked judgment calls

**"Design a chunking strategy for RAG over 50,000 legal briefs — they have Case Name → Section → Subsection → Paragraph hierarchies."** Structural-first, the textbook case (DR1). Parse the heading hierarchy, subdivide by paragraph within sections — this is PGC territory because legal arguments are organized in paragraph units (DR3). Target 512–768 tokens per paragraph group, within the model's optimal range (Stage B). Heading-path metadata on every chunk (Case → Section → Subsection) so retrieval can expand context and cite. No semantic chunking unless evaluation proves structural insufficient (DR6). Fixed-character chunking explicitly rejected (DR5). The trap to avoid: defaulting to "512 tokens with 10% overlap" with no structural parsing — that throws away the author's segmentation on a corpus that hands it to you for free.

**"Set up chunking for product search."** Different strategy entirely — this is the anti-V24 case. Product listings are structured fields (SKU, name, description, specifications), so use **attribute-aware structural** chunking: concatenate SKU + name + description as the chunk text, keep specification fields as structured metadata for filtering, not as prose to embed. Fixed-character rejected. Say explicitly that this is a *different* strategy from a legal/narrative corpus — applying paragraph-group or a "default 512-token" here would bury the high-signal fields (DR1–DR4). The downstream consumer (a shopper scanning results) wants the identifying fields tight and findable, not a 512-token blob.

**"Configure chunking for these Chinese-language research papers (mixed CJK and English terminology)."** Script-aware splitting, not token-count sizing — token targets are language-biased and mis-size CJK (DR4, DR10). Use character-level sizing for the CJK segments, NFC normalization for token efficiency, and structural boundaries (headings, paragraphs) for semantic coherence where the papers provide them. Reject Latin-optimized token-count sizing explicitly. And flag the honesty caveat: CJK corpus support is a single mechanism — this is the right rule, but deeper CJK strategy guidance is lower-confidence, so recommend validating on a CJK query set before committing.

**"Our RAG retrieval is bad — chunks are 256-token fixed-size, no overlap. Should we switch embedding models?"** No — check the chunks first (Law 2, R16). 256-token *fixed-size* is the signature of the worst-performing strategy in the benchmark (V21); inspect a sample of boundaries — if they end mid-sentence, chunking is the root cause, full stop. Recommend a structural-first redesign and measure nDCG on current chunking vs a structural baseline on the *same* model and reranker (DR15). Compute the chunk-boundary failure rate. Do **not** recommend a model swap until chunking is cleared — that's exactly the misdiagnosis that burns hours pulling the wrong lever.

**"We want to deploy int8 quantization to save storage — current chunking is 512-token structural. Safe?"** Verify chunking quality first (DR15). If the boundary-failure rate is under 5% and chunks pass the standalone test, int8 is reasonable — but measure quantization tolerance on *production* chunking quality, never on MTEB-style academic benchmarks (DR17). The benchmark's 94–98% recall retention is measured on clean chunks and overstates what you'll get; chunking and quantization losses compound multiplicatively (V22). If the boundary-failure rate is above 5%, fix chunking before quantizing — you'd be compressing away the only signal that was separating relevant from irrelevant.

**"How should we chunk documents for multi-hop RAG?"** Chunking *enables* multi-hop through metadata; it doesn't design the multi-hop loop. Put parent-child links and adjacent-chunk retrieval support in the metadata schema (Stage E) so answers spanning multiple chunks can be reassembled (prevents Failure R6). Configure chunk-to-answer traceability to track multi-hop success specifically. Then state the boundary clearly: the multi-hop retrieval *methodology* (retrieve → read → retrieve more) is owned by `[[Retrieval Pipeline Design]]` — flag the assumption that it will handle the iterative logic, and that it must be verified before implementation. Designing the full multi-hop pipeline inside the chunking artifact would cross the ownership line.
