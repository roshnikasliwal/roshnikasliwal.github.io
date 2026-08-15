---
title: "Postmortem Format for Agent Incidents That Actually Gets Read"
date: 2026-05-30
mermaid: true
categories: [AI, Agentic AI]
tags: [postmortem, incident-response, retro, agentic-ai-series]
author: Roshni Kasliwal
description: A standard postmortem template asks questions built for deterministic software failures. Agent incidents need a few additions, or the document misses exactly the parts worth learning from.
---

A standard incident postmortem template — timeline, root cause, impact, action items — transfers mostly intact to agent incidents, with a few sections that need to be added or they'll quietly get skipped, and those are usually where the most useful lessons live.

## What a Standard Template Misses

**"Why did the model do this?"** is a different question from "what code path executed," and a postmortem that only documents the latter misses the reasoning failure entirely. For an agent incident, the trace of what the model actually reasoned through — not just which function got called — belongs in the timeline.

**"Was this in the eval set, and if not, why not?"** A code incident's postmortem asks whether a test would have caught the bug. An agent incident's equivalent question is whether the eval set covered this case — and if it didn't, whether that was a genuine gap or a case nobody could reasonably have anticipated, which calls for different follow-up actions.

**"Did this look different from a traditional incident, and did that delay detection?"** Referenced in the [on-call runbook post](/posts/on-call-runbook-agent-incidents/) earlier in this series — agent incidents often don't trip standard error-rate alerts. Worth documenting explicitly whether detection was delayed for that reason, because it's an actionable finding about monitoring gaps, not just about the incident itself.

## A Template That Covers This

```markdown
## Incident Postmortem: [Title]

### Timeline
- What happened, when — including the model's reasoning trace where relevant, not just system events

### Root Cause
- What specifically caused the bad output/behavior (prompt, model version, retrieved context, tool failure)

### Detection
- How was this detected? Was it a standard alert, or did it require the pattern-based triage from the on-call runbook?
- Time from occurrence to detection, and to resolution

### Eval Coverage
- Was this case in the eval set? If not, why not — genuine gap, or reasonably unanticipated?
- Eval case(s) added as a result of this incident

### Impact
- Scope: how many requests/users affected
- Severity: was this a quality regression, a safety-relevant failure, or something else

### Action Items
- [ ] Eval case added
- [ ] Guardrail/config change made (if applicable)
- [ ] Monitoring gap addressed (if detection was delayed)
```

## The Section Most Teams Skip and Shouldn't

"Eval case(s) added as a result of this incident" is the section most likely to be left blank under time pressure, and it's the one that actually prevents recurrence. A postmortem with a thorough root-cause analysis and no corresponding eval case is a document that explains what happened without doing anything to stop it from happening again the next time a similar input arrives.

## Key Takeaways

1. **Document the model's reasoning trace, not just the system event timeline** — that's where an agent incident's actual cause usually lives
2. **Explicitly ask whether the case was in the eval set** — and whether the gap was reasonable or should have been caught
3. **Note whether standard alerting delayed detection** — agent incidents often don't trip traditional error-rate alerts
4. **Never close a postmortem without a corresponding eval case added** — that's the step that actually prevents recurrence

---

*Part of the [Agentic AI in Practice series](/tags/agentic-ai-series/) — lessons from building production multi-agent systems.*
