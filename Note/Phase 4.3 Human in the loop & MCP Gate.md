# ✅ **1. MCP Overview — Source**

This covers:

- What MCP is
- Why it exists
- Clients vs. servers
- Tools, sessions, messages

MCP (Model Context Protocol) is an open-source **standard** for connecting *AI applications to external systems*.

thru MCP, AI app like Claude/ChatGPT can connect to data sources (db, files), tools (search engines, calculators) and workflow (e.g. specialized prompts) →  enabling them to access key info and perform tasks

!image.png

# ✅ **2. Tools & Servers — Source**

**Sources:**

**Tool Definition**`https://modelcontextprotocol.org/docs/concepts/tools` (modelcontextprotocol.org in Bing)

**Server Capabilities**`https://modelcontextprotocol.org/docs/concepts/capabilities` (modelcontextprotocol.org in Bing)

**Tool Calls**`https://modelcontextprotocol.org/docs/concepts/tool-calls` (modelcontextprotocol.org in Bing)

These explain:

- How servers expose tools
- How clients discover them
- How the model calls tools
- How results flow back

The Model Context Protocol (MCP) allows servers to expose tools that can be invoked by language models

## User Interaction Model

tools in MCP are model-controlled → LM can discover and invoke tools automatically based on its contextual understanding and the user’s prompts

## Capabilities

servers that support tools MUST declare the `tools` capability:

```python
{
  "capabilities": {
    "tools": {
      "listChanged": true
    }
  }
}
```

`listChanged` means the server can notify clients when tools are added or removed.

## Three main operations

### Listing Tools

client discovers tools via:

```python
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list"
}

#example response
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "get_weather",
        "title": "Weather Information Provider",
        "description": "Get current weather information",
        "inputSchema": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string"
            }
          },
          "required": ["location"]
        }
      }
    ]
  }
}
```

### Call Tool

client sends `tools/call`  with tool name, and atguments. Server executes the tool and returns results

```python
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "location": "New York"
    }
  }
}
```

### List Changed Notification

if tools are added or removed, server can send `notifications/tools/list_changed`

```python
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
```

## Tool Definition

Field

name

title

description

inputSchema

outputSchema

annotations

execution

Purpose

unique identifier

human-readable name

tool explanation

expected inputs

expected outputs

tool behavior metadata

execution settings

### Input Schema

Tools use JSON Schema to define: required params - types - validation rules

```python
{
  "type": "object",
  "properties": {
    "location": {
      "type": "string"
    }
  },
  "required": ["location"]
}
```

### Output Schema

A tool can also define the structure of its response.

```python
{
  "type": "object",
  "properties": {
    "temperature": {
      "type": "number"
    },
    "humidity": {
      "type": "number"
    }
  }
}
```

### **Error Handling**

MCP defines two different error categories.

#### **Protocol Errors**

Something is wrong with the request itself. ( unknown tool, invalid JSON-RPC request, missing required fields)

```python
{
  "error": {
    "code": -32602,
    "message": "Unknown tool"
  }
}
```1

---

## Tool Execution Errors

The tool executed but encountered a business problem.

Examples:

- Invalid dates
- API failure
- Validation failure

```json
{
  "result": {
    "isError": true
  }
}

```
# MCP

# ✅ **1. MCP Overview — Source**

This covers:

- What MCP is
- Why it exists
- Clients vs. servers
- Tools, sessions, messages

MCP (Model Context Protocol) is an open-source **standard** for connecting *AI applications to external systems*.

thru MCP, AI app like Claude/ChatGPT can connect to data sources (db, files), tools (search engines, calculators) and workflow (e.g. specialized prompts) →  enabling them to access key info and perform tasks

!image.png

# ✅ **2. Tools & Servers — Source**

**Sources:**

**Tool Definition**`https://modelcontextprotocol.org/docs/concepts/tools` (modelcontextprotocol.org in Bing)

**Server Capabilities**`https://modelcontextprotocol.org/docs/concepts/capabilities` (modelcontextprotocol.org in Bing)

**Tool Calls**`https://modelcontextprotocol.org/docs/concepts/tool-calls` (modelcontextprotocol.org in Bing)

These explain:

- How servers expose tools
- How clients discover them
- How the model calls tools
- How results flow back

The Model Context Protocol (MCP) allows servers to expose tools that can be invoked by language models

## User Interaction Model

tools in MCP are model-controlled → LM can discover and invoke tools automatically based on its contextual understanding and the user’s prompts

## Capabilities

servers that support tools MUST declare the `tools` capability:

```python
{
  "capabilities": {
    "tools": {
      "listChanged": true
    }
  }
}
```

`listChanged` means the server can notify clients when tools are added or removed.

## Three main operations

### Listing Tools

client discovers tools via:

```python
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list"
}

#example response
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "get_weather",
        "title": "Weather Information Provider",
        "description": "Get current weather information",
        "inputSchema": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string"
            }
          },
          "required": ["location"]
        }
      }
    ]
  }
}
```

### Call Tool

client sends `tools/call`  with tool name, and atguments. Server executes the tool and returns results

```python
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "location": "New York"
    }
  }
}
```

### List Changed Notification

if tools are added or removed, server can send `notifications/tools/list_changed`

```python
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
```

## Tool Definition

Field

name

title

description

inputSchema

outputSchema

annotations

execution

Purpose

unique identifier

human-readable name

tool explanation

expected inputs

expected outputs

tool behavior metadata

execution settings

### Input Schema

Tools use JSON Schema to define: required params - types - validation rules

```python
{
  "type": "object",
  "properties": {
    "location": {
      "type": "string"
    }
  },
  "required": ["location"]
}
```

### Output Schema

A tool can also define the structure of its response.

```python
{
  "type": "object",
  "properties": {
    "temperature": {
      "type": "number"
    },
    "humidity": {
      "type": "number"
    }
  }
}
```

### **Error Handling**

MCP defines two different error categories.

#### **Protocol Errors**

Something is wrong with the request itself. ( unknown tool, invalid JSON-RPC request, missing required fields)

```python
{
  "error": {
    "code": -32602,
    "message": "Unknown tool"
  }
}
```1

---

## Tool Execution Errors

The tool executed but encountered a business problem.

Examples:

- Invalid dates
- API failure
- Validation failure

```json
{
  "result": {
    "isError": true
  }
}

```

# ✅ **3. Permissions — Source**

**Sources:**

**Permissions Overview**`https://modelcontextprotocol.org/docs/concepts/permissions` (modelcontextprotocol.org in Bing)

**Permission Modes**`https://modelcontextprotocol.org/docs/concepts/permissions#permission-modes` (modelcontextprotocol.org in Bing)

**Allow / Deny / Ask Rules**

`https://modelcontextprotocol.org/docs/concepts/permissions#rules` (modelcontextprotocol.org in Bing)

**canUseTool Callback**`https://modelcontextprotocol.org/docs/concepts/permissions#canusetool` (modelcontextprotocol.org in Bing)

These cover:

- Permission modes
- Tool approval logic
- Human‑in‑the‑loop
- Draft → Approve → Execute

# ✅ **4. Elicitation — Source**

**Source:Elicitation**`https://modelcontextprotocol.org/docs/concepts/elicitation` (modelcontextprotocol.org in Bing)

This covers:

- Form mode
- URL mode
- Secure user input
- Client responsibilities

# **5. Tool Exposure Annotations — Source**

**Source:Tool Annotations**`https://modelcontextprotocol.org/docs/concepts/tool-annotations` (modelcontextprotocol.org in Bing)

This covers:

- readOnlyHint
- destructiveHint
- idempotentHint
- openWorldHint

# ✅ **6. MCP Client Specification — Source**

**Source:Client Spec**`https://modelcontextprotocol.org/docs/specification/client` (modelcontextprotocol.org in Bing)

This covers:

- How clients display tools
- How they show drafts
- How they handle approvals
- How they manage sessions

# ✅ **7. MCP Server Specification — Source**

**Source:Server Spec**`https://modelcontextprotocol.org/docs/specification/server` (modelcontextprotocol.org in Bing)

This covers:

- How servers expose tools
- How they send annotations
- How they request elicitation
- How they manage state
- 

# Learning Plan: Draft-Approve-Execute Workflow & Human-in-the-Loop Control Gate

## Objective

Build the skills and a working deliverable that demonstrates:

1. A **structured draft-approve-execute workflow** for risky (write/side-effect) operations: a tool call is first *drafted* (proposed, not run), surfaced for **explicit human approval**, and only *executed* after that approval is granted — never before.
2. Exploration of **MCP (Model Context Protocol) tool exposure and consumption patterns** — how a server exposes tools with risk metadata, and how a client/host consumes them safely, including where MCP already has (or lacks) primitives for human-in-the-loop gating.
3. A **human-in-the-loop interactive control gate**: a chokepoint that no write action can bypass, presenting the caller with what's about to happen and blocking execution until a human explicitly approves, denies, or edits the proposed action.

**End deliverable:** A draft-approve-execute layer sitting in front of your existing tool registry's `WRITE` tools, so that `ingest_document` (or any future write tool) requires human sign-off before it actually runs — with drafts, approvals, denials, and edits all visible in your existing metrics.

---

## Phase 1 — The Draft-Approve-Execute Pattern

**Goal:** Understand why "ask permission before every write" is a distinct pattern from "check permission before every write" (which you already have), and where each one applies.

### What to learn

- The difference between **authorization** (does this caller have the scope?) and **approval** (should this *specific* action happen, right now, given its actual arguments?). Scope answers "can this caller ever ingest documents" — approval answers "should *this* document, with *this* content, be ingested *right now*."
- Why this needs three distinct states, not two: `DRAFTED` (proposed, nothing has happened yet), `APPROVED`/`DENIED` (human decision recorded), `EXECUTED`/`FAILED` (the actual side effect, only reachable from `APPROVED`).
- Idempotency and staleness: what happens if the underlying data changes between draft and approval? A draft should carry enough context (and ideally a freshness check) that a stale approval doesn't execute against outdated assumptions.
- Prior art: CI/CD deployment approval gates, database migration review workflows, and "confirm before send" patterns in email/chat clients are all the same shape applied to different domains.

### Sources

- Anthropic — Claude Agent SDK permission modes / human-in-the-loop docs: https://docs.claude.com/en/docs/claude-code/sdk (concept reference — approval/permission callback patterns)
- Martin Fowler — Two-Phase Commit / Saga pattern overview (for the state-machine framing, not the distributed-transactions part): https://martinfowler.com/articles/patterns-of-distributed-systems/two-phase-commit.html
- OWASP — Business Logic Testing / approval workflow tampering considerations: https://cheatsheetseries.owasp.org/cheatsheets/Business_Logic_Cheat_Sheet.html

### Example

```python
from enum import Enum
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from uuid import uuid4

class DraftStatus(str, Enum):
    DRAFTED = "drafted"
    APPROVED = "approved"
    DENIED = "denied"
    EXECUTED = "executed"
    EXPIRED = "expired"

@dataclass
class ActionDraft:
    draft_id: str
    tool_name: str
    args: dict
    caller_id: str
    status: DraftStatus = DraftStatus.DRAFTED
    created_at: datetime = field(default_factory=datetime.utcnow)
    approved_by: str | None = None
    ttl: timedelta = timedelta(minutes=15)

    def is_stale(self) -> bool:
        return datetime.utcnow() - self.created_at > self.ttl
```

# **1. How the agent loop works**

## ⭐ Core idea

The Agent SDK runs a **continuous loop** where Claude evaluates the current state, proposes actions (tool calls), receives results, and repeats until it produces a final text-only answer. Each cycle is a **turn**.

## 🔄 The agent loop, step by step

### 1. **Receive prompt**

Claude gets:

- your prompt
- system prompt
- tool definitions
- conversation history

The SDK emits a **SystemMessage (init)** with session metadata.

### 2. **Evaluate and respond**

Claude decides what to do next:

- produce text
- request tool calls
- or both

The SDK emits an **AssistantMessage** containing text + tool call blocks.

### 3. **Execute tools**

The SDK:

- runs each requested tool
- collects results
- sends results back to Claude

You can intercept or block tool calls using **hooks**.

A **UserMessage** is emitted with the tool results.

### 4. **Repeat**

Claude uses the new state + tool results to decide the next action.
This continues until Claude produces a response **with no tool calls**.

### 5. **Return final result**

The SDK emits:

- a final **AssistantMessage** (text-only)
- a **ResultMessage** with final text, cost, usage, and session ID

## 🔁 What counts as a “turn”

A **turn** = Claude proposes tool calls → SDK executes them → results go back to Claude.
Turns continue until Claude stops calling tools.

You can limit the loop with:

- **max_turns**
- **max_budget_usd**

## 🧱 Message types (the loop’s building blocks)

- **SystemMessage** — lifecycle events (init, compaction, shutdown)
- **AssistantMessage** — Claude’s text + tool calls
- **UserMessage** — tool results or user input
- **StreamEvent** — partial streaming chunks (optional)
- **ResultMessage** — final output + cost + session ID

## 🛠 Tool execution model

Tools give Claude the ability to act (read files, edit code, run commands, fetch web pages, etc.).

Key behaviors:

- Read-only tools can run **in parallel**
- State-modifying tools run **sequentially**
- Permissions determine whether tools auto-run, require approval, or are blocked

## 🧠 Context window behavior

Everything accumulates across turns:

- system prompt
- tool definitions
- conversation history
- tool inputs/outputs

When near the limit, the SDK **automatically compacts** older messages into summaries.

## 🔌 Hooks (control points in the loop)

Hooks let you intercept or modify behavior:

- **PreToolUse** — validate or block tool calls
- **PostToolUse** — audit outputs
- **UserPromptSubmit** — inject context
- **Stop** — run logic when the agent finishes
- **PreCompact** — act before compaction
- **SubagentStart / SubagentStop** — track subagent activity

# **Handle approvals and user input**

Claude somtimes needs user approval or clarifying input while executing tasks. Your app must detect these moments, surface the questions to users, and return their decisions so Claude can continue

## When Claude requests input

CLaude pauses execution and triggers your `canUseTool` callback in 2 situations:

- Tool approval needed: claude wants to use a tool this isnt auto-approved
- clarifying questions: uses `AskUserQuestions` tool to ask multiple-choice questions

```python
async def handle_tool_request(tool_name, input, context):
 ...
 
options = ClaudeAgentOptions(can_use_tool=handle_tool_request)
```

## Detecting requests

u provide a `canUseTool` callback in ur agent options. It receives: 

- `toolName` - the tool Claude wants to use
- `input` - tool params
- `context` - extra info like suggestions or cancellation signals

*If `toolName=="AskUserQuestions"` , handle it as a clarifying questions

```python
import asyncio

from claude_agent_sdk import ClaudeAgentOptions, ResultMessage, query
from claude_agent_sdk.types import (
    HookMatcher,
    PermissionResultAllow,
    PermissionResultDeny,
    ToolPermissionContext,
)

async def can_use_tool(
    tool_name: str, input_data: dict, context: ToolPermissionContext
) -> PermissionResultAllow | PermissionResultDeny:
    # Display the tool request
    print(f"\nTool: {tool_name}")
    if tool_name == "Bash":
        print(f"Command: {input_data.get('command')}")
        if input_data.get("description"):
            print(f"Description: {input_data.get('description')}")
    else:
        print(f"Input: {input_data}")

    # Get user approval
    response = input("Allow this action? (y/n): ")

    # Return allow or deny based on user's response
    if response.lower() == "y":
        # Allow: tool executes with the original (or modified) input
        return PermissionResultAllow(updated_input=input_data)
    else:
        # Deny: tool doesn't execute, Claude sees the message
        return PermissionResultDeny(message="User denied this action")

# Required workaround: dummy hook keeps the stream open for can_use_tool
async def dummy_hook(input_data, tool_use_id, context):
    return {"continue_": True}

async def prompt_stream():
    yield {
        "type": "user",
        "message": {
            "role": "user",
            "content": "Create a test file in /tmp and then delete it",
        },
    }

async def main():
    async for message in query(
        prompt=prompt_stream(),
        options=ClaudeAgentOptions(
            can_use_tool=can_use_tool,
            hooks={"PreToolUse": [HookMatcher(matcher=None, hooks=[dummy_hook])]},
        ),
    ):
        if isinstance(message, ResultMessage) and message.subtype == "success":
            print(message.result)

asyncio.run(main())
```

## Handling Tool Approvals

ur callback can:

- allow/modify/deny  the tool
- approve and remember
- suggest alt
- redirect entirely

(u display the tool request to the user, collect their decision, and return the appropriate response object)

```python
async def can_use_tool(tool_name, input_data, context):
    print(f"Claude wants to use {tool_name}")
    approved = await ask_user("Allow this action?")

    if approved:
        return PermissionResultAllow(updated_input=input_data)
    return PermissionResultDeny(message="User declined")
    
    
async def can_use_tool(tool_name, input_data, context):
    if tool_name == "Bash":
        # User approved, but scope all commands to sandbox
        sandboxed_input = {**input_data}
        sandboxed_input["command"] = input_data["command"].replace(
            "/tmp", "/tmp/sandbox"
        )
        return PermissionResultAllow(updated_input=sandboxed_input)
    return PermissionResultAllow(updated_input=input_data)
    
async def can_use_tool(tool_name, input_data, context):
    if tool_name == "Bash" and "rm" in input_data.get("command", ""):
        # User doesn't want to delete, suggest archiving instead
        return PermissionResultDeny(
            message="User doesn't want to delete files. They asked if you could compress them into an archive instead."
        )
    return PermissionResultAllow(updated_input=input_data)
```

## Handling Clarifying Questions

when Claude needs guidance:

1. Include `AskUserQuestion` in ur tools list if u restrict tools
2. Detect `toolName== "AskUserQuestion` 

```python
async def can_use_tool(tool_name: str, input_data: dict, context):
    if tool_name == "AskUserQuestion":
        # Your implementation to collect answers from the user
        return await handle_clarifying_questions(input_data)
    # Handle other tools normally
    return await prompt_for_approval(tool_name, input_data)
```

1. Parse the `questions` array:
    1. each question has text, header, options, and whether multi-select is allowed

```python
{
  "questions": [
    {
      "question": "How should I format the output?",
      "header": "Format",
      "options": [
        { "label": "Summary", "description": "Brief overview" },
        { "label": "Detailed", "description": "Full explanation" }
      ],
      "multiSelect": false
    },
    {
      "question": "Which sections should I include?",
      "header": "Sections",
      "options": [
        { "label": "Introduction", "description": "Opening context" },
        { "label": "Conclusion", "description": "Final summary" }
      ],
      "multiSelect": true
    }
  ]
}
```

1. Present questions to the user and collect answers
2. return an object containing:
    1. the og questions
    2. an answer mapping from question text → selected option

```python
return PermissionResultAllow(
    updated_input={
        "questions": input_data.get("questions", []),
        "answers": {
            "How should I format the output?": "Summary",
            "Which sections should I include?": ["Introduction", "Conclusion"],
        },
    }
)
```

# **Configure permissions**

## Permission Eval Order

Claude checks permissions:

1. Hooks: run first. Can allow, deny, or modify the tool call
2. Deny rules: bare-name deny (”Bash” removes the tool entirely), scoped deny (`Bash(rm *)` blocks only matching calls
3. Ask rules: if matched, the call goes to `canUseTool` /Tools requiring user interaction
4. Permission mode: determines global behavior (e.g: auto-approve, deny prompts, planning-only)
5. Allow rules: auto-approve matching calls
6. `canUseTool` callback: final fallback unless in `dontAsk` mode

## Important Behaviors

- auto-aproved tools skip `canUseTool` , except those requiring user interaction
- Bare allow rules (e.g:”Read”): approve all calls to that tool
- Scoped allow rules (`Bash(ls *)`) approve only matching calls
- Use hooks if u need checks on every tool call

## Allow & Deny rules

### Allow rules (`allowed_tools` )

approve tools auto: `allowed_tools: ["Read", "Grep"]` 

### Deny rules

remove or block tools: `disallowed_tools: ["Bash"]` `disallowed_tools: ["Bash(rm *)"]` `disallowed_tools: [*]` 

*Glob rules: `"*"`* matches all tool / `"mcp__*` matches all MCP tools

## Permission Modes

```python
| Mode | Behavior |
| --- | --- |
| **default** | Standard behavior; unresolved calls go to ``canUseTool``. |
| **dontAsk** | Deny instead of prompting; ``canUseTool`` never called. |
| **acceptEdits** | Auto-approve file edits + filesystem ops. |
| **bypassPermissions** | Approve everything (except deny/ask rules). |
| **plan** | Read-only planning; edits always prompt. |
| **auto** | Model classifier decides allow/deny. |
```

- Subagent inheritance: modes like `bypassPermission` , `acceptEdits`, and `auto` propagate to subagents

## Setting Permission Mode

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    async for message in query(
        prompt="Help me refactor this code",
        options=ClaudeAgentOptions(
            permission_mode="default",
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)

asyncio.run(main())
```

## Mode Details

**acceptEdits**: auto-approves: `Edit` , `Write`, `mkdir`, `rm`, `mv`, `cp`, `sed`  → only inside working dir

**dontAsk**: anything not pre-approved is denied/ No `canUseTool` calls

**bypassPermissions**: approves everything except deny/ask rules - Use with extreme caution

**plan**: Claude explores codebase w/o editing -  All edits prompt via `canUseTool` 

Declarative Rules

u can also configure allow/deny/ask rules in: `.claude/settings.json` - make sure `setting_sources` includes `"project"` 

# **Intercept and control agent behavior with hooks**

hooks are **callback functions** that let u intercept, observe, modify, or block agent behavior at key executions points. They run when events occur (a tool being called, a tool finishing, a subagent starting, or the agent stopping.

Hooks give you fine-grained control to:

- block dangerous operations (e.g: destructive Bash cmd)
- log or audit tool  usage
- modify tool I/O
- Inject context or credentials
- require human approval
- track session cycle

## How Hooks Work

1. Event fires: `PreToolUse`, `PostToolUse`, `SubagentStart` , `Stop`
2. SDK collects hooks: include hooks from `options.hooks`  and setting files
3. Matchers filter hooks: pattern like `"Write|Edit"` or regex `^mcp__` determine which hooks run
4. Callbacks execute: receives event details: tool name, input, session ID
5. Callback returns a decision: deny, allow, ask, defer, modify i/o, inject context, log async

## Available Hook Types

**PreToolUse** (intercept tool calls) - **PostToolUse**/**PostToolUseFailure** (inspect results or errors) - **Notification** (fwd agent noti) - **SubagentStart**/**SubagentStop** (track parallel tasks) - **Stop** (cleanup on exit) - **UserPromptSubmit** (intercept user prompts) - **PermissionRequest** (custom permission handling) - **PreCompact** (archive transcript before compaction)

## Matchers

deterime when hooks fire

- exact match: `"Write|Edit"`
- Wildcard: `"*"`
- Regex: `"^mcp__"`
- No matcher: runs for every event

tool names follow patterns like: 

- built-ins: `Bash` , `Read`, `Write`, `Edit` , `Glob`, `Grep`
- MCP tools: `mcp__<server>__<action>`

## Hook Callback Inputs

every callback receives:

- `input_data` - event details (tool name, tool_input, message)
- `tool_use_id` - correlates pre/post events
- `context` - includes `AbortSignal in T.S

## Hook Callback Outputs

top-level fields:

- `systemMessage` - message shown to user
- `continue` - whether agent continues

event-specific fields (inside `hookSpecificOutput`):

- `permissionDecision`: `"allow"|"deny"|"ask"|"defer"`
- `permissionDecisionReason`
- `updatedInput`
- `updatedToolOutput`
- `additionalContext`
- priority: deny > defer > ask > allow

## Async Hooks

you can return `{ "async_": True, "asyncTimeout": 30000 }` 

→ agent continue immediately while ur hook logs or sends webhooks in the background

## Examples

### Block `.env` file edits

```python
async def protect_env_files(input_data, tool_use_id, context):
    file_path = input_data["tool_input"].get("file_path", "")
    if file_path.endswith(".env"):
        return {
            "hookSpecificOutput": {
                "hookEventName": input_data["hook_event_name"],
                "permissionDecision": "deny",
                "permissionDecisionReason": "Cannot modify .env files",
            }
        }
    return {}
```

### Redirect all writes to `/sandbox`

```python
async def redirect_to_sandbox(input_data, tool_use_id, context):
    if input_data["tool_name"] == "Write":
        original = input_data["tool_input"]["file_path"]
        return {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "allow",
                "updatedInput": {
                    **input_data["tool_input"],
                    "file_path": f"/sandbox{original}",
                },
            }
        }
    return {}

```

Auto-approved read-only tools

```python
async def auto_approve_read_only(input_data, tool_use_id, context):
    if input_data["tool_name"] in ["Read", "Glob", "Grep"]:
        return {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "allow",
                "permissionDecisionReason": "Read-only tool auto-approved",
            }
        }
    return {}
```

Forward noti to Slack

```python
async def notification_handler(input_data, tool_use_id, context):
    await asyncio.to_thread(_send_slack_notification, input_data.get("message", ""))
    return {}
```

Track subagent completion

```python
async def subagent_tracker(input_data, tool_use_id, context):
    print(f"[SUBAGENT] Completed: {input_data['agent_id']}")
    print(f"Transcript: {input_data['agent_transcript_path']}")
    return {}
```

Common pitfalls

- Hook not firing → wrong matcher or event name
- modified input ignored → must include `permissionDecision: "allow"`
- Unexpected blocking → another hook returned `"deny"`
- session hooks missing in Python → available via settings files

---

## Phase 2 — The Human-in-the-Loop Control Gate

**Goal:** Build the actual chokepoint: nothing marked `WRITE` in your registry can reach `spec.fn` without passing through an approval check first.

### What to learn

- Where the gate belongs in the call chain: after validation (no point asking a human to approve malformed arguments) but before resilient execution (no point retrying something that was never approved).
- Presenting the draft for approval: what a human reviewer actually needs to see to make a good decision — the tool name, the fully validated arguments (not raw JSON), a plain-language description of the effect, and what tenant/scope it's running under.
- Synchronous vs asynchronous approval: a CLI/chat context can block and wait for a human's next message; a production system usually can't block a thread indefinitely and instead needs a callback/webhook/polling model. Understand both, build the simpler synchronous one first.
- Denial and edit paths: approval isn't just yes/no — a good gate lets a human edit the arguments before approving (e.g. correcting a wrong tenant_id) rather than forcing a full reject-and-redraft cycle.

### Sources

- Anthropic — tool use + human confirmation patterns in agent loops: https://docs.claude.com/en/docs/agents-and-tools/tool-use/implement-tool-use
- Anthropic Cookbook — human-in-the-loop tool use examples (search the cookbook repo for "human in the loop"): https://github.com/anthropics/anthropic-cookbook
- Slack's "confirm" block pattern (a well-known UX reference for approve/deny/edit surfaces, read for the interaction pattern not the API): https://api.slack.com/surfaces/modals

### Example

```python
from typing import Callable
from app.registry.access import CallerContext, PermissionDeniedError
from app.registry.errors import ToolCallError

class ApprovalRequiredError(ToolCallError):
    def __init__(self, draft_id: str):
        self.draft_id = draft_id
        super().__init__(
            f"This write action requires human approval before it can run. "
            f"Draft {draft_id} has been created and is pending review."
        )

class ApprovalDeniedError(ToolCallError):
    def __init__(self, draft_id: str, reason: str | None):
        super().__init__(
            f"Draft {draft_id} was denied by a human reviewer"
            + (f": {reason}" if reason else ".")
        )

def create_draft(tool_name: str, validated_args: dict, ctx: CallerContext, store: dict) -> str:
    draft_id = str(uuid4())
    store[draft_id] = ActionDraft(
        draft_id=draft_id, tool_name=tool_name, args=validated_args, caller_id=ctx.tenant_context.user_id
    )
    return draft_id

def approve_draft(draft_id: str, store: dict, approver_id: str) -> ActionDraft:
    draft = store[draft_id]
    if draft.is_stale():
        draft.status = DraftStatus.EXPIRED
        raise ApprovalDeniedError(draft_id, "draft expired before approval")
    draft.status = DraftStatus.APPROVED
    draft.approved_by = approver_id
    return draft
```

Tool selectionTool call generationTool input validationTool executionTool results returned to model

- Tool selection
- Tool call generation
- Tool input validation
- Tool execution
- Tool results returned to model
- When tools should execute automatically
- When tools need confirmation
- Risky vs safe tools
- User authorization before execution

---

## Phase 3 — MCP Tool Exposure & Consumption Patterns

**Goal:** Understand how this same draft-approve-execute idea maps onto MCP specifically — both as a server exposing risky tools, and as a client/host consuming them.

### What to learn

- MCP's tool schema: how a server declares a tool's `inputSchema`, and what metadata (if any) exists today for signaling "this is a write/destructive action" — MCP does not yet have a standardized risk-level field, so servers currently communicate this only through naming/description conventions or host-side heuristics.
- MCP's **elicitation** capability: a newer part of the spec letting a server ask the client to collect additional input or confirmation mid-call — the closest existing MCP primitive to "pause and ask a human."
- Host-side responsibility: most MCP clients (Claude Desktop, Claude Code, etc.) currently implement approval gating themselves, at the host level, rather than relying on the server to enforce it — meaning your draft-approve-execute gate is naturally a **host/client-side** concern, sitting between "server proposed a tool call" and "client actually invokes it."
- Comparing two exposure patterns: (a) a server exposes both a "propose" and a "commit" tool as two separate MCP tools, forcing two-step interaction at the protocol level, vs (b) a single tool with the approval gate enforced entirely in your own host/registry code (what Phase 2 builds). Understand the tradeoff: (a) is visible to any MCP client automatically; (b) is enforced but invisible outside your own registry.

### Example — two-step MCP tool exposure (pattern a)

```python
# Server-side: two separate MCP tools instead of one
{
    "name": "propose_ingest_document",
    "description": "Draft a document-ingestion action for human review. Does not modify any data.",
    "inputSchema": IngestDocumentInput.model_json_schema(),
},
{
    "name": "commit_ingest_document",
    "description": "Execute a previously approved ingestion draft. Requires draft_id from a human-approved draft.",
    "inputSchema": {"type": "object", "properties": {"draft_id": {"type": "string"}}, "required": ["draft_id"]},
}
```

A client consuming this server sees two distinct tools and can enforce — at the UI level — that `commit_ingest_document` is only ever called after a human has seen the corresponding draft, without needing any special logic beyond "this tool name looks committal."

# What tool Annotations are

labels that describe a tool’s behavior:

- readOnlyHint → i dont change anything
- destructiveHint → i might delete or overwrite stuff
- idempotentHint → u can run me twice w/o breaking things
- openWorldHint → i talk to things outside this computer

→ help the client (app using Claude) understand risk

## Why they are only hints

because servers can lie. If a random server says: “Trust me, I’m read-only”… it might still delete ur files → so clients must treat these hints as **untrusted unless the server is trusted**

## Why This Matters

the lethal trifecta:

- Tool can read private data
- Tool can see untrusted content
- tool can send data out

if all 3 exist in one session, an attacker could trick the model into leaking your data → Annotations help detect when this dangerous combo might happen

## What Annotations can do

- help decide when to show a confirmation (if a tool is marked destructive, the client can show a “Are you sure?” pop-up
- improve UX (showing a friendly tool name `title`
- Help enterprise enforce rules (”no destructive tools unless approved)
- help detect risky tool combi (closed-world data + open-world communication = daner)

## What Annotations can’t do

- they dont stop prompt injection
- they dont enforce safety
- they dont guarantee truth
- they dont describe session-level risk

## **Security Considerations**

1. Servers **MUST**:
    - Validate all tool inputs
    - Implement proper access controls
    - Rate limit tool invocations
    - Sanitize tool outputs
2. Clients **SHOULD**:
    - Prompt for user confirmation on sensitive operations
    - Show tool inputs to the user before calling the server, to avoid malicious or accidental data exfiltration
    - Validate tool results before passing to LLM
    - Implement timeouts for tool calls
    - Log tool usage for audit purposes

## What Elicitation

Elicitation is how an MCP server politely asks the user for extra information **through the client**.

Think of it like the server saying:

> “Hey, I need something from the user — can you ask them for me?”
> 

The client shows the request to the user, the user responds, and the client sends the answer back.

### 1. **Form Mode** — Ask inside the app

This is like a small form popping up inside the MCP client.

- Server sends a message: “Please fill this in.”
- Client shows a form with fields (text, number, yes/no, dropdown, multi-select).
- User fills it out.
- Client sends the data back.

***Important rules:** No sensitive info (passwords, API keys, payment details) - Only simple data types (string, number, boolean, enums) - Client must let the user review and edit before submitting.

### 2. **URL Mode** — Ask outside the app

This is for **sensitive or secure actions** that must NOT pass through the MCP client.

Examples:

- Entering an API key
- Doing an OAuth login
- Completing a payment

Flow:

1. Server sends a URL + message.
2. Client shows the URL and asks for user consent.
3. User clicks → browser opens the page.
4. User completes the action directly on the secure website.
5. Server may send a “complete” notification later

**🧸 Important rules:**

- Client must show the **full URL** clearly.
- Client must NOT auto-open the link.
- Server must NOT put sensitive info inside the URL.
- Server must verify the user who opened the URL is the same user who requested it (to prevent phishing).

### 🔄 **Three User Actions**

Every elicitation request can end in one of three ways:

- **accept** → user agrees (and submits data if form mode)
- **decline** → user explicitly says no
- **cancel** → user closes/dismisses without choosing

Servers must handle each case properly.

---

## Phase 4 — Integration: The Control Gate

**Goal:** Wire draft-approve-execute into your existing registry so it's the mandatory path for every `WRITE` tool, with nothing able to skip it.

### Build checklist

1. `app/registry/drafts.py` — `ActionDraft`, `DraftStatus`, an in-memory `DraftStore` (swap for a real DB/queue later) with `create()`, `get()`, `approve()`, `deny()`, `edit()`.
2. `app/registry/approval.py` — `ApprovalRequiredError`, `ApprovalDeniedError`, and `require_approval_for_write()` — a check inserted into `guarded_execute()` (or a new `gated_execute()` wrapping it) that, for any `ToolKind.WRITE` tool, short-circuits into drafting instead of running `spec.fn` unless a valid, non-stale, approved `draft_id` is present in the call.
3. Update `app/registry/tools.py`'s `ingest_document` registration: no code change to the tool itself — the gate wraps around it at the execution layer, same pattern as how permission scoping wraps around it today.
4. Extend `Metrics` with `drafts_created_total`, `drafts_approved_total`, `drafts_denied_total`, `drafts_expired_total` per tool.
5. `scripts/approval_demo.py` — end-to-end CLI: propose an ingest → show the draft → simulate a human `approve`/`deny`/`edit` input → show the tool either running or being blocked, with metrics printed at the end.
6. `tests/test_approval_gate.py` — a write tool cannot execute without an approved draft; a denied draft cannot execute; a stale draft cannot execute even if approved; an edited draft executes with edited args; a read tool is completely unaffected (no draft required).

### Stretch goals

- Multi-approver policies: require 2 distinct approvers for destructive actions above some threshold (e.g. bulk delete).
- Persist drafts outside process memory (SQLite/Redis) so approval can happen from a different process than the one that proposed the draft — closer to how a real chat-based approval flow would work.
- Explore MCP elicitation directly: have a real MCP server call back into the client mid-tool-call to request the approval, rather than handling it entirely in your own registry.

---

## Resume-Ready Framing (once complete)

> Designed and implemented a draft-approve-execute workflow enforcing explicit human approval before any write/side-effect tool call executes, including a human-in-the-loop control gate integrated into an existing centralized tool registry, and evaluated MCP tool exposure patterns for surfacing risky actions to human reviewers.
>