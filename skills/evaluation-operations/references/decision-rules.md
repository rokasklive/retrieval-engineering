# Decision rules — operational gates (DR1–DR20)

These are gates, not suggestions. Each has a rule and the trigger that fires it. Cite them by number in design artifacts. Grouped by concern.

## Evaluation foundation

| # | Rule | Trigger |
|---|------|---------|
| DR1 | **No eval baseline → establish it before ANY retrieval change.** A runnable evaluation command and a recorded baseline are the first deliverable; without them every change is noise (R3, Law 12). | Any system with no command that returns a single quality number |
| DR2 | **Coverage@K must be reported alongside every quality metric — no exceptions.** A bare metric is not a result. | Every evaluation run, every metric report |
| DR3 | **Coverage@10 < 0.50 → flag all metrics UNRELIABLE; do not use them for decisions.** Re-pool / re-annotate first (Law 11, EV4). | Top-10 coverage below threshold |
| DR4 | **Offline metrics are FILTERS (block regressions), never DECISION GATES (ship improvements).** | Every shipping decision |

## Judgment pool construction

| # | Rule | Trigger |
|---|------|---------|
| DR5 | **Judgment pools must include results from ≥2 retrieval paradigms.** The pooling system defines "relevant"; one paradigm = self-fulfilling prophecy (Law 11, EV1). | Any new judgment-pool construction |
| DR6 | **Lexical-only pool evaluating a dense retriever → REJECT; re-pool with diverse sources.** Dense suffers 5–10× higher annotation hole rates (EV1). | A dense retriever judged against a BM25-only pool |
| DR7 | **Graded relevance (0–3), not binary, for hybrid / multi-path search.** Binary can't capture the match-quality differences fusion turns on. | Multiple retrieval paths with differing match quality |
| DR8 | **Report Coverage per retrieval method, not just overall.** A healthy aggregate can hide one starved path. | Any evaluation with ≥2 paradigms |
| DR9 | **Unjudged documents must NOT be treated as non-relevant in precision@K.** Unjudged is unknown, not bad (Law 11). | Unjudged documents present in top-K |

## Metric selection

| # | Rule | Trigger |
|---|------|---------|
| DR10 | **Single correct answer → MRR primary, P@1 secondary.** | Known-item search, factoid QA, canary queries |
| DR11 | **Graded relevance, general search → NDCG@10 primary, ERR secondary.** | General/hybrid/RAG search |
| DR12 | **Binary relevance, full recall → MAP primary, Recall@K secondary.** | E-discovery, patent search, comprehensive research |
| DR13 | **User scanning/stopping → ERR primary.** | Navigational search, autocomplete, typeahead |

## Evaluation quality

| # | Rule | Trigger |
|---|------|---------|
| DR14 | **Aggregate metric green + any per-signal metric red (>20% below aggregate) → REJECT the change.** A class has collapsed under the average (EV5). | Per-signal decomposition shows a red signal |
| DR15 | **LLM-as-Judge as sole arbiter without weekly human calibration → REJECT.** Use a different judge model than the generator, randomize order, multi-sample vote; human correlation <0.85 → suspend (EV-judge). | Evaluation relying exclusively on LLM judges |
| DR16 | **Demo / curated query performance as quality evidence → REJECT.** PPS-sample from production logs (R15). | A quality claim citing demo performance |
| DR17 | **Offline-online correlation < 0.5 → offline metrics not predictive; increase coverage, diversify pooling, recalibrate.** | After 3+ A/B tests showing weak correlation |

## Shipping

| # | Rule | Trigger |
|---|------|---------|
| DR18 | **Offline improvement → proceed to A/B test; offline regression → REJECT.** | Every proposed retrieval change |
| DR19 | **Ship only if the A/B test shows a business-metric improvement with statistical significance.** | Every shipping decision |
| DR20 | **No A/B feasible → maximize offline rigor (≥2 pooling paradigms, Coverage ≥0.90, human gold set) and flag that offline alone cannot guarantee a production improvement.** | Domains where live experimentation is prohibited (healthcare, legal) or traffic is too low |

## Canary design (this skill owns design; operation → `[[Drift Management & Freshness]]`)

| # | Rule | Trigger |
|---|------|---------|
| DR-canary | **Design 20–50 fixed queries with known answers spanning ALL query types (not just easy ones); baseline MRR per-query and aggregate; >7% week-over-week MRR drop = investigate.** Standard monitoring (latency, errors, throughput) catches none of retrieval's positional degradation. | Any production retrieval system without a canary set |
