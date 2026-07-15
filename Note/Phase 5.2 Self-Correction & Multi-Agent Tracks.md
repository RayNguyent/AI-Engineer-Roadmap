# Learning Objectives

By end of day you should be able to explain:

### Reflection & Critique Loops

- What agent reflection is
- Why self-critique improves outputs
- Risks of recursive reasoning
- How to enforce bounded execution

### Multi-Agent Systems

- Supervisor-worker architecture
- Task delegation
- Specialized agents
- Coordination overhead

### Architecture Decisions

- When deterministic pipelines win
- When agents provide value
- Costs and tradeoffs

# Understanding Reflection Loops

Question
↓
Generate Answer
↓
Critique Answer
↓
Revise Answer
↓
Return Final Output

https://www.anthropic.com/research/constitutional-ai-harmlessness-from-ai-feedback

https://cookbook.openai.com (reasoning, reflection ,evaluation)

# **Bounded Reflection**

Without limits: Generate → Critique → Revise → Generate → Critique → Revise → ……

⇒ Bound the loop

- MAX_REFLECTIONS = 3
- Confidence Threshold: quality_score ≥ 0.9
- Cost Budget:  if tokens_used > budget ⇒ break
- Time Budget: if elapsed > 10 ⇒ break

```python
for i in range(3):
    answer = generate()

    critique = critique_answer(answer)

    if critique.passed:
        break

    answer = improve(answer, critique)
```

# Supervisor-Worker Multi-Agent Pattern

### Supervisor

- receives request
- decomposes task
- delegates work
- merges outputs

### Workers

Specialists.

ex: retrieval agen → find docs, sql agent → run queries, research agent → web search, compliance agent → policy checks

### Drawbacks

More:

- latency
- cost
- orchestration complexity
- debugging difficulty