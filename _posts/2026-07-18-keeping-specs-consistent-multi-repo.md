---
title: "Keeping Specs Consistent Across a Multi-Repo Codebase"
date: 2026-07-18
mermaid: true
categories: [AI Engineering, Spec-Driven Development]
tags: [spec-driven-development, multi-repo, architecture, sdd-series]
author: Roshni Kasliwal
description: Colocating specs with code works cleanly in a single repo. Once a system spans several repos, the same discipline needs a cross-repo strategy or specs drift apart from each other, not just from the code.
---

The colocation approach from earlier in this series — specs live in the same repo as the code they describe — works cleanly for a single-repo system. Once a system spans multiple repos (a common outcome as services split apart), a naive extension of colocation puts each service's specs only in that service's repo, and nothing keeps specs that describe a shared contract (an API one service calls on another, a shared data format) consistent with each other across repo boundaries.

## Where Cross-Repo Drift Actually Shows Up

```mermaid
flowchart TD
    A[Service A repo] --> AS[Service A's local spec: "Service B returns field X"]
    B[Service B repo] --> BS[Service B's local spec: field renamed to Y]
    AS -.no structural link.-> BS
    Note[Service A's spec silently wrong the moment B's contract changes]
```

Colocation prevents a service's spec from drifting from its *own* code. It does nothing to prevent one service's spec from drifting from *another service's* actual behavior, because there's no structural connection between repos the way there is between a spec file and the code in the same repo.

## A Shared Contract Repo for Cross-Cutting Specs

The practical fix: specs describing a contract between services (API schemas, shared event formats, integration behavior) live in a separate, shared repository that both services' repos reference — not duplicated into each service's own spec directory, where the two copies would inevitably diverge.

```
contracts-repo/
  specs/
    service-a-to-service-b-api.md
    shared-event-schema.md

service-a-repo/
  specs/
    internal-features/...      # service A's own internal behavior
  # references contracts-repo for anything crossing the boundary

service-b-repo/
  specs/
    internal-features/...
  # references contracts-repo for anything crossing the boundary
```

## Enforcing the Reference, Not Just Recommending It

A convention to "check the contracts repo" that isn't enforced gets skipped under deadline pressure, the same way any unenforced convention does. Where feasible, pin the contracts repo as a versioned dependency (a git submodule, a package reference, whatever your tooling supports) so a service's build can actually detect when it's implementing against an outdated version of a shared contract, rather than relying on developers to remember to check.

```python
def check_contract_version_currency(local_contract_version: str, contracts_repo_latest: str) -> bool:
    if local_contract_version != contracts_repo_latest:
        logger.warning(f"Local contract pin ({local_contract_version}) is behind latest ({contracts_repo_latest})")
        return False
    return True
```

## Changing a Shared Contract Needs Cross-Repo Review

A change to a spec in the contracts repo affects every service that depends on it — the review process for a contract change needs visibility from every affected team, not just the team proposing the change. This is slower than a single-repo spec change by design; a shared contract genuinely has more stakeholders, and the review process should reflect that rather than trying to move as fast as an internal-only spec change.

## Key Takeaways

1. **Single-repo colocation doesn't prevent drift between specs describing a shared contract across repo boundaries**
2. **Put cross-cutting contract specs in a shared repo**, referenced by each service's repo, rather than duplicated into each
3. **Enforce the reference through tooling (versioned pins) where possible** — an unenforced convention gets skipped under pressure
4. **Changes to a shared contract need cross-repo review**, reflecting that it has more stakeholders than an internal-only spec change

---

*Part of the [Spec-Driven Development series](/tags/sdd-series/) — how agentic coding goes from vibe-coded prototypes to production-grade systems.*
