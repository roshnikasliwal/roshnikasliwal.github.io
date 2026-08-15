---
title: "Human-in-the-Loop Interrupts in LangGraph"
date: 2026-06-13
mermaid: true
categories: [AI Engineering, Architecture]
tags: [langgraph, human-in-the-loop, agents, python]
author: Roshni Kasliwal
description: LangGraph's interrupt mechanism, combined with checkpointing, is what makes a genuinely durable approval pause possible — one that survives a process restart while it's waiting on a human.
---

A human-approval step in an agent workflow needs to actually *pause* — not poll in a loop, not hold a process open waiting — potentially for hours or days until someone reviews it. LangGraph's `interrupt` mechanism, combined with the checkpointing covered in the previous post, is what makes that pause durable: the graph's execution state is checkpointed at the interrupt point, the process can shut down entirely, and execution resumes exactly where it left off whenever the human approval actually arrives.

## The Pattern

```python
from langgraph.types import interrupt, Command

def request_approval(state: GraphState) -> GraphState:
    decision = interrupt({
        "action": state["proposed_action"],
        "reason": state["reasoning"],
        "risk_level": state["risk_assessment"],
    })
    # Execution pauses here — the process can fully shut down.
    # It resumes from this exact point once resumed externally.
    if decision["approved"]:
        return {"status": "approved", "approver": decision["approver"]}
    return {"status": "rejected", "reason": decision.get("reason")}
```

```python
# Resuming later — potentially in a completely different process
graph.invoke(
    Command(resume={"approved": True, "approver": "jsmith"}),
    config={"configurable": {"thread_id": thread_id}},
)
```

The `thread_id` is what ties the resume call back to the exact checkpointed state — this is why checkpointing and interrupts are inseparable in practice: an interrupt without durable checkpointing can only pause within a single running process, which defeats the purpose for anything beyond very short waits.

## Design the Approval Payload for the Human, Not the Model

```mermaid
flowchart LR
    A[interrupt() call] --> B[Payload shown to human reviewer]
    B --> C{Clear enough to decide without digging into logs?}
    C -->|Yes| D[Fast, confident approval]
    C -->|No| E[Reviewer has to reconstruct context — slow, error-prone]
```

The same principle from the earlier post on [escalation design patterns](/posts/escalation-design-patterns-agent-ask-human/) applies directly here — the `interrupt` payload is the interface a human actually interacts with, and if it's raw internal state rather than a clear summary of what's being asked and why, the interrupt mechanism works correctly at the infrastructure level while still producing a bad human experience at the UX level.

## Multiple Interrupt Points in One Graph

A graph can have several distinct interrupt points for different kinds of decisions — one for high-value action approval, another for ambiguous-case escalation, a third for final output review before it's sent externally. Each resumes independently, keyed on the same `thread_id` but distinguishable by which node raised the interrupt, letting a single conversation flow through multiple human touchpoints without needing separate graph executions for each.

## Timeout Handling for Interrupts

An interrupt with no response — a human reviewer who never gets to it — needs a defined behavior, not an indefinitely paused graph. A separate monitoring process checking interrupt age against a timeout, and escalating (re-notify, route to a different reviewer, or auto-reject per policy) past that threshold, prevents a stalled approval queue from silently blocking work forever.

## Key Takeaways

1. **`interrupt` combined with a durable checkpointer lets a graph pause for hours or days**, fully shutting down the process in between
2. **`thread_id` ties a later resume call back to the exact checkpointed state** — this pairing is what makes long pauses possible
3. **Design the interrupt payload for the human reviewer**, not as a raw state dump — same discipline as any escalation UX
4. **Handle interrupt timeouts explicitly** — an unattended approval request needs a defined escalation path, not an indefinite pause

---

*Tags: LangGraph, human-in-the-loop, agents, AI engineering*
