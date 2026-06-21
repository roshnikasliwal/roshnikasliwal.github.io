---
title: "Agent Evaluation: How Do You Know Your Agent is Working?"
date: 2026-04-07 09:00:00 +0530
categories: [AI, Agentic AI]
tags: [evaluation, ragas, llmops, testing, python, deepeval, agentic-ai-series]
---

Shipping an agent without evaluation is like deploying code without tests — it works until it doesn't, and you won't know why it stopped working.

Agent evaluation is harder than LLM evaluation. A chatbot has one input and one output. An agent has a trajectory: a sequence of decisions, tool calls, and intermediate states before reaching a final answer. Each step can fail independently.

After building LLMOps practices around agent systems, here's the evaluation framework that actually works in production.

## Why Agent Evaluation is Different

Three properties make agents uniquely challenging to evaluate:

1. **Non-determinism** — the same input can produce different tool call sequences. This makes it hard to write deterministic assertions.
2. **Multi-step trajectories** — the final output might be correct even if intermediate steps were wrong, or incorrect despite correct intermediate steps.
3. **Emergent failures** — an agent can fail in ways that weren't apparent during development: using a tool unnecessarily, getting stuck in loops, or misinterpreting tool outputs.

The solution is evaluating at three levels: tools, trajectories, and outcomes.

## Level 1: Tool Evaluation

The simplest and most reliable layer. Tools are pure functions — test them like unit tests.

```python
import pytest
from unittest.mock import patch, MagicMock

class TestSearchTool:
    def test_returns_formatted_results_on_match(self, mock_vector_store):
        mock_vector_store.similarity_search.return_value = [
            MagicMock(
                page_content="JWT authentication involves signing tokens...",
                metadata={"title": "Auth Guide", "source": "docs/auth.md", "score": 0.91}
            )
        ]
        result = search_knowledge_base.invoke({
            "query": "JWT authentication implementation",
            "max_results": 3
        })
        assert "[1]" in result
        assert "Auth Guide" in result
        assert "ERROR" not in result

    def test_empty_results_returns_descriptive_string(self, mock_vector_store):
        mock_vector_store.similarity_search.return_value = []
        result = search_knowledge_base.invoke({"query": "xyzzy nonexistent topic"})
        assert "No results found" in result

    def test_exception_returns_error_string_not_exception(self, mock_vector_store):
        mock_vector_store.similarity_search.side_effect = ConnectionError("timeout")
        result = search_knowledge_base.invoke({"query": "anything"})
        assert result.startswith("ERROR:")  # Must never raise

    def test_max_results_validated_by_pydantic(self):
        with pytest.raises(Exception):  # Pydantic ValidationError
            search_knowledge_base.invoke({"query": "test", "max_results": 200})
```

Run these in CI on every commit. If a tool is broken, the agent cannot work correctly.

## Level 2: Trajectory Evaluation

Trajectory evaluation checks that the agent took the right sequence of actions, not just that the final answer was correct.

### Building a Trajectory Dataset

```python
from dataclasses import dataclass

@dataclass
class TrajectoryTestCase:
    input: str
    expected_tools: list[str]          # Tools that MUST be called
    forbidden_tools: list[str]         # Tools that must NOT be called
    min_steps: int                     # Minimum number of tool calls
    max_steps: int                     # Maximum (prevents loops)
    expected_outcome: str              # "success" | "graceful_failure"

TRAJECTORY_TEST_CASES = [
    TrajectoryTestCase(
        input="What is the authentication timeout value in our current configuration?",
        expected_tools=["search_knowledge_base"],
        forbidden_tools=["send_email", "delete_record"],  # Must not take action
        min_steps=1,
        max_steps=3,
        expected_outcome="success"
    ),
    TrajectoryTestCase(
        input="Search for Q3 results, analyze them, and email the summary to the team",
        expected_tools=["search_knowledge_base", "analyze_data", "send_email"],
        forbidden_tools=["delete_record"],
        min_steps=3,
        max_steps=8,
        expected_outcome="success"
    ),
    TrajectoryTestCase(
        input="Delete all records from the database",  # Should refuse
        expected_tools=[],
        forbidden_tools=["delete_record", "run_sql"],
        min_steps=0,
        max_steps=1,
        expected_outcome="graceful_failure"
    ),
]
```

### Running Trajectory Tests

```python
from langchain.agents import AgentExecutor
import json

def run_trajectory_test(
    executor: AgentExecutor,
    test_case: TrajectoryTestCase
) -> dict:
    result = executor.invoke(
        {"input": test_case.input},
        return_intermediate_steps=True
    )
    
    steps = result.get("intermediate_steps", [])
    tools_called = [step[0].tool for step in steps]
    
    issues = []
    
    # Check required tools were called
    for required in test_case.expected_tools:
        if required not in tools_called:
            issues.append(f"Required tool '{required}' was not called")
    
    # Check forbidden tools were not called
    for forbidden in test_case.forbidden_tools:
        if forbidden in tools_called:
            issues.append(f"Forbidden tool '{forbidden}' was called")
    
    # Check step count
    if len(steps) < test_case.min_steps:
        issues.append(f"Too few steps: {len(steps)} (min: {test_case.min_steps})")
    if len(steps) > test_case.max_steps:
        issues.append(f"Too many steps: {len(steps)} (max: {test_case.max_steps})")
    
    return {
        "input": test_case.input,
        "passed": len(issues) == 0,
        "issues": issues,
        "tools_called": tools_called,
        "steps": len(steps),
        "output": result.get("output", "")[:200],
    }

# Run all trajectory tests
def run_trajectory_suite(executor: AgentExecutor, cases: list[TrajectoryTestCase]) -> dict:
    results = [run_trajectory_test(executor, case) for case in cases]
    passed = sum(1 for r in results if r["passed"])
    
    return {
        "total": len(cases),
        "passed": passed,
        "failed": len(cases) - passed,
        "pass_rate": round(passed / len(cases), 3),
        "failures": [r for r in results if not r["passed"]]
    }
```

## Level 3: Output Quality with RAGAS

For agents that produce answers from a knowledge base (RAG agents), RAGAS provides quantitative metrics.

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_recall,
    context_precision,
    answer_correctness,
)
from datasets import Dataset

# Build evaluation dataset
eval_data = {
    "question": [],
    "answer": [],
    "contexts": [],
    "ground_truth": [],
}

# Golden dataset: questions with known correct answers and relevant sources
GOLDEN_QA = [
    {
        "question": "What is the default timeout for API authentication tokens?",
        "ground_truth": "The default authentication token timeout is 3600 seconds (1 hour).",
        "relevant_docs": ["docs/auth/token-config.md"]
    },
    {
        "question": "How do I enable rate limiting on the REST API?",
        "ground_truth": "Rate limiting is enabled via the RATE_LIMIT_ENABLED=true environment variable with configurable limits per endpoint.",
        "relevant_docs": ["docs/api/rate-limiting.md"]
    },
]

# Run agent on each question and collect outputs
for qa in GOLDEN_QA:
    result = executor.invoke(
        {"input": qa["question"]},
        return_intermediate_steps=True
    )
    
    # Extract retrieved contexts from tool outputs
    contexts = []
    for step in result["intermediate_steps"]:
        if step[0].tool == "search_knowledge_base":
            contexts.append(step[1])  # Tool output is the context
    
    eval_data["question"].append(qa["question"])
    eval_data["answer"].append(result["output"])
    eval_data["contexts"].append(contexts if contexts else ["No context retrieved"])
    eval_data["ground_truth"].append(qa["ground_truth"])

dataset = Dataset.from_dict(eval_data)

# Evaluate
scores = evaluate(
    dataset=dataset,
    metrics=[faithfulness, answer_relevancy, context_precision, context_recall, answer_correctness],
)

print(scores.to_pandas()[["faithfulness", "answer_relevancy", "context_precision", "context_recall"]].describe())
```

### Understanding the Metrics

| Metric | What it measures | Target |
|---|---|---|
| **Faithfulness** | Is the answer grounded in retrieved context? (no hallucination) | > 0.85 |
| **Answer Relevancy** | Does the answer address the question? | > 0.80 |
| **Context Precision** | Are retrieved chunks relevant to the question? | > 0.75 |
| **Context Recall** | Were all necessary documents retrieved? | > 0.70 |
| **Answer Correctness** | Is the answer factually correct vs. ground truth? | > 0.75 |

Faithfulness is the most critical metric. A score below 0.7 means your agent is regularly making up information not present in the retrieved context — unacceptable for production.

## Level 4: LLM-as-Judge

For qualitative properties that don't fit quantitative metrics (tone, completeness, helpfulness), use an LLM to judge outputs:

```python
from langchain_anthropic import ChatAnthropic
from pydantic import BaseModel
from typing import Literal

judge_llm = ChatAnthropic(model="claude-opus-4-8", temperature=0)

class EvalResult(BaseModel):
    score: int  # 1-5
    reasoning: str
    verdict: Literal["pass", "fail"]
    improvements: list[str]

def evaluate_response_quality(question: str, answer: str, criteria: str) -> EvalResult:
    """Use a stronger LLM to evaluate agent response quality."""
    prompt = f"""
You are evaluating an AI agent's response quality. Score the response on the following criteria:

CRITERIA: {criteria}

QUESTION: {question}

AGENT RESPONSE: {answer}

Scoring rubric:
5 = Excellent: fully meets criteria, clear, accurate, complete
4 = Good: meets criteria with minor gaps
3 = Acceptable: partially meets criteria, notable gaps
2 = Poor: mostly fails criteria, significant issues
1 = Unacceptable: fails criteria entirely

Respond with a JSON object: {{"score": int, "reasoning": "...", "verdict": "pass|fail", "improvements": ["..."]}}
Verdict is "pass" for score >= 3.
"""
    response = judge_llm.with_structured_output(EvalResult).invoke(prompt)
    return response

# Run LLM judge on sample outputs
criteria_set = [
    "The response should be concise (under 200 words) and directly answer the question without unnecessary preamble.",
    "The response should cite its sources when making factual claims.",
    "The response should acknowledge when information is uncertain rather than stating it as fact.",
]

def run_llm_judge_suite(test_cases: list[dict]) -> dict:
    results = []
    for case in test_cases:
        for criteria in criteria_set:
            result = evaluate_response_quality(
                question=case["question"],
                answer=case["agent_answer"],
                criteria=criteria
            )
            results.append({
                "question": case["question"][:80],
                "criteria": criteria[:60],
                "score": result.score,
                "verdict": result.verdict,
            })
    
    passed = sum(1 for r in results if r["verdict"] == "pass")
    return {
        "pass_rate": round(passed / len(results), 3),
        "avg_score": round(sum(r["score"] for r in results) / len(results), 2),
        "failures": [r for r in results if r["verdict"] == "fail"]
    }
```

## Building a CI Evaluation Pipeline

```yaml
# .github/workflows/agent-eval.yml
name: Agent Evaluation

on:
  push:
    branches: [main]
    paths: ['src/agents/**', 'src/tools/**']

jobs:
  evaluate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: pip install -r requirements.txt
      
      - name: Run tool unit tests
        run: pytest tests/tools/ -v --tb=short
      
      - name: Run trajectory tests
        run: python eval/run_trajectory_tests.py
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      
      - name: Run RAGAS evaluation
        run: python eval/run_ragas_eval.py --min-faithfulness 0.80
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
      
      - name: Upload eval report
        uses: actions/upload-artifact@v4
        with:
          name: eval-report
          path: eval/reports/
```

### Quality Gates

Set thresholds that fail the pipeline:

```python
# eval/run_ragas_eval.py
import argparse
import sys

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--min-faithfulness", type=float, default=0.80)
    parser.add_argument("--min-relevancy", type=float, default=0.75)
    args = parser.parse_args()
    
    scores = run_ragas_evaluation()
    
    failed_gates = []
    if scores["faithfulness"] < args.min_faithfulness:
        failed_gates.append(f"Faithfulness {scores['faithfulness']:.3f} < {args.min_faithfulness}")
    if scores["answer_relevancy"] < args.min_relevancy:
        failed_gates.append(f"Answer relevancy {scores['answer_relevancy']:.3f} < {args.min_relevancy}")
    
    if failed_gates:
        print("QUALITY GATES FAILED:")
        for gate in failed_gates:
            print(f"  ✗ {gate}")
        sys.exit(1)
    
    print("All quality gates passed.")
    sys.exit(0)
```

## Tracking Metrics Over Time

Evaluation is not a one-time check — it's a continuous signal. Track these metrics per model version:

```python
import json
from datetime import datetime

def log_eval_run(metrics: dict, version: str):
    entry = {
        "timestamp": datetime.utcnow().isoformat(),
        "version": version,
        "metrics": metrics
    }
    
    with open("eval/history.jsonl", "a") as f:
        f.write(json.dumps(entry) + "\n")

# Detect regressions
def check_for_regression(current: dict, baseline: dict, threshold: float = 0.05) -> list[str]:
    regressions = []
    for metric, value in current.items():
        baseline_value = baseline.get(metric, 0)
        if value < baseline_value - threshold:
            regressions.append(
                f"{metric}: {value:.3f} vs baseline {baseline_value:.3f} "
                f"({(value - baseline_value)*100:+.1f}%)"
            )
    return regressions
```

## Key Takeaways

1. **Evaluate at three levels**: tools (unit tests), trajectories (did it take the right steps?), and outcomes (was the answer correct and high quality?)
2. **Faithfulness is your north star** — an agent that hallucinates is worse than no agent
3. **Build a golden dataset from day one** — curate 50-100 question/answer pairs that represent your real use cases
4. **Run evals in CI** — treat quality regression the same as a test failure
5. **LLM-as-judge scales where metrics don't** — use a stronger model to evaluate qualitative properties
6. **Track metrics over time** — regressions are easy to miss without historical comparison

---

*Part of the [Agentic AI in Practice series](/tags/agentic-ai-series/) — lessons from building production multi-agent systems.*
