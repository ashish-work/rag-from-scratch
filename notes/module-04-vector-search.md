# Module 4 — Vector search & indexes

> Track 1 (RAG) · Status: ✅ passed (2026-06-09)

## TL;DR

- Search = given a query vector, find the top-k nearest chunk vectors. Two ways: **brute-force (exact)** or **ANN (approximate)**.
- **Brute force**: compare against *every* vector. O(N·d), 100% recall, perfect up to ~100k–1M vectors. This is v0 (numpy). Underrated — don't reach for a vector DB prematurely.
- **ANN** trades exactness for speed: pre-organize vectors so a query only inspects a candidate subset → sublinear, but **recall < 100%**.
- Two families: **IVF(-PQ)** (cluster into cells, probe a few; memory-efficient at scale) and **HNSW** (navigable graph; best recall/latency, costs RAM).
- The triangle: **recall ↔ latency ↔ memory** (+ build time / update-ability). Pick your tradeoff and *measure recall@k vs brute-force truth*.
- "Vector search" = a library (faiss/hnswlib/sqlite-vec). "Vector database" = ops (persistence, filtering, upsert/delete, multi-tenancy). Don't conflate.

## Brute-force (flat / exact kNN)

- similarity(query, every vector) → sort → top-k. With numpy/BLAS it's one matrix-vector product + `argpartition`.
- O(N·d) time, stores all vectors. **Exact** (recall = 1.0).
- Fine to ~100k–1M vectors (dims/hardware dependent); a few hundred k × 384-dim = milliseconds. It is the *correct default* until N forces you off it. **This is v0.**

## Why ANN

At large N you can't touch every vector per query. ANN indexes pre-structure the data so search is sublinear — at the cost of occasionally missing a true neighbor (recall < 1). You *tune* how much recall you trade for speed.

## The two main families

**IVF (Inverted File / cell-probe)**
- k-means the vectors into `nlist` cells; at query time search only the `nprobe` nearest cells.
- Knob: `nprobe` ↑ → more cells searched → higher recall, slower.
- Memory-efficient; pairs with **PQ (Product Quantization)** to compress vectors (IVF-PQ) → big memory savings at some recall cost. Good for large, fairly static datasets. Centroids drift as data changes → may need retraining.

**HNSW (Hierarchical Navigable Small World)**
- Multi-layer proximity graph; greedy hop from a top entry point down to the base layer.
- De-facto default for in-memory ANN; excellent recall/latency.
- Costs: **high memory** (graph + vectors), heavier build, awkward deletes. Knobs: `M` (links/node), `efConstruction` (build quality), `efSearch` (query recall/speed; ↑ = better recall, slower).

**One-liner:** HNSW = best recall/latency, pay RAM. IVF-PQ = memory-efficient at scale, pay recall + tuning.

## The tradeoff triangle: recall ↔ latency ↔ memory

| Want | Use | Pay |
|---|---|---|
| exact (recall = 1) | brute force | latency/compute at scale |
| fast + high recall | HNSW | memory |
| low memory at huge scale | IVF-PQ | recall + tuning |

4th axis: **build time & update-ability** — matters for incremental indexing (Module 12). HNSW graphs dislike deletes; IVF centroids go stale on drift.

## recall@k (how you measure ANN)

recall@k = of the *true* top-k (by brute force), how many did the ANN return? Measured **against brute-force ground truth on a sample**. Tune `nprobe`/`efSearch` until recall ≥ target (e.g. 0.95) at acceptable latency.

## When do you actually need a vector DB?

- **Prototype / <~100k–1M vectors** → numpy flat or faiss-flat, in-memory, **exact**. No DB.
- **ANN library** (faiss / hnswlib / sqlite-vec) → when N blows your latency budget.
- **Vector database** (Qdrant, Weaviate, Milvus, pgvector…) → when you need *operational* features: persistence, metadata filtering at scale, concurrent writes, replication, upsert/delete, multi-tenancy. The "DB" is the **ops**, not the math.
- Our path: **v0 numpy → v2 faiss / sqlite-vec** (local, free). No hosted DB ($) on purpose.

## Primary sources & honesty notes

- **Malkov & Yashunin, 2018** — HNSW.
- **Jégou et al., 2011** — Product Quantization.
- **Johnson, Douze, Jégou, 2017** — FAISS ("billion-scale similarity search").
- **ANN-Benchmarks** (Aumüller et al.) — standard recall/latency comparison.
- **Hype check:** vector DBs are over-marketed; most early projects don't need one. "Approximate" sounds scary but 95–99% recall is usually invisible end-to-end (reranker + LLM absorb a missed borderline neighbor). The recall that matters is *end-to-end, measured* (Module 10) — not index recall in isolation.

## Self-quiz

**Q1.** Brute force vs ANN, and the core tradeoff?
<details><summary>answer</summary>Brute force compares against every vector → exact (recall 1) but O(N·d). ANN pre-structures vectors → sublinear search but recall < 1. Trade recall ↔ latency ↔ memory (+ build/update cost).</details>

**Q2 (tradeoff).** 50k-chunk internal tool — "we need Pinecone with HNSW." Right?
<details><summary>answer</summary>No. 50k vectors is trivial for exact brute-force / faiss-flat in-memory — milliseconds, 100% recall, zero infra. A hosted vector DB adds cost + ops for no benefit. "Vector search" (library) ≠ "vector database" (operational infra). Revisit only if N grows or you need persistence/filtering/multi-tenancy.</details>

**Q3.** "recall@10 = 0.9" — what does it mean, measured against what?
<details><summary>answer</summary>Of the true top-10 nearest (computed by exact brute force on a sample), the ANN returned 9. Ground truth = brute-force results.</details>

**Q4 (failure mode).** When does ANN silently degrade?
<details><summary>answer</summary>When the recall knob is too low (`nprobe`/`efSearch` too small) the true neighbor sits in an unsearched cell/graph region → silently dropped; or IVF centroids go stale as data drifts. Silent because, without measuring recall vs brute force, you never see the neighbor you missed.</details>

## ✍️ In my own words

_(fill 2–4 lines after you pass the checkpoint)_
