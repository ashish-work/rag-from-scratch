# Module 7 — Reranking

> Track 1 (RAG) · Status: ✅ passed (2026-06-09)

## TL;DR

- Stage-1 retrieval (hybrid + RRF) is cheap, approximate, high-**recall** but poorly **ordered**. A reranker re-scores the candidates with a slower, smarter model for **precision**.
- **Bi-encoder** (your embedder): encodes query & doc *separately* → cosine. Fast, precomputable, scales to millions, but coarse (all interaction crushed into one dot product).
- **Cross-encoder** (the reranker): feeds query+doc *together* through the transformer → token-level cross-attention → far more accurate, but **one forward pass per (query, doc) pair** at query time → can't scale to a corpus.
- **Two-stage** = retrieve wide (bi-encoder+BM25, whole corpus) → rerank narrow (cross-encoder, top-N only). Cross-encoder accuracy at bi-encoder scale.
- **Reranking fixes precision, not recall.** If the answer isn't in the top-N, the reranker can't save you — fix retrieval instead.

## Bi-encoder vs cross-encoder (the core distinction)

| | **Bi-encoder** | **Cross-encoder** |
|---|---|---|
| Input | query and doc encoded **separately** | query + doc **together** (`[CLS] q [SEP] d`) |
| Interaction | one dot product (coarse) | full token-level cross-attention (rich) |
| Precompute | yes — docs embedded offline | no — pair doesn't exist until query time |
| Cost | cheap, scales to millions | 1 forward pass **per candidate** |
| Role | stage-1 retrieval | stage-2 reranking |

The bi-encoder's separate encoding is the *bottleneck*: the query never "sees" the doc, so relevance is compressed into a single similarity. The cross-encoder removes that bottleneck at the price of no precomputation.

## Why two-stage (the resolution of recall vs precision)

- **Stage 1 — retrieve:** bi-encoder + BM25 over the whole corpus → top-N (N≈50–200). Cheap, high recall.
- **Stage 2 — rerank:** cross-encoder over **only those N** → reorder → final top-k (k≈3–8). Expensive but accurate.
- Net: cross-encoder quality without cross-encoding the corpus, because it only ever sees N docs.

## Latency — the real tradeoff

A cross-encoder runs once per candidate → reranking 100 candidates = 100 forward passes. On CPU (our $0 path) this is usually the slowest step; on GPU it's fine. Knobs: **N** (more candidates = more recall into the reranker, more latency) and **model size** (MiniLM cross-encoder = small/fast; larger = better/slower).

## When it pays off — and when it doesn't

- **Pays off:** stage-1 recall is good but ordering/precision is poor — the right doc is *in* the top-N but not the top-k. Directly fixes the "topical but not answer-bearing" failure (cross-encoder reads them together and demotes the empty-but-similar chunk). Biggest quality lever after hybrid for many systems.
- **Doesn't help:** stage-1 **recall** is bad (answer not in top-N) — a reranker only reorders what it's given. Also: pure latency cost on easy queries.
- **Rule:** *reranking fixes precision, not recall.* Recall problem → fix chunking / hybrid / query transformation, not the reranker.

## Practical (v3)

Local cross-encoder via sentence-transformers (`cross-encoder/ms-marco-MiniLM-L-6-v2`); rerank top ~50–100 hybrid candidates → top-5. Free, CPU-runnable. (Late-interaction models like **ColBERT** sit between bi- and cross-encoders — token-level but precomputable — survey only.)

## Primary sources & honesty notes

- **Nogueira & Cho, 2019** — "Passage Re-ranking with BERT" (cross-encoder reranking).
- **Reimers & Gurevych, 2019** — SBERT; bi- vs cross-encoder framing.
- **Khattab & Zaharia, 2020** — ColBERT (late interaction).
- **Hype check:** rerankers are a reliable quality boost *but they are latency* — on CPU they can dominate response time. Useless if retrieval recall is bad — people bolt on a reranker to paper over a chunking/retrieval problem and wonder why nothing improves. Measure recall@N before, precision after (Module 10).

## Self-quiz

**Q1.** Bi- vs cross-encoder, and why two-stage?
<details><summary>answer</summary>Bi-encoder encodes q & d separately → cheap dot product, precomputable, scales, coarse. Cross-encoder feeds q+d together → token-level attention, accurate, but one pass per pair, no precompute. Two-stage: bi-encoder retrieves wide cheaply (recall) → cross-encoder reranks the top-N accurately (precision) → cross-encoder quality at corpus scale.</details>

**Q2 (tradeoff).** Why not just cross-encode the whole corpus?
<details><summary>answer</summary>Cost: a forward pass per (query, doc) pair, nothing precomputable. O(corpus) passes per query is infeasible at millions of docs. So restrict the expensive model to a small candidate set (top-N) produced by cheap retrieval.</details>

**Q3.** "Reranking fixes precision, not recall" — scenario where a reranker changes nothing, and what you'd fix?
<details><summary>answer</summary>If the answer-bearing chunk never enters the top-N candidate pool (retrieval recall failure), reranking only reorders junk → no improvement. Fix recall: better chunking, hybrid search, query transformation, or larger N — not the reranker.</details>

**Q4 (failure mode).** Right doc IS in the candidate list but the reranker still doesn't rank it #1 — when?
<details><summary>answer</summary>Domain/distribution mismatch — a reranker trained on web/MS-MARCO queries mis-scores legal/medical/code jargon; or the query is underspecified/ambiguous so even full cross-attention can't judge true relevance; or relevance is reasoning-based, not surface (needs query transformation). Cross-encoders are accurate, not omniscient.</details>

## ✍️ In my own words

_(fill 2–4 lines after you pass the checkpoint)_
