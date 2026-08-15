---
title: "Caching Retrieved Context Without Serving Stale Answers"
date: 2026-04-25
mermaid: true
categories: [AI, Agentic AI]
tags: [rag, caching, performance, field-notes, agentic-ai-series]
author: Roshni Kasliwal
description: Caching retrieval results is an easy latency and cost win, right up until the underlying documents change and the cache doesn't know.
---

Caching a RAG pipeline's retrieval step is one of the cheapest wins available — repeated or near-duplicate queries skip the vector search entirely. It's also the easiest way to quietly start serving answers grounded in documents that have since been edited, deleted, or corrected, because a cache doesn't know anything changed unless you tell it to.

## What's Actually Safe to Cache

**The embedding of a query string** — deterministic for a given model and input, safe to cache indefinitely keyed on the exact query text (or a normalized version of it).

**The retrieval result for a query, keyed with a document-version fingerprint** — safe to cache, but only if invalidated the moment any retrieved document changes, not on a fixed TTL alone.

**The final generated answer** — the riskiest to cache, because it's furthest from the source of truth. Caching final answers is reasonable for genuinely static content (a product FAQ that changes rarely) and risky for anything time-sensitive or frequently corrected.

## Fingerprinting Instead of Time-Based Expiry

A fixed TTL cache (e.g. "expire after 1 hour") is either too aggressive (discarding a valid cache entry for content that hasn't changed) or too stale (serving outdated content for up to an hour after a document was corrected). A content fingerprint solves both:

```python
import hashlib

def document_fingerprint(doc_ids: list[str], doc_store) -> str:
    """Hash of the last-modified timestamps of every doc that could be retrieved for this query."""
    versions = sorted(f"{doc_id}:{doc_store.get_version(doc_id)}" for doc_id in doc_ids)
    return hashlib.sha256("|".join(versions).encode()).hexdigest()[:16]

def get_cached_retrieval(query: str, candidate_doc_ids: list[str]):
    fingerprint = document_fingerprint(candidate_doc_ids, doc_store)
    cache_key = f"retrieval:{hash(query)}:{fingerprint}"
    cached = cache.get(cache_key)
    if cached:
        return cached
    result = run_retrieval(query)
    cache.set(cache_key, result, ttl=86400)  # long TTL is safe — the fingerprint invalidates on change
    return result
```

Because the fingerprint is derived from document versions, not wall-clock time, a long TTL becomes safe — the cache key itself changes the instant a relevant document is edited, so there's no window where a stale result can be served under a valid-looking key.

## The Harder Problem: Knowing Which Documents "Could Be Retrieved"

Fingerprinting works cleanly when you know in advance which documents a query might touch. For genuinely open-ended retrieval across a large corpus, that's not knowable ahead of time. The practical compromise: fingerprint the *retrieved set*, not the full candidate space, and accept a narrower cache-invalidation guarantee — a newly-edited document that would have ranked in the top-k for this query but wasn't retrieved before the edit won't trigger invalidation of an existing cache entry. Combine this with a short-enough TTL as backstop for that edge case.

## Key Takeaways

1. **Cache query embeddings freely — they're deterministic and cheap to key**
2. **Cache retrieval results keyed on a document-version fingerprint, not a fixed TTL alone**
3. **Be most cautious caching final generated answers** — they're furthest from the source of truth
4. **A version-fingerprinted cache key lets you use long TTLs safely**, since the key itself changes when content changes

---

*Part of the [Agentic AI in Practice series](/tags/agentic-ai-series/) — lessons from building production multi-agent systems.*
