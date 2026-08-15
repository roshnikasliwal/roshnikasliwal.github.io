---
title: "Field Note: Debugging a CrewAI Agent That Wouldn't Stop Looping"
date: 2026-04-09
mermaid: true
categories: [AI, Agentic AI]
tags: [crewai, debugging, agents, python, field-notes, agentic-ai-series]
author: Roshni Kasliwal
description: A concrete walkthrough of an agent stuck re-delegating the same subtask forever, why max_iter alone didn't fix it, and the three checks that actually catch this class of bug.
---

Last week a CrewAI orchestrator agent burned through 40,000 tokens and $3 on a single request before hitting its iteration ceiling — for a task that should have taken four tool calls. This is the debugging session, because the failure mode is common enough to be worth writing down.

## What It Looked Like

The crew had an orchestrator delegating research to a specialist. The specialist would return a result, and the orchestrator would... delegate the same subtask again, with nearly identical wording. Not an infinite loop in the literal sense — `max_iter` eventually cut it off — but a loop in substance: work was repeating without making progress.

```python
crew = Crew(
    agents=[orchestrator, specialist],
    tasks=[research_task],
    process=Process.hierarchical,
    manager_llm=llm,
    verbose=True,
)
```

`verbose=True` is where this debugging session started, and it's the first thing to turn on when an agent looks stuck — the raw thought/action trace shows you what the model *believes* it's doing, which is usually different from what you assumed.

## The Real Cause

The specialist's tool was returning a result, but the result string didn't clearly signal *completion*. It read like a progress update ("Found 3 relevant sources, continuing to verify...") rather than a final answer. The orchestrator, reading that back, reasonably concluded the subtask wasn't done and delegated it again.

This is a framing bug, not a logic bug. The tool was working correctly. The orchestrator was reasoning correctly given the input. The mismatch was in what the tool's output *implied* to a model reading it as natural language.

## Three Checks for This Class of Bug

**1. Read the last tool output like the model does.** Not what the data contains — what the phrasing implies about task state. Ambiguous words like "continuing," "still," or "in progress" in a supposedly-final result are a signal.

**2. Check whether `expected_output` on the Task is specific enough to serve as a stopping condition.**

```python
research_task = Task(
    description="Research {topic} and identify the top sources.",
    expected_output=(
        "A final list of exactly 3 sources with title, URL, and one-line "
        "relevance summary each. This is the complete, final answer — do not "
        "delegate this task again once this list is produced."
    ),
    agent=specialist,
)
```

That last sentence looks redundant until you've watched an orchestrator re-delegate a technically-complete result three times.

**3. Log delegation count per subtask, not just total iterations.** `max_iter` tells you the crew stopped; it doesn't tell you *why* it needed that many iterations. A simple counter keyed on task description similarity catches the repeat-delegation pattern before it burns your whole budget.

```python
from collections import Counter

delegation_log = Counter()

def track_delegation(task_description: str):
    key = task_description[:80]  # crude but effective similarity proxy
    delegation_log[key] += 1
    if delegation_log[key] > 2:
        logger.warning(f"Possible delegation loop: {key!r} delegated {delegation_log[key]}x")
```

## Key Takeaways

1. **`verbose=True` first** — the raw reasoning trace shows what the model believes, which is often not what you assumed
2. **Tool output phrasing matters as much as tool output correctness** — ambiguous "in progress" language reads as incomplete even when the task is done
3. **Make `expected_output` a stopping condition, not just a format spec**
4. **Track delegation counts per subtask** — `max_iter` alone won't tell you where the budget actually went

---

*Part of the [Agentic AI in Practice series](/tags/agentic-ai-series/) — lessons from building production multi-agent systems.*
