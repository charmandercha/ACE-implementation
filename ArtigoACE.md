# Agentic Context Engineering: Evolving Contexts for Self-Improving Language Models

Qizheng Zhang 1*  Changran Hu 2*  Shubhangi Upasani 2  Boyuan Ma 2  Fenglu Hong 2  
Vamsidhar Kamanuru 2  Jay Rainton 2  Chen Wu 2  Mengmeng Ji 2  Hanchen Li 3  
Urmish Thakker 2  James Zou 1  Kunle Olukotun 1

1 Stanford University  2 SambaNova Systems, Inc.  3 UC Berkeley  \* equal contribution

qizhengz@stanford.edu, changran.hu@sambanovasystems.com

---

## Abstract

Large language model (LLM) applications such as agents and domain-specific reasoning increasingly rely on _context adaptation_—modifying inputs with instructions, strategies, or evidence, rather than weight updates. Prior approaches improve usability but often suffer from brevity bias, which drops domain insights for concise summaries, and from context collapse, where iterative rewriting erodes details over time.

Building on the adaptive memory introduced by Dynamic Cheatsheet, we introduce ACE (Agentic Context Engineering), a framework that treats contexts as evolving playbooks that accumulate, refine, and organize strategies through a modular process of generation, reflection, and curation. ACE prevents collapse with structured, incremental updates that preserve detailed knowledge and scale with long-context models.

Across agent and domain-specific benchmarks, ACE optimizes contexts both offline (e.g., system prompts) and online (e.g., agent memory), consistently outperforming strong baselines: +10.6% on agents and +8.6% on finance, while significantly reducing adaptation latency and rollout cost.

Notably, ACE could adapt effectively without labeled supervision and instead by leveraging natural execution feedback. On the AppWorld leaderboard, ACE matches the top-ranked production-level agent on the overall average and surpasses it on the harder test-challenge split, despite using a smaller open-source model.

These results show that comprehensive, evolving contexts enable scalable, efficient, and self-improving LLM systems with low overhead.

---

## 1 Introduction

Modern AI applications based on large language models (LLMs), such as LLM agents and compound AI systems, increasingly depend on context adaptation. Instead of modifying model weights, context adaptation improves performance after model training by incorporating clarified instructions, structured reasoning steps, or domain-specific input formats directly into the model's inputs.

Adapting through contexts rather than weights offers several key advantages:

- Contexts are interpretable and explainable for users and developers
- Allow rapid integration of new knowledge at runtime
- Can be shared across models or modules in a compound system

Meanwhile, advances in long-context LLMs and context-efficient inference such as KV cache reuse are making context-based approaches increasingly practical for deployment.

### Limitations of Existing Context Adaptation Methods

**The Brevity Bias**: Many prompt optimizers prioritize concise, broadly applicable instructions over comprehensive accumulation. For example, GEPA highlights brevity as a strength, but such abstraction can omit domain-specific heuristics, tool-use guidelines, or common failure modes that matter in practice.

**Context Collapse**: Methods that rely on monolithic rewriting by an LLM often degrade into shorter, less informative summaries over time, causing sharp performance declines. In domains such as interactive agents, domain-specific programming, and financial or legal analysis, strong performance depends on retaining detailed, task-specific knowledge rather than compressing it away.

---

## 2 Background and Motivation

### Context Adaptation

Context adaptation (or context engineering) refers to methods that improve model behavior by constructing or modifying inputs to an LLM, rather than altering its weights. The current state of the art leverages natural language feedback. Representative methods include:

- **Reflexion**: Reflects on failures to improve agent planning
- **TextGrad**: Optimizes prompts via gradient-like textual feedback
- **GEPA**: Refines prompts iteratively based on execution traces
- **Dynamic Cheatsheet**: Constructs an external memory that accumulates strategies and lessons from past successes and failures during inference

---

## 3 Agentic Context Engineering (ACE)

We present ACE (Agentic Context Engineering), a framework for scalable and efficient context adaptation in both offline (e.g., system prompt optimization) and online (e.g., test-time memory adaptation) scenarios.

Instead of condensing knowledge into terse summaries or static instructions, ACE treats contexts as evolving playbooks that continuously accumulate, refine, and organize strategies over time.

### Architecture

ACE adopts an agentic architecture with three specialized components:

1. **Generator**: Produces reasoning trajectories for new queries, which surface both effective strategies and recurring pitfalls.

2. **Reflector**: Critiques these traces to extract lessons, optionally refining them across multiple iterations.

3. **Curator**: Synthesizes these lessons into compact delta entries, which are merged deterministically into the existing context by lightweight, non-LLM logic.

This mirrors how humans learn—experimenting, reflecting, and consolidating—while avoiding the bottleneck of overloading a single model with all responsibilities.

### Key Innovations

1. **Dedicated Reflector**: Separates evaluation and insight extraction from curation, improving context quality and downstream performance.

2. **Incremental Delta Updates**: Replaces costly monolithic rewrites with localized edits, reducing both latency and compute cost.

3. **Grow-and-Refine Mechanism**: Balances steady context expansion with redundancy control.

### 3.1 Incremental Delta Updates

A core design principle of ACE is to represent context as a collection of _structured, itemized bullets_, rather than a single monolithic prompt. Each bullet consists of:

- Metadata: unique identifier and counters tracking how often it was marked helpful or harmful
- Content: a small unit such as a reusable strategy, domain concept, or common failure mode

This itemized design enables three key properties:

- **Localization**: Only the relevant bullets are updated
- **Fine-grained retrieval**: The Generator can focus on the most pertinent knowledge
- **Incremental adaptation**: Allowing efficient merging, pruning, and de-duplication during inference

### 3.2 Grow-and-Refine

Beyond incremental growth, ACE ensures that contexts remain compact and relevant through periodic or lazy refinement:

- Bullets with new identifiers are appended
- Existing bullets are updated in place (e.g., incrementing counters)
- A de-duplication step prunes redundancy by comparing bullets via semantic embeddings

This refinement can be performed proactively (after each delta) or lazily (only when the context window is exceeded).

---

## 4 Results

### Tasks and Datasets

We evaluate ACE on two categories of LLM applications:

1. **LLM Agent: AppWorld**: A suite of autonomous agent tasks involving API understanding, code generation, and environment interaction. It provides a realistic execution environment with common applications and APIs (e.g., email, file system) and tasks of two difficulty levels (normal and challenge).

2. **Financial Analysis**:
   - **FiNER**: Requires labeling tokens in XBRL financial documents with one of 139 fine-grained entity types
   - **Formula**: Focuses on extracting values from structured XBRL filings and performing computations to answer financial queries

### Methods

- **Base LLM**: The base model evaluated directly on each benchmark without any context engineering
- **ICL (In-Context Learning)**: Provides the model with task demonstrations in the input prompt
- **MIPROv2**: A popular prompt optimizer that works by jointly optimizing system instructions and in-context demonstrations via bayesian optimization
- **GEPA (Genetic-Pareto)**: A sample-efficient prompt optimizer based on reflective prompt evolution
- **Dynamic Cheatsheet (DC)**: Test-time learning approach that introduces an adaptive external memory of reusable strategies
- **ACE (ours)**: Our proposed framework

### Results on Agent Benchmark

| Method               | GT Labels | Test-Normal TGC | Test-Normal SGC | Test-Challenge TGC | Test-Challenge SGC | Average  |
| -------------------- | --------- | --------------- | --------------- | ------------------ | ------------------ | -------- |
| ReAct (Base)         | -         | 63.7            | 42.9            | 41.5               | 21.6               | 42.4     |
| ReAct + ICL          | ✓         | 64.3            | 46.4            | 46.0               | 27.3               | 46.0     |
| ReAct + GEPA         | ✓         | 64.9            | 44.6            | 46.0               | 30.2               | 46.4     |
| ReAct + ACE          | ✓         | **76.2**        | **64.3**        | **57.3**           | **39.6**           | **59.4** |
| ReAct + ACE          | ✗         | 75.0            | 64.3            | 54.4               | 35.2               | 57.2     |
| ReAct + DC (Online)  | ✗         | 65.5            | 58.9            | 52.3               | 30.8               | 51.9     |
| ReAct + ACE (Online) | ✗         | **69.6**        | **53.6**        | **66.0**           | **48.9**           | **59.5** |

**Key Findings:**

- ACE consistently outperforms strong baselines by an average of **10.6%**
- ACE achieves good performance even without access to ground-truth labels during adaptation
- On the AppWorld leaderboard, ReAct + ACE (59.4%) matches the top-ranked IBM CUGA (60.3%), despite using a smaller open-source model (DeepSeek-V3.1)

### Results on Domain-Specific Benchmark (Financial Analysis)

| Method       | GT Labels | FiNER (Acc) | Formula (Acc) | Average  |
| ------------ | --------- | ----------- | ------------- | -------- |
| Base LLM     | -         | 70.7        | 67.5          | 69.1     |
| ICL          | ✓         | 72.3        | 67.0          | 69.6     |
| MIPROv2      | ✓         | 72.4        | 69.5          | 70.9     |
| GEPA         | ✓         | 73.5        | 71.5          | 72.5     |
| ACE          | ✓         | **78.3**    | **85.5**      | **81.9** |
| ACE          | ✗         | 71.1        | 83.0          | 77.1     |
| DC (Online)  | ✓         | 74.2        | 69.5          | 71.8     |
| ACE (Online) | ✓         | **76.7**    | **76.5**      | **76.6** |

**Key Findings:**

- ACE delivers an average performance gain of **8.6%** over strong baselines on financial analysis
- Particularly effective when tasks require precise domain knowledge (e.g., financial concepts, XBRL rules)

### Ablation Study

| Method                           | GT Labels | Test-Normal TGC | Test-Normal SGC | Test-Challenge TGC | Test-Challenge SGC | Average  |
| -------------------------------- | --------- | --------------- | --------------- | ------------------ | ------------------ | -------- |
| ACE w/o Reflector or multi-epoch | ✓         | 68.9            | 53.2            | 49.0               | 30.9               | 50.5     |
| ACE w/o multi-epoch              | ✓         | 72.3            | 60.3            | 53.2               | 36.9               | 55.7     |
| ACE (full)                       | ✓         | **76.2**        | **64.3**        | **57.3**           | **39.6**           | **59.4** |

The Reflector with iterative refinement and multi-epoch adaptation each contribute substantially to performance.

### Cost and Speed Analysis

- ACE requires **86.9% lower adaptation latency** on average
- Achieves these gains with fewer rollouts and lower token dollar costs

---

## 5 Discussion

### Longer Context ≠ Higher Serving Cost

ACE achieves substantial improvements while maintaining efficiency through incremental updates rather than full context regeneration.

### Implications for Online and Continuous Learning

ACE demonstrates that self-improving LLM systems can be achieved with both higher accuracy and lower overhead, enabling:

- Continuous accumulation of domain-specific knowledge
- Effective adaptation without labeled supervision
- Scalable context engineering for long-horizon applications

---

## Appendix A: Related Work on Agent Memory

ACE builds upon and extends work in agent memory systems, particularly Dynamic Cheatsheet, by adding structured reflection and incremental updates.

---

## Appendix B: Limitations and Challenges

- Context adaptation depends critically on feedback quality
- In the absence of reliable feedback signals (e.g., ground-truth labels or execution outcomes), both ACE and other adaptive methods may degrade
- The constructed context can be polluted by spurious or misleading signals

---

## Appendix C: AppWorld Leaderboard Snapshot (09/2025)

On the AppWorld leaderboard, ACE matches the top-ranked production-level agent IBM-CUGA on average and surpasses it on the harder test-challenge split.

---

## Appendix D: Prompts

(Contains the detailed prompts used in the experiments)

---

## References

[1-57] Various references to related work on LLM agents, context adaptation, prompt optimization, and memory systems.
