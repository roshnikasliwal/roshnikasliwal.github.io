---
title: "Episodic vs Semantic Memory in Practice"
date: 2026-08-10
mermaid: true
categories: [AI Engineering, Agent Infrastructure]
tags: [agent-memory, architecture, agent-infra-series]
author: Roshni Kasliwal
description: Most agent memory implementations store everything as one undifferentiated stream of facts. Separating episodic from semantic memory, the way cognitive science distinguishes them, changes both what gets retrieved and how well it ages.
---

Cognitive science distinguishes episodic memory (specific events — "we discussed the Q3 budget on Tuesday and you said you'd follow up") from semantic memory (general facts, decoupled from any specific occasion — "the user's budget authority is $50k"). Most agent memory implementations don't make this distinction, storing both as equivalent entries in one undifferentiated store — which loses information about *why* a fact is true and makes both retrieval and eviction less precise than they could be.

## Why the Distinction Matters Practically

```mermaid
flowchart TD
    E[Episodic: "On Tuesday, discussed Q3 budget, user said they'd follow up"] --> D1[Ages naturally — the follow-up either happened or the episode becomes historical]
    S[Semantic: "User's budget authority is $50k"] --> D2[Persists until explicitly superseded, no natural time decay]
```

An episodic memory has a natural relationship to time — it's tied to a specific occasion, and its relevance genuinely does decay as that occasion recedes into the past (the earlier eviction-policy post's time-based decay category maps cleanly onto episodic memory specifically). A semantic fact extracted from that episode doesn't have the same relationship to time — "$50k budget authority" stays true regardless of which conversation it was first mentioned in, until something explicitly changes it.

## Extracting Semantic Facts From Episodic Interactions

```python
def extract_semantic_facts(episodic_memory: dict) -> list[dict]:
    prompt = f"""From this conversation excerpt, extract durable facts about
the user that would remain true independent of this specific conversation.
Do not extract facts that are specific to this one interaction (e.g., "asked
about X today") — only extract genuinely durable information (preferences,
attributes, ongoing commitments).

Conversation: {episodic_memory['content']}"""
    facts = llm.invoke(prompt)
    return [{"fact": f, "source_episode": episodic_memory["id"], "extracted_at": time.time()} for f in parse_facts(facts)]
```

This extraction step is what turns a raw conversation log into structured, durable semantic memory — the episodic record stays as the source of truth for "when and how did we learn this," while the semantic fact becomes what's actually retrieved for most ongoing-context purposes, since it's more directly useful than the full conversation it was extracted from.

## Two Separate Retrieval Paths

```mermaid
flowchart LR
    Query[Agent needs context] --> Sem[Semantic retrieval: current facts about user/domain]
    Query --> Epi[Episodic retrieval: relevant to "what happened when" questions]
    Sem --> Most[Used for most ongoing interactions]
    Epi --> Specific[Used when the question specifically needs history: "what did we discuss last time"]
```

Most agent interactions benefit primarily from semantic retrieval — the current, durable facts relevant to the task at hand. Episodic retrieval matters specifically when a request needs history — "what did we discuss last time," "has this come up before" — and treating these as the same retrieval call, undifferentiated, means neither is well-optimized: semantic queries get polluted with episode-specific noise, and episodic queries compete against a store dominated by extracted facts rather than actual event records.

## The Storage and Query Cost Tradeoff

Maintaining two memory types and an extraction pipeline between them is genuinely more infrastructure than one undifferentiated store. It's worth the cost specifically when an agent's usefulness depends on both durable personalization *and* accurate recall of specific past interactions — a system that only ever needs "what does this user generally want" can reasonably skip episodic memory and semantic extraction entirely, keeping the simpler single-store model.

## Key Takeaways

1. **Episodic and semantic memory have different relationships to time**, and conflating them in one store loses that distinction for both retrieval and eviction
2. **Extract durable semantic facts from episodic conversation records**, keeping the episode as source-of-truth provenance
3. **Separate retrieval paths serve different query types** — most interactions need semantic facts; "what happened when" questions need episodic records
4. **This adds real infrastructure complexity** — worth it when both durable personalization and accurate event recall genuinely matter, not a default for every system

---

*Part of the [Agent Infrastructure series](/tags/agent-infra-series/) — the plumbing layer underneath production agentic systems.*
