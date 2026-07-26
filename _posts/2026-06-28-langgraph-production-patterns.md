---
title: "LangGraph in Production: Patterns for Reliable Multi-Step Agents"
date: 2026-06-28 09:00:00 +0000
mermaid: true
categories: [AI Engineering, Architecture]
tags: [langgraph, agents, architecture, production, python]
author: Roshni Kasliwal
description: The patterns that separate a LangGraph prototype from a graph that survives real production traffic — checkpointing, interrupts, subgraphs, and streaming.
---

I've shipped several LangGraph-based agents to production at this point, and the gap between a working demo graph and one that survives real traffic is consistently the same handful of patterns. None of these are exotic — they're mostly things LangGraph already supports natively — but they're easy to skip when a graph works fine in local testing and only matter once real users and real failures show up.

## Pattern 1: Durable Checkpointing, Not In-Memory

The default in-memory checkpointer is fine for a notebook. In production, a process restart mid-execution loses every in-flight conversation. Swap it for a durable backend from day one:

```python
from langgraph.checkpoint.postgres import PostgresSaver

with PostgresSaver.from_conn_string(DATABASE_URL) as checkpointer:
    checkpointer.setup()  # creates tables on first run
    graph = workflow.compile(checkpointer=checkpointer)

    config = {"configurable": {"thread_id": conversation_id}}
    result = graph.invoke(initial_state, config=config)
```

The `thread_id` is what makes this durable across restarts — as long as you persist it (in your session store, your database row, wherever the conversation is tracked), a graph that gets interrupted by a deploy can resume exactly where it left off instead of restarting the whole trajectory.

## Pattern 2: Human-in-the-Loop via Interrupts

I covered the general principle of gating high-risk actions behind human approval in an earlier post. In LangGraph specifically, this is a first-class primitive — `interrupt` pauses graph execution before a node runs, and the graph resumes only when you explicitly feed back a decision:

```python
from langgraph.types import interrupt, Command

def execute_refund_node(state: AgentState) -> dict:
    if state["refund_amount"] > AUTO_APPROVE_LIMIT:
        decision = interrupt({
            "action": "approve_refund",
            "amount": state["refund_amount"],
            "customer": state["customer_id"],
        })
        if decision != "approved":
            return {"status": "denied", "reason": decision}
    process_refund(state["customer_id"], state["refund_amount"])
    return {"status": "completed"}

# First invoke — hits the interrupt and pauses
result = graph.invoke(initial_state, config=config)
print(result["__interrupt__"])  # {"action": "approve_refund", "amount": 5000, ...}

# Resume after a human reviews it, potentially hours later
result = graph.invoke(Command(resume="approved"), config=config)
```

Because this is backed by the same checkpointer, the pause can last minutes or days — the state is durable, not held in a live process waiting on a callback. That distinction matters for anything that routes to a human queue instead of a synchronous approval dialog.

## Pattern 3: Subgraphs for Reusable Components

Once you have more than one graph that needs "retrieve, grade, rewrite if needed" — the [agentic RAG loop](/posts/agentic-rag-tool-using-agents/) is a good example — pull it out as a subgraph instead of copy-pasting the nodes into every parent graph:

```python
def build_retrieval_subgraph() -> StateGraph:
    sub = StateGraph(RetrievalState)
    sub.add_node("retrieve", retrieve_node)
    sub.add_node("grade", grade_node)
    sub.add_node("rewrite", rewrite_node)
    sub.set_entry_point("retrieve")
    sub.add_edge("retrieve", "grade")
    sub.add_conditional_edges("grade", route_after_grade, {"rewrite": "rewrite", "done": END})
    sub.add_edge("rewrite", "retrieve")
    return sub.compile()

retrieval_subgraph = build_retrieval_subgraph()

main_workflow = StateGraph(MainState)
main_workflow.add_node("retrieval", retrieval_subgraph)  # a compiled graph is a valid node
main_workflow.add_node("generate_final_answer", generate_node)
main_workflow.add_edge("retrieval", "generate_final_answer")
```

Subgraphs get their own checkpointed state and can be tested in complete isolation from whatever parent graph eventually embeds them — I test the retrieval subgraph on its own golden set before it's ever wired into a larger pipeline.

## Pattern 4: Streaming Intermediate Steps

Users staring at a blank screen for 15 seconds while a multi-node graph runs assume it's broken. Stream node-level progress instead of waiting for the final state:

```python
for event in graph.stream(initial_state, config=config, stream_mode="updates"):
    for node_name, node_output in event.items():
        send_progress_to_client({
            "step": node_name,
            "status": "completed",
        })
```

`stream_mode="values"` gives you the full state after each node if the UI needs to show intermediate content (like a partial draft), while `"updates"` gives just the diff — cheaper to ship over a websocket when you only need to show step progress rather than full state.

## Pattern 5: Node-Level Retry with Backoff

A single flaky node (a rate-limited API call, a transient DB timeout) shouldn't fail the whole trajectory. LangGraph lets you attach retry policy per node:

```python
from langgraph.pregel import RetryPolicy

workflow.add_node(
    "call_external_api",
    call_external_api_node,
    retry=RetryPolicy(max_attempts=3, backoff_factor=2.0, retry_on=(TimeoutError, RateLimitError)),
)
```

Scope the retry to the exception types that are actually transient — retrying a `ValidationError` three times just delays a failure that was never going to succeed, and burns the iteration budget I'd rather spend on genuine retries elsewhere in the graph.

## Pattern 6: Testing Graphs, Not Just Nodes

Node-level unit tests are necessary but not sufficient — the routing logic between nodes is where most of the bugs I've hit actually live. Test the graph's *paths*, not just its individual functions:

```python
def test_graph_takes_revision_loop_when_review_fails():
    initial_state = {"topic": "test", "revision_count": 0, "status": "researching"}
    result = graph.invoke(initial_state, config={"configurable": {"thread_id": "test-1"}})
    assert result["revision_count"] >= 1  # confirms the loop actually executed

def test_graph_stops_after_max_revisions():
    # inject a review_node mock that always requests revision
    with patch("graph_module.review_node", return_value={"status": "writing", "review_feedback": "needs work"}):
        result = graph.invoke(initial_state, config={"configurable": {"thread_id": "test-2"}})
        assert result["revision_count"] <= MAX_REVISIONS  # confirms the cap actually bites
```

The second test is the one I'd have skipped early on, and it's the one that would have caught a real incident — a routing condition that referenced the wrong state key and let a revision loop run unbounded until it hit the LLM provider's own error, not our intended cap.

## Summary Table

| Pattern                  | Problem It Solves                                     |
| --------------------------- | --------------------------------------------------------- |
| Durable checkpointing        | Process restarts losing in-flight conversations           |
| Human-in-the-loop interrupts | High-risk actions executing without review                |
| Subgraphs                    | Duplicated node logic across multiple parent graphs        |
| Streaming intermediate steps | Users assuming a working multi-step graph has hung         |
| Node-level retry with backoff | A single transient failure killing the whole trajectory   |
| Testing graph paths, not just nodes | Routing bugs that only show up under specific state transitions |

## Key Takeaways

1. **Swap the in-memory checkpointer for a durable one before anything reaches production** — a process restart shouldn't lose conversation state
2. **Use `interrupt` for anything that needs human approval**, and lean on the checkpointer to make the pause last as long as the approval queue needs
3. **Extract repeated node sequences into subgraphs** — test them in isolation, then embed them wherever the pattern recurs
4. **Stream progress for anything that takes more than a couple seconds** — a silent multi-node graph reads as broken to users
5. **Test routing logic explicitly, not just node functions** — that's where the bugs that actually reach production tend to live
