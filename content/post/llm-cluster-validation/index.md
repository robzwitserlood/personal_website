---
title: Teaching an LLM to spot the odd one out
draft: false
date: 2026-01-15
---

## Introduction

This post walks through a project I built to optimize an LLM without manual prompt engineering. The task is simple — given six Dutch keywords, find the one that doesn't belong — but the pipeline around it covers the full DSPy workflow: defining a signature, running a baseline, applying progressively heavier optimizers, and fine-tuning a small local model (Ministral). By the end, you'll have a clear picture of how DSPy structures LLM programs and what each optimization stage actually buys you.

---

## The Task

The concrete problem is intruder detection: given a set of six keywords, five come from the same semantic cluster and one comes from a different cluster. The LLM's job is to identify which keyword is the outlier.

An example Dutch input looks like this:

```
Keywords: rechter, vonnis, rechtbank, straf, verdachte, klimaat
```

The first five are all from a cluster about the Dutch legal system. `klimaat` (climate) is the intruder. For a model that understands Dutch, this is straightforward. But clusters from real-world news data are messier — topics bleed into each other, and a keyword like `maatregel` (measure) plausibly belongs in a legal context, a climate policy context, or a public health context. The task is easy to define and easy to evaluate, which makes it a good vehicle for comparing optimization strategies.

---

## The Data

The keyword clusters come from Dutch news articles processed through BERTopic. BERTopic is a topic modelling approach that uses sentence embeddings and hierarchical density clustering (HDBSCAN) to group documents into topics, then represents each topic by its most distinctive keywords via a class-based TF-IDF variant (c-TF-IDF).

The output is a set of topics, each described by a ranked list of keywords. To construct intruder detection examples, two topics are sampled, five keywords are drawn from one, one keyword is drawn from the other, and the six are shuffled into a fixed-width input. Ground truth is the position of the sampled intruder. Raw examples are stored in `data/raw_examples.json` and loaded once at the start of each script.

---

## The Optimization Pipeline

The pipeline runs four stages in order of increasing computational cost. Each stage builds on the previous one: later stages need training signal that earlier stages generate.

### Stage 1: Baseline

The baseline is a zero-shot `dspy.Predict` module with no examples and no optimized prompt — just the signature docstring and field descriptions turned into a prompt by DSPy's default template. This establishes the floor: what can the model do out of the box, before any optimization?

`[PLACEHOLDER: baseline accuracy]`

### Stage 2: BootstrapFewShot

`BootstrapFewShot` is DSPy's lightest optimizer. It runs the model over a small subset of the training set, collects examples where the model answered correctly, and automatically selects a handful to prepend as few-shot demonstrations. No gradient updates, no external LM calls — just prompt-level improvement from examples the model already knows how to solve.

```python
optimizer = BootstrapFewShot(metric=exact_match, max_bootstrapped_demos=4)
compiled_program = optimizer.compile(program, trainset=trainset)
```

`[PLACEHOLDER: BootstrapFewShot accuracy vs. baseline]`

### Stage 3: GEPA

GEPA (Generalized Prompt Optimization) goes a step further by using a separate, more capable model — here `gpt-4o-mini` — as a *reflection LM*. It looks at examples where the student model failed and rewrites the instruction string to be more explicit or better targeted at the failure mode. The result is a new instruction that the student model didn't write for itself.

This is the first stage that incurs external API cost, which is why it sits after the cheaper prompt-level optimizer rather than before it.

`[PLACEHOLDER: GEPA accuracy vs. BootstrapFewShot]`

### Stage 4: BootstrapFinetune

The primary objective. `BootstrapFinetune` uses the examples and traces collected in earlier stages as a synthetic fine-tuning dataset, then updates the weights of the student model — `meta-llama/Llama-3.2-1B-Instruct` — directly. This is no longer prompt engineering; it's a weight update.

The rationale for the ordering is that you don't want to commit to the cost of fine-tuning before you know what a well-prompted version of the model can achieve. The earlier stages generate better training signal than raw gold labels alone, because they include reasoning traces and examples that the model architecture can actually learn from.

`[PLACEHOLDER: BootstrapFinetune accuracy vs. GEPA]`

---

## Experiment Tracking with MLflow

DSPy has native MLflow autologging. Enabling it takes one line:

```python
import mlflow
mlflow.dspy.autolog()
```

From that point, every optimizer run logs the compiled program state, per-example predictions, and aggregate metrics to an MLflow experiment. The compiled program state is particularly useful: it's a serialized snapshot of the optimized prompt or fine-tuned adapter that can be loaded back without re-running the optimizer.

Fine-tuned models are registered in the MLflow model registry and served locally:

```bash
mlflow models serve -m "models:/intruder-detector/1" -p 5001
```

This exposes the same OpenAI-compatible endpoint that DSPy's LM client expects — so the deployed model is a drop-in replacement for the development model without any code changes.

`[PLACEHOLDER: MLflow run comparison screenshot]`

---

## Lessons Learned & What's Next

- **The signature is the interface contract.** Getting the field descriptions right matters more than it first appears. Vague descriptions produce inconsistent output formats; precise ones make parsing reliable and give the optimizer something concrete to work with.

- **Run the cheap optimizers first.** BootstrapFewShot costs almost nothing and often closes most of the gap between baseline and a well-tuned model. GEPA and BootstrapFinetune are worth running, but the incremental gain over BootstrapFewShot tells you whether the task is prompt-starved or weight-starved.

- **Local models change the economics.** Running `ministral-3:3b` via SGLang means the baseline and BootstrapFewShot stages are effectively free. That lowers the threshold for experimentation — you can iterate on the signature, the metric, and the training split without watching a cost meter.

- **What's next:** Scaling the student model to a larger variant, adding more topically diverse BERTopic clusters to the training set, and evaluating whether the fine-tuned model transfers to clusters from a different corpus entirely.

---

## Conclusion

DSPy reframes LLM development from prompt authorship to program optimization. The intruder detection task is simple enough to reason about clearly, which makes it a useful lens for understanding what each optimization stage actually does: BootstrapFewShot finds examples, GEPA rewrites instructions, BootstrapFinetune updates weights. Each stage is independently useful and composable. The full pipeline — from a zero-shot baseline to a fine-tuned local model tracked in MLflow — is a blueprint that transfers to more complex tasks with very few structural changes.
