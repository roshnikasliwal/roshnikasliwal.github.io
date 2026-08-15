---
title: "Budgeting a Context Window Like It's a Scarce Resource"
date: 2026-08-01
mermaid: true
categories: [AI Engineering, Agent Infrastructure]
tags: [context-engineering, agent-memory, field-notes, agent-infra-series]
author: Roshni Kasliwal
description: A large context window feels like it removes the constraint entirely. It doesn't — it just changes where the cost shows up, from truncation bugs to degraded attention and higher latency.
---

A 200K-token context window feels, at first, like it removes context management as a real engineering concern — just put everything relevant in and let the model sort it out. It doesn't remove the constraint; it moves where the cost shows up, from hard truncation failures (an obviously broken response when context is cut off) to soft degradation (a model that technically has all the information but attends to it less reliably as the context fills up, plus real latency and cost that scale with token count regardless of whether every token was useful).

## Treat the Budget as a Line Item, Not an Afterthought

```python
CONTEXT_BUDGET = {
    "system_prompt": 500,
    "constitution/steering": 1500,
    "conversation_history": 4000,
    "retrieved_context": 6000,
    "tool_definitions": 2000,
    "reserve_for_response": 4000,
}
TOTAL_BUDGET = sum(CONTEXT_BUDGET.values())  # explicit target, well under the hard model limit
```

Setting an explicit budget per category — well under the model's actual maximum — forces a decision about what gets cut when something exceeds its allocation, rather than discovering the tradeoff implicitly when the model's behavior degrades in a way that's hard to diagnose back to a specific cause.

## Where the Budget Actually Gets Spent Without Anyone Deciding To

```mermaid
flowchart TD
    A[Common uncontrolled context growth] --> B[Full conversation history, never summarized]
    A --> C[Full tool results returned verbatim, not shaped]
    A --> D[All available tool schemas passed, not retrieved subset]
    A --> E[Retrieved context over-fetched "just in case"]
```

Each of these individually seems reasonable in the moment a feature is built, and in aggregate they consume budget nobody explicitly allocated to them. The skill-discovery-at-scale post earlier in this blog covered the tool-schema case directly; the same discipline — retrieve or shape the relevant subset rather than including everything — applies to each of the other categories too.

## Measure Actual Token Spend Per Category, Not Just Total

```python
def log_context_composition(request_id: str, prompt_parts: dict):
    for category, text in prompt_parts.items():
        token_count = count_tokens(text)
        log_metric("context_tokens", category=category, count=token_count, request_id=request_id)
```

Aggregate token count tells you the total cost. Per-category breakdown tells you *why* — the same distinction between aggregate and attributed cost from the earlier post on [attributing LLM cost to teams](/posts/attributing-llm-cost-to-teams/), applied to context composition instead of team ownership. Without it, "our context is too full" has no actionable next step; with it, the specific category consuming an unexpected share of the budget is immediately visible.

## A Full Budget Isn't Automatically a Problem — An Unmeasured One Is

The point isn't that every request should minimize token usage — some tasks genuinely need extensive context. The point is that spend should be a deliberate allocation decision, visible and measured, rather than an emergent consequence of nobody having looked. A context window feeling unconstrained is exactly the condition under which nobody looks, until degraded response quality or a cost review forces the question.

## Key Takeaways

1. **A large context window shifts the cost from hard truncation to soft degradation and higher latency/cost** — it doesn't remove the constraint
2. **Set an explicit per-category token budget**, well under the model's actual limit, to force deliberate tradeoff decisions
3. **Uncontrolled growth usually comes from unshaped tool results, full history, and over-fetched retrieval** — the same shaping discipline from earlier posts applies across all of them
4. **Measure token spend per category, not just in aggregate** — that's what makes "context is too full" an actionable finding instead of a vague complaint

---

*Part of the [Agent Infrastructure series](/tags/agent-infra-series/) — the plumbing layer underneath production agentic systems.*
