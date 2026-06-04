# Design inputs — what to extract and why it changes the plan

Before designing a drift-management plan, gather these inputs. For each: what to extract, and why it moves a decision. When the user's brief is thin, this is your checklist of what to ask for. Don't design on silent assumptions — if an input is unknown, name it as an assumption and flag it for verification.

## 1. Current production quality baseline
**Extract.** Current canary-query MRR (per-query and aggregate)? Cosine-similarity distribution (mean, variance)? Current ef_search / nprobe values? Last REINDEX date? Is `model_version` metadata on every vector? Index size and growth rate? When was the last baseline measured?
**Why.** You cannot detect drift without a baseline — the delta from baseline *is* the signal. If no baseline exists, establishing one is the entire first job (Stage A, DR1). Without `model_version` metadata, model-level drift diagnosis is impossible. Drives whether Stage A is the whole project.

## 2. Data velocity and access patterns
**Extract.** How fast does data change — new docs/day, updated docs/day? Is access Pareto-skewed? Which data is "hot" (frequently accessed, time-sensitive) vs. "cold" (archival, stable)?
**Why.** Determines freshness strategy and reindex cadence. High velocity → frequent reindex / CDC pipeline. Skewed access → hot/cold tiering (≈82% cost reduction). Uniform access → a flat strategy. Real-time needs → sub-second SLA; static archives → quarterly reindex suffices. Drives Stage D and Stage F.

## 3. Model upgrade plans
**Extract.** Are embedding-model upgrades planned, and when? Current model? Target model? Are vectors version-tagged? Is there a rollback procedure? What's the full-reindex time?
**Why.** Upgrades require blue-green index deployment (Stage E). Without `model_version`, mixed-version degradation is inevitable (Failure V25). Full-reindex time sets the migration window; rollback capability sets risk tolerance. Budget ~10–15% of embedding spend for reindexing. Drives Stage E.

## 4. Index infrastructure
**Extract.** Which ANN algorithm (HNSW, IVFFlat)? Which vector DB (pgvector, Weaviate, Milvus, Pinecone)? Index size (vector count, dimension, precision)? Update pattern (bulk, incremental, CDC)? Soft-delete support? Automatic compaction available?
**Why.** Degradation mode depends on index type. IVFFlat → centroid collapse (monitor centroid entropy). HNSW → soft-delete accumulation, graph degradation (monitor index size vs. live rows). pgvector → manual REINDEX required. Weaviate/Milvus → automatic compaction may handle some of this. Drives the structural signals in Stage B and the cure in Stage F.

## 5. Existing monitoring coverage
**Extract.** What monitoring exists today? Standard service monitoring (latency, error rate, throughput)? Any search-specific signals (canary MRR, cosine distribution, parameter inflation)? Alert thresholds? Drift-detection cadence? Who owns quality monitoring?
**Why.** Standard monitoring does NOT capture quality degradation (Law 8) — MRR dropped 0.85 → 0.62 with zero alerts. If only standard monitoring exists, drift is undetected, and closing that gap is the entire rationale for this skill. Drives Stage B.

## 6. Canary query infrastructure
**Extract.** Are canary queries already designed (design is owned by `[[Evaluation Operations]]`)? Baseline MRR established? Is weekly tracking operational? What's the degradation threshold? Who maintains the set?
**Why.** Canary MRR is the primary continuous quality signal. If no set exists, you can't *operate* one — flag the dependency to `[[Evaluation Operations]]` and proceed on a placeholder (e.g., "assume a 20–50 query set with baseline MRR"). You own operation; they own design. Drives Stage A and Stage B.

## 7. Reindexing cost and capacity
**Extract.** Full-reindex time? Incremental-reindex time? Embedding-API cost per reindex? Can a duplicate index fit in memory/disk during blue-green? Embedding-cache pre-warming capacity?
**Why.** Reindex cost determines the tiering strategy — a 48-hour full reindex makes hot/cold tiering essential. Blue-green needs duplicate infrastructure during migration. Embedding-cache cold-start causes latency spikes if not pre-warmed (Failure V31). Drives Stages D, E, F and the cache cold-start plan.
