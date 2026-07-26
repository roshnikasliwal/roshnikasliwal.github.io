---
title: "Wiring Up Ragas: A Hands-On Guide to RAG Evaluation"
date: 2026-06-24 09:00:00 +0000
mermaid: true
categories: [AI Engineering, Evaluation]
tags: [rag, ragas, evaluation, python, testing]
author: Roshni Kasliwal
description: The theory of RAG evaluation is one thing. Actually wiring Ragas into your own pipeline — judge model config, dataset construction, debugging a low score — is where most of the friction lives.
---

I've written about the theory of RAG evaluation before — [what to measure and why](/posts/rag-evaluation-ragas-faithfulness/) — but theory doesn't tell you what to do when `pip install ragas` works fine and your first real run comes back with a `context_precision` of 0.31 and no obvious explanation. This post is the setup-and-debug walkthrough I wish I'd had the first time.

## Ragas Defaults to OpenAI — Point It at Your Actual Model

The first friction point: Ragas' default judge model is OpenAI, and if your production stack runs on Claude, your evaluation judge should match — or at least be deliberately chosen, not left as an accident of the library default:

```python
from ragas.llms import LangchainLLMWrapper
from ragas.embeddings import LangchainEmbeddingsWrapper
from langchain_anthropic import ChatAnthropic
from langchain_openai import OpenAIEmbeddings

judge_llm = LangchainLLMWrapper(ChatAnthropic(model="claude-sonnet-4-6", temperature=0))
judge_embeddings = LangchainEmbeddingsWrapper(OpenAIEmbeddings(model="text-embedding-3-large"))

from ragas import evaluate
from ragas.metrics import faithfulness, context_precision, context_recall, answer_relevancy

results = evaluate(
    eval_dataset,
    metrics=[faithfulness, context_precision, context_recall, answer_relevancy],
    llm=judge_llm,
    embeddings=judge_embeddings,
)
```

Set `temperature=0` on the judge model specifically. A judge that isn't deterministic makes your regression suite flaky in a way that's genuinely hard to distinguish from a real regression — I lost half a day to exactly that before pinning it down.

## Building the Dataset from Real Traces, Not Hand-Written Examples

The fastest way to get a misleading eval is to hand-write the dataset yourself — you'll only think of the question phrasings you'd naturally use, which is a biased sample of what real users type. Build it from production traces instead:

```python
def build_eval_dataset_from_traces(trace_logs: list[dict]) -> Dataset:
    """trace_logs come from your existing observability pipeline —
    each entry already has the question, retrieved chunks, and generated answer."""
    return Dataset.from_dict({
        "question": [t["query"] for t in trace_logs],
        "contexts": [[c["text"] for c in t["retrieved_chunks"]] for t in trace_logs],
        "answer": [t["generated_answer"] for t in trace_logs],
        "ground_truth": [t.get("human_verified_answer", "") for t in trace_logs],
    })

# Sample across query categories, not just the most recent N —
# recency-biased sampling misses whatever category broke three weeks ago
recent_sample = sample_stratified(trace_logs, by="query_category", n_per_category=20)
eval_dataset = build_eval_dataset_from_traces(recent_sample)
```

`ground_truth` is the expensive part — someone has to verify the correct answer for each sampled trace. I treat this as an ongoing task, not a one-time setup cost: every week, a small batch of traces gets human-verified and added to the golden set, so the dataset grows and stays representative of what's actually being asked.

## Debugging a Low Score

A `context_precision` of 0.31 means most of what got retrieved wasn't relevant to the question — but that number alone doesn't tell you *which* questions or *why*. Pull the per-row breakdown, not just the aggregate:

```python
results_df = results.to_pandas()
worst_rows = results_df.sort_values("context_precision").head(10)

for _, row in worst_rows.iterrows():
    print(f"Q: {row['question']}")
    print(f"Score: {row['context_precision']:.2f}")
    print(f"Retrieved: {row['contexts'][:2]}")  # inspect what actually came back
    print("---")
```

In practice this almost always resolves to one of three things: a chunking boundary that split the relevant fact in two (fix: adjust chunk size/overlap), an embedding model that doesn't distinguish two similar-sounding but distinct concepts in your domain (fix: hybrid retrieval, or a domain-tuned embedding model), or — this one surprised me the first time — questions about content that was deleted or moved since the trace was logged, meaning the "failure" isn't a retrieval bug at all, just a stale eval set.

## Adding a Custom Metric

The built-in metrics cover general RAG quality, but domain-specific correctness usually needs its own metric. Ragas metrics are just classes with a `_ascore` method, so writing one is straightforward:

```python
from ragas.metrics.base import MetricWithLLM
from dataclasses import dataclass

@dataclass
class PolicyCompliance(MetricWithLLM):
    name: str = "policy_compliance"

    async def _ascore(self, row: dict) -> float:
        prompt = (
            f"Does this answer comply with our policy that refund amounts must never "
            f"be stated without mentioning the 30-day window?\n\nAnswer: {row['answer']}\n\n"
            "Respond with only a number: 1.0 if compliant, 0.0 if not."
        )
        response = await self.llm.agenerate([prompt])
        return float(response.generations[0][0].text.strip())

policy_compliance = PolicyCompliance()
results = evaluate(eval_dataset, metrics=[faithfulness, policy_compliance], llm=judge_llm)
```

Custom metrics like this are where Ragas earns its keep beyond the generic faithfulness/relevancy pair — the moment you have a compliance rule, a brand-voice requirement, or a domain-specific correctness constraint, you can fold it into the same evaluation run instead of building a separate check.

## Wiring It into CI Without Breaking the Bank

Running the full eval suite against a live judge model on every commit gets expensive fast. I run a small, fast subset on every PR and the full suite nightly:

```python
import pytest

FAST_SUBSET_SIZE = 15
BASELINE = {"faithfulness": 0.90, "context_recall": 0.70}

@pytest.fixture
def fast_eval_dataset():
    return build_eval_dataset_from_traces(sample_stratified(golden_traces, n_per_category=3))

def test_rag_quality_fast_gate(fast_eval_dataset):
    results = evaluate(fast_eval_dataset, metrics=[faithfulness, context_recall], llm=judge_llm)
    for metric, threshold in BASELINE.items():
        assert results[metric] >= threshold, f"{metric} regressed: {results[metric]:.2f} < {threshold}"
```

The nightly run uses the full golden set and writes results to a dashboard rather than gating a merge — that's where I catch slower drift (a metric trending down over two weeks) that a per-PR threshold check wouldn't flag on any single commit.

## Gotchas I Hit Setting This Up

| Symptom                                   | Cause                                              | Fix                                             |
| ------------------------------------------ | ----------------------------------------------------- | -------------------------------------------------- |
| Scores vary between identical runs          | Judge model temperature not pinned to 0                | Set `temperature=0` on the judge LLM                |
| Evaluation hits rate limits mid-run         | Ragas batches async calls aggressively by default      | Lower batch concurrency, add retry/backoff          |
| Non-English content scores poorly on relevancy | Default embedding model isn't multilingual            | Swap to a multilingual embedding model for that content |
| CI run takes 10+ minutes                   | Running the full golden set on every PR                | Fast subset per-PR, full suite nightly              |
| A metric that was passing suddenly fails    | Underlying content changed, eval set is now stale       | Treat golden-set staleness as its own alert, not just a code regression |

## Key Takeaways

1. **Point the judge model at what you actually run in production** — an unpinned default judge is an easy source of eval drift
2. **Build the dataset from real traces, sampled across categories** — hand-written examples miss the query phrasings real users actually use
3. **Debug per-row, not just the aggregate score** — the aggregate tells you something's wrong, the row-level contexts tell you what
4. **Custom metrics are where Ragas pays off beyond the defaults** — wire in your own compliance and correctness checks the same way
5. **Split fast and full eval runs** — a small per-PR subset for the gate, a full nightly run for tracking slower drift

This is the concrete tool I'd point at Layer 3 of the [skill evaluation framework](/posts/evaluating-agent-skills-framework/) for any skill that's RAG-backed — Ragas measures the retrieval-and-generation half; the selection-accuracy layer still needs the scenario-based approach from that post.
