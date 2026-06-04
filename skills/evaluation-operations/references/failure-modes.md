# Failure modes — symptom, cause, prevention

The evaluation failures the corpus characterizes well. Each gives the symptom you'll observe, the underlying cause, the design prevention, and the governing law. Cite these by code when you flag a problem.

---

## R3 — No Eval Baseline

**Symptom.** The team can't answer "what's our quality?" User complaints are the only signal of degradation. When something breaks, there's no way to know what broke or how badly.

**Cause.** RAG systems built as prototypes without evaluation infrastructure. The system "works" in demos, so evaluation is deprioritized — until production reveals there was never a reference point.

**Prevention.** DR1 — establish the evaluation baseline as the FIRST step, before any retrieval change including the initial deployment. Minimum viable baseline: weekly canary-query MRR tracking. The deliverable is a runnable evaluation command and a recorded "NDCG@10 = X, Coverage@10 = Y on [date]."

**Law.** 12 (Eval-Gated Complexity) — without a baseline there is no symptom visibility, so no layer can be justified.

---

## EV1 — Annotation Selection Bias (the self-fulfilling "no value" prophecy)

**Symptom.** Dense retrievers appear worse than BM25 on benchmarks. Re-annotation reveals 10–30 percentile-point jumps. "The new method is bad" conclusions turn out to be artifacts of an old pool.

**Cause.** Test collections built via lexical pooling systematically miss the documents dense retrievers find. The pooler system labels what *it* finds; a new system finds unlabeled documents; those count as "not relevant." The bias is self-reinforcing — the pool is never updated, so the verdict never changes.

**Prevention.** DR5–DR6 — diverse pooling from ≥2 paradigms, pooled at depth N≈100 per system. Re-annotate the top unjudged documents when coverage drops. Report coverage per retrieval method, not just overall (DR8). Never conclude a dense retriever is worse on a lexical-only pool.

**Law.** 11 (Coverage Dependency).

---

## EV4 — Low Coverage Making Metrics Unreliable

**Symptom.** Coverage@10 < 0.50. Metrics look stable, but user-reported quality is declining. New methods show "worse" metrics specifically because they surface unjudged documents.

**Cause.** Most top-K results lack judgments. NDCG is structurally biased against retrievers that find documents outside the pool, and unjudged docs are counted as "not relevant" in precision@K, making it a lower bound rather than a measurement.

**Prevention.** DR2–DR3 — Coverage@K reported alongside every metric; <0.50 → flag UNRELIABLE and do not decide on it. Pool at depth N≈100, supplement with diverse pooling, re-annotate holes.

**Law.** 11 (Coverage Dependency).

---

## EV5 — Single-Signal Evaluation Hiding Per-Signal Failures

**Symptom.** Aggregate NDCG looks good but specific query types are failing. Exact-identifier queries have 0% recall while semantic queries are excellent. Aggregate "green" masks per-signal "red."

**Cause.** Evaluating with a single aggregate metric without decomposing by query type, signal, or domain. A system with strong semantic NDCG can have 0% recall on exact IDs and still post an acceptable aggregate — the average launders the disaster.

**Prevention.** DR14 — decompose by query type (exact, keyword, semantic, entity), by signal (precision, recall, MRR, coverage), and by domain. Flag any signal >20% below aggregate. Green blended + red per-signal = REJECT the change.

**Law.** 7 (Evaluation Gap) — aggregate offline numbers diverge from per-class production reality.

---

## EV9 — Offline/Online Gap Causing Wrong Decisions

**Symptom.** Changes with improved offline metrics fail A/B tests. Business metrics don't correlate with NDCG/MRR. AUC inversion: the better offline model (0.92) produced the worse business outcome ($1,840/mo vs $2,320/mo for the 0.78 model).

**Cause.** Three structural gaps: annotation selection bias (pools miss what dense finds), coverage scarcity (unjudged = invisible), and business-vs-relevance mismatch (a click is not a relevance judgment). Offline metrics measure relevance against a fixed pool; the business measures user behavior.

**Prevention.** DR4, DR17, DR18–DR19 — offline as filter only; online A/B for all shipping decisions; track offline-online correlation and act when it drops below 0.5; canary-query MRR as the continuous signal between formal evals.

**Law.** 7 (Evaluation Gap).

---

## R15 — Demo-to-Production Gap

**Symptom.** Systems with perfect demo behavior achieve 8.4% confidence and 31.2% accuracy on real queries. Demo queries were engineered to show the system at its best; real users exercise every edge case.

**Cause.** Evaluation query sets built from curated examples instead of sampled from production. Demos use queries that match the system's strengths; real users ask unpredictable, ambiguous, edge-case, and typo-laden questions.

**Prevention.** DR16 — PPS (probability-proportional-to-size) sampling from production logs. Never use demo-query performance as quality evidence. Include typo, edge-case, and ambiguous queries in the judged set. Add production failures to the eval set monthly so it gets harder, not easier.

**Law.** 12 (Eval-Gated Complexity).

---

## EV-judge — Uncalibrated LLM-as-Judge

**Symptom.** An LLM judge reports a confident score ("faithfulness = 0.92") that overstates quality. Degradation goes undetected because the judge keeps approving plausible-but-wrong answers.

**Cause.** LLM judges share the generator's knowledge boundaries, so they can't catch domain errors they don't know about. They are systematically lenient on plausible-but-wrong answers, exhibit position bias (prefer the first result), and show self-enhancement bias (inflate the same model's output).

**Prevention.** DR15 — weekly calibration against a human gold set (50–100 query-result pairs). Use a *different* judge model than the generator. Randomize document order. Multi-sample vote (3–5, majority). Track human correlation; below ~0.85, recalibrate or suspend the judge for decisions. Never let an uncalibrated judge be the sole arbiter.

**Law.** 7 (Evaluation Gap) — an uncalibrated judge is another offline proxy that diverges from ground truth.
