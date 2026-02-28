# ACE-implementation

Implementation of **Agentic Context Engineering (ACE)** framework from the paper by Qizheng Zhang et al. (Stanford/SambaNova).

## What is ACE?

ACE is a framework that treats contexts as **evolving playbooks** that accumulate, refine, and organize strategies through a modular process. It addresses two key limitations in existing context adaptation methods:

- **Brevity Bias**: Optimizers that prioritize concise instructions lose domain-specific details
- **Context Collapse**: Monolithic rewrites degrade over time, losing valuable information

## Architecture

```
Generator → Reflector → Curator
   ↓           ↓           ↓
Produces    Extracts    Merges
trajectories lessons    deltas
```

- **Generator**: Produces reasoning trajectories for new queries
- **Reflector**: Analyzes executions and extracts actionable lessons
- **Curator**: Synthesizes lessons into compact entries that are merged incrementally

## Results (from paper)

- **+10.6%** on agent benchmarks (AppWorld)
- **+8.6%** on financial analysis (FiNER, Formula)
- **86.9%** lower adaptation latency

## Implementation Stages

1. Foundation (Schema, Storage)
2. Agentic Roles (Reflector, Curator, Generator)
3. Semantic Intelligence (Embeddings)
4. Core Integration
5. Multi-Epoch Reflection
6. Offline Preparation
7. Evaluation & Monitoring
8. Maintenance & Scaling
9. User Interface

See `SKILL.md` for the complete implementation guide.