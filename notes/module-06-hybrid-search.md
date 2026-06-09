# Module 6 — Hybrid search

> Track 1 (RAG) · Status: ✅ passed (2026-06-09)

## TL;DR

- **Dense** (embeddings) is great at *meaning*, blind to *exact tokens*. **Lexical/sparse (BM25)** is great at exact tokens, blind to meaning. Their failures are largely **disjoint** → run both and **fuse**.
- **BM25** = TF-IDF evolved: term frequency × inverse-document-frequency, with **saturation** + **length normalization**.
- **Why combine:** dense catches paraphrase BM25 misses; BM25 catches the SKU/error-code dense smears away. Hybrid beats either alone on most real (mixed) corpora — high ROI, low cost (this is v3).
- **Fusion problem:** dense (cosine ∈[−1,1]) and BM25 (unbounded) scores aren't on the same scale. **RRF (Reciprocal Rank Fusion)** fuses by **rank, not score** → no tuning, robust → the practical default.

## Lexical / sparse retrieval (BM25)

- Workhorse keyword ranking. Score ≈ Σ over query terms of: `TF (saturating) × IDF (rarer term = higher weight)`, normalized for document length.
  - **Saturation:** the 10th occurrence of a term adds less than the 2nd (diminishing returns).
  - **Length normalization:** long docs don't win just for being long.
- "Sparse" = representation is a huge mostly-zero vector over the vocabulary (one dim per term). Exact-token matching.
- **Wins:** exact identifiers/codes/names, rare terms, precise phrases, out-of-domain jargon the embedder never saw. Zero training, cheap, interpretable.
- **Loses:** semantics — "reset my password" ✗ "recover login credentials" (no shared terms).

## Dense retrieval (Modules 2–4)

- **Wins:** meaning, paraphrase, synonyms. **Loses:** exact tokens, rare identifiers (Module 2 failure mode).

## Why combine — complementary, not redundant

The failure sets are mostly disjoint, so union ≫ either alone. Especially on mixed content (prose + code + IDs + jargon). One of the few near-universal wins in RAG.

## Fusion: RRF vs weighted score-sum

**Weighted sum:** min-max normalize each score, then `α·dense + (1−α)·sparse`. Works, but `α` is finicky and per-query score distributions vary → brittle (same disease as fixed similarity thresholds).

**Reciprocal Rank Fusion (RRF, Cormack et al. 2009):**
```
RRF_score(doc) = Σ over result lists  1 / (k + rank_in_list)      # k ≈ 60
```
- Fuses by **rank**, which is comparable across systems → sidesteps incompatible scores entirely. No tuning, robust → **practical default**.
- A doc ranked high in *both* lists rises to the top.
- **Tradeoff:** discards score *magnitude* (a wildly-more-similar doc and a barely-more-similar one at the same rank count equally). Usually a good trade.

## Pipeline placement

Run BM25 (inverted index; `rank_bm25` locally) + dense (vector index) → top-N each → **RRF** → final top-k → **rerank** (Module 7). Hybrid raises **recall** (two nets = the "retrieve wide" half of Module 5); reranking restores **precision**.

## Primary sources & honesty notes

- **Robertson & Zaragoza, 2009** — BM25 / Probabilistic Relevance Framework.
- **Cormack, Clarke & Büttcher, 2009** — Reciprocal Rank Fusion.
- **Anthropic Contextual Retrieval, 2024** — contextual BM25 + contextual embeddings + RRF; strong real-world reference.
- **Hype check:** hybrid is a rare near-universal win; RRF is boringly effective and weighted sums rarely beat it in practice. Matters less on pure-prose corpora with no identifiers/jargon (dense alone may do).

## Self-quiz

**Q1.** Lexical vs dense — why complementary, and what does hybrid buy?
<details><summary>answer</summary>Dense = meaning (paraphrase/synonyms), blind to exact tokens. BM25 = exact tokens (IDs/codes/rare terms), blind to meaning. Failures are disjoint → hybrid raises recall by covering both, especially on mixed prose+code+ID corpora.</details>

**Q2 (tradeoff).** Why RRF over weighted score-sum? What does RRF give up?
<details><summary>answer</summary>RRF fuses by rank, so it ignores the incompatible score scales (cosine vs unbounded BM25) and needs no per-query α tuning → robust default. It gives up score *magnitude* — strength-of-match nuance is flattened to rank.</details>

**Q3.** One query where BM25 wins/dense fails, one where dense wins/BM25 fails?
<details><summary>answer</summary>BM25 wins: "error code ER-4021" / a SKU / a function name — exact token, dense smears it. Dense wins: "how do I reset my password" vs a doc titled "recovering your login credentials" — no shared terms, pure semantic match.</details>

**Q4 (failure mode).** When does even hybrid + RRF return a bad set?
<details><summary>answer</summary>When the answer needs *reasoning/synthesis* rather than matching (neither lexical nor semantic surface signal points to it); or the answer-bearing passage shares neither key terms nor close semantics with the query (needs query transformation, Module 8); or both lists rank the right doc mediocrely so RRF can't lift it. Hybrid widens the net but can't conjure a signal that isn't there.</details>

## ✍️ In my own words

_(fill 2–4 lines after you pass the checkpoint)_
