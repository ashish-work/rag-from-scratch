# Module 2 — Embeddings

> Track 1 (RAG) · Status: ✅ passed (2026-06-08)

## TL;DR

- An **embedding** is a learned function `text → fixed-length vector of floats` (e.g. 384/768/1024). The vector is a *coordinate* in an N-dimensional space.
- **Meaning becomes geometry:** the model is trained so similar-meaning text → nearby points. Proximity = semantic similarity.
- Embeddings power the **retrieval/search system** — the LLM never sees the vector; it sees the retrieved *text*.
- **Cosine** = angle only (magnitude-invariant) → default for text. **On normalized (unit) vectors, cosine = dot, and both rank identically to L2.** Vector DBs normalize then use dot (cheaper, same ranking).
- Embeddings are **model-specific** — never mix vectors from two models. Dimensions are a memory/latency vs. (maybe) accuracy tradeoff, not "bigger = better."

## What a vector is here

A transformer encoder maps a text chunk to N floats. Key facts:

- **Dimensions aren't human-interpretable** — no "dim 47 = royalty." Meaning is *distributed* across all dims. From the **distributional hypothesis** (Firth, 1957): text in similar contexts has similar meaning. "king"≈"queen" because of shared contexts, not shared letters.
- **Model-specific:** model A's vectors are gibberish to model B; they carve the space differently.

## Similarity: cosine vs dot vs L2

| Measure | Formula | Measures | Sense |
|---|---|---|---|
| **Dot** | `a·b = Σ aᵢbᵢ` | direction **and** magnitude | bigger = more similar (unbounded) |
| **Cosine** | `a·b / (‖a‖‖b‖)` | **angle only** | [−1,1], 1 = same direction |
| **Euclidean (L2)** | `‖a−b‖` | straight-line distance | smaller = more similar |

**Why cosine for text:** we want *direction = meaning*, not vector length (length often just tracks document length — noise).

**Key identity — on unit vectors (‖v‖=1):**
- `dot = cosine` (denominator is 1)
- `L2² = 2 − 2·cosine`
- ⇒ maximizing dot ≡ maximizing cosine ≡ minimizing L2 (**same ranking**).

That's why "normalize, then use dot" is standard: cheaper than cosine (no division), identical ordering.

```python
import numpy as np
def cosine(a, b):
    return a @ b / (np.linalg.norm(a) * np.linalg.norm(b))
# on unit vectors, a @ b alone == cosine(a, b)
```

## Normalization

Divide a vector by its L2 norm → length 1. Matters because (a) makes dot == cosine (fast + correct), (b) stops long docs from inflating dot scores. Many models emit normalized vectors (`normalize_embeddings=True`). **Gotcha:** be consistent — mixing normalized + raw vectors makes every score garbage.

## Model choice — what moves the needle

- **Dimensions (384 vs 768 vs 1536):** higher *can* capture more nuance but costs memory (`≈ n_vectors × dims × 4 bytes` for f32) and slows search. Not automatically better — diminishing returns. (5M × 1536 × 4B ≈ 30 GB vs 5M × 384 × 4B ≈ 7.7 GB.)
- **Asymmetric models (E5, BGE):** want `query:` / `passage:` prefixes — questions and docs have different distributions. Wrong prefix → recall tanks (silent).
- **Max sequence length:** models truncate past their limit (often 256–512 tokens). Overflow is *silently ignored* → ties directly to chunking (Module 3).
- **Local / $0:** `all-MiniLM-L6-v2` (384-dim, CPU-fast), `bge-small`, `e5-small` run free on a laptop.

## Primary sources & honesty notes

- **Mikolov et al., 2013** — word2vec (the "king − man + woman ≈ queen" embeddings).
- **Reimers & Gurevych, 2019** — Sentence-BERT (SBERT); practical sentence embeddings (our library lineage).
- **MTEB (Muennighoff et al., 2022)** — standard benchmark. *Caveat:* leaderboard rank ≠ best for *your* data — always eval on your corpus.
- **Hype check:** "cosine vs dot" are the same on unit vectors. "Bigger embedding = better" is a myth. Embeddings can be blind to **negation** ("good"≈"not good") and weak on niche domains.

## Self-quiz

**Q1.** Why is cosine the default, and when do cosine/dot/L2 stop mattering?
<details><summary>answer</summary>Cosine measures angle (meaning), ignoring length (noise). On normalized vectors all three give identical rankings, so the choice is just compute cost → normalize + dot.</details>

**Q2 (tradeoff).** 384 vs 1536 dims, 5M chunks, latency matters?
<details><summary>answer</summary>384: 4× less memory/compute per query, lower latency, usually "good enough." 1536: maybe more nuance but 4× cost and not guaranteed better. Latency-bound → 384, and *verify* with eval rather than assuming higher = better.</details>

**Q3.** Embed docs with model A, query with model B (same dim)?
<details><summary>answer</summary>Retrieval is effectively random. Different models build *unaligned coordinate systems*; comparing B's query vector to A's doc vectors is comparing two different languages. Same dimension ≠ compatible.</details>

**Q4 (failure mode).** When does embedding retrieval break (that keyword search would get right)?
<details><summary>answer</summary>Exact identifiers / novel acronyms / OOV terms (error code `ER-4021`, a SKU). Embeddings smear text into a smooth semantic space; rare tokens have no meaningful neighborhood → nearest-neighbor returns adjacent-but-wrong items. → motivates hybrid search (Module 6).</details>

## ✍️ In my own words

_(fill 2–4 lines)_
