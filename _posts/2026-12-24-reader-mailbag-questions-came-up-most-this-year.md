---
title: "Reader Mailbag: The Questions That Came Up Most This Year"
date: 2026-12-24
mermaid: true
categories: [AI Engineering, Road to 2027]
tags: [reader-questions, retro, road-to-2027-series]
author: Roshni Kasliwal
description: "A holiday-week roundup of the questions that recurred most across a year of posts and comments — answered directly, with pointers back to where each was covered in depth."
---

A quieter post for a quiet week: a roundup of the questions that came up most persistently across this year of daily writing, answered directly and pointed back to where each was covered in fuller depth for readers who want to go deeper.

## "How do you actually decide narrow-vertical-agent versus general-purpose assistant?"

The single most recurring question, spanning from April's architecture posts through October's Agent Economy series. The short answer: start with a bounded, measurable business process (procurement, ticket triage — October's specific examples), prove it, expand deliberately. The long answer is the business case post from October and everything in that series' first week.

## "Is spec-driven development overkill for a small team?"

Asked constantly after July's SDD series. The honest answer, from the [SDD vs TDD comparison post](/posts/spec-driven-development-vs-test-driven-development/): match formality to stakes. A small team doesn't need full EARS-notation rigor on every change — the lighter-weight templates from that series' resource post capture most of the benefit at a fraction of the ceremony.

## "What's the single highest-leverage guardrail to implement first?"

From this year's guardrails series: there isn't one, and that's the actual answer — the [layered defense post](/posts/layered-defense-one-guardrail-never-enough/) argued this directly. If forced to rank, output-side action-consistency checking (does the action match the stated request) tends to catch the widest range of failure modes for the least implementation cost, but it's a floor, not a complete answer.

## "Does this apply to us if we're not EU-based?"

The most common question after November's EU AI Act coverage. Per the U.S. state law convergence post: the substantive architecture (risk-tiered oversight, data minimization, audit trails) transfers regardless of jurisdiction, even though specific documentation and enforcement differ. Build the substance; adapt the paperwork per jurisdiction.

## "Small model or frontier model for [specific task]?"

Recurring throughout the year, most directly addressed this month. The 2.6B-vs-671B post and the sizing framework that followed it: narrow and well-defined favors small; broad, novel, or genuinely open-ended favors frontier. Measure on your own eval set rather than trusting either intuition.

## What These Questions Have in Common

```mermaid
flowchart TD
    A[Recurring questions] --> B[All ask "which extreme should I pick"]
    B --> C[Nearly every honest answer this year was "it depends, here's the deciding factor" — not a universal rule]
```

Looking back across a full year of these questions, the pattern is that readers consistently ask for a universal rule, and the honest answer — consistent with this blog's throughline argued in the November retrospective — was almost always "it depends on stakes/scope/task-narrowness, here's the specific factor that decides it for your case." That's not evasiveness; it's the actual, considered position this blog has tried to argue for consistently rather than offering false universal certainty.

## Key Takeaways

1. **Vertical-vs-general, spec formality, guardrail prioritization, jurisdiction applicability, and model sizing were this year's most recurring questions**
2. **Every one has a pointer back to a specific earlier post with the fuller reasoning**, not just a restated summary
3. **The consistent pattern across answers: match the decision to actual stakes and scope**, not a universal rule — this was a deliberate, considered position, not an evasive non-answer
4. **This mailbag post is itself a small demonstration of this blog's cross-referencing structure** — a year of interconnected posts readers can navigate by the specific question they actually have

---

*Part of the [Road to 2027 series](/tags/road-to-2027-series/) — edge agents, coding agent maturity, orchestration, and where agentic AI stands as the year closes.*
