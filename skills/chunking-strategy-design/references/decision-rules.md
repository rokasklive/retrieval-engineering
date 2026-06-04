# Decision rules — operational gates (DR1–DR17)

These are gates, not suggestions. Each has a rule and the trigger that fires it. Cite them by number in design artifacts. Grouped by concern.

## Strategy selection

| # | Rule | Trigger |
|---|------|---------|
| DR1 | **Structural boundaries present → structural-first chunking; never skip to semantic.** Headings, sections, and paragraphs are author-supplied semantic segmentation — exploit them before inventing boundaries. | Any content type with parseable headings or paragraphs |
| DR2 | **No structural boundaries + narrative/technical prose → DFC (Dynamic Token Chunking) with semantic boundary detection.** | Unstructured narrative/technical content (essays, articles, docs) |
| DR3 | **No structural boundaries + highly structured content → PGC (Paragraph Group Chunking).** Arguments are organized in paragraph units; group them. | Legal, academic, specifications |
| DR4 | **CJK or non-Latin script content → script-aware splitter or character-level n-grams; never token-count-based sizing.** | Any CJK / non-Latin content, regardless of other structural markers |
| DR5 | **Fixed-character chunking (FCC) → REJECT, regardless of content type.** Cutting at a byte/character offset ignores every semantic, syntactic, and structural boundary; it is the worst performer in the 36-strategy benchmark (nDCG@5 = 0.244). This is a hard rejection (Failure V21). | Any design proposing fixed-character / fixed-byte chunking |
| DR6 | **Semantic chunking as the default → REJECT; accept only as the fallback when structural + DFC + PGC are all unavailable.** Semantic chunking adds embedding cost and complexity; structural-first usually wins and is cheaper (Failure V23 prevention). | Any design defaulting to semantic chunking without exhausting structural options |

## Size and configuration

| # | Rule | Trigger |
|---|------|---------|
| DR7 | **Chunk-size distribution P95 must be ≤ embedding model max input length.** Above it, the longest chunks are silently truncated and lose information. | Any chunking configuration |
| DR8 | **Chunk-size variation beyond ±50% of target → document and diagnose the skew.** The distribution shape reveals whether the strategy is working; a long tail means boundaries aren't landing where intended. | P5 or P95 outside ±50% of target size |
| DR9 | **Overlap > 20% → flag; overlap is masking boundary problems, not solving them.** If chunks end mid-sentence, 20% overlap just puts the fragment in two chunks. Overlap is a tuning parameter for redundant coverage, not a cure for bad boundaries. | Any configuration with > 20% overlap |
| DR10 | **CJK content: never use token-count-based sizing → use character-level sizing.** Token-count sizing is language-biased; CJK produces more tokens per semantic unit, so token targets systematically mis-size CJK chunks. | CJK content present in the corpus |
| DR11 | **Non-MRL embedding model + truncation below 256 dimensions → REJECT.** Non-MRL models degrade sharply when truncated (Failure V3); only MRL models tolerate dimension reduction gracefully. | Non-MRL embedding model selected with sub-256-dim target |

## Quality gates

| # | Rule | Trigger |
|---|------|---------|
| DR12 | **Chunk-boundary failure rate > 5% → chunking is the bottleneck; fix it before tuning any other parameter.** | After initial deployment, with traceability instrumentation in place |
| DR13 | **> 10% of chunks not independently interpretable → the chunking strategy is failing.** A chunk a human can't understand in isolation cannot be retrieved or used by an LLM (Failure V23). | Human spot-check or automated coherence test |
| DR14 | **Mid-sentence chunk boundaries detected → chunking quality is low; the strategy must be fixed.** Mid-sentence splits are the signature of fixed-character chunking (Failure V21). | Automated boundary quality check (chunk does not end on a sentence terminator) |
| DR15 | **Chunking evaluation (same model, same reranker, vary only chunking) must precede any quantization or MRL optimization.** | Before any precision/compression optimization (Failure V22) |

## Compound optimization

| # | Rule | Trigger |
|---|------|---------|
| DR16 | **Optimize chunking before any compression strategy (quantization, MRL dimension reduction).** You cannot tune your way out of bad boundaries; compression only multiplies the loss. | Before applying any compound-optimization / compression stack |
| DR17 | **Quantization tolerance must be measured on production chunking quality, not academic benchmarks.** Benchmark quantization recall (94–98%) is measured on clean academic chunks and overstates production tolerance on poorly chunked text (Failure V22). | When evaluating quantization for deployment |
