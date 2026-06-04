# Design inputs — what to extract and why it changes the design

Before designing a chunking strategy, gather these inputs. For each: what to extract, and why it moves a design decision. When the user's brief is thin, this is your checklist of what to ask for. Don't design on silent assumptions — if an input is unknown, name it as an assumption and flag it for verification.

## 1. Content types present
**Extract.** Categorize the documents by type: narrative prose (essays, articles), structured documents (legal contracts, academic papers with headings), tabular/structured (product catalogs, specifications), code (functions, classes), CJK / non-Latin text, multilingual content, multimodal (images + text, tables + text). Estimate the proportion of each.
**Why.** Content type determines the strategy *family* — this is the first and most important input. Legal → PGC. Narrative/technical → DFC. Code → function/class boundaries. CJK → script-aware splitter. Product catalog → attribute-aware structural. The wrong strategy can cost a 1.88× nDCG difference (Law 2). Drives Stage A and Stage C.

## 2. Embedding model input length
**Extract.** Maximum token length the embedding model accepts. Optimal input length for quality (usually not the maximum — often 80–90% of max). Whether the model supports MRL (Matryoshka Representation Learning).
**Why.** Chunk size must not exceed max input length, or the longest chunks (P95+) get silently truncated and lose information (DR7). The optimal range, not the max, is the real ceiling. Non-MRL models degrade sharply when truncated (Failure V3), which constrains later dimension reduction. Drives Stage B and Stage D.

## 3. Retrieval goal and downstream use
**Extract.** What consumes the retrieved chunks? An LLM for RAG (needs coherent, self-contained context)? A human reader (needs readable, citation-friendly segments)? Multi-hop retrieval (needs cross-chunk linking)? Single-answer lookup (needs precise targeting)? Summarization (needs comprehensive coverage)?
**Why.** Determines the chunk size and overlap tradeoff. RAG needs independently interpretable chunks with enough LLM context. Q&A needs smaller, precisely targeted chunks. Summarization tolerates larger chunks with lower specificity. Multi-hop needs parent-child links in the metadata schema. Design from what the consumer needs, not from "512 is standard" (Pattern Seed 13). Drives Stage D and Stage E.

## 4. Document structure availability
**Extract.** Do documents have parseable structural markers — heading hierarchies, section boundaries, paragraph breaks, tables with captions, code with function/class boundaries? Or are they unstructured prose with no internal organization?
**Why.** Structural-first requires parseable structure. Its presence or absence is the *first branch* of the strategy-selection decision tree (Stage C). Structure present → structural-first (DR1). No structure + narrative → DFC (DR2). No structure + highly structured content → PGC (DR3). No structure at all → semantic chunking as the last-resort fallback (DR6). Drives Stage C.

## 5. CJK and non-Latin script presence
**Extract.** The proportion of content in Chinese, Japanese, Korean, Arabic, or other non-Latin scripts. Whether token-count-based sizing is feasible or will produce systematically mis-sized chunks.
**Why.** Token-count sizing is language-biased: CJK characters produce more tokens per semantic unit than Latin scripts, so token targets mis-size CJK chunks, and BERT-style tokenizers split logograms non-semantically. Script-aware splitters or character-level n-grams are required, with NFC normalization for token efficiency (DR4, DR10). Drives Stage C and Stage D. Note: corpus support here is thin (single mechanism) — flag CJK guidance as lower-confidence.

## 6. Retrieval failure logs (if a production system exists)
**Extract.** For existing systems with failures: what fraction of retrieval failures trace to chunk boundaries? Are chunks ending mid-sentence? Are key passages split across boundaries? Are chunks independently interpretable in isolation?
**Why.** Existing failure patterns directly diagnose the current strategy's weaknesses. A chunk-boundary failure rate > 5% means chunking is the bottleneck (DR12). Mid-sentence splits confirm fixed-character chunking is in use (Failure V21). Non-interpretable chunks confirm a semantic-coherence violation (Failure V23). This is what separates a redesign from a tuning exercise. Drives Stage F and Stage G.

## 7. Metadata requirements
**Extract.** What metadata must each chunk carry? Source document ID? Position/offset? Heading path? Language? Content type? Version timestamp?
**Why.** Metadata enables chunk-to-answer traceability (Pattern Seed 24): without source + position you cannot determine whether a retrieval failure is a boundary problem (Failure R16). Heading path enables context expansion; language tag enables language-aware retrieval; parent-child links enable multi-hop reassembly (Failure R6). Drives Stage E.
