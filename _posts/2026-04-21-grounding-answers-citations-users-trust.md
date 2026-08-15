---
title: "Grounding Answers with Citations Users Actually Trust"
date: 2026-04-21
mermaid: true
categories: [AI, Agentic AI]
tags: [rag, citations, grounding, trust, agentic-ai-series]
author: Roshni Kasliwal
description: A citation that doesn't survive a click is worse than no citation at all. Here's how to build a citation pipeline that points to the exact passage, not just the source document.
---

The fastest way to lose a user's trust in a RAG system isn't a wrong answer — it's a wrong *citation*. A confidently wrong answer with no citation reads as a model limitation. A confidently wrong answer with a citation that, when clicked, doesn't actually say what the answer claims — that reads as the system lying, and it's far more damaging to trust.

## Document-Level Citations Aren't Enough

The easy version of citations — "according to [Document Title]" — technically satisfies a requirement to cite sources, and it's nearly useless for verification. A 40-page document doesn't help a user confirm a claim; they'd have to search it themselves, which defeats the purpose of citing it at all.

## Passage-Level Grounding

The fix is carrying the exact retrieved chunk through to the citation, not just its parent document:

```python
def generate_grounded_answer(query: str, retrieved_chunks: list[dict]) -> dict:
    context = "\n\n".join(
        f"[{i+1}] {chunk['text']}" for i, chunk in enumerate(retrieved_chunks)
    )
    prompt = f"""Answer the question using ONLY the numbered sources below.
Cite the source number in brackets after each claim, e.g. [1].
If the sources don't contain the answer, say so explicitly.

Sources:
{context}

Question: {query}"""

    response = llm.invoke(prompt)
    return {
        "answer": response.content,
        "sources": [
            {"id": i + 1, "text": c["text"], "url": c["source_url"], "location": c.get("section_heading")}
            for i, c in enumerate(retrieved_chunks)
        ],
    }
```

The `[1]`-style inline citation, resolved against the actual chunk text and a deep link (URL plus section anchor where possible), lets a user click through and land on the exact passage — not the top of a document they then have to search themselves.

## Verify the Citations, Don't Just Generate Them

Models occasionally cite a source number that doesn't actually support the claim next to it — a subtler failure than outright hallucination, because the citation *looks* rigorous. A lightweight post-generation check catches the worst of this:

```python
def verify_citations(answer: str, sources: list[dict]) -> list[dict]:
    issues = []
    claims = extract_cited_claims(answer)  # regex or LLM-based claim extraction
    for claim in claims:
        source = next((s for s in sources if s["id"] == claim["source_id"]), None)
        if source is None:
            issues.append({"claim": claim["text"], "issue": "cites nonexistent source"})
            continue
        entailment = judge_llm.invoke(
            f"Does this source support this claim?\nSource: {source['text']}\nClaim: {claim['text']}\nAnswer yes or no."
        )
        if "no" in entailment.content.lower():
            issues.append({"claim": claim["text"], "issue": "source does not support claim"})
    return issues
```

This doesn't need to run on every request in a latency-sensitive path — sampling a percentage of production traffic for citation-accuracy monitoring is often the right tradeoff between coverage and cost.

## Key Takeaways

1. **Cite the exact passage, not just the parent document** — document-level citations don't let users actually verify anything
2. **Deep-link to the passage location** where the source format supports it (URL + anchor, page + paragraph)
3. **Verify citation-claim entailment**, at least via sampling — models can cite a real source that doesn't actually support the claim
4. **A wrong citation is worse than no citation** — it converts a visible limitation into an invisible one

---

*Part of the [Agentic AI in Practice series](/tags/agentic-ai-series/) — lessons from building production multi-agent systems.*
