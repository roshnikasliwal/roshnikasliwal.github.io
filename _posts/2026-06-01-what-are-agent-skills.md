---
title: "What Are Agent Skills? A Practical Introduction"
date: 2026-06-01 09:00:00 +0000
categories: [AI Engineering, Agent Design]
tags: [agents, skills, llm, tool-use]
author: Roshni Kasliwal
description: Agent skills are the modular building blocks that give LLMs the ability to act. This post breaks down what they are, how they work, and why getting the design right matters.
---

If you've spent any time building with LLMs, you've probably hit the wall where the model "knows" something but can't *do* anything with that knowledge. Enter **agent skills** — the mechanism that bridges knowledge and action.

## The Core Idea

An agent skill is a well-defined, callable capability that an AI agent can invoke during reasoning. Think of it as a function the model can reach for: search the web, query a database, send a message, run a calculation. The model decides *when* to use a skill; the skill handles *how* to actually do it.

At the implementation level, most frameworks represent skills as tool definitions — a name, a description, and a schema for inputs and outputs. The LLM sees the description and learns to match intent to capability.

```json
{
  "name": "search_knowledge_base",
  "description": "Search the internal knowledge base for documentation, policies, or reference material. Use when the user asks about internal processes or company-specific information.",
  "parameters": {
    "query": {
      "type": "string",
      "description": "A concise search query capturing the user's information need"
    }
  }
}
```

Simple enough. But the design decisions behind that small JSON object have enormous downstream consequences.

## Why the Description Is Everything

The model doesn't see your code. It sees the description. That description is what it uses to decide whether to invoke the skill at all — and how to construct the inputs.

A bad description leads to:
- **Under-use**: the model doesn't recognize when the skill is relevant
- **Over-use**: the model reaches for it in cases it wasn't designed for
- **Bad inputs**: the model passes in something malformed because it wasn't clear what was expected

A good description is specific about *when* to use the skill, not just *what* it does. "Searches the knowledge base" is weaker than "Searches the internal knowledge base for documentation, policies, or reference material — use when the user asks about internal processes or company-specific information."

## The Three Layers of a Skill

I think of agent skills as having three distinct layers, each with its own concerns:

### 1. The Interface Layer
This is what the model sees: name, description, input schema. The job here is to communicate intent precisely enough that the model makes good decisions. No implementation details leak through here.

### 2. The Execution Layer
This is where the skill actually runs: API calls, database queries, computations. The job here is reliability. Skills should be idempotent where possible, handle errors gracefully, and return structured, predictable output.

### 3. The Output Layer
This is what gets fed back to the model: the result of the skill execution. The job here is to return *just enough* information for the model to continue reasoning — not a raw API dump, but a distilled, model-friendly representation.

## Atomic vs. Composite Skills

One of the most important design decisions is granularity. Should you build one big "do everything" skill, or lots of small, focused ones?

**Atomic skills** do one thing. `get_customer_by_id`, `send_email`, `calculate_tax`. They're easy to test, easy to compose, and the model can chain them in sequences you didn't anticipate.

**Composite skills** bundle multiple operations. `research_and_summarize`, `create_and_assign_task`. They reduce the number of model decisions and can be faster, but they're less flexible and harder to debug when something goes wrong.

My general rule: start atomic, compose when you see a pattern. If the model is always calling skill A immediately before skill B, that's a signal to consider a composite.

## What Makes a Skill Production-Ready

There's a big gap between a skill that works in a notebook and one you'd trust in production. A few things that separate them:

- **Input validation** — don't trust the model to pass well-formed inputs. Validate and return clear errors.
- **Timeouts** — skills that hang will stall your entire agent loop.
- **Structured output** — return consistent types. If you sometimes return a list and sometimes a string, the model will get confused.
- **Error messages the model can reason about** — "404: resource not found" is better than a stack trace. The model may be able to recover and try something different.

## Coming Up

In the next post, I'll walk through building your first agent skill end-to-end — from writing the interface definition to handling errors in the execution layer. We'll build something real: a skill that searches GitHub issues and returns a structured summary.

---

*Tags: agent design, tool use, LLM, AI engineering*
