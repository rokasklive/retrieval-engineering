# Retrieval Engineering

> Distilled skills for AI agents building production search pipelines.

## About

These skills were methodically synthesized from a focused corpus of literature — research papers, engineering blogs, and published media on semantic search, retrieval, and RAG systems — as part of an amateur research effort.

This is not settled science, a formal academic result, or a claim of best practice. It is a transparent, benchmarked attempt to distill recurring retrieval-engineering patterns into reusable agent skills.

**Corpus**: Hybrid Search Corpus v0.1

> **Note**: These skills are designed for the **design and verification phases** of search system development. They prioritize correctness, traceability, and thoroughness — expect higher token usage and longer runtimes than general-purpose skills. Token cost and latency were not design constraints.

**Generated**: 2026-06-04 15:15 UTC

## Skills

| Skill | Purpose |
|-------|---------|
| **[Chunking Strategy Design](skills/chunking-strategy-design/)** | Where to place chunk boundaries, what size and overlap, and how to validate chunk quality before tuning anything downstream. |
| **[Drift Management And Freshness](skills/drift-management-and-freshness/)** | Detecting embedding drift and index decay, enforcing model versions, and keeping production search from degrading. |
| **[Evaluation Operations](skills/evaluation-operations/)** | Building judged sets, picking the right metric, detecting when metrics lie, and gating whether a change ships. |
| **[Hybrid Search Architecture](skills/hybrid-search-architecture/)** | How to fuse lexical and dense results, choose fusion methods, and route queries to the right retrieval path. |
| **[Retrieval Pipeline Design](skills/retrieval-pipeline-design/)** | When dense, lexical, or hybrid retrieval is justified; when a reranker helps; how many retrieval stages you actually need. |

## Eval Results

See [`evals/README.md`](evals/README.md) for aggregated benchmark scores — per-skill breakdowns, before/after deltas, and per-eval detail.

## License

Copyright 2026 Rokas Kučinskas. Licensed under the [Apache License, Version 2.0](LICENSE).
