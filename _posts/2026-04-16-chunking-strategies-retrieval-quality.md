---
title: "Chunking Strategies That Actually Affect Retrieval Quality"
date: 2026-04-16
mermaid: true
categories: [AI, Agentic AI]
tags: [rag, chunking, embeddings, vector-search, agentic-ai-series]
author: Roshni Kasliwal
description: Fixed-size chunking is the default in every tutorial and the wrong choice for most real documents. Here's what actually improves retrieval quality.
---

Fixed-size chunking — split every 500 tokens, done — is where every RAG tutorial starts and where most production pipelines should have stopped starting. It's not wrong exactly; it's just leaving retrieval quality on the table for free.

## Why Fixed-Size Chunking Underperforms

A fixed window doesn't know where a thought ends. It routinely splits a sentence in half, separates a heading from the paragraph it introduces, or merges the tail of one section with the head of the next. Each of those degrades the embedding of both resulting chunks — they now represent a mash of two unrelated ideas instead of one coherent one, which weakens the similarity score for both.

## What Works Better

**Structure-aware chunking** splits on document structure first — headings, paragraphs, list boundaries — and only falls back to a size limit within a structural unit that's too long. For Markdown or HTML sources this is close to free:

```python
from langchain_text_splitters import MarkdownHeaderTextSplitter, RecursiveCharacterTextSplitter

headers_to_split_on = [("#", "h1"), ("##", "h2"), ("###", "h3")]
header_splitter = MarkdownHeaderTextSplitter(headers_to_split_on)
sections = header_splitter.split_text(document_markdown)

# Only recursively split sections that exceed the size budget
size_splitter = RecursiveCharacterTextSplitter(chunk_size=800, chunk_overlap=100)
chunks = []
for section in sections:
    if len(section.page_content) > 800:
        chunks.extend(size_splitter.split_documents([section]))
    else:
        chunks.append(section)
```

**Semantic chunking** goes further, embedding sentence-level windows and splitting where consecutive-sentence similarity drops — a proxy for a genuine topic shift rather than an arbitrary character count. It costs more compute at index time and is worth it for high-value, low-volume corpora (legal, medical, internal policy docs) where retrieval precision matters more than indexing speed.

## Overlap Is a Tradeoff, Not a Default

A 10-20% overlap between chunks prevents a fact from being fully lost when it lands right at a boundary. But overlap isn't free — it multiplies index size and introduces near-duplicate chunks that can crowd out genuinely different results in a top-k retrieval. Start with structure-aware splitting *before* reaching for overlap as the fix; overlap is a patch for boundary loss, not a substitute for splitting in the right place.

## Keep the Parent Context

Even a well-chunked passage loses the heading and surrounding context that made it unambiguous. Retrieving "the timeout is 30 seconds" without knowing which system it's about is nearly useless. Store a lightweight parent reference — the section heading, document title — as chunk metadata, and inject it back into the retrieved context even though it wasn't part of what was embedded:

```python
chunk_metadata = {
    "section_heading": "## Retry Configuration",
    "document_title": "Payment Service Runbook",
    "chunk_index": 3,
}
```

## Key Takeaways

1. **Split on document structure before falling back to a size limit** — most formats give you this for near-free
2. **Semantic chunking is worth the extra indexing cost for high-value, low-volume corpora**
3. **Overlap patches boundary loss — it doesn't replace splitting in the right place, and it isn't free**
4. **Carry parent context (headings, titles) as metadata** so a retrieved chunk isn't ambiguous on its own

---

*Part of the [Agentic AI in Practice series](/tags/agentic-ai-series/) — lessons from building production multi-agent systems.*
