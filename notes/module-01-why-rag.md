# Module 1 — Why RAG

> Track 1 (RAG) · Status: ✅ passed (2026-06-08)
> Reference notes. The gate still requires my own-words checkpoint — see bottom.

## TL;DR

- A plain LLM is a **parametric** system: all knowledge is frozen into its weights at training time.
- That design fails four predictable ways: **hallucination, staleness, no access to private data, no citations.**
- **RAG = parametric (the model's reasoning) + non-parametric (an external, updatable knowledge store).** At query time it **retrieves** relevant text and **injects** it into the prompt so the model answers *grounded in* that text.
- **RAG supplies knowledge. Fine-tuning changes behavior/style. Long-context is the special case where the "retriever" is `select *`.** They compose.
- Use RAG when knowledge is **too big, too private, or too fresh** to live in the weights. Don't, when the task needs no external knowledge.

## The problem: how plain LLMs fail

1. **Hallucination** — trained to produce *plausible*, not *true*, text. No built-in "I'm unsure" gate, so it confabulates confidently.
2. **Knowledge cutoff / staleness** — knows nothing after training, and nothing that changes after.
3. **No private/proprietary data** — your wiki, PDFs, tickets were never in pretraining.
4. **No attribution** — even when right, it can't say *where* the answer came from. Unacceptable for legal/medical/support/compliance.

## What RAG actually does (mechanically)

Keep the LLM as a reasoning/writing engine. Split the system into:

- **Parametric memory** — the model's learned weights (reasoning, language, general knowledge).
- **Non-parametric memory** — a searchable store *you control* (documents → chunks → embeddings in a vector index).

Query-time flow:

```
user query
  → embed query (embedding model)
  → similarity search over the vector index  → top-k relevant chunks
  → stuff those chunks into the prompt alongside the query
  → LLM generates an answer grounded in the retrieved text (+ citations)
```

The retrieved **text** goes to the LLM; the **vectors** are only for finding it. Keep those two lanes separate.

## RAG vs fine-tuning vs long-context

| | Changes | Good for | Bad at / costs |
|---|---|---|---|
| **RAG** | *what the model sees* | large/changing/private facts, freshness, **citations**, cheap updates (re-index, don't retrain) | retrieval can miss → GIGO; more moving parts; per-query retrieval latency |
| **Fine-tuning** | *how the model behaves* (weights) | teaching **form/skill/tone/format** | poor at changing facts; slow + expensive to update; still hallucinates; no citations |
| **Long-context** | *how much you stuff in* | small/bounded corpora that just fit | cost & latency scale with tokens *every query*; doesn't scale; accuracy degrades ("lost in the middle") |

**"Lost in the Middle" (Liu et al., 2023):** LLMs use info best at the *start/end* of context and worse in the *middle*. More context ≠ better answers → relevance and placement matter → that's what retrieval buys you.

## When NOT to use RAG

- Knowledge already lives in the model (general world facts).
- Corpus is small and static and fits cheaply → long-context is simpler.
- The task is behavior/format/style, not facts → fine-tune or just prompt.
- A reasoning *skill* is needed → RAG hands over reference text, it doesn't make the model smarter.
- Hard latency budget where the retrieval round-trip is unacceptable *and* the knowledge could be baked in.

**One-liner:** RAG is for knowledge that is too big, too private, or too fresh to live in the weights.

## Primary source & honesty notes

- **Lewis et al., 2020 — "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks" (NeurIPS).** Origin of the term. Note: the *original* RAG **jointly trained** a neural retriever (DPR over a Wikipedia index) *with* the generator (BART). Today's "RAG" — a **frozen** LLM + vector DB + prompt stuffing — is a simpler *descendant*, same parametric+non-parametric intuition.
- **Hype check:** "long context kills RAG" is recurring and wrong — it *moves the boundary* of when RAG is worth it. Cost-per-query, freshness, citations, and lost-in-the-middle keep retrieval relevant at scale. Genuine disagreement: how much engineering RAG deserves vs. more context for *mid-size* corpora.

## Self-quiz

**Q1.** What are the four failure modes of a plain LLM that RAG addresses?
<details><summary>answer</summary>Hallucination, staleness/knowledge-cutoff, no access to private data, no citations.</details>

**Q2 (tradeoff).** Support bot gives outdated pricing and won't cite sources — "let's fine-tune on our docs." Right call?
<details><summary>answer</summary>No. Pricing is a *changing fact*; fine-tuning bakes facts into weights → expensive + slow to retrain on every change, and still gives no citations. Fix with RAG: update the index (not the weights) and the retrieved chunk *is* the citation. Fine-tuning teaches behavior/style; this is a knowledge problem.</details>

**Q3 (failure mode).** Where does even a well-built RAG system break?
<details><summary>answer</summary>Aggregation/global queries ("most common complaint across 10k comments") — top-k retrieval only ever shows the LLM a few chunks, never the whole corpus, so it answers from a tiny silent sample. Also: retrieval misses → generator confabulates over irrelevant context (a *silent* failure).</details>

## ✍️ In my own words

_(fill 2–4 lines — this is the part that makes it stick)_
