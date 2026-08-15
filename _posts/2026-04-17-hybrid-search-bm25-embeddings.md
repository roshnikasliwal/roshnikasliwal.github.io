---
title: "Hybrid Search: Combining BM25 and Embeddings Without the Guesswork"
date: 2026-04-17
mermaid: true
categories: [AI, Agentic AI]
tags: [rag, hybrid-search, bm25, vector-search, agentic-ai-series]
author: Roshni Kasliwal
description: Pure vector search misses exact matches on product codes, error strings, and acronyms that keyword search catches instantly. Hybrid search fixes it — if you fuse the two result sets correctly.
---

Pure embedding search is bad at exactly the queries you'd expect a search engine to nail: an error code, a product SKU, an acronym. Embeddings capture semantic similarity, not lexical exactness — `ERR_429` and `ERR_503` can end up with similar embeddings because they're syntactically alike, even though they mean completely different things. BM25 keyword search gets this right instantly and is bad at the semantic queries embeddings handle well. Hybrid search runs both and fuses the results.

## Why Not Just Pick One

| Query type | BM25 | Embeddings |
|---|---|---|
| Exact error code / SKU / ID | Strong | Weak |
| Acronym or jargon match | Strong | Weak |
| Paraphrased natural-language question | Weak | Strong |
| Conceptually related but zero shared words | Weak | Strong |

Most real query sets are a mix of both types. Committing to one search method means silently failing on the other half.

## Fusing Two Ranked Lists

The hard part isn't running both searches — most vector databases support BM25 alongside vector search natively now. The hard part is combining two differently-scaled ranking signals into one ordered list. **Reciprocal Rank Fusion (RRF)** is the simplest approach that works well in practice, because it uses rank position rather than raw scores, sidestepping the scale-mismatch problem entirely:

```python
def reciprocal_rank_fusion(result_lists: list[list[str]], k: int = 60) -> list[str]:
    scores = {}
    for results in result_lists:
        for rank, doc_id in enumerate(results):
            scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank + 1)
    return sorted(scores, key=scores.get, reverse=True)

bm25_results = bm25_index.search(query, top_k=20)
vector_results = vector_index.search(query, top_k=20)
fused = reciprocal_rank_fusion([bm25_results, vector_results])[:10]
```

The constant `k=60` (from the original RRF paper) dampens the influence of any single very-high rank from one method dominating the fused list — a document ranked #1 by BM25 doesn't automatically outrank ten documents the vector search found more relevant.

## Weighting Isn't Usually Necessary — Until It Is

RRF with equal weighting is a solid default. If your query logs show a consistent skew — mostly exact-match technical queries, say — a weighted fusion that favors BM25 can outperform the unweighted version:

```python
def weighted_rrf(result_lists, weights, k=60):
    scores = {}
    for results, weight in zip(result_lists, weights):
        for rank, doc_id in enumerate(results):
            scores[doc_id] = scores.get(doc_id, 0) + weight / (k + rank + 1)
    return sorted(scores, key=scores.get, reverse=True)
```

Don't guess the weights — tune them against a labeled query set, and only bother once RRF's equal weighting has actually shown a measurable gap for your query distribution.

## Key Takeaways

1. **BM25 and embeddings fail on opposite query types** — exact/lexical vs. semantic/paraphrased
2. **Reciprocal Rank Fusion combines ranked lists without needing to normalize incompatible score scales**
3. **Equal-weighted RRF is a strong default** — only move to weighted fusion once query logs show a real skew
4. **Most modern vector databases support hybrid search natively** — this is largely a fusion-logic problem, not an infrastructure one

---

*Part of the [Agentic AI in Practice series](/tags/agentic-ai-series/) — lessons from building production multi-agent systems.*
