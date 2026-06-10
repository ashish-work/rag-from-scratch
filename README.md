# rag-from-scratch — AI Application Engineer, the hard way

Learning AI engineering by **building**, not copying. No orchestration frameworks
(LangChain / LlamaIndex / Haystack are banned) — every line of glue is hand-written,
and everything runs locally for ~$0.

This repo is my learn-in-public log: concept notes in my own words, decision logs, and code.

## Curriculum

A single committed, T-shaped path (broad base, depth in LLM/agent engineering + LLMOps),
adapted from [rohitg00/ai-engineering-from-scratch](https://github.com/rohitg00/ai-engineering-from-scratch)
— I **implement** the high-leverage tracks and **skim** the rest.

| # | Track | Status |
|---|-------|--------|
| 0 | Foundations primer (numpy backprop + self-attention) | ⬜ todo |
| 1 | **RAG from scratch** — the depth spine | 🟨 in progress |
| 2 | LLM Engineering core (tiny GPT, evals, guardrails) | ⬜ todo |
| 3 | Tools, MCP & Agents | ⬜ todo |
| 4 | LLMOps / Production (serving, observability, cost) | ⬜ todo |
| 5 | Breadth skim + portfolio polish | ⬜ todo |

**Definition of done:** 3 deployed + documented capstones, can whiteboard attention,
can argue RAG / agent / serving tradeoffs.

### Track 1 — RAG modules (current)

Learn (gated) → Build. No pipeline code until modules 1–10 are passed.

- [x] 1. Why RAG
- [x] 2. Embeddings
- [x] 3. Chunking
- [x] 4. Vector search & indexes
- [x] 5. Retrieval quality
- [x] 6. Hybrid search (BM25 + dense, RRF)
- [x] 7. Reranking (cross-encoders)
- [x] 8. Query transformation
- [x] 9. Context construction & prompting
- [x] 10. Evaluation
- [ ] 11. Advanced architectures
- [ ] 12. Production concerns

**Build ladder** (unlocks after modules 1–10):
`v0` numpy brute-force cosine → `v1` real PDF + my own chunker + sentence-transformers →
`v2` faiss / sqlite-vec → `v3` hybrid (BM25 + dense) + RRF + cross-encoder reranker →
`v4` query rewriting + contextual headers + eval harness → `v5` production hardening
(embedding cache, incremental indexing, cost/latency logging).

## Learning plan — theory-first (changed 2026-06-10)

Strategy: cover the **theory of the whole curriculum first**, then one concentrated build phase. The v0→v5 build ladder above + capstones are **deferred** until the theory phase is complete.

**Theory phase — Parts A–F** (checkpoint-gated, one reference note each):

- **A · RAG** — [x] 1–10 · [ ] 11 Advanced architectures · [ ] 12 Production concerns
- **B · Foundations** — [ ] Math for ML · [ ] ML fundamentals · [ ] Deep learning core · [ ] Transformers
- **C · LLM Engineering** — [ ] How LLMs are built · [ ] Prompting & context · [ ] Fine-tuning · [ ] Function calling · [ ] Eval/guardrails/cost · [ ] Quantization & inference
- **D · Tools, MCP & Agents** — [ ] Tools deep dive · [ ] MCP · [ ] Agent loop & patterns · [ ] Multi-agent · [ ] Agent eval & observability
- **E · LLMOps** — [ ] Serving internals · [ ] Inference metrics & quant · [ ] Caching/gateways/cost · [ ] Deploy/observability/security
- **F · Breadth** — [ ] Generative models · [ ] Multimodal & VLMs · [ ] Safety & alignment

## Repo layout

- `notes/` — concept notes + checkpoint answers, in my own words
- `decisions/` — 3-line decision logs per build step (what / decision vs. rejected alternative / mistake avoided)
- `src/` — the pipeline code (Phase B only)

## Stack

Python · numpy · sentence-transformers · faiss-cpu / sqlite-vec · rank_bm25 ·
pypdf / pymupdf · tiktoken · Ollama (local generation). Single-purpose libraries only.

## How I commit

- Pass a module checkpoint → commit my `notes/` (`docs: pass module N (...)`)
- Finish a build rung → commit `src/` + `decisions/` (`feat: vN ...`)
