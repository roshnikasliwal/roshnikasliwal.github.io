---
title: "Query Rewriting: Turning Bad User Questions into Good Retrieval Queries"
date: 2026-04-28
mermaid: true
categories: [AI, Agentic AI]
tags: [rag, query-rewriting, retrieval, agentic-ai-series]
author: Roshni Kasliwal
description: The question a user types and the query that retrieves the right documents are often not the same string. Query rewriting closes that gap before retrieval, not after.
---

A user types "it's broken again, same as last time" into a support search box. Embedded and retrieved as-is, that query matches almost nothing useful — it has no nouns a vector search can anchor to. The fix isn't a better embedding model; it's rewriting the query before it ever reaches retrieval.

## Where Raw Queries Fail Retrieval

- **Pronoun and context dependence**: "same as last time" refers to something only present in prior conversation turns, not in the query string itself
- **Vague or colloquial phrasing**: "it's broken" carries no retrievable specifics
- **Compound questions**: "what's our refund policy and does it apply to gift cards" is really two retrieval queries bundled into one
- **Query-document vocabulary mismatch**: a user says "can't log in," the documentation says "authentication failure" — semantically related, but exact-match components (like the BM25 half of a hybrid search) will miss it entirely

## The Rewrite Step

```python
def rewrite_query(raw_query: str, conversation_history: list[str]) -> list[str]:
    prompt = f"""Conversation so far:
{format_history(conversation_history)}

Latest user message: "{raw_query}"

Rewrite this into one or more standalone, specific search queries suitable
for a retrieval system. Resolve any pronouns or references to earlier
conversation. If the message contains multiple distinct questions, return
multiple queries. Return a JSON list of strings."""

    response = llm.invoke(prompt)
    return parse_json_list(response.content)

# "it's broken again, same as last time" + history mentioning "password reset email"
# -> ["password reset email not being delivered", "recurring password reset failures"]
```

Returning a *list* rather than a single rewritten string matters — compound questions retrieve better as separate queries fused afterward (the same [reciprocal rank fusion](/posts/hybrid-search-bm25-embeddings/) approach used for combining BM25 and vector results works here too) than as one merged query that dilutes both intents.

## Don't Discard the Original Query

Rewriting improves retrieval, but the original query is still the most reliable signal for two things: understanding what the user actually meant for the final generation step, and catching a rewrite that went wrong. Keep both, and pass the original alongside the rewritten version(s) into the generation prompt:

```python
def retrieve_and_generate(raw_query: str, history: list[str]) -> str:
    rewritten = rewrite_query(raw_query, history)
    all_results = []
    for q in rewritten:
        all_results.extend(retriever.search(q, top_k=5))
    fused = deduplicate_and_rank(all_results)
    return generate_answer(original_query=raw_query, context=fused)
```

## When Rewriting Isn't Worth It

For a system where users already type well-formed, self-contained questions — a documentation search bar with no conversation history, for instance — query rewriting adds an LLM call's worth of latency for marginal gain. It earns its cost specifically in conversational contexts where queries depend on prior turns, and in domains with a strong vocabulary mismatch between how users ask and how source documents are written.

## Key Takeaways

1. **Raw user queries often lack the specifics a retriever needs** — pronouns, vagueness, and vocabulary mismatch are the usual causes
2. **Rewrite into one or more standalone queries**, splitting compound questions rather than merging their intents
3. **Keep the original query for the generation step** — it's still the best signal for what the user actually meant
4. **Reserve rewriting for conversational or vocabulary-mismatched contexts** — it's not free, and not every system needs it

---

*Part of the [Agentic AI in Practice series](/tags/agentic-ai-series/) — lessons from building production multi-agent systems.*
