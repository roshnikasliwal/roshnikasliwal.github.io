---
title: "Building Agent Skills as Reusable Modules"
date: 2026-04-04 09:00:00 +0530
mermaid: true
categories: [AI, Agentic AI]
tags: [agent-tools, python, langchain, crewai, architecture, reusability, agentic-ai-series]
---

The difference between a brittle agent demo and a production-grade agent system often comes down to one thing: how well you've designed your tools.

An agent's capability is defined entirely by its tool library. A poorly designed tool leads to parsing errors, ambiguous inputs, and agents making wrong decisions. A well-designed tool is self-documenting, testable in isolation, and reusable across multiple agents without modification.

After building 10+ agent skill modules that span code generation, data querying, automation, and testing — here's what the architecture looks like.

## What is an Agent Skill?

In LangChain and most agent frameworks, a "skill" is a tool: a Python function exposed to the LLM with a description that tells it what the function does, what parameters it accepts, and what it returns. The LLM reads these descriptions and decides when and how to call each tool.

The quality of your tool description is directly proportional to how well the agent uses it. A vague description produces incorrect calls. A precise one produces correct ones.

## The Anatomy of a Well-Designed Tool

```python
from langchain_core.tools import tool
from pydantic import BaseModel, Field
from typing import Optional

class SearchInput(BaseModel):
    query: str = Field(description="The search query. Be specific — include entity names, dates, or technical terms.")
    max_results: int = Field(default=5, ge=1, le=20, description="Maximum number of results to return (1-20).")
    filter_type: Optional[str] = Field(
        default=None,
        description="Optional filter: 'documentation', 'tutorial', 'release_notes'. Leave empty to search all."
    )

@tool("search_knowledge_base", args_schema=SearchInput)
def search_knowledge_base(query: str, max_results: int = 5, filter_type: Optional[str] = None) -> str:
    """
    Search the internal knowledge base for relevant documents.

    Use this tool when you need to find information about product features,
    troubleshooting steps, API documentation, or release notes.

    Returns a list of relevant document excerpts with their source titles.
    If no relevant results are found, returns an empty result set.
    Do NOT use this tool for general internet searches — it only searches internal content.
    """
    try:
        results = vector_store.similarity_search(
            query=query,
            k=max_results,
            filter={"type": filter_type} if filter_type else None
        )
        
        if not results:
            return f"No results found for query: '{query}'"
        
        formatted = []
        for i, doc in enumerate(results, 1):
            formatted.append(
                f"[{i}] {doc.metadata.get('title', 'Untitled')}\n"
                f"Source: {doc.metadata.get('source', 'Unknown')}\n"
                f"Content: {doc.page_content[:500]}..."
            )
        
        return f"Found {len(results)} results:\n\n" + "\n\n".join(formatted)
    
    except Exception as e:
        return f"ERROR: Search failed: {str(e)}"
```

Notice what makes this well-designed:
1. **Pydantic schema** — typed, validated inputs with field-level descriptions the LLM reads
2. **Negative instructions** — "Do NOT use this tool for..." prevents misuse
3. **Return errors as strings** — the agent can reason about failures, not crash
4. **Capped results** — prevents the agent from accidentally requesting 1000 results

## The Skills Registry Pattern

When you have 10+ tools, hardcoding them into every agent becomes unmanageable. A registry lets you compose agent capabilities from a central catalog:

```python
from dataclasses import dataclass, field
from typing import Callable, List, Optional
from langchain_core.tools import BaseTool

@dataclass
class SkillMetadata:
    name: str
    category: str
    description: str
    requires_auth: bool = False
    rate_limited: bool = False

class SkillRegistry:
    _skills: dict[str, tuple[BaseTool, SkillMetadata]] = {}

    @classmethod
    def register(cls, metadata: SkillMetadata):
        """Decorator to register a tool with metadata."""
        def decorator(tool_func: BaseTool):
            cls._skills[metadata.name] = (tool_func, metadata)
            return tool_func
        return decorator

    @classmethod
    def get_skills(cls, categories: List[str] = None, exclude_auth: bool = False) -> List[BaseTool]:
        """Retrieve tools filtered by category or auth requirement."""
        tools = []
        for name, (tool, meta) in cls._skills.items():
            if categories and meta.category not in categories:
                continue
            if exclude_auth and meta.requires_auth:
                continue
            tools.append(tool)
        return tools

    @classmethod
    def list_all(cls) -> dict:
        return {name: meta for name, (_, meta) in cls._skills.items()}


# Register skills with metadata
@SkillRegistry.register(SkillMetadata(
    name="search_knowledge_base",
    category="search",
    description="Search internal documentation",
))
@tool("search_knowledge_base", args_schema=SearchInput)
def search_knowledge_base(query: str, max_results: int = 5, filter_type: Optional[str] = None) -> str:
    """Search the internal knowledge base for relevant documents."""
    ...

@SkillRegistry.register(SkillMetadata(
    name="create_jira_ticket",
    category="project_management",
    description="Create a Jira issue",
    requires_auth=True,
))
@tool
def create_jira_ticket(summary: str, description: str, priority: str = "Medium") -> str:
    """Create a new Jira issue in the current project."""
    ...

@SkillRegistry.register(SkillMetadata(
    name="run_unit_tests",
    category="development",
    description="Execute the test suite and return results",
))
@tool
def run_unit_tests(test_path: str = "tests/") -> str:
    """Run the unit test suite using pytest and return a summary of results."""
    ...


# Compose agent capabilities from the registry
support_agent_tools = SkillRegistry.get_skills(categories=["search"])
dev_agent_tools = SkillRegistry.get_skills(categories=["development", "search"])
pm_agent_tools = SkillRegistry.get_skills(categories=["project_management", "search"], exclude_auth=False)
```

This pattern means each new agent gets exactly the skills it needs with one line of code.

## The Single Responsibility Principle for Tools

Each tool should do one thing and do it well. The temptation is to build "smart" tools that handle multiple cases:

```python
# Bad: does too much, hard to test, ambiguous for the agent
@tool
def handle_data(operation: str, data: str, target: str = None) -> str:
    """Handle data operations: search, update, delete, or export."""
    if operation == "search":
        ...
    elif operation == "update":
        ...
    elif operation == "delete":
        ...
```

The agent has to guess what `operation` to pass, what `data` means in each context, and what `target` does. This leads to incorrect calls.

```python
# Good: three separate tools, each with a clear purpose
@tool
def search_records(query: str, limit: int = 10) -> str:
    """Search records by keyword. Returns matching record IDs and summaries."""
    ...

@tool
def update_record(record_id: str, field: str, new_value: str) -> str:
    """Update a single field in a record. Returns confirmation or error."""
    ...

@tool
def export_records(record_ids: list[str], format: str = "csv") -> str:
    """Export specified records to CSV or JSON. Returns download URL."""
    ...
```

## Testing Skills Independently

Tools must be testable without running the full agent. This is where most teams cut corners — and then can't debug why the agent is behaving unexpectedly.

```python
import pytest
from unittest.mock import patch, MagicMock

class TestSearchKnowledgeBase:
    def test_returns_formatted_results(self, mock_vector_store):
        mock_vector_store.similarity_search.return_value = [
            MagicMock(
                page_content="FastAPI authentication guide content...",
                metadata={"title": "Auth Guide", "source": "docs/auth.md"}
            )
        ]
        result = search_knowledge_base.invoke({
            "query": "how to implement JWT authentication",
            "max_results": 3
        })
        assert "[1]" in result
        assert "Auth Guide" in result
        assert "ERROR" not in result

    def test_empty_results_returns_descriptive_message(self, mock_vector_store):
        mock_vector_store.similarity_search.return_value = []
        result = search_knowledge_base.invoke({"query": "nonexistent topic xyz"})
        assert "No results found" in result
        assert "nonexistent topic xyz" in result

    def test_exception_returns_error_string_not_exception(self, mock_vector_store):
        mock_vector_store.similarity_search.side_effect = ConnectionError("DB unavailable")
        result = search_knowledge_base.invoke({"query": "anything"})
        assert result.startswith("ERROR:")
        # Critical: the agent must receive a string, never an unhandled exception

    def test_max_results_capped_at_twenty(self, mock_vector_store):
        # Even if someone calls with max_results=100, Pydantic rejects it
        with pytest.raises(ValidationError):
            search_knowledge_base.invoke({"query": "test", "max_results": 100})
```

## Adding Rate Limiting and Caching

Production tools need guardrails beyond just input validation:

```python
import functools
import time
from collections import defaultdict

def rate_limited(max_calls: int, period: float):
    """Decorator to rate-limit tool calls."""
    calls = defaultdict(list)

    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            now = time.time()
            # Purge old calls outside the window
            calls[func.__name__] = [t for t in calls[func.__name__] if now - t < period]
            
            if len(calls[func.__name__]) >= max_calls:
                wait = period - (now - calls[func.__name__][0])
                return f"ERROR: Rate limit reached. Retry after {wait:.1f} seconds."
            
            calls[func.__name__].append(now)
            return func(*args, **kwargs)
        return wrapper
    return decorator

def cached(ttl: int = 300):
    """Simple TTL cache for deterministic tool results."""
    cache = {}
    
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            key = str(args) + str(sorted(kwargs.items()))
            if key in cache:
                result, ts = cache[key]
                if time.time() - ts < ttl:
                    return result
            result = func(*args, **kwargs)
            cache[key] = (result, time.time())
            return result
        return wrapper
    return decorator

# Apply to tools
@tool
@rate_limited(max_calls=10, period=60.0)
@cached(ttl=300)
def fetch_external_data(resource_id: str) -> str:
    """Fetch data from the external API. Results cached for 5 minutes."""
    ...
```

## The Complete Skill Module Structure

For a production skill library, organize your code this way:

```
agent_skills/
├── __init__.py
├── registry.py          # SkillRegistry class
├── search/
│   ├── __init__.py
│   ├── knowledge_base.py
│   └── web_search.py
├── development/
│   ├── __init__.py
│   ├── code_runner.py
│   ├── test_runner.py
│   └── git_operations.py
├── project_management/
│   ├── __init__.py
│   ├── jira.py
│   └── confluence.py
└── tests/
    ├── test_search.py
    ├── test_development.py
    └── test_project_management.py
```

Each skill module is independently deployable and testable. When you need a new agent, you compose its capability from the registry:

```python
from agent_skills.registry import SkillRegistry

# Support agent: only needs search and documentation
support_agent = Agent(
    role="Support Engineer",
    tools=SkillRegistry.get_skills(categories=["search"]),
    ...
)

# Developer agent: needs dev tools + search
developer_agent = Agent(
    role="Senior Developer",
    tools=SkillRegistry.get_skills(categories=["development", "search"]),
    ...
)
```

## Key Takeaways

1. **Tool descriptions are prompts** — write them as carefully as you write system prompts
2. **Return errors as strings** — agents must be able to reason about failures
3. **One tool = one responsibility** — avoid multi-mode tools that the LLM has to "configure"
4. **Test tools independently** — isolate them from the agent loop in unit tests
5. **Use a registry** — composing agent capabilities from a catalog scales to 10+ tools cleanly
6. **Add rate limits and caching** — production tools need guardrails beyond just input validation

---

*Part of the [Agentic AI in Practice series](/tags/agentic-ai-series/) — lessons from building production multi-agent systems.*
