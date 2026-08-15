---
title: "Vector Database Shootout: pgvector, Pinecone, Qdrant, and Weaviate"
date: 2026-04-20
mermaid: true
categories: [AI, Agentic AI]
tags: [rag, vector-database, pgvector, pinecone, qdrant, comparison, agentic-ai-series]
author: Roshni Kasliwal
description: "A practical comparison of four vector database options across the axes that actually decide a production choice: operational burden, filtering, hybrid search, and cost at scale."
---

Every one of these four can serve as the vector store for a RAG pipeline. The differences that matter aren't raw recall-at-k benchmarks — they're mostly tied — they're operational: who runs it, how filtering works, and what it costs once you're past the free tier.

## The Comparison

| | pgvector | Pinecone | Qdrant | Weaviate |
|---|---|---|---|---|
| **Operational model** | Self-hosted (it's a Postgres extension) | Fully managed only | Self-hosted or managed | Self-hosted or managed |
| **Best fit** | Already running Postgres, want one less system | Want zero ops, will pay for it | Self-hosting with strong filtering needs | Need built-in hybrid search + modules |
| **Metadata filtering** | Full SQL — extremely expressive | Good, but a distinct DSL from your app's queries | Strong, purpose-built for it | Strong, GraphQL-based |
| **Hybrid search (built-in)** | No — build it yourself alongside full-text search | Yes | Yes | Yes |
| **Cost at scale** | Your existing Postgres infra cost | Scales with vector count + queries, can get expensive | Self-hosted: infra cost only | Self-hosted: infra cost only |

## The Case for pgvector

If the rest of your application already runs on Postgres, pgvector is the pragmatic default: one fewer system to operate, one fewer data-consistency boundary between your relational data and your vectors, and you can join a vector search against relational filters in a single SQL query. The tradeoff is that it's not purpose-built for vector search at very large scale — for corpora in the tens of millions of vectors with high query throughput, a dedicated vector database's indexing (HNSW variants tuned specifically for this) tends to outperform pgvector's.

```sql
-- pgvector: vector search joined with relational filters in one query
SELECT id, content, embedding <=> $1 AS distance
FROM documents
WHERE tenant_id = $2 AND status = 'published'
ORDER BY distance
LIMIT 10;
```

## The Case for Pinecone

Zero operational burden — no cluster to size, no index to tune, no upgrades to schedule. That's genuinely valuable for a small team without dedicated infrastructure capacity. The cost is real money that scales with usage, and a metadata filtering DSL that's yet another system-specific syntax to learn, separate from wherever your other application queries live.

## The Case for Qdrant or Weaviate

Both give you native hybrid search and strong filtering without giving up self-hosting control (or you can use their managed offerings if you want the ops taken off your plate while keeping the cost model closer to infrastructure-based than per-query). Weaviate's module ecosystem (built-in rerankers, generative modules) is convenient if you want fewer moving pieces to wire up yourself; Qdrant's filtering performance under high cardinality metadata tends to hold up particularly well.

## The Decision That Actually Matters

Before comparing feature checklists, answer one question: **do you want to operate infrastructure, or pay to not operate it?** That single axis eliminates half the field immediately and makes the rest of the comparison much faster.

## Key Takeaways

1. **Already on Postgres? pgvector removes a system instead of adding one** — until scale genuinely demands a dedicated index
2. **Pinecone trades money for zero operational burden** — a legitimate tradeoff for small teams, a real line item at scale
3. **Qdrant and Weaviate offer native hybrid search with self-hosting control** — a middle ground worth evaluating before defaulting to a fully managed option
4. **Decide "operate vs. pay to not operate" first** — it narrows the field faster than any feature comparison

---

*Part of the [Agentic AI in Practice series](/tags/agentic-ai-series/) — lessons from building production multi-agent systems.*
