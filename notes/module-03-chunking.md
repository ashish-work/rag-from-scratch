# Module 3 — Chunking

> Track 1 (RAG) · Status: ✅ passed (2026-06-08)

## TL;DR

- You can't embed a whole document as one vector: it overflows the model's token limit and produces a **blurry average** vector near nothing ("semantic smearing"). So split docs into **chunks**.
- **Chunking sets the ceiling on the whole pipeline** — if the answer isn't inside a well-formed chunk, no embedding model or reranker downstream can recover it.
- **Size tension:** too small → loses context, ambiguous; too large → blurry vector, imprecise retrieval, wasted context budget.
- **Strategies (simple→smart):** fixed-size → recursive/structure-aware (practical default) → semantic → layout-aware.
- Chunk by **tokens, not chars** (models think in tokens). Chunking **destroys** coreference, global structure, and aggregation — irreversibly.

## Why chunk — three distinct reasons

1. **Model token limits** — overflow is silently truncated.
2. **Retrieval precision** — retrieve the *passage*, not the book; one vector per huge doc is imprecise and floods the context (→ lost in the middle).
3. **Embedding fidelity** — a chunk spanning many topics has a vector near none of them. One chunk ≈ one idea keeps vectors sharp.

## The size tradeoff

| | Too **small** | Too **large** |
|---|---|---|
| Problem | loses context, ambiguous | blurry vector, imprecise, wastes budget |
| Example | *"He signed it in 1801."* (who? what?) | 2-page chunk covering 5 subtopics |
| Failure | retrieves but doesn't help | retrieves too much, dilutes answer |

Starting point: **~200–500 tokens, ~10–20% overlap** — a *hyperparameter to tune with eval* (Module 10), not a constant.

**Overlap** = repeat the last N tokens of chunk *k* at the start of *k+1*, so a fact straddling a boundary isn't split across two chunks that each hold half. Cost: duplication (storage + same content retrieved twice → dedup later). More overlap = safer boundaries, more redundancy.

## Strategies (simplest → smartest)

1. **Fixed-size** (chars/tokens): split every N units. Trivial, fast, blindly cuts mid-sentence. Baseline (v0/v1).
2. **Recursive / structure-aware** (*practical default*): split on a hierarchy of separators — paragraphs → sentences → words — dropping finer only when a piece is still too big. Respects natural boundaries. **Hand-written in v1.**
3. **Semantic:** embed sentences, start a new chunk when adjacent-sentence similarity drops (topic shift). More faithful, but costs embeddings at ingest + a threshold to tune. *Often marginal gains over good recursive — practitioners disagree it's worth it.*
4. **Layout-aware:** use the doc's own structure (headings, sections, tables, markdown, code). Often **highest ROI** — real docs encode meaning in structure.

## Token- vs char-based

Embedding models count **tokens**, not characters. "Every 1000 chars" can blow past a 512-token limit unpredictably (≈4 chars/token in English; wildly different for code/other languages → silent truncation). **Token-based** chunking (model tokenizer / `tiktoken`) controls exactly what the model sees and respects its max length.

## What chunking destroys (the honest part)

- **Coreference / cross-chunk context:** "The company" in chunk 5 → "Acme Corp," named only in chunk 1. Split → chunk 5 is unanchored. → fix with **Contextual Retrieval** (Anthropic, 2024): prepend a short context header to each chunk *before* embedding (v4).
- **Global structure:** tables split across chunks; a list severed from its heading.
- **Aggregation / long-range:** the "most common complaint across 10k comments" problem (Module 1 failure mode).
- **Irreversible** — destroyed at ingestion, unrecoverable downstream.

## Primary sources & honesty notes

- **Anthropic, "Contextual Retrieval" (2024)** — chunking destroys context; fix by prepending per-chunk context; includes hard numbers.
- **Liu et al., "Lost in the Middle" (2023)** — why over-stuffing chunks/context hurts.
- **RAPTOR (Sarthi et al., 2024)** — hierarchical summaries for aggregation (Module 11 survey).
- **Hype check:** no universal best chunk size (corpus + model dependent). "Semantic chunking" is trendier than decisive — most wins come from respecting structure + adding context, not a clever splitter.

## Self-quiz

**Q1.** Why chunk (≥2 reasons), and what breaks at each size extreme?
<details><summary>answer</summary>Reasons: model token limits, retrieval precision, embedding fidelity. Too small → ambiguous, loses context; too large → blurry vector, imprecise retrieval, wasted context budget.</details>

**Q2 (tradeoff).** When ship plain recursive instead of semantic chunking?
<details><summary>answer</summary>When the extra ingest cost + threshold tuning of semantic isn't justified by measurable recall gains — which is often. Recursive respects paragraph/sentence boundaries cheaply and deterministically; reach for semantic only if eval shows recursive is splitting meaning badly.</details>

**Q3.** 512-token model — why token-based over "every 2000 chars"?
<details><summary>answer</summary>Char count ≠ token count (≈4:1 but variable). 2000 chars can exceed 512 tokens → the model silently truncates the tail, so part of the chunk is never embedded. Token-based chunking guarantees you stay within the limit.</details>

**Q4 (failure mode).** When does chunking silently cap quality (no reranker can fix)?
<details><summary>answer</summary>When the answer-bearing text is split so no single chunk contains it whole, or a chunk loses its antecedent/context (coreference). Retrieval can only rank what was chunked; if meaning was destroyed at ingestion it's unrecoverable downstream.</details>

## ✍️ In my own words

_(fill 2–4 lines after you pass the checkpoint)_
