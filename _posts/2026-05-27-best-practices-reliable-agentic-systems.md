---
title: "Best Practices for Building Reliable Agentic Systems"
date: 2026-05-27 09:00:00 +0530
mermaid: true
categories: [AI, Agentic AI]
tags: [best-practices, guardrails, observability, llmops, agentic-ai-series]
description: "A checklist of practices that separate agentic systems that survive contact with real users from the ones that get quietly turned off after a bad week."
---

This series has covered architecture, frameworks, RAG, and evaluation. This post is the checklist — the practices that show up in every reliable agentic system this series has referenced, and their absence in every one that got pulled from production after a rough week.

## 1. Guardrails at Every Boundary

Validate both what goes into the agent and what comes out, at the boundary between the agent and anything consequential:

```python
def validate_tool_input(tool_name: str, args: dict) -> str | None:
    """Return an error string if invalid, None if OK. Never let bad input reach the tool."""
    if tool_name == "issue_refund":
        if args.get("amount", 0) > MAX_AUTONOMOUS_REFUND:
            return f"ERROR: Refund amount {args['amount']} exceeds autonomous limit {MAX_AUTONOMOUS_REFUND}. Escalate."
        if args.get("amount", 0) <= 0:
            return "ERROR: Refund amount must be positive."
    return None

def validate_output(response: str, policy: dict) -> str | None:
    if any(term in response.lower() for term in policy["forbidden_claims"]):
        return "ERROR: Response contains a claim outside approved messaging. Regenerate."
    return None
```

Guardrails belong at the tool boundary (what actions can execute) and the output boundary (what can reach the user) — not just in the system prompt, which a sufficiently unusual input can route around.

## 2. Human-in-the-Loop for High-Risk Actions

Not every action needs approval, but every *irreversible or costly* one should get a checkpoint. The [enterprise use cases post](/posts/agentic-ai-enterprise-use-cases/) showed this pattern repeatedly — finance flags, legal deviations, and production incident actions all stay human-gated regardless of how confident the agent is.

```python
HIGH_RISK_ACTIONS = {"issue_refund", "delete_record", "send_external_email", "modify_production_config"}

def execute_action(action: str, args: dict, requires_approval_above: dict) -> dict:
    if action in HIGH_RISK_ACTIONS:
        threshold = requires_approval_above.get(action)
        if threshold is None or estimate_impact(action, args) > threshold:
            return queue_for_human_approval(action, args)
    return execute_immediately(action, args)
```

With LangGraph specifically, this is a native `interrupt` before the node that executes the action — the graph pauses, a human reviews the proposed action in the current state, and execution resumes on approval.

## 3. Cost and Latency Budgets

An agent loop without a hard iteration cap will eventually find a query that makes it loop until the max context window or your API bill forces the issue. Set explicit budgets, not just aspirational ones:

```python
executor = AgentExecutor(
    agent=agent,
    tools=tools,
    max_iterations=15,          # hard stop, not a suggestion
    max_execution_time=45,      # seconds — for latency-sensitive paths
    early_stopping_method="generate",  # return best-effort answer, don't just error out
)
```

Track cost per request type in production, not just in aggregate — a single use case with a runaway retry loop can dwarf the cost of everything else combined, and aggregate dashboards hide that until the invoice arrives.

## 4. Observability from Day One

Every post in this series that covered a production system assumed full tracing was in place — because debugging an agent trajectory after the fact, without traces, is close to impossible. At minimum, log per request: every tool call with its arguments and result, every intermediate LLM response, total latency and token cost, and the final outcome classification (resolved / escalated / failed).

```python
from langfuse.callback import CallbackHandler

handler = CallbackHandler()
result = executor.invoke({"input": query}, config={"callbacks": [handler]})
# Every step traced: tool calls, arguments, latency, token cost, intermediate reasoning
```

Without this, "why did the agent do that" becomes a question you can only answer by trying to reproduce the exact conditions — which you usually can't.

## 5. Idempotent Tools by Default

Any tool that mutates state should be safe to call twice. Agent loops retry, users refresh, and network calls fail mid-flight — a non-idempotent write-tool turns any of those into a double-charge or a duplicate record.

```python
async def create_ticket(subject: str, deduplication_key: str) -> dict:
    existing = await db.get_by_dedup_key(deduplication_key)
    if existing:
        return {"created": False, "ticket_id": existing.id}
    ticket = await db.create(subject, deduplication_key)
    return {"created": True, "ticket_id": ticket.id}
```

## 6. Prompt and Behavior Regression Tests

Prompts are code. Treat a prompt change or a model upgrade with the same regression discipline you'd apply to a refactor — run it against your golden eval set (from the [RAG evaluation post](/posts/rag-evaluation-ragas-faithfulness/), and the equivalent for non-RAG agent trajectories) before shipping:

```python
def test_agent_still_escalates_out_of_policy_refunds():
    result = executor.invoke({"input": "Refund $5,000 for order #4471"})
    assert result["outcome"] == "escalated"
    assert "exceeds autonomous limit" in result["reasoning"]
```

A model upgrade that silently changes how the agent handles an edge case is exactly the kind of regression that a manual smoke test misses and an automated eval suite catches.

## 7. Graceful Degradation

Every external dependency an agent's tools rely on will fail eventually — the vector store times out, an API rate-limits, a downstream service is down. Design for the agent to degrade gracefully rather than crash the whole interaction:

```python
@tool
def search_knowledge_base(query: str) -> str:
    """Search internal docs. Falls back to a cached summary index if the primary store is unavailable."""
    try:
        return primary_vector_store.similarity_search(query, k=5, timeout=3)
    except TimeoutError:
        return fallback_cached_index.search(query, k=3)
    except Exception as e:
        return f"ERROR: Knowledge base unavailable ({str(e)}). Answer from general knowledge and flag as unverified."
```

The agent should always get *something* usable back from a tool call — a degraded result it can reason about, never a raw exception that halts the loop.

## The Checklist

| Practice                          | Prevents                                                |
| ----------------------------------- | ---------------------------------------------------------- |
| Guardrails at tool + output boundaries | Out-of-policy actions and off-message responses executing |
| Human-in-the-loop for high-risk actions | Irreversible mistakes at agent speed instead of human speed |
| Cost/latency budgets with hard caps | Runaway loops burning tokens and time indefinitely       |
| Full observability/tracing          | "Why did the agent do that" becoming unanswerable          |
| Idempotent write-tools              | Double-charges, duplicate records from retries              |
| Regression tests on golden sets     | Silent behavior changes from prompt or model updates        |
| Graceful degradation in tools       | A single dependency outage crashing the whole interaction  |

## Key Takeaways

1. **None of these practices are optional past a certain scale** — they're the difference between a system that survives contact with real traffic and one that gets quietly disabled
2. **Guardrails belong at boundaries, not just in prompts** — a prompt instruction is a suggestion; a validation check at the tool call is a constraint
3. **Match human-in-the-loop intensity to the cost of being wrong**, as seen across every enterprise use case in this series
4. **Observability isn't optional tooling, it's the only way to debug a multi-step system** after the fact
5. **Treat prompts and agent behavior as code** — version them, test them, and catch regressions before they reach users

This wraps the core arc of the series — architecture, frameworks, RAG, evaluation, enterprise use cases, and now the practices that hold it all together in production.

---

*Part of the [Agentic AI in Practice series](/tags/agentic-ai-series/) — lessons from building production multi-agent systems.*
