---
title: "Agent Skill Design Patterns: What Works in Production"
date: 2026-06-14 09:00:00 +0000
categories: [AI Engineering, Architecture]
tags: [agents, skills, design-patterns, architecture, production]
author: Roshni Kasliwal
description: After building dozens of agent skills across different domains, some patterns consistently work and some consistently don't. Here's what I've learned.
---

I've built a lot of agent skills at this point — across customer support, internal tooling, developer assistants, and data analysis workflows. Some patterns keep coming up in the ones that work well. Others are traps I've fallen into more than once.

This is my current thinking on skill design, organized as patterns. None of these are universal laws — context matters — but they've been reliable enough that they're my starting assumptions.

## Pattern 1: One Description Line, One Responsibility

The hardest part of skill design isn't the code. It's writing a description you can summarize in a single sentence that precisely captures *when* to call this skill.

If you can't write that sentence, the skill is probably trying to do too many things.

```python
# ❌ Too broad — the model can't reliably know when to use this
{
    "name": "manage_user",
    "description": "Manage user data including lookups, updates, and preferences."
}

# ✅ Clear boundary — the model knows exactly when to reach for this
{
    "name": "get_user_by_email",
    "description": "Look up a user account by email address. Use when you need "
                   "to verify a user exists or retrieve their account ID before "
                   "taking action on their account."
}
```

The "manage_user" skill forces the model to figure out *how* to use it based on context. The `get_user_by_email` skill handles exactly one operation, and the description tells the model the precise situation where it applies.

## Pattern 2: Design the Output for the Next Reasoning Step

Skills aren't just about getting data. They're about setting up the model for its *next* thought.

Ask yourself: given this skill's output, what will the model do next? Then design the output to make that next step easy.

```python
# ❌ Raw API response — makes the model work too hard
{
    "id": "usr_8f7d3a",
    "email": "alice@example.com",
    "created_at": "2024-01-15T08:32:11.000Z",
    "updated_at": "2026-05-30T14:21:07.000Z",
    "subscription": {"plan": "pro", "status": "active", "renewal": "2026-12-15"},
    "metadata": {"source": "organic", "referrer": null, "campaign_id": null},
    # ... 30 more fields
}

# ✅ Shaped for reasoning — model gets what it needs to decide next
{
    "found": True,
    "user_id": "usr_8f7d3a",
    "email": "alice@example.com",
    "account_status": "active",        # Clear, normalized
    "plan": "pro",                     # Extracted from nested object
    "plan_renewal_date": "2026-12-15", # Formatted, not ISO timestamp
}
```

The second version eliminates the work of parsing nested objects and normalizing timestamps. The model reasons better when it's doing *less* data wrangling and *more* task reasoning.

## Pattern 3: Fail Loudly with Actionable Errors

When a skill fails, the message the model gets back determines whether it recovers gracefully or loops forever. Most production incidents I've debugged come down to the model receiving an unhelpful error and not knowing what to do next.

The pattern is: **what happened + what the model can do about it**.

```python
def handle_skill_error(exception: Exception, context: dict) -> dict:
    if isinstance(exception, RateLimitError):
        return {
            "error": True,
            "code": "rate_limited",
            "message": "API rate limit reached. Wait 60 seconds before retrying, "
                       "or reduce the number of results requested.",
            "retry_after_seconds": 60,
        }
    if isinstance(exception, NotFoundError):
        return {
            "error": True,
            "code": "not_found",
            "message": f"No record found for {context.get('id')}. "
                       "Verify the ID is correct or search by email instead.",
        }
    # Generic fallback
    return {
        "error": True,
        "code": "unexpected_error",
        "message": "An unexpected error occurred. If this persists, escalate to support.",
    }
```

Notice the rate limit error includes `retry_after_seconds`. This gives the model concrete information to include in its response to the user instead of vague "try again later" language.

## Pattern 4: The Skill Pair

Some operations naturally belong together. When I find myself building a "search" skill, I almost always end up needing a "get details" companion.

| Search Skill | Detail Skill |
|---|---|
| `search_knowledge_base` | `get_article_by_id` |
| `find_customer` | `get_customer_details` |
| `list_open_tickets` | `get_ticket_thread` |

The search skill returns brief, scannable results — enough for the model to identify what it needs. The detail skill fetches the full content for the item the model decided it wants.

This keeps your search results compact (no context window waste) while giving the model a clean path to go deeper when needed.

## Pattern 5: Idempotent by Default

Write-skills (anything that creates or mutates state) should be idempotent wherever possible. If the same skill call is made twice — because of a network retry, a model loop, or a user refreshing the page — it should produce the same end state without double-posting, double-charging, or double-booking.

```python
async def create_or_get_support_ticket(
    customer_id: str,
    subject: str,
    deduplication_key: str,  # caller provides this
) -> dict:
    """
    Creates a ticket, or returns the existing one if the key was already used.
    Safe to call multiple times with the same deduplication_key.
    """
    existing = await db.get_ticket_by_dedup_key(deduplication_key)
    if existing:
        return {
            "created": False,
            "ticket_id": existing.id,
            "message": "Ticket already exists for this request."
        }

    ticket = await db.create_ticket(customer_id, subject, deduplication_key)
    return {
        "created": True,
        "ticket_id": ticket.id,
        "message": f"Support ticket #{ticket.id} created successfully."
    }
```

The deduplication key can be derived from the conversation context — a hash of the user ID, the session, and the action intent. This makes the write-skill safe to retry without side effects.

## Pattern 6: Skill Scoping via System Prompt

Not every skill should be available in every context. A customer support agent doesn't need access to `deploy_to_production`. An internal search tool doesn't need `send_customer_email`.

I build skills as a full library and then scope them per agent configuration:

```python
SKILL_REGISTRY = {
    "search_knowledge_base": search_knowledge_base_tool,
    "get_article_by_id": get_article_by_id_tool,
    "create_support_ticket": create_support_ticket_tool,
    "get_customer_by_email": get_customer_by_email_tool,
    "escalate_to_human": escalate_to_human_tool,
    # ... more skills
}

AGENT_CONFIGS = {
    "customer_support": [
        "search_knowledge_base",
        "get_article_by_id",
        "create_support_ticket",
        "get_customer_by_email",
        "escalate_to_human",
    ],
    "internal_ops": [
        "search_knowledge_base",
        "get_article_by_id",
    ]
}

def get_tools_for_agent(agent_type: str) -> list:
    skill_names = AGENT_CONFIGS.get(agent_type, [])
    return [SKILL_REGISTRY[name] for name in skill_names]
```

This keeps the model's tool list short and relevant, which reduces both token cost and the chance of the model reaching for the wrong skill.

## The One Anti-Pattern I See Most

**Returning too much data from skills.**

I see this constantly. Engineers build a skill that mirrors an API endpoint — and API endpoints return everything, because they have to serve many different clients. But your agent only needs what it needs for *this* task.

When a skill returns 50 fields, the model has to read and process all 50. That's wasted context. Worse, it sometimes latches onto a field you didn't expect and reasons about it in ways you didn't intend.

Shape your output. Extract what matters. Format it for reasoning, not for completeness.

---

The next post in this series will be on **skill evaluation** — how to know, with actual numbers, whether your skills are performing well enough to trust in production.
