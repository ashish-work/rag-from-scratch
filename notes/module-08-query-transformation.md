# Module 8 — Query transformation

> Track 1 (RAG) · Status: ✅ passed (2026-06-10)

## TL;DR

- Everything so far assumes the **user's query is a good search probe**. Often it isn't: underspecified, conversational, pronoun-laden, phrased unlike the docs, too broad, or actually several questions.
- Query transformation **rewrites / expands / decomposes the query *before* retrieval** so the probe matches the corpus. It's the **input-side** complement to reranking's output-side fix.
- Techniques: **rewriting** (coref/normalize), **multi-query** (paraphrase + RRF fuse), **HyDE** (embed a hypothetical *answer*), **decomposition** (sub-questions), **step-back** (ask a more general question first).
- Mostly raises **recall** (better/wider probe) → pair with reranking to re-tighten precision. Every technique adds an LLM call → latency, cost, and new failure modes.

## The techniques (each fixes a different query pathology)

**1. Query rewriting / normalization.** Resolve coreference from chat history ("its pricing" → "Acme's pricing"), expand abbreviations, clean phrasing. Critical in multi-turn chat where the latest message is meaningless alone. Cheap, near-universal ROI. (Ma et al., 2023.)

**2. Multi-query (expansion).** LLM generates several paraphrases; retrieve for each; **fuse with RRF**. One phrasing may miss the right vocabulary; many phrasings widen the semantic net → higher recall. Cost: N× retrieval + a generation call.

**3. HyDE — Hypothetical Document Embeddings (Gao et al., 2022).** Instead of embedding the *question*, have an LLM write a *hypothetical answer passage*, embed **that**, and search with it.
- Why it works: question↔document is **asymmetric** (a question and an answer look different in embedding space). A hypothetical answer *looks like* a real answer doc → **answer↔answer (symmetric)** matching is easier.
- The hypothetical can be factually **wrong** and still retrieve the right real docs — it just needs the right *shape/vocabulary*.
- Risk: extra LLM latency; drifts on niche topics where the LLM hallucinates off-distribution.

**4. Decomposition (sub-questions).** Split a complex query into parts, retrieve each, combine. "Compare domestic vs international refund policies" → two sub-queries. Attacks the multi-hop/composition failure (Module 6c).

**5. Step-back prompting (Zheng et al., 2023).** Ask a more *abstract* question first, retrieve the governing context, then answer the specific one. "Was X eligible for Y in 1801?" → "What are the eligibility rules for Y?" → retrieve principles → reason.

## The unifying idea

**The user's literal query is rarely the optimal retrieval probe.** Transformation bridges *how users ask* vs *how documents are written* — the main lever against the "needs reasoning, not resemblance" ceiling (Modules 6c/7).

## Tradeoffs & honest notes

- Every technique adds an **LLM call before retrieval** → latency, cost, and **error propagation** (a rewriter/HyDE generator that misreads intent or hallucinates poisons everything downstream).
- They mostly improve **recall** → must pair with **reranking** to restore precision. Opposite ends of the pipeline, complementary.
- Not free wins: on simple, well-phrased queries over a clean corpus, transformation is wasted latency; multi-query/HyDE can *hurt* if they drift. **Eval before adopting (Module 10).**
- Production reality: **query rewriting (coref for chat) is the near-universal one**; multi-query/HyDE/decomposition are situational.

## Primary sources

- **Gao et al., 2022** — HyDE ("Precise Zero-Shot Dense Retrieval without Relevance Labels").
- **Zheng et al., 2023** — Step-Back Prompting.
- **Ma et al., 2023** — "Query Rewriting for Retrieval-Augmented LLMs."
- Decomposition ↔ least-to-most prompting / multi-hop RAG.

## Self-quiz

**Q1.** What does query transformation attack, and name 2–3 techniques + what each fixes?
<details><summary>answer</summary>The input side — the user's query is a poor retrieval probe. Rewriting fixes coref/phrasing (chat); multi-query fixes vocabulary mismatch via paraphrase+RRF; HyDE fixes question↔doc asymmetry; decomposition fixes multi-part/multi-hop; step-back fixes too-narrow queries needing broader context.</details>

**Q2 (tradeoff).** Why does embedding a *hypothetical answer* (HyDE) beat embedding the question? Risk?
<details><summary>answer</summary>Questions and answer-passages are asymmetric in embedding space, so question→doc matching is hard. A hypothetical answer has the *shape/vocabulary* of a real answer → answer→answer (symmetric) matching is easier; it can be factually wrong and still retrieve correctly. Risk: extra LLM latency + hallucination drift on niche topics.</details>

**Q3.** Does query transformation mainly improve recall or precision? What must you pair it with?
<details><summary>answer</summary>Recall — it casts a better/wider net before retrieval. Pair with reranking (and dedup) to restore precision, or you just flood the context with more candidates.</details>

**Q4 (failure mode).** When does query transformation make retrieval *worse*?
<details><summary>answer</summary>When the pre-retrieval LLM misreads intent or hallucinates — the rewritten/expanded/hypothetical query drifts from what the user meant, and that error propagates to every downstream stage. Multi-query/HyDE add noise on simple queries; a bad rewrite is worse than the raw query.</details>

## ✍️ In my own words

_(fill 2–4 lines after you pass the checkpoint)_
