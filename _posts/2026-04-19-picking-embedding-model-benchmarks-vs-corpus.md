---
title: "Picking an Embedding Model: Benchmarks vs Your Actual Corpus"
date: 2026-04-19
mermaid: true
categories: [AI, Agentic AI]
tags: [rag, embeddings, benchmarking, mteb, agentic-ai-series]
author: Roshni Kasliwal
description: MTEB leaderboard position is a weak predictor of how an embedding model performs on your specific corpus. Here's how to actually evaluate one before committing.
---

The MTEB leaderboard is the default place people pick an embedding model, and it's a reasonable starting filter — but leaderboard rank is an average across dozens of unrelated tasks and domains, and your corpus is neither. A model ranked #3 overall can underperform a model ranked #15 on your specific mix of documents and query patterns.

## Why the Leaderboard Undersells the Gap

MTEB aggregates performance across retrieval, classification, clustering, and more, across general-domain datasets like Wikipedia and news. If your corpus is internal engineering documentation full of code snippets, API names, and jargon, a model's Wikipedia-retrieval score tells you very little about how well it'll embed a stack trace or a config key.

The fix isn't to ignore the leaderboard — it's a reasonable shortlist filter — but to treat its ranking as a starting point, not a decision.

## Building a Corpus-Specific Eval

The only reliable signal is testing candidate models against your actual documents and realistic queries:

```python
from sentence_transformers import SentenceTransformer
import numpy as np

def eval_embedding_model(model_name: str, eval_pairs: list[tuple[str, str]]) -> float:
    """eval_pairs: list of (query, correct_document_id) from your own labeled set."""
    model = SentenceTransformer(model_name)
    hits = 0
    for query, correct_doc_id in eval_pairs:
        query_emb = model.encode(query)
        scores = {doc_id: np.dot(query_emb, doc_embeddings[doc_id]) for doc_id in doc_embeddings}
        top_result = max(scores, key=scores.get)
        hits += (top_result == correct_doc_id)
    return hits / len(eval_pairs)

candidates = ["text-embedding-3-small", "bge-large-en-v1.5", "e5-mistral-7b-instruct"]
results = {m: eval_embedding_model(m, your_labeled_eval_set) for m in candidates}
```

The `eval_pairs` are the part that takes real effort: 50-100 real or realistic queries against your corpus, each labeled with the document that should actually be retrieved. This is the same investment as building a golden dataset for any other eval — worth doing once and reusing every time you reconsider the embedding model.

## Dimensions Beyond Accuracy

Accuracy on your eval set is the primary signal, but three other factors matter in production:

- **Embedding dimension** — higher-dimensional embeddings (3072 vs 384) cost more to store and search, with diminishing accuracy returns past a point that varies by corpus
- **Inference cost and latency** — a hosted API model has a per-call cost; a self-hosted open model has infrastructure cost and needs GPU capacity for acceptable indexing throughput
- **Context length support** — if your chunks run long, verify the model's actual effective context length, not just its stated maximum, which sometimes degrades well before the limit

## Key Takeaways

1. **MTEB leaderboard rank is a filter for a shortlist, not the final decision**
2. **Build a small, labeled, corpus-specific eval set** — 50-100 real queries with known-correct documents
3. **Weigh embedding dimension and inference cost alongside accuracy** — the best-scoring model isn't always the best-fit model
4. **Reuse the eval set** every time the embedding model choice is reconsidered, not just once at launch

---

*Part of the [Agentic AI in Practice series](/tags/agentic-ai-series/) — lessons from building production multi-agent systems.*
