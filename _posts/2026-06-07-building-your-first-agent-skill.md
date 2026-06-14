---
title: "Building Your First Agent Skill: From Definition to Production"
date: 2026-06-07 09:00:00 +0000
categories: [AI Engineering, Tutorials]
tags: [agents, skills, python, tutorial, tool-use]
author: Roshni Kasliwal
description: A hands-on walkthrough of building a real agent skill in Python — covering interface design, execution, error handling, and output formatting for the model.
---

Last week I covered what agent skills *are* conceptually. Today let's build one from scratch. We'll create a GitHub issue search skill — something that pulls relevant issues from a repo and returns a model-friendly summary.

This is a deliberately practical example. It hits most of the tricky parts: an external API, structured output, error states, and rate limits.

## Setting Up the Skill Definition

Before writing a single line of execution code, I write the interface — the part the model sees. This forces me to think from the model's perspective first.

```python
SEARCH_GITHUB_ISSUES_TOOL = {
    "name": "search_github_issues",
    "description": (
        "Search for GitHub issues in a repository by keyword or topic. "
        "Use when the user asks about known bugs, feature requests, or "
        "existing discussions related to a specific topic. "
        "Returns a list of matching issues with titles, status, and summaries."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "repo": {
                "type": "string",
                "description": "Repository in 'owner/name' format, e.g. 'anthropics/anthropic-sdk-python'"
            },
            "query": {
                "type": "string",
                "description": "Search terms describing the issue topic or error"
            },
            "state": {
                "type": "string",
                "enum": ["open", "closed", "all"],
                "description": "Filter by issue state. Default: 'open'",
                "default": "open"
            }
        },
        "required": ["repo", "query"]
    }
}
```

A few things worth noting here:
- The description says *when* to use it, not just what it does
- The `repo` field has an example — this meaningfully helps the model format the input correctly
- `state` has a default so the model doesn't have to specify it every time

## The Execution Layer

Now the actual implementation. I'm wrapping the GitHub Search API:

```python
import httpx
from dataclasses import dataclass

@dataclass
class Issue:
    number: int
    title: str
    state: str
    url: str
    body_preview: str
    labels: list[str]
    created_at: str

async def search_github_issues(
    repo: str,
    query: str,
    state: str = "open",
    github_token: str | None = None,
) -> dict:
    """
    Execute the GitHub issue search skill.
    Returns a dict formatted for the model to consume.
    """
    # Input validation — don't trust the model
    if "/" not in repo:
        return {
            "error": True,
            "message": f"Invalid repo format '{repo}'. Expected 'owner/name', e.g. 'anthropics/sdk'."
        }

    headers = {"Accept": "application/vnd.github.v3+json"}
    if github_token:
        headers["Authorization"] = f"Bearer {github_token}"

    search_query = f"repo:{repo} {query}"
    if state != "all":
        search_query += f" is:{state}"

    try:
        async with httpx.AsyncClient(timeout=10.0) as client:
            response = await client.get(
                "https://api.github.com/search/issues",
                params={"q": search_query, "per_page": 5, "sort": "relevance"},
                headers=headers,
            )
            response.raise_for_status()
            data = response.json()

    except httpx.TimeoutException:
        return {"error": True, "message": "GitHub API request timed out. Try again."}
    except httpx.HTTPStatusError as e:
        if e.response.status_code == 403:
            return {"error": True, "message": "GitHub rate limit exceeded. Try again in a minute."}
        if e.response.status_code == 422:
            return {"error": True, "message": f"Invalid search query: '{query}'."}
        return {"error": True, "message": f"GitHub API error: {e.response.status_code}"}

    issues = []
    for item in data.get("items", []):
        issues.append({
            "number": item["number"],
            "title": item["title"],
            "state": item["state"],
            "url": item["html_url"],
            "preview": (item.get("body") or "")[:200].strip(),
            "labels": [l["name"] for l in item.get("labels", [])],
            "created_at": item["created_at"][:10],  # just the date
        })

    return {
        "error": False,
        "total_count": data.get("total_count", 0),
        "issues": issues,
        "repo": repo,
        "query": query,
    }
```

## Wiring It Into an Agent Loop

Here's the full agent loop that uses this skill:

```python
import anthropic
import asyncio
import json

client = anthropic.Anthropic()

async def run_agent(user_message: str):
    messages = [{"role": "user", "content": user_message}]
    tools = [SEARCH_GITHUB_ISSUES_TOOL]

    while True:
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            tools=tools,
            messages=messages,
        )

        # Add assistant response to history
        messages.append({"role": "assistant", "content": response.content})

        # If no tool calls, we're done
        if response.stop_reason == "end_turn":
            for block in response.content:
                if hasattr(block, "text"):
                    return block.text

        # Handle tool calls
        tool_results = []
        for block in response.content:
            if block.type != "tool_use":
                continue

            print(f"[skill] invoking: {block.name}({block.input})")

            # Route to the right skill
            if block.name == "search_github_issues":
                result = await search_github_issues(**block.input)
            else:
                result = {"error": True, "message": f"Unknown skill: {block.name}"}

            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": json.dumps(result),
            })

        # Feed results back
        messages.append({"role": "user", "content": tool_results})


# Try it
result = asyncio.run(
    run_agent("Are there any open issues about rate limiting in anthropics/anthropic-sdk-python?")
)
print(result)
```

## The Output Shaping Decision

Notice that I truncate the issue body to 200 characters and only return 5 results. This is intentional. The model doesn't need the full issue text — it needs enough to reason with. Returning full bodies for 30 issues would:

1. Burn context window
2. Bury the relevant signal in noise
3. Slow down the response

If the model needs more, it can ask. That's a second skill call — or a different, more targeted query.

## Testing the Skill in Isolation

I always test skills independently of the model first:

```python
import asyncio

async def test():
    # Happy path
    result = await search_github_issues(
        repo="anthropics/anthropic-sdk-python",
        query="rate limit"
    )
    assert not result["error"]
    assert "issues" in result

    # Bad input
    result = await search_github_issues(repo="not-valid", query="anything")
    assert result["error"]
    assert "owner/name" in result["message"]

    # Empty results are fine
    result = await search_github_issues(
        repo="anthropics/anthropic-sdk-python",
        query="xyzzy nonexistent gibberish"
    )
    assert not result["error"]
    assert result["total_count"] == 0

asyncio.run(test())
```

This catches most bugs before you ever involve the model, which makes debugging vastly easier.

## What I'd Add Next

This is a working v1. In production I'd add:

- **Caching** — GitHub search results for the same query are valid for a few minutes
- **Pagination skill** — a companion `get_github_issue_details` for when the model wants more than the preview
- **Auth injection** — pull the token from a secrets manager rather than passing it as a parameter

Next post: how to evaluate whether your skills are actually working — beyond "it ran without errors."
