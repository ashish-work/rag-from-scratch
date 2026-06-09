# Module 10 — Evaluation

> Track 1 (RAG) · Status: ✅ passed (2026-06-10) — Phase A complete, build gate unlocked

## TL;DR

- Every prior module ended in a **knob** (chunk size, k, hybrid α/RRF, rerank N, transform on/off, prompt). **You can't tune what you can't measure.** No eval = optimizing by vibes = silent regressions.
- RAG has **two independent failure stages** → measure them **separately**: **retrieval** (did we fetch the right chunks?) and **generation** (given context, was the answer good?).
- **Retrieval metrics:** recall@k (the big one), precision@k, MRR, nDCG. **Generation metrics:** faithfulness, answer relevance, context relevance (the RAGAS triad).
- **Golden Q&A set** = (question, ideal answer, relevant doc IDs) over *your* corpus — your regression test suite. 30–100 good ones > thousands of random.
- **Eval-first** beats vibes: turns choices into measured decisions, localizes failures, catches "fixed one query, broke ten."

## Why eval is first-class

Failures are often **silent** (Modules 1c, 4c, 9c). Two stages fail independently. The only way to know if a change helped — not just helped *the three queries you eyeballed* — is to measure. Build the harness alongside the pipeline, not after.

## A) Retrieval metrics — "did we fetch the right chunks?"

Needs ground-truth relevant docs per query.

- **Recall@k** — of the relevant docs, how many are in the top-k? **The most important RAG retrieval metric**: if it's not retrieved, generation can't recover (Module 7's rule).
- **Precision@k** — of the top-k, how many are relevant? (Context budget / noise.)
- **MRR (Mean Reciprocal Rank)** — `1/rank` of the *first* relevant doc, averaged. Rewards getting *a* good hit high. Use when one good hit suffices.
- **nDCG** — rank-aware + **graded** relevance; rewards highly-relevant docs near the top, discounts by position. Richest ranking metric; use when relevance has degrees and order matters (informs reranking).

## B) Generation metrics — "given the context, was the answer good?"

- **Faithfulness / groundedness** — is every claim *supported by* the retrieved context? Targets the Module 9c failure (cited but unfaithful). Usually **LLM-as-judge**: decompose answer into claims, check each vs context.
- **Answer relevance** — does it actually address the *question*? (You can be faithful but unhelpful.)
- **Context relevance/precision** — was the retrieved context on-topic? (Bridges A and B.)
- The **RAGAS triad** = faithfulness + answer relevance + context relevance. Tools: RAGAS, DeepEval, TruLens.
- **Catch:** LLM-as-judge is itself noisy/biased (position, verbosity, self-preference). Calibrate against human labels; use for *relative* comparison, not absolute truth.

## The golden Q&A set (the foundation)

- Each entry: **(question, ideal answer, relevant doc IDs)** over *your* corpus.
- Sources: real user queries (best) > expert-written > LLM-generated **that a human curates** (don't trust raw synthetic).
- Cover: easy, hard/multi-hop, adversarial, and **"should abstain"** (no answer in corpus) — the last forces you to test honest "I don't know" and catch hallucination.
- Payoff: turns "I think it's better" into "recall@5 0.72→0.88, faithfulness 0.81→0.90." RAG changes become regression-tested like code.

## Why eval-first beats vibes

- Dozens of choices = dozens of hypotheses. Eval makes each a measured decision and catches regressions.
- **Localizes failure:** low recall@k → retrieval problem (fix chunking/hybrid/transform); good recall but low faithfulness → generation/prompt problem. The split tells you *where* it broke.

## Primary sources & honesty notes

- **Es et al., 2023** — RAGAS.
- **Järvelin & Kekäläinen, 2002** — nDCG.
- **Zheng et al., 2023** — "Judging LLM-as-a-Judge" (judge biases).
- Classic IR — recall / precision / MRR.
- **Hype check:** most RAG projects skip eval and pay for it. LLM-as-judge is useful but imperfect — anchor with human labels. Golden sets rot as the corpus changes — maintain them. End-to-end task success is the metric that *matters*; the decomposed metrics tell you *where to fix*.

## Self-quiz

**Q1.** Why is eval first-class, and why measure retrieval and generation separately?
<details><summary>answer</summary>You can't tune knobs you can't measure, and failures are silent. RAG fails in two independent stages; separate metrics localize the failure — low recall = retrieval problem, good recall + low faithfulness = generation problem.</details>

**Q2 (tradeoff).** Recall@k vs nDCG — when do you care about which?
<details><summary>answer</summary>Recall@k: "is the answer in the context at all?" — the gate for whether generation *can* succeed; binary relevance, set-based. nDCG: ranking *quality* with graded relevance and position discount — when order matters (e.g., tuning the reranker, or when only the top 1–2 chunks get read).</details>

**Q3.** What's in a golden-set entry, and why include "should abstain" cases?
<details><summary>answer</summary>(question, ideal answer, relevant doc IDs). "Should abstain" (no answer in corpus) tests honest "I don't know" and catches the model hallucinating an answer when it should refuse — otherwise your eval only rewards confident answers and hides that failure.</details>

**Q4 (failure mode).** Eval dashboard all green, real users still get bad answers — when?
<details><summary>answer</summary>The golden set is unrepresentative of real queries (you optimized for the test, not production), the LLM-judge is miscalibrated/biased, you measured convenient metrics not the ones that matter, or you overfit the pipeline to the golden set. Green eval ≠ good product if the eval doesn't mirror reality.</details>

## ✍️ In my own words

_(fill 2–4 lines after you pass the checkpoint — this one unlocks the build!)_
