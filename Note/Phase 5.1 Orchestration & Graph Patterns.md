# Agentic Workflow Design: Orchestration & Graph Patterns

---

## 🎯 Learning objectives

- [ ]  Explain the difference between a **workflow** (predefined code path) and an **agent** (LLM directs its own process)
- [ ]  Differentiate 4 orchestration shapes: deterministic workflow, router, planner-executor, multi-agent
- [ ]  Choose the right shape for a task using a risk-based decision framework, not "which is coolest"
- [ ]  Build a stateful graph in LangGraph (nodes, edges, conditional edges, checkpointing)
- [ ]  Build the same workflow in Agno (Agent, Team, Workflow) and the OpenAI Agents SDK (Agent, handoffs, agents-as-tools)
- [ ]  Add a human-in-the-loop / approval checkpoint to at least one pattern

---

## 🧠 Core mental model

Anthropic's framing is the cleanest starting point for this whole topic:

> **Workflows** are systems where LLMs and tools are orchestrated through **predefined code paths**.
**Agents** are systems where LLMs **dynamically direct their own process and tool usage**.
> 

Everything else in this plan is a spectrum between those two poles. The four patterns below are ordered roughly by how much control you hand from your code to the model.

📖 Read first: Anthropic — Building Effective Agents
This single post defines prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer, and the autonomous agent loop — the vocabulary the rest of this field uses.yu

---

## 🧩 The four patterns, compared

| Pattern | Control flow | Model's role | When it fits | Failure mode if misused |
| --- | --- | --- | --- | --- |
| **Simple deterministic workflow** | Fixed sequence of steps in your code (prompt chaining, parallelization) | Fills in one step at a time; never chooses what happens next | Task is well-specified, steps are known in advance, high reliability matters more than flexibility | Using an LLM step where a plain function would do — adds latency/cost/risk for nothing |
| **Router pattern** | Your code classifies input, then dispatches to one of N fixed handlers | Chooses *which* fixed branch to take | Task falls into a small number of known categories, each needing different instructions/tools | Too many overlapping routes; router becomes a brittle guess instead of a clean classifier |
| **Planner-executor** | A planning step (often one LLM call) produces a plan/subtask list; an executor loop works through it, can replan | Decides *what the steps should be*, not just which branch | Task decomposition can't be hardcoded, but each step is independently checkable ("ground truth" from tool results, tests, etc.) | No verification step — the plan silently drifts and errors compound across steps |
| **Multi-agent system** | Multiple agents with distinct roles/tools coordinate (handoffs, or a manager calling agents-as-tools) | Each agent directs its own subtask; may also decide when to hand off | Distinct specialties genuinely need separate context/tools/policies, and the task is parallelizable or requires escalation between roles | Coordination overhead, context fragmentation, hardest to debug and cheapest to over-engineer into |

### Risk-based selection — the actual decision framework

Ask these in order; stop at the first "yes":

1. **Can this be fully hardcoded?** → Plain code, no LLM. (Not a workflow pattern at all — the most common mistake is skipping this check.)
2. **Is the task one well-defined LLM call with good context?** → Single augmented LLM call. Don't build a graph for this.
3. **Are the steps and their order fixed and known?** → **Deterministic workflow** (prompt chaining / parallelization). Best predictability, easiest to test, cheapest to run.
4. **Is there a small, closed set of categories the input falls into, each needing different handling?** → **Router**. Still fully predictable per-branch; only the classification step is "agentic."
5. **Does the task need to be broken into subtasks that can't be enumerated ahead of time, but each subtask's success is verifiable (tests pass, data matches schema, tool returns expected shape)?** → **Planner-executor**. You're trading some predictability for adaptability, but you still have a checkable ground truth at each step — this is what makes it safe to hand control to the model here.
6. **Do genuinely different specialties/tools/policies need to own different parts of the conversation, or does the work parallelize across independent workers that then get synthesized?** → **Multi-agent** (orchestrator-workers, or handoffs for conversation ownership). Reserve this for when 1–5 don't fit — it's the most expensive and hardest-to-debug option.

**Risk multipliers that push you toward *more* structure (lower in this list = more caution needed), regardless of which pattern you pick:**

- Irreversible/write actions (deploys, payments, deletes) → add a human-approval gate as a graph node, not a prompt instruction
- High cost-per-error → prefer evaluator-optimizer loops or added verification steps over raw autonomy
- Long-horizon tasks → checkpointing/persistence become mandatory, not optional (this is where LangGraph's durable execution or Agno's session storage matters)
- Multiple agents touching the same state → you need an explicit state schema and single source of truth, not each agent keeping its own memory

---

## 🗓 4-week plan

### Week 1 — Foundations: augmented LLM, workflows vs agents

## 🧩 What AI Agents Are

- **Workflows:** Predefined sequences where LLMs follow fixed code paths. Predictable and consistent.
- **Agents:** LLMs dynamically decide how to accomplish tasks, using tools and planning autonomously.

Anthropic treats both as *agentic systems* but distinguishes them by how much autonomy the LLM has.

## ⚖️ When to Use Agents (and When Not To)

- Start with the **simplest solution** (single LLM call + retrieval).
- Use **workflows** for predictable, well‑structured tasks.
- Use **agents** when:
    - Tasks are open‑ended
    - Steps can’t be predetermined
    - Tool use and planning must adapt dynamically
- Agents trade **higher cost & latency** for **flexibility & performance**.

## 🏗️ Building Blocks & Common Patterns

Anthropic outlines a progression from simple to complex agentic patterns:

### 1. **Augmented LLM**

LLM + retrieval + tools + memory
Often implemented via **Model Context Protocol (MCP)**.

### 2. **Prompt Chaining**

Sequential LLM calls with checks between steps. Useful when tasks can be cleanly decomposed.

ex: Directing different types of customer service queries (general questions, refund requests, technical support) into different downstream processes, prompts, and tools.

!image.png

### 3. **Routing**

Classify input → send to specialized prompts or models.
Great for customer support or multi‑model optimization.

!image.png

### 4. **Parallelization**

- **Sectioning:** Split subtasks and run in parallel
- **Voting:** Multiple attempts → aggregate results
Useful for guardrails, evals, code review, content moderation.

!image.png

### 5. **Orchestrator–Workers**

A central LLM dynamically creates subtasks and delegates.
Ideal for coding tasks or multi‑source search. Well-suited for complex tasks where you can’t predict the subtasks needed (in coding, for example, the number of files that need to be changed and the nature of the change in each file likely depend on the task)

!image.png

### 6. **Evaluator–Optimizer**

LLM generates → another LLM critiques → iterate.
Useful for translation, complex search, refinement loops. Effective when we have clear evaluation criteria, and when iterative refinement provides measurable value.

!image.png

## 🤖 Autonomous Agents

Agents operate independently once given a goal:

- Plan steps
- Use tools
- Gather ground truth from environment
- Ask for human input when needed
- Stop based on completion or iteration limits
- 

Best for:

- Open‑ended tasks
- Multi‑step reasoning
- Scalable automation

Examples from Anthropic:

- Coding agents solving SWE‑bench tasks
- “Computer use” agents that operate a computer

---

### Week 2 — Deterministic workflows & the router pattern (LangGraph)

**Read**

- [ ]  LangGraph, Overview — concepts: `StateGraph`, nodes, edges, conditional edges.
- [ ]  LangGraph Academy course: Introduction to LangGraph — Module 1 (state, nodes, edges) and the module on conditional routing.
- [ ]  LangGraph API reference for `StateGraph` basics: reference.langchain.com/python/langgraph

**Do**

- [ ]  Build a **deterministic workflow**: a 3-node prompt chain (e.g., draft → critique → revise) in LangGraph, with a plain-code gate between steps that checks the output before continuing.
- [ ]  Build a **router**: one node classifies an input into 2–3 categories, a conditional edge dispatches to different downstream nodes/prompts. Test it with inputs designed to hit each branch.
- [ ]  Add a checkpointer (LangGraph's built-in persistence) and confirm you can resume a run after interrupting it.

---

### Week 3 — Planner-executor & orchestrator-workers (Agno)

**Read**

- [ ]  Agno docs home: docs.agno.com — read the sections on `Agent`, `Team`, and `Workflow` (Agno's names for single-agent, multi-agent, and structured multi-step patterns).
- [ ]  Agno GitHub README: github.com/agno-agi/agno — note the built-in "Human approval" and "Guardrails" primitives; these are what you'd wire a HITL gate into.
- [ ]  Re-read Anthropic's **orchestrator-workers** section for the conceptual pattern this maps to.

**Do**

- [ ]  Build a **planner-executor**: one Agno `Agent`/step produces a subtask list (structured output), a loop executes each subtask with its own tool calls, and a final step synthesizes results.
- [ ]  Add a verification step after each subtask (e.g., schema validation, a re-check LLM call, or a real test/assertion) — this is the "ground truth" check that makes planner-executor safe to use.
- [ ]  Compare: rebuild the same planner-executor task as a LangGraph graph with a `plan` node and a loop back-edge. Note what each framework makes easy vs. awkward.

---

### Week 4 — Multi-agent systems & production concerns (OpenAI Agents SDK)

**Read**

- [ ]  OpenAI, Orchestration and handoffs — the **handoffs vs. agents-as-tools** distinction is the key concept this week.
- [ ]  OpenAI Agents SDK, Agent orchestration — "orchestrating via code" vs "orchestrating via LLM" tradeoffs.
- [ ]  OpenAI Agents SDK, Agents — manager pattern vs. handoff pattern code examples.
- [ ]  OpenAI, Guardrails and human review (linked from the Agents SDK guide) for how resumable approval flows are modeled.

# Agent Definitions

An agent is the core unit of an SDK-based workflow. It packages a model, instructions, and optional runtime behavior such as tools, guardrails, MCP servers, handoffs, and structured outputs.

```python

from agents import Agent, function_tool

@function_tool
def get_weather(city: str) -> str:
    """Return the weather for a given city."""
    return f"The weather in {city} is sunny."

agent = Agent(
    name="Weather bot",
    instructions="You are a helpful weather bot.",
    model="gpt-5.6",
    tools=[get_weather],
)

```

## **Shape instructions, handoffs, and outputs**

Three configuration choices deserve extra care:

- Start with static `instructions`. When the guidance depends on the current user, tenant, or runtime context, switch to a dynamic instructions callback instead of stitching strings together at the call site.
- Keep `handoff_description` short and concrete so routing agents know when to pick this specialist.
- Use `output_type` when downstream code needs typed data rather than free-form prose.

```python

import asyncio

from pydantic import BaseModel

from agents import Agent, Runner

class CalendarEvent(BaseModel):
    name: str
    date: str
    participants: list[str]

agent = Agent(
    name="Calendar extractor",
    instructions="Extract calendar events from text.",
    output_type=CalendarEvent,
)

async def main() -> None:
    result = await Runner.run(
        agent,
        "Dinner with Priya and Sam on Friday.",
    )
    print(result.final_output)

if __name__ == "__main__":
    asyncio.run(main())
```

## **Keep local context separate from model context**

The SDK lets you pass application state and dependencies into a run without sending them to the model. Use this for data like authenticated user info, database clients, loggers, and helper functions.

```python

import asyncio
from dataclasses import dataclass

from agents import Agent, RunContextWrapper, Runner, function_tool

@dataclass
class UserInfo:
    name: str
    uid: int

@function_tool
async def fetch_user_age(wrapper: RunContextWrapper[UserInfo]) -> str:
    """Fetch the age of the current user."""
    return f"The user {wrapper.context.name} is 47 years old."

agent = AgentUserInfo

async def main() -> None:
    result = await Runner.run(
        agent,
        "What is the age of the user?",
        context=UserInfo(name="John", uid=123),
    )
    print(result.final_output)

if __name__ == "__main__":
    asyncio.run(main())
```

The important boundary is:

- Conversation history is what the model sees.
- Run context is what your code sees.

If the model needs a fact, put it in instructions, input, retrieval, or a tool. If only your runtime needs it, keep it in local context.

## **When to split one agent into several**

Split an agent when one specialist shouldn’t own the full reply or when separate capabilities are materially different. Common reasons are:

- A specialist needs a different tool or MCP surface.
- A specialist needs a different approval policy or guardrail.
- One branch of the workflow needs a different model or output style.
- You want explicit routing in traces rather than a single large prompt.

# Orchestration

Multi-agent workflows are useful when specialists should own different parts of the job. The first design choice is deciding who owns the final user-facing answer at each branch of the workflow.

## **Choose the orchestration pattern**

| **Pattern** | **Use it when** | **What happens** |
| --- | --- | --- |
| Handoffs | A specialist should take over the conversation for that branch of the work | Control moves to the specialist agent |
| Agents as tools | A manager should stay in control and call specialists as bounded capabilities | The manager keeps ownership of the reply |

## **Use handoffs for delegated ownership**

Handoffs are the clearest fit when a specialist should own the next response rather than merely helping behind the scenes.

```python

from agents import Agent, handoff

billing_agent = Agent(name="Billing agent")
refund_agent = Agent(name="Refund agent")

triage_agent = Agent(
    name="Triage agent",
    handoffs=[billing_agent, handoff(refund_agent)],
)
```

Keep the routing surface legible:

- Give each specialist a narrow job.
- Keep `handoff_description` short and concrete.
- Split only when the next branch truly needs different instructions, tools, or policy.

## **Use agents as tools for manager-style workflows**

Use `agent.as_tool()` when the main agent should stay responsible for the final answer and call specialists as helpers.

```python

from agents import Agent

summarizer = Agent(
    name="Summarizer",
    instructions="Generate a concise summary of the supplied text.",
)

main_agent = Agent(
    name="Research assistant",
    tools=[
        summarizer.as_tool(
            tool_name="summarize_text",
            tool_description="Generate a concise summary of the supplied text.",
        )
    ],
)
```

This is usually the better fit when:

- the manager should synthesize the final answer
- the specialist is doing a bounded task like summarization or classification
- you want one stable outer workflow with nested specialist calls instead of ownership transfer

# **Guardrails and human review**

Use guardrails for automatic checks and human review for approval decisions. Together, they define when a run should continue, pause, or stop

- **Guardrails** validate input, output, or tool behavior automatically.
- **Human review** pauses the run so a person or policy can approve or reject a sensitive action

| **Use case** | **Start with** |
| --- | --- |
| Block disallowed user requests before the main model runs | Input guardrails |
| Validate or redact the final output before it leaves the system | Output guardrails |
| Check arguments or results around a function tool call | Tool guardrails |
| Pause before side effects like cancellations, edits, shell commands, or sensitive MCP actions | Human-in-the-loop approvals |

## **Add a blocking guardrail**

Use input guardrails when you want a fast validation step to run before the expensive or side-effecting part of the workflow starts.

```python
import asyncio

from pydantic import BaseModel

from agents import (
    Agent,
    GuardrailFunctionOutput,
    InputGuardrailTripwireTriggered,
    RunContextWrapper,
    Runner,
    TResponseInputItem,
    input_guardrail,
)

class MathHomeworkOutput(BaseModel):
    is_math_homework: bool
    reasoning: str

guardrail_agent = Agent(
    name="Homework check",
    instructions="Detect whether the user is asking for math homework help.",
    output_type=MathHomeworkOutput,
)

@input_guardrail
async def math_guardrail(
    ctx: RunContextWrapper[None],
    agent: Agent,
    input: str | list[TResponseInputItem],
) -> GuardrailFunctionOutput:
    result = await Runner.run(guardrail_agent, input, context=ctx.context)
    return GuardrailFunctionOutput(
        output_info=result.final_output,
        tripwire_triggered=result.final_output.is_math_homework,
    )

agent = Agent(
    name="Customer support",
    instructions="Help customers with support questions.",
    input_guardrails=[math_guardrail],
)

async def main() -> None:
    try:
        await Runner.run(agent, "Can you solve 2x + 3 = 11 for me?")
    except InputGuardrailTripwireTriggered:
        print("Guardrail blocked the request.")

if __name__ == "__main__":
    asyncio.run(main())
    
#Use blocking execution when the cost or risk of starting the main agent is too high. 
#Use parallel guardrails when lower latency matters more than avoiding speculative work.
```

## **Pause for human review**

Approvals are the human-in-the-loop path for tool calls. The model can still decide that an action is needed, but the run pauses until you approve or reject it.

```python

import asyncio

from agents import Agent, Runner, function_tool

@function_tool(needs_approval=True)
async def cancel_order(order_id: int) -> str:
    return f"Cancelled order {order_id}"

agent = Agent(
    name="Support agent",
    instructions="Handle support requests and ask for approval when needed.",
    tools=[cancel_order],
)

async def main() -> None:
    result = await Runner.run(agent, "Cancel order 123.")

    if result.interruptions:
        state = result.to_state()
        for interruption in result.interruptions:
            state.approve(interruption)
        result = await Runner.run(agent, state)

    print(result.final_output)

if __name__ == "__main__":
    asyncio.run(main())

```

## **Approval lifecycle**

When a tool call needs review, the SDK follows the same pattern every time:

1. The run records an approval interruption instead of executing the tool.
2. The result returns `interruptions` plus a resumable `state`.
3. Your application approves or rejects the pending items.
4. You resume the same run from `state` instead of starting a new user turn.

If the review might take time, serialize `state`, store it, and resume later. That’s still the same run.

## **Workflow boundaries matter**

Agent-level guardrails don’t run everywhere:

- Input guardrails run only for the first agent in the chain.
- Output guardrails run only for the agent that produces the final output.
- Tool guardrails run on the function tools they’re attached to.

# **Results and state**

When you run an agent, the result is more than just the final answer. It’s also the handoff boundary, the next-turn continuation surface, and the resumable snapshot when a run pauses for review.

| **If you need** | **Use** |
| --- | --- |
| The final answer to show the user | `final_output` |
| Local replay-ready history | `to_input_list()` |
| The specialist that should usually own the next turn | `last_agent` |
| OpenAI-managed response chaining | `last_response_id` |
| Pending approvals and a resumable snapshot | `interruptions` plus `to_state()` |

Use the result in a way that matches your continuation strategy:

- If your application owns full local history, reuse `to_input_list()`.
- If you are using a session, keep passing the same session and let the SDK load and persist history for you.
- If you are using server-managed continuation, pass only the new user input and reuse the stored ID instead of replaying the full transcript.
- After handoffs, reuse `last_agent` when that specialist should stay in control for the next turn.

---

## Glossary (quick reference)

---

| Term | Meaning |
| --- | --- |
| Workflow | LLM/tool calls orchestrated through code you wrote in advance |
| Agent | LLM decides its own next action based on environment feedback |
| Router | A classification step that dispatches to one of several fixed downstream paths |
| Planner-executor | A planning step generates subtasks; an execution loop carries them out, can replan |
| Orchestrator-workers | A central LLM decomposes work and delegates to worker LLMs, then synthesizes |
| Handoff | One agent transfers full ownership of the conversation to another |
| Agent-as-tool | A specialist agent is called like a function by a manager agent, which keeps final control |
| Checkpointing | Persisting graph/agent state so a run can pause, fail, and resume without starting over |
| Ground truth | Verifiable feedback from the environment (test results, tool output, schema validation) used to check agent progress |