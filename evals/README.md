# Eval Results — Retrieval Engineering

> Generated: 2026-06-04 15:15 UTC
> Repository: [https://github.com/rokasklive/retrieval-engineering](https://github.com/rokasklive/retrieval-engineering)

## Aggregate Scores

| Skill | With Skill | Without Skill | Δ (pp) | Assertions |
|-------|-----------|---------------|--------|------------|
| **chunking-strategy-design** | 100.0% | 84.1% | +15.9 | 41 |
| **drift-management-and-freshness** | 100.0% | 48.8% | +51.2 | 41 (+12 partial) |
| **evaluation-operations** | 100.0% | 83.0% | +17.0 | 44 |
| **hybrid-search-architecture** | 100.0% | 85.9% | +14.1 | 39 |
| **retrieval-pipeline-design** | 100.0% | 92.9% | +7.1 | 42 |

**Overall**: 100% with-skill across 207 assertions. Without-skill (weighted avg): 79.0%.

## Changes Since Last Export

Previous: 2026-06-04 15:07 UTC
Current:  2026-06-04 15:14 UTC

| Skill | Previous (w/o skill) | Current (w/o skill) | Δ |
|-------|----------------------|---------------------|---|
| chunking-strategy-design | 84.1% | 84.1% | +0.0pp |
| drift-management-and-freshness | 48.8% | 48.8% | +0.0pp |
| evaluation-operations | 83.0% | 83.0% | +0.0pp |
| hybrid-search-architecture | 85.9% | 85.9% | +0.0pp |
| retrieval-pipeline-design | 92.9% | 92.9% | +0.0pp |

## Per-Skill Breakdown

### chunking-strategy-design

| Eval | With Skill | Without Skill | Δ | Note |
|------|-----------|---------------|----|------|
| 1: legal-corpus-structural-first | 6/6 (100.0%) | 4.0/6 (66.7%) | 2.0 | ... |
| 2: product-catalog-attribute-aware | 5/5 (100.0%) | 5.0/5 (100.0%) | 0.0 | ... |
| 3: cjk-script-aware | 5/5 (100.0%) | 2.5/5 (50.0%) | 2.5 | ... |
| 4: diagnose-before-swapping-model | 5/5 (100.0%) | 5.0/5 (100.0%) | 0.0 | ... |
| 5: quantization-validate-chunking-first | 5/5 (100.0%) | 4.5/5 (90.0%) | 0.5 | ... |
| 6: multi-hop-metadata-defer-pipeline | 5/5 (100.0%) | 2.5/5 (50.0%) | 2.5 | ... |
| 7: overlap-is-not-a-cure | 5/5 (100.0%) | 5.0/5 (100.0%) | 0.0 | ... |
| 8: semantic-not-the-default | 5/5 (100.0%) | 5.0/5 (100.0%) | 0.0 | ... |

---

### drift-management-and-freshness

*Scoring: Pass/Fail/Partial (partial = 0 credit)*

| Eval | With Skill | Without Skill | Δ | Note |
|------|-----------|---------------|----|------|
| 1: new-system-set-up-drift-monitoring | 7/7 (100%) | 1/7 (14%) | 6 | The with_skill output is a clean sweep — all 7 assertions pass, delivering a com... |
| 2: parameter-inflation-not-a-fix | 6/6 (100%) | 5/6 (83%) | 1 | Both outputs correctly reject ef_search=120 as a permanent baseline and prescrib... |
| 3: blue-green-model-migration | 6/6 (100%) | 2/6 (33%) | 4 | The with_skill output achieves a perfect score (6/6 PASS), demonstrating compreh... |
| 4: cosine-shift-without-mrr-drop | 5/5 (100%) | 2/5 (40%) | 3 | The with_skill output delivers a clean sweep — all five assertions are fully sat... |
| 5: cold-tier-staleness-check | 4/4 (100%) | 3/4 (75%) | 1 | Both outputs deliver strong, substantive answers to the cold-tier staleness ques... |
| 6: detect-existing-mixed-version-index | 5/5 (100%) | 4/5 (80%) | 1 | Both outputs correctly diagnose the mixed-version index root cause, explain the ... |
| 7: undeclared-freshness-sla | 4/4 (100%) | 3/4 (75%) | 1 | With_skill produced a comprehensive drift-management design artifact that treats... |
| 8: ownership-boundary-defense | 4/4 (100%) | 0/4 (0%) | 4 | This eval tests the skill's ability to defend its ownership boundary — recognizi... |

---

### evaluation-operations

| Eval | With Skill | Without Skill | Δ | Note |
|------|-----------|---------------|----|------|
| 1: eval-1 | 6/6 (100.0%) | 4/6 (67.0%) | 2 | ... |
| 2: eval-2 | 6/6 (100.0%) | 5/6 (83.0%) | 1 | ... |
| 3: eval-3 | 5/5 (100.0%) | 5/5 (100.0%) | 0 | ... |
| 4: eval-4 | 6/6 (100.0%) | 4/6 (67.0%) | 2 | ... |
| 5: eval-5 | 6/6 (100.0%) | 6/6 (100.0%) | 0 | ... |
| 6: eval-6 | 5/5 (100.0%) | 4/5 (80.0%) | 1 | ... |
| 7: eval-7 | 4/4 (100.0%) | 4/4 (100.0%) | 0 | ... |
| 8: eval-8 | 6/6 (100.0%) | 4/6 (67.0%) | 2 | ... |

---

### hybrid-search-architecture

*Scoring: Per-assertion: 0 (not met), 1 (partially met), 2 (fully met). Totals normalized to percentages.*

| Eval | With Skill | Without Skill | Δ | Note |
|------|-----------|---------------|----|------|
| 1: dense-only-exact-identifiers | 100.0 | 90.0 | 1 | without_skill misses the structured 'fail in opposite directions' complementarit... |
| 2: raw-score-fusion | 100.0 | 80.0 | 2 | without_skill misses 'silent breakage on model upgrade' and doesn't gate score-l... |
| 3: third-path-no-complementarity | 100.0 | 90.0 | 1 | without_skill produces a strong rejection but uses failure-mode diagnosis rather... |
| 4: hybrid-below-best-single | 100.0 | 90.0 | 1 | without_skill is comprehensive on diagnosis but doesn't frame 'fused ≥ best sing... |
| 5: unpinned-vendor-fusiontype | 100.0 | 80.0 | 2 | without_skill centers the alpha shift (0.5→0.75) rather than the fusionType chan... |
| 6: pre-fusion-metadata-filter | 100.0 | 87.5 | 1 | without_skill recommends post-fusion filtering but doesn't explicitly state the ... |
| 7: labeled-data-cc-upgrade | 100.0 | 80.0 | 2 | without_skill misses CC/convex combination entirely as the specific labeled-data... |
| 8: agent-facing-confidence | 100.0 | 90.0 | 1 | without_skill doesn't present the normalized top-gap formula c(q) — recommends d... |

---

### retrieval-pipeline-design

| Eval | With Skill | Without Skill | Δ | Note |
|------|-----------|---------------|----|------|
| 1: small-corpus-lexical-enough | 6/6 (100.0%) | 6/6 (100.0%) | 0 | ... |
| 2: dense-only-with-exact-identifiers | 4/4 (100.0%) | 4/4 (100.0%) | 0 | ... |
| 3: raw-score-fusion | 5/5 (100.0%) | 4/5 (80.0%) | 1 | ... |
| 4: reranker-without-recall-baseline | 5/5 (100.0%) | 5/5 (100.0%) | 0 | ... |
| 5: arbitrary-chunking-legal | 5/5 (100.0%) | 4/5 (80.0%) | 1 | ... |
| 6: production-no-model-versioning | 5/5 (100.0%) | 4/5 (80.0%) | 1 | ... |
| 7: complex-case-justified-build | 7/7 (100.0%) | 7/7 (100.0%) | 0 | ... |
| 8: retrieval-vs-reranking-triage | 5/5 (100.0%) | 5/5 (100.0%) | 0 | ... |

---

## Export History

| Date | Skills Exported | Notes |
|------|----------------|-------|
| 2026-06-04 15:07 UTC | 5 | Update — 5 skills |
| 2026-06-04 15:14 UTC | 5 | Update — 5 skills |
