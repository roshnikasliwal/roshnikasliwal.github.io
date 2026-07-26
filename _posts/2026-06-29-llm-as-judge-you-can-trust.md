---
title: "Building an LLM-as-Judge You Can Actually Trust"
date: 2026-06-29 09:00:00 +0000
mermaid: true
categories: [AI Engineering, Evaluation]
tags: [llm-as-judge, evaluation, llmops, testing, python]
author: Roshni Kasliwal
description: LLM judges fail in predictable, well-documented ways. Here's how I design rubrics, calibrate against human labels, and know when a judge score is actually trustworthy.
---

I've leaned on LLM-as-judge scoring in both the [Ragas evaluation setup](/posts/wiring-up-ragas-rag-evaluation/) and the [skill evaluation framework](/posts/evaluating-agent-skills-framework/) — anywhere a task-level correctness question doesn't reduce to a deterministic check. The failure mode I want to cover today is trusting a judge before you've earned the right to. LLM judges are biased in specific, well-documented ways, and an uncalibrated judge is worse than no judge at all — it gives you a false sense of measurement.

## The Biases That Show Up Every Time

**Position bias**: in pairwise comparisons ("which response is better, A or B"), judges systematically favor whichever response appears first, independent of quality. I've measured this at 10-15 percentage points of swing just from flipping the order.

**Verbosity bias**: longer responses get scored as "more thorough" even when the extra length is padding, not substance.

**Self-preference bias**: a model judging outputs — including its own past outputs or outputs from the same model family — tends to rate them more favorably than outputs from a different model, independent of actual quality.

**Leniency drift**: judges asked to score on a 1-10 scale cluster heavily toward 7-9 regardless of actual quality distribution, which compresses your ability to distinguish "good" from "great."

None of these are exotic edge cases — they show up in the first batch of judge outputs you look at closely, if you look.

## Pointwise vs. Pairwise: Pick Based on What You're Actually Measuring

| Approach      | Question Answered                          | Bias Risk                         | Best For                                  |
| -------------- | --------------------------------------------- | ------------------------------------ | -------------------------------------------- |
| **Pointwise**  | "Does this response meet the bar?"            | Leniency drift                     | Regression gates against a fixed threshold  |
| **Pairwise**   | "Which of these two is better?"               | Position bias, self-preference     | A/B comparing two prompts, models, or versions |

Pick pointwise when you have a golden set with a known acceptable bar. Pick pairwise when you're comparing two variants and don't have (or don't trust) an absolute reference — but budget for running both orderings to catch position bias.

## Rubric Design: Decompose, Don't Ask for a Vibe

The single biggest lever for judge reliability is the rubric. "Rate the quality of this response from 1-10" produces noisy, low-agreement scores. A rubric that decomposes the judgment into specific binary checks produces something you can actually calibrate:

```python
JUDGE_RUBRIC = """
Evaluate the response against each criterion below. For each, answer YES or NO,
then give a final verdict.

1. FACTUAL: Does every claim trace back to the provided context? (not outside knowledge)
2. COMPLETE: Does it address every part of the question asked?
3. CONCISE: Is it free of padding, repetition, or irrelevant elaboration?
4. FORMATTED: Does it follow the required output format (if one was specified)?

After evaluating all four, give a final verdict: PASS only if all four are YES,
otherwise FAIL.

Question: {question}
Context: {context}
Response: {response}

Format your answer as:
FACTUAL: [YES/NO] - [one sentence reason]
COMPLETE: [YES/NO] - [one sentence reason]
CONCISE: [YES/NO] - [one sentence reason]
FORMATTED: [YES/NO] - [one sentence reason]
VERDICT: [PASS/FAIL]
"""
```

Forcing the chain-of-thought reasoning *before* the verdict — rather than asking for a verdict alone — measurably improves consistency. The model has to justify itself, which makes it harder to default to a lenient rubber-stamp.

```python
def judge_response(question: str, context: str, response: str, llm) -> dict:
    prompt = JUDGE_RUBRIC.format(question=question, context=context, response=response)
    output = llm.invoke(prompt).content

    verdict_line = [l for l in output.split("\n") if l.startswith("VERDICT:")][0]
    verdict = "PASS" in verdict_line

    return {"verdict": verdict, "reasoning": output}
```

## Calibration: The Step Most Teams Skip

A rubric that looks reasonable on inspection can still disagree with human judgment in practice. The only way to know is to check — take a sample the judge has scored, get human labels on the same sample, and measure agreement:

```python
from sklearn.metrics import cohen_kappa_score

def calibrate_judge(sample: list[dict], llm) -> dict:
    judge_verdicts = []
    human_verdicts = []

    for item in sample:
        result = judge_response(item["question"], item["context"], item["response"], llm)
        judge_verdicts.append(1 if result["verdict"] else 0)
        human_verdicts.append(1 if item["human_label"] == "PASS" else 0)

    kappa = cohen_kappa_score(human_verdicts, judge_verdicts)
    disagreements = [
        sample[i] for i in range(len(sample))
        if judge_verdicts[i] != human_verdicts[i]
    ]
    return {"kappa": kappa, "disagreements": disagreements}
```

`kappa` above 0.7 is generally considered strong agreement; below 0.4 means the judge isn't measuring what you think it's measuring, and shipping it as a quality gate will train you to trust a number that doesn't track reality. The `disagreements` list is where I actually spend time — reading through cases where the judge and a human disagreed almost always reveals a rubric criterion that's ambiguous or missing entirely, and fixing that criterion is a faster path to a trustworthy judge than swapping models.

## Mitigating Position Bias in Pairwise Judging

If you're doing pairwise comparison, run both orderings and only trust a result that's consistent across the swap:

```python
def pairwise_judge_debiased(question: str, response_a: str, response_b: str, llm) -> str:
    verdict_1 = pairwise_judge(question, response_a, response_b, llm)  # A first
    verdict_2 = pairwise_judge(question, response_b, response_a, llm)  # B first, swapped

    # Normalize verdict_2 back to the original A/B frame
    verdict_2_normalized = "B" if verdict_2 == "A" else "A"

    if verdict_1 == verdict_2_normalized:
        return verdict_1
    return "TIE"  # inconsistent across order — don't force a winner
```

This doubles judge cost per comparison, but a pairwise result that flips depending on presentation order isn't a result — it's noise, and reporting it as a decisive win for one variant is actively misleading.

## When Not to Reach for LLM-as-Judge at All

The [skill evaluation framework post](/posts/evaluating-agent-skills-framework/) makes this point in miniature and it's worth restating on its own: **prefer a deterministic check whenever the task has one.** If you can verify correctness with a field match, a regex, a schema validation, or an exact-string comparison, do that instead — it's cheaper, faster, has zero bias to correct for, and never needs calibration. Reserve LLM-as-judge for genuinely open-ended quality dimensions — tone, completeness, faithfulness to unstructured context — where no deterministic check is possible.

## Key Takeaways

1. **Judges are biased in specific, measurable ways** — position, verbosity, self-preference, leniency drift all show up from the first batch you inspect
2. **Decompose the rubric into binary criteria with required reasoning** — a single "rate 1-10" prompt is the least reliable version of an LLM judge you can build
3. **Calibrate against human labels before trusting the judge as a gate** — Cohen's kappa below 0.4 means the judge isn't measuring what you think
4. **Debias pairwise comparisons by running both orderings** — an order-dependent verdict is noise, not signal
5. **Reach for a deterministic check first** — LLM-as-judge is for the quality dimensions that genuinely have no verifiable ground truth, not a default

---

*Continues the evaluation thread from [Wiring Up Ragas](/posts/wiring-up-ragas-rag-evaluation/) and [Evaluating Agent Skills](/posts/evaluating-agent-skills-framework/).*
