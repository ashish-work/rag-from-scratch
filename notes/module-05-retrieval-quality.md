# Module 5 — Retrieval quality

> Track 1 (RAG) · Status: ✅ passed (2026-06-09)

## TL;DR

- You can now *find* nearest neighbors — but **"nearest" ≠ "useful."** Quality comes from *curating* candidates, not just fetching them.
- **top-k**: too few → miss the answer (low recall); too many → flood context, dilute signal, trigger lost-in-the-middle, cost ↑. Sweet spot small (≈3–8), eval-tuned.
- **Retrieval wants recall (cast wide); generation wants precision (few clean chunks).** Reranking bridges them: retrieve many → rerank to few.
- **Similarity thresholds** let the system say "no relevant docs" instead of returning the least-bad garbage — but absolute cutoffs are brittle.
- **Lost in the Middle**: LLMs use the start/end of context best → order chunks deliberately, don't over-stuff.
- **Duplication** wastes context budget on one idea → dedup / MMR for diverse evidence.

## top-k — how many to retrieve

| Too few (k=1–2) | Too many (k=20) |
|---|---|
| miss the answer-bearing chunk (low recall) | flood context, dilute signal, cost+latency ↑ |
| bad when answer spans chunks | triggers lost-in-the-middle; answer quality *drops* |

Higher k = higher recall, lower precision. **More context is not more better.** Optimal k is corpus/model/task-specific → eval it (Module 10).

## The recall/precision tension (key mental model)

Retrieval and generation want **opposite** things:
- **Retrieval** wants **high recall** — cast a wide net (big k) so the answer is *somewhere* in the candidates.
- **Generation** wants **high precision** — few, on-point chunks so the LLM isn't distracted.

Resolution: **retrieve many (recall) → rerank/dedup down to few (precision)** before the prompt. That's why Module 7 (reranking) exists.

## Similarity thresholds

Drop chunks below a score cutoff so an irrelevant corpus returns *nothing* rather than the least-bad k. Without one, top-k **always** returns k chunks even when none are relevant → the LLM is handed junk and hallucinates ("I must answer from this"). A threshold enables an honest "I don't know."

**Catch:** thresholds aren't portable — cosine distributions vary by model and query, so an absolute "0.7" is brittle. Many teams skip fixed thresholds and lean on **reranker scores** instead.

## Lost in the Middle (Liu et al., 2023)

LLMs attend best to the **beginning and end** of the context; info buried in the **middle** of a long context is used worse (can fall toward closed-book accuracy). Implications:
1. Don't over-stuff — fewer, better chunks.
2. **Order deliberately** — place the strongest chunks at the *edges*. Reordering is free and recovers accuracy.

## Duplication / redundancy

Real corpora have near-duplicates (your own chunk overlap, boilerplate, the same FAQ in 3 places). Top-k will happily return 5 near-identical chunks → context budget spent on **one** idea, starving the answer of *diverse* evidence.
- **Dedup**: exact (hash) or near-dup (high mutual similarity).
- **MMR (Maximal Marginal Relevance, Carbonell & Goldstein 1998)**: pick chunks relevant to the query *but* dissimilar to already-selected ones — trade a little relevance for coverage/diversity.

## Primary sources & honesty notes

- **Liu et al., 2023** — "Lost in the Middle."
- **Carbonell & Goldstein, 1998** — MMR.
- **Hype check:** "more context = better" is the seductive wrong intuition this whole module fights. Fixed similarity thresholds are widely used but finicky — rerankers often replace them. Optimal k is not a constant; eval it.

## Self-quiz

**Q1.** Why is "nearest ≠ useful," and what breaks at each end of the top-k dial?
<details><summary>answer</summary>Nearest neighbors can be near-duplicates, off-topic-but-similar, or missing context. Too-small k → miss the answer (low recall); too-large k → dilution, lost-in-the-middle, cost ↑, lower answer quality.</details>

**Q2 (tradeoff).** Retrieval wants recall, generation wants precision — how does a pipeline get both?
<details><summary>answer</summary>Retrieve a wide candidate set (high recall) then rerank + dedup down to a small, clean final set (high precision) before building the prompt.</details>

**Q3.** You raise k from 3 → 20 and answers get *worse*. Two distinct reasons?
<details><summary>answer</summary>(1) Lost-in-the-middle: the answer chunk sits in the middle of a long context and is under-used. (2) Signal dilution / near-duplicates: 17 weak-or-redundant chunks drown the 3 good ones and waste budget. (Also: cost/latency ↑.)</details>

**Q4 (failure mode).** High-similarity chunks retrieved, answer still bad — when?
<details><summary>answer</summary>When the top-k are near-duplicates of one fact (no diverse evidence), or the answer requires info spread across more chunks than k, or there's no threshold so an irrelevant corpus still yields k chunks the LLM hallucinates from — or the right chunk is buried mid-context.</details>

## ✍️ In my own words

_(fill 2–4 lines after you pass the checkpoint)_
