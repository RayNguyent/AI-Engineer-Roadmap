# Learning Plan: Structured Tool I/O Specs & Permission-Scoped Tool Registry

## Objective

Build the skills and a working deliverable that demonstrates:

1. Designing structured tool input/output specifications using **Pydantic** and **JSON Schema**, so an LLM (or any caller) can reliably invoke tools with validated arguments and typed responses.
2. Implementing a **centralized tool registry** that strictly segregates **read-only** tools (safe, idempotent, no side effects) from **write/side-effect** tools (mutate state, call external systems, cost money, etc.).
3. Defining **explicit access parameters** (who/what can call which tool, under what scope) that are enforced at execution time — not just documented.
4. Wrapping the above in **monitoring/tracking**: every tool call is logged with permission scope, latency, success/failure, and caller identity, producing metrics usable in a real internal business application.

**End deliverable:** A monitored tool registry (Python package) enforcing permission-scoped access, with tracking metrics, ready to plug into an agentic system (e.g. your existing RAG pipeline repo).

---

## Phase 1 — Structured Tool I/O with Pydantic & JSON Schema

---

**Goal:** Every tool has a strict, machine-readable contract for its inputs and outputs.

### What to learn

- Pydantic v2 `BaseModel`, field validation, `Field(..., description=...)` for LLM-facing docs
- Generating JSON Schema from a Pydantic model (`model_json_schema()`)
- Why LLM function-calling APIs (OpenAI, Anthropic) consume JSON Schema directly
- Validating a tool's *output*, not just its input (return schemas catch silent failures)

### Sources

- Pydantic docs — Models & JSON Schema: https://docs.pydantic.dev/latest/concepts/json_schema/
- Anthropic tool use docs (input_schema format): https://docs.claude.com/en/docs/build-with-claude/tool-use
- JSON Schema spec basics: https://json-schema.org/understanding-json-schema/

### Example

```python
from pydantic import BaseModel, Field
from typing import Literal

class SearchDocsInput(BaseModel):
    query: str = Field(..., description="Search query text")
    top_k: int = Field(5, ge=1, le=50, description="Number of results to return")
    tenant_id: str = Field(..., description="Tenant scope for the search")

class SearchDocsOutput(BaseModel):
    results: list[str]
    latency_ms: float
    status: Literal["ok", "empty", "error"]

# Auto-generate the JSON schema an LLM/tool-caller consumes
print(SearchDocsInput.model_json_schema()
```

In an AI tool system, every tool should have an explicit contract.

```python
from pydantic import BaseModel, Field

class SearchCustomerInput(BaseModel):
    customer_id: str = Field(
        ...,
        description="Unique customer identifier"
    )
```

The description becomes part of the generated JSON Schema and helps the LLM understand how to use the tool. Pydantic supports schema metadata such as:

```python
title=
description=
examples=
json_schema_extra=
```

Pydantic can convert a model directly into JSON Schema.

```python
schema = SearchCustomerInput.model_json_schema()
"""
{
  "type": "object",
  "properties": {
    "customer_id": {
      "type": "string",
      "description": "Unique customer identifier"
    }
  },
  "required": ["customer_id"]
}
"""
```

Pydantic does more than generate docs. It validates values. Validation guarantees: correct types - required fields - ranges - constraints - patterns - enums

Always create output schemas. After tool execution:

```python
result = tool.run(...)
validated = SearchCustomerOutput.model_validate(result)
```

Tool use : a contract between your app and model. You specify the operations + input and output structures; Claude decides when and how to call them (model never executes on its own. model emits a request, ur code/Anthropic server runs the operation)

## Type of tools

### 1. User-defined Tools (client-executed)

- u define the schema and run the code
- claude sends a `tool_use`  block, your app executes and return a `tool_result`
- Ideal for custom logic, internal APIs, proprietary data

### 2. Anthropic-schema tools (client-executed)

- predefined schemas: memory, bash, text_editor, computer
- same execution model as user-defined tools

### 3. Server-executed tools

- web_search, web_fetch, code_execution, tool_research
- anthropic runs the code, u simply enable the tool
- results are fed back automatically unless the loop pauses

## The Agentic Loop

### Client-side loop (foor client-executed tools)

a round-trip cycle:

1. calude calls a tool → `stop_reason: "tool_use"` 
2. u app executes the tool
3. u return `tool_results` 
4. claude continues until a non-tool stop reason appears

### Server-side loop (for server-executed tools)

- Claude iterates internally (searching, fetching, executing)
- if it hits an iteratin cap, you get `pause_turn` and must resend the convo to continue

## When to use tools

- side effects (write files, send emails, update records)
- fresh/external data (current prices, weather, db contents)
- structured outputs (strict JSON)
- integration with existing systems

```python
# Ring 5: The Tool Runner SDK abstraction.

import json

import anthropic
from anthropic import beta_tool

client = anthropic.Anthropic()

@beta_tool
def create_calendar_event(
    title: str,
    start: str,
    end: str,
    attendees: list[str] | None = None,
    recurrence: dict | None = None,
) -> str:
    """Create a calendar event with attendees and optional recurrence.

    Args:
        title: Event title.
        start: Start time in ISO 8601 format.
        end: End time in ISO 8601 format.
        attendees: Email addresses to invite.
        recurrence: Dict with 'frequency' (daily, weekly, monthly) and 'count'.
    """
    if attendees and len(attendees) > 10:
        raise ValueError("Too many attendees (max 10)")
    return json.dumps({"event_id": "evt_123", "status": "created", "title": title})

@beta_tool
def list_calendar_events(date: str) -> str:
    """List all calendar events on a given date.

    Args:
        date: Date in YYYY-MM-DD format.
    """
    return json.dumps({"events": [{"title": "Existing meeting", "start": "14:00", "end": "15:00"}]})

final_message = client.beta.messages.tool_runner(
    model="claude-opus-4-8",
    max_tokens=1024,
    tools=[create_calendar_event, list_calendar_events],
    messages=[
        {
            "role": "user",
            "content": "Check what I have next Monday, then schedule a planning session that avoids any conflicts.",
        }
    ],
).until_done()

for block in final_message.content:
    if block.type == "text":
        print(block.text)
```

## What a Tool Definition Contains

each client-executed tool must include:

- `name` : must match a strict regex (`[a-zA-Z0-9_]{1,64}$`
- description: most important part, must be detailed and explicit
- input_schema: JSON schema defining params
- input_examples: helps Claude understand valid inputs

*API constructs a tool-use system prompt from these definitions to instruct Claude on how to use them

## Best practices for defining tools

1. Write extremely detailed descriptions: what it does, when it is used, when it is not used, what each param means 
2. use input_examples for complex schemas
3. Consolidate related operations
4. use meaningful namespacing: prefix tools with service names (github_list_prs, slack_send_message)
5. return only high-signal info (stable identifiers, fields Claude needs for the next steps)

```python
{
  "name": "get_stock_price",
  "description": "Retrieves the current stock price for a given ticker symbol. The ticker symbol must be a valid symbol for a publicly traded company on a major US stock exchange like NYSE or NASDAQ. The tool will return the latest trade price in USD. It should be used when the user asks about the current or most recent price of a specific stock. It will not provide any other information about the stock or company.",
  "input_schema": {
    "type": "object",
    "properties": {
      "ticker": {
        "type": "string",
        "description": "The stock ticker symbol, e.g. AAPL for Apple Inc."
      }
    },
    "required": ["ticker"]
  }
}
```

## Controlling Tool Use (via tool_choice)

```python
import anthropic

client = anthropic.Anthropic()

tools = [
    {
        "name": "get_weather",
        "description": "Get the current weather in a given location",
        "input_schema": {
            "type": "object",
            "properties": {
                "location": {
                    "type": "string",
                    "description": "The city and state, e.g. San Francisco, CA",
                }
            },
            "required": ["location"],
        },
    }
]

response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    tools=tools,
    tool_choice={"type": "tool", "name": "get_weather"}, #tool_choice controls
    messages=[{"role": "user", "content": "What's the weather like in San Francisco?"}],
)

print(response)
```

Four modes:

- `auto` - Claude decides
- `any` - Claude must use one tool
- `tool`- Claude must use a specific tool
- `none` - Claude cannot use tools

How Claude responds when using Tools

Claude may explain what its doing then emit a `tool_use` block

## Client Tool Call Cycle

when Claude uses a client tool, the assistant message includes:

- `stop_reason: “tool_use”`
- A `tool_use` block containing:
    - id
    - name
    - input - json object matching your tool’s schema

Your system must then:

1. Extract the tool name, id, and input
2. execute the tool in ur own code
3. Reply wit a user message containing:
    1. one or more `tool_results`  blocks
    2. optional text after all tool_restults blocks

## Tool Result Formatting

A `tool_result` block includes:

- `tool_use_id` - must match the og tool call
- `content` - string, nested contain blocks,
- `is_error` - mark the tool execution as failed

```python
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
      "content": "15 degrees"
    }
  ]
}
```

## What Tool Runner automatically handles

- run tools when Claude calls them
- sends tool results back to Claude
- Manages message history
- wraps errors and enforces type safety
- validate tool inputs using JSON schema
- supports multimodal tool outputs (text, image, doc)

## Basic Usage

u define tools using SDK helpers (decorator `@beta_tool`):

- Sends ur initial messages
- iterates thru Claude’s responses
- automatically executes tools when requested
- continues until Claude produces a final non-tool message

you can:

- iterate over each message
- or call `runner.until_done()` to get the final answer directly

```python
import json
from anthropic import Anthropic, beta_tool

client = Anthropic()

@beta_tool
def get_weather(location: str, unit: str = "fahrenheit") -> str:
    """Get the current weather in a given location.

    Args:
        location: The city and state, e.g. San Francisco, CA
        unit: Temperature unit, either 'celsius' or 'fahrenheit'
    """
    return json.dumps({"temperature": "20°C", "condition": "Sunny"})

@beta_tool
def calculate_sum(a: int, b: int) -> str:
    """Add two numbers together.

    Args:
        a: First number
        b: Second number
    """
    return str(a + b)

runner = client.beta.messages.tool_runner(
    model="claude-opus-4-8",
    max_tokens=1024,
    tools=[get_weather, calculate_sum],
    messages=[
        {
            "role": "user",
            "content": "What's the weather like in Paris? Also, what's 15 + 27?",
        }
    ],
)
for message in runner:
    print(message)
```

## Iteration Lifecycle

each loop iteration follows this pattern:

- runner sends current state to Claude
- Claude responds
- ur code can inspect or modify the response
- runner decides whether to:
    - append the message automatically
    - run tools
    - exit the loop
    - or use ur custom-mod state

## Strict Tool Use

Strict tool use enforces guaranteed JSON Schema compliance for tool inputs by using grammar-constrained sampling

Why it matters?

solves the biggest reliability issues in agentic systems:

- prevent type mismatches (e.g: “2” vs 2)
- ensures required fields are alwasy present
- eliminattes invalid enum values
- guarantees tool names are valid
- removes the need for manual validation + retry loops

## How Strict mode work

1. u define a JSON Schema for the tool’s `input_schema` 
2. add `"strict": true` at the top level of the tool definition
3. claude’s token sampling is constrained so it cannot output invalid params
4. Tool calls always match the schema exactly

```python
client = Anthropic()
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": "Help me plan a trip from New York to Paris for 2 people, departing June 1, 2026",
        }
    ],
    tools=[
        {
            "name": "search_flights",
            "strict": True, #strict mode
            "input_schema": {
                "type": "object",
                "properties": {
                    "origin": {"type": "string"},
                    "destination": {"type": "string"},
                    "departure_date": {"type": "string", "format": "date"},
                    "travelers": {"type": "integer", "enum": [1, 2, 3, 4, 5, 6]},
                },
                "required": ["origin", "destination", "departure_date"],
                "additionalProperties": False,
            },
        },
        {
            "name": "search_hotels",
            "strict": True,
            "input_schema": {
                "type": "object",
                "properties": {
                    "city": {"type": "string"},
                    "check_in": {"type": "string", "format": "date"},
                    "guests": {"type": "integer", "enum": [1, 2, 3, 4]},
                },
                "required": ["city", "check_in"],
                "additionalProperties": False,
            },
        },
    ],
)

print(response)
```

---

## Phase 2 — Centralized Tool Registry with Read/Write Segregation

**Goal:** A single registry object is the only way tools get discovered and invoked — no ad-hoc function calls scattered through the codebase.

### What to learn

- Registry pattern (decorator-based registration, e.g. `@registry.register(...)`)
- Modeling a tool as metadata + Pydantic schemas + a callable, not just a bare function
- Tagging tools by **capability class**: `READ` vs `WRITE` (and optionally `WRITE_DESTRUCTIVE`)
- Why this split matters: read tools can be auto-approved/looped freely by an agent; write tools need confirmation, rate limits, or human-in-the-loop gates
- Design references: LangChain's `Tool`/`StructuredTool` abstraction, Anthropic's tool-use loop, MCP's resource vs tool distinction

### Sources

- LangChain tools & toolkits concept docs: https://python.langchain.com/docs/concepts/tools/
- Model Context Protocol (MCP) spec — Tools section: https://modelcontextprotocol.io/docs/concepts/tools
- Anthropic — building effective agents (tool design section): https://www.anthropic.com/research/building-effective-agents

### Example

```python
from enum import Enum
from dataclasses import dataclass
from typing import Callable, Type
from pydantic import BaseModel

class ToolKind(str, Enum):
    READ = "read"
    WRITE = "write"

@dataclass
class ToolSpec:
    name: str
    kind: ToolKind
    input_model: Type[BaseModel]
    output_model: Type[BaseModel]
    fn: Callable
    required_scope: str  # e.g. "docs:read", "crm:write"

class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, ToolSpec] = {}

    def register(self, spec: ToolSpec):
        if spec.name in self._tools:
            raise ValueError(f"Tool '{spec.name}' already registered")
        self._tools[spec.name] = spec

    def read_tools(self) -> list[ToolSpec]:
        return [t for t in self._tools.values() if t.kind == ToolKind.READ]

    def write_tools(self) -> list[ToolSpec]:
        return [t for t in self._tools.values() if t.kind == ToolKind.WRITE]

    def get(self, name: str) -> ToolSpec:
        return self._tools[name]

registry = ToolRegistry()

registry.register(ToolSpec(
    name="search_docs",
    kind=ToolKind.READ,
    input_model=SearchDocsInput,
    output_model=SearchDocsOutput,
    fn=lambda i: SearchDocsOutput(results=["doc1"], latency_ms=12.4, status="ok"),
    required_scope="docs:read",
))
```

Tool Registry

central directory of all tools an AI agent can use

Without a registry:

```python
Agent
  ├─ search_customer()
  ├─ update_customer()
  ├─ send_email()
  └─ create_order()
```

The agent has no centralized way to know:

- What tools exist
- Their schemas
- Required permissions
- Whether they're read/write tools
- How to validate inputs/outputs

With a registry:

```python
Agent
   ↓
Tool Registry
   ├─ search_customer
   ├─ update_customer
   ├─ send_email
   └─ create_order
```

---

## Phase 3 — Explicit Access Parameters & Permission-Scoped Enforcement

**Goal:** Access control is enforced in code at call time, not assumed by convention.

### What to learn

- Scope/permission modeling: caller identity → allowed scopes (e.g. `{"user_role": "analyst", "scopes": ["docs:read"]}`)
- Enforcing the check inside a single `execute()` chokepoint (never call `spec.fn` directly elsewhere)
- Difference between **authentication** (who is calling) and **authorization** (what they're allowed to do) — keep them as separate, composable checks
- Patterns: RBAC (role-based) vs ABAC (attribute-based) access control, and when the extra complexity of ABAC is worth it
- Raising typed, catchable permission errors rather than silent no-ops

### Sources

- OWASP Authorization Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html
- NIST RBAC overview (concept reference, skim): https://csrc.nist.gov/projects/role-based-access-control
- Anthropic MCP — Authorization spec (good reference for scoped tool access): https://modelcontextprotocol.io/docs/concepts/architecture

### Example

```python
class PermissionDeniedError(Exception):
    pass

@dataclass
class CallerContext:
    caller_id: str
    scopes: set[str]

def execute(tool_name: str, raw_input: dict, ctx: CallerContext):
    spec = registry.get(tool_name)

    if spec.required_scope not in ctx.scopes:
        raise PermissionDeniedError(
            f"Caller '{ctx.caller_id}' lacks scope '{spec.required_scope}' "
            f"for tool '{tool_name}'"
        )

    validated_input = spec.input_model.model_validate(raw_input)
    result = spec.fn(validated_input)
    return spec.output_model.model_validate(result)

# Read caller — allowed
ctx_analyst = CallerContext(caller_id="hoang", scopes={"docs:read"})
execute("search_docs", {"query": "vinyl inventory", "tenant_id": "t1"}, ctx_analyst)

# Same caller tries a write tool — denied
# execute("update_crm_record", {...}, ctx_analyst)  -> PermissionDeniedError
```

### Proof-of-skill

- Implement `CallerContext` + scope enforcement.
- pytest matrix: (read tool × read scope) passes, (write tool × read-only scope) raises `PermissionDeniedError`, (unknown tool) raises a clean `KeyError`/custom `ToolNotFoundError`.

---

## Phase 4 — Monitoring & Tracking Metrics

**Goal:** Every call is observable: who called what, was it permitted, how long it took, did it succeed — feeding into metrics usable by a real internal application (dashboards, alerting, audit logs).

### What to learn

- Structured logging (JSON logs) vs print-debugging
- Basic metrics concepts: counters (calls total, denials total), histograms (latency), labels/tags (tool name, kind, caller, scope)
- Minimal viable observability stack for a side project: Python `logging` + `time.perf_counter` + an in-memory or SQLite metrics store before reaching for Prometheus/Grafana
- How this maps to production tools (Prometheus client, OpenTelemetry) so you can name-drop and later upgrade

### Sources

- Python `logging` cookbook: https://docs.python.org/3/howto/logging-cookbook.html
- Prometheus client (Python) — for when you outgrow the toy version: https://github.com/prometheus/client_python
- OpenTelemetry Python getting started (concept only, optional): https://opentelemetry.io/docs/languages/python/getting-started/

### Example

```python
import time, logging, json

logger = logging.getLogger("tool_registry")

class Metrics:
    def __init__(self):
        self.calls_total = 0
        self.denials_total = 0
        self.errors_total = 0
        self.latency_ms: list[float] = []

    def record(self, tool_name, kind, caller_id, scope, ok, denied, duration_ms):
        self.calls_total += 1
        if denied:
            self.denials_total += 1
        if not ok and not denied:
            self.errors_total += 1
        self.latency_ms.append(duration_ms)
        logger.info(json.dumps({
            "tool": tool_name, "kind": kind, "caller": caller_id,
            "scope": scope, "ok": ok, "denied": denied,
            "duration_ms": round(duration_ms, 2),
        }))

metrics = Metrics()

def monitored_execute(tool_name: str, raw_input: dict, ctx: CallerContext):
    spec = registry.get(tool_name)
    start = time.perf_counter()
    denied, ok = False, False
    try:
        result = execute(tool_name, raw_input, ctx)
        ok = True
        return result
    except PermissionDeniedError:
        denied = True
        raise
    finally:
        duration_ms = (time.perf_counter() - start) * 1000
        metrics.record(tool_name, spec.kind, ctx.caller_id,
                        spec.required_scope, ok, denied, duration_ms)
```

### Proof-of-skill

- Run 20+ mixed calls (read/write, allowed/denied) through `monitored_execute`.
- Produce a small summary report: total calls, denial rate, p50/p95 latency per tool. This is your "tracking metrics" evidence.

---

## Phase 5 — Integration: The Actual Deliverable

**Goal:** Combine Phases 1–4 into a small internal-app-ready package.

### Build checklist

1. `models.py` — Pydantic input/output models for 6–8 real tools (mix of read/write; reuse ideas from your RAG pipeline: `search_docs` [read], `rerank_results` [read], `ingest_document` [write], `delete_document` [write], `update_tenant_config` [write]).
2. `registry.py` — `ToolRegistry` with `register`, `read_tools()`, `write_tools()`, `get()`.
3. `access.py` — `CallerContext`, scope enforcement, `PermissionDeniedError`.
4. `execution.py` — `execute()` and `monitored_execute()` chokepoints; **no code outside this module ever calls `spec.fn` directly**.
5. `metrics.py` — counters/latency store, a `summary()` method returning a dict suitable for a dashboard or JSON API endpoint.
6. `tests/` — pytest suite covering: schema validation failures, duplicate registration, permission denial matrix, metrics accuracy.
7. `README.md` — architecture diagram (even ASCII) showing: LLM/agent → registry lookup → permission check → execution → metrics/log sink.

### Stretch goals (if time allows)

- Wire this registry into your Anthropic/OpenAI tool-calling loop so the LLM only ever sees `input_schema` JSON generated from your Pydantic models.
- Add a `WRITE_DESTRUCTIVE` tier requiring an extra human-confirmation callback before execution.
- Expose `metrics.summary()` via a tiny FastAPI endpoint — turns this into an actual "monitored service," not just a library.

---

## Suggested Timeline

| Phase | Focus | Est. time |
| --- | --- | --- |
| 1 | Pydantic + JSON Schema | 1–2 days |
| 2 | Registry + read/write segregation | 2 days |
| 3 | Access control enforcement | 2 days |
| 4 | Monitoring & metrics | 1–2 days |
| 5 | Integration + tests + README | 2–3 days |

**Total: ~1.5–2 weeks** at a steady pace, fits well as the next proof-of-skill deliverable alongside your existing RAG pipeline repo.

---

## Resume-Ready Framing (once complete)

> Designed structured tool I/O specs (Pydantic/JSON Schema) and built a centralized tool registry enforcing strict read/write segregation with scope-based access control; instrumented with call-level monitoring producing latency and permission-denial metrics for internal agentic workflows.
>