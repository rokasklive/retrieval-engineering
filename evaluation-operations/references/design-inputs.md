# Design inputs — what to extract and why it changes the design

Before designing an evaluation program, gather these inputs. For each: what to extract, and why it moves a design decision. When the user's brief is thin, this is your checklist of what to ask for. Don't design on silent assumptions — if an input is unknown, name it as an assumption and flag it for verification.

## 1. Search task type
**Extract.** What task does the user actually perform? Navigational (find a known item), informational (learn about a topic), transactional (complete an action), known-item lookup, comprehensive research, entity resolution.
**Why.** Task type *determines the metric*. Single correct answer → MRR. Graded relevance, general search → NDCG@10. Binary relevance + full recall → MAP. User scans and stops → ERR. The wrong metric produces confidently misleading conclusions. This is the most important input for Stage C.

## 2. Query distribution
**Extract.** The actual query-log distribution — not curated examples. Which types are present (exact identifiers, short keywords, natural language, entity lookups, typo-afflicted), the head-vs-tail frequency distribution, and the language distribution.
**Why.** The judged set must mirror production, not a demo. PPS sampling from real logs is what keeps evaluation honest; demo queries engineered to flatter the system miss real failure modes (Failure R15: 8.4% confidence, 31.2% accuracy on real queries). Drives Stage A.

## 3. Retrieval paradigms in use
**Extract.** Which retrieval methods are deployed or planned — BM25/lexical, dense, hybrid, fuzzy, SPLADE? Are they independently measurable? Can each be queried in isolation?
**Why.** Judgment pools must be built from ALL active paradigms. A lexical-only pool systematically disadvantages dense retrievers (5–10× higher annotation hole rates; Failure EV1). Diverse pooling (≥2 paradigms) is mandatory for a fair evaluation. Drives Stage B.

## 4. Existing evaluation infrastructure
**Extract.** Does a judgment list exist? What metrics are tracked? What's the current Coverage@K? When was the last eval run? Can any team member run `eval` and get a single quality number? Is there a baseline before any change?
**Why.** Sets the starting point. No judgment list → start with construction. Coverage unknown → measure it immediately. No baseline → establish it before ANY other work — that's the root cause under nearly every production failure (Failure R3). Drives whether Stage A is the whole project.

## 5. Annotation budget and cadence
**Extract.** How many queries can be judged? By whom (domain experts, crowd-sourced, author-as-judge)? At what cadence (weekly, monthly, quarterly)? What relevance scale (binary 0/1 vs graded 0–3)? Is an LLM judge available? Is there a human-calibration budget?
**Why.** Budget determines feasibility. <50 queries → focus on canary queries for continuous monitoring. 50–200 → judgment-list construction + canary. 200+ → a full evaluation program. Graded relevance (0–3) costs more annotation effort but captures the resolution hybrid search needs (DR7). Shapes Stages B, D, E.

## 6. Online A/B testing feasibility
**Extract.** Is online A/B testing possible? Is there enough traffic for statistical significance? Are business metrics (conversion, revenue, retention) defined? Is interleaving available? Are there regulatory constraints (healthcare, legal may forbid live experimentation)?
**Why.** Offline metrics are FILTERS, not DECISION GATES (Law 7). If A/B is impossible, offline evaluation must be maximally rigorous — highest possible coverage, ≥2 pooling paradigms, a human gold set — and the residual risk flagged explicitly (DR20). The offline-online gap can't be *closed* without online testing, only minimized. Drives Stage F.

## 7. Canary query requirements
**Extract.** Are there 20–50 queries with known correct answers? Can they be maintained as a fixed reference set? Is weekly tracking feasible? Who maintains the set?
**Why.** Canary queries are the continuous production quality signal — they detect positional degradation standard monitoring misses (MRR drifted 0.85 → 0.62 over six months with zero alerts). You design and baseline them here; operation is owned by `[[Drift Management & Freshness]]`. Drives Stage D.
