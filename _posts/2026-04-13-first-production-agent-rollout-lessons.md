---
title: "Field Note: What Broke in Our First Production Agent Rollout"
date: 2026-04-13
mermaid: true
categories: [AI, Agentic AI]
tags: [production, incident, lessons, agentic-ai, field-notes, agentic-ai-series]
author: Roshni Kasliwal
description: An honest account of the failures in the first week of a production agent rollout — none of them were the failure mode we'd prepared for.
---

We spent three weeks hardening an agent before its first production rollout: guardrails, fallbacks, evaluation gates, the works. The failures that actually happened in week one weren't in any of that. Writing them down because the pattern — prepared for the obvious failures, blindsided by the boring ones — is worth naming.

## Failure 1: The Timeout We Set for the Wrong Thing

We set a 30-second timeout on the agent's overall response, reasoning that no user should wait longer than that. What we hadn't accounted for: a legitimate multi-step research task could genuinely need 45 seconds when the underlying API was having a slow day. The timeout wasn't protecting against a bug — it was cutting off correct, in-progress work.

The fix wasn't a longer timeout. It was separating "the agent is stuck" from "the agent is working slowly but making progress":

```python
async def run_with_progress_timeout(agent_run, no_progress_timeout=15, hard_timeout=90):
    last_progress = time.monotonic()
    async for event in agent_run:
        if event.type == "progress":
            last_progress = time.monotonic()
        if time.monotonic() - last_progress > no_progress_timeout:
            raise NoProgressError("No progress in 15s — likely stuck")
        if time.monotonic() - start > hard_timeout:
            raise HardTimeoutError("Exceeded absolute time budget")
        yield event
```

A hard ceiling still exists, but it's generous, because the thing actually worth failing fast on is *no progress*, not *total elapsed time*.

## Failure 2: A Guardrail That Was Too Good at Its Job

Our output guardrail flagged responses containing what looked like unverified claims about specific numbers. It worked exactly as designed — and it also flagged the agent correctly quoting a number the user had provided earlier in the conversation, because the guardrail had no way to distinguish "the model invented this number" from "the model is repeating a number the user gave it."

The guardrail wasn't wrong to be cautious. It was missing a distinction it needed to make that distinction correctly. We ended up passing the guardrail a diff of what appeared in context versus what didn't, rather than scanning the output in isolation.

## Failure 3: Real Users Don't Type Like Test Users

Our eval set — carefully built, genuinely representative of the use cases we designed for — had zero examples of a user pasting a wall of unformatted text with the actual question buried in the third paragraph. Real users do this constantly. The agent handled clean, well-formed requests well and struggled to even locate the actual ask in messy ones.

This is the failure mode that stung the most, because it's not a bug you can architecture your way out of — it's a gap in what you tested against. The fix was mundane: sample real (anonymized) user inputs from the first 48 hours and add the messy ones to the eval set, rather than trying to anticipate messiness in advance.

## The Actual Lesson

None of these three were the "agent hallucinates" or "agent goes off the rails" failure mode we'd spent three weeks preparing for. They were boundary conditions in infrastructure we thought we'd already gotten right — timeouts, guardrails, and eval coverage — that only showed up under real, messy, high-volume traffic.

## Key Takeaways

1. **Distinguish "stuck" from "slow but working"** — a hard timeout alone punishes legitimate multi-step work
2. **Guardrails need context, not just output text** — a rule that can't tell "invented" from "repeated from context" will misfire
3. **Real user input breaks assumptions your eval set didn't test** — sample actual production inputs early and often
4. **The failures that actually happen are rarely the ones you spent the most time preparing for** — budget time for post-launch observation, not just pre-launch hardening

---

*Part of the [Agentic AI in Practice series](/tags/agentic-ai-series/) — lessons from building production multi-agent systems.*
