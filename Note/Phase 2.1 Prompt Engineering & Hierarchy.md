# **Significance of Structured Roles**

many real-world enterprise applications require the LLM to maintain context across multiple interactions, such as a customer support chatbot or an intelligent assistant. Modern LLMs are often designed to process a "message list" or "conversation history" rather than just a single prompt.

This structured message list is where system, user, and assistant roles become *essential*. They help the LLM understand:

- **Who is speaking:** Is this an instruction from the application, input from the end-user, or a previous response from the AI itself?
- **The intent of each message:** Is this a general guideline, a specific request, or part of an ongoing dialogue?
- **The flow of the conversation:** By providing a chronological history of messages, the LLM can "remember" earlier parts of the interaction.

# **The System Role**

The system role is typically the *first message* in a sequence and acts as the foundational instruction for the LLM. It defines the model's overall behavior, persona, constraints, and general rules for the entire interaction or session. 

**Effectiveness:** The "system" message provides a stable "operating environment" for the LLM, ensuring *consistent behavior* and adherence to established guidelines throughout a conversation, regardless of subsequent user inputs. It significantly *reduces "concept drift"* and helps the LLM stay aligned with your application's intent.

**Developer Aspect:** This message is usually static or semi-static within your application's code, defined once at the beginning of an interaction.

```json
{
  "role": "system",
  "content": "You are a helpful assistant for SAP Logistics. Your goal is to provide accurate and concise information about supply chain processes. Always refer to official SAP terminology where possible. If you don't know the answer, state that you don't have enough information."
}

```

# The Developer Role

contains **task‑specific operational rules**, such as:

## 1. Task Behavior Rules

- How the assistant should perform *this task*
- Level of strictness (creative vs deterministic)
- ex: “Use only provided context. Do not infer missing data.”

## 2. Output Constraints

- Format requirements
- Length limits
- Schema enforcement
- ex: “Output must be valid JSON. No additional text.”

## 3. Context Handling Rules

- What context is allowed
- Whether memory is used
- Truncation or refusal behavior
- ex: “Ignore conversation history unless explicitly injected.”

## 4. Determinism Rules

- No creativity
- Fixed refusal phrases
- Reproducibility guarantees
- ex: “If information is missing, return INSUFFICIENT_INFORMATION”

### 5. Token & Boundary Policies

- Input/output limits
- Failure behavior
- ex: “If input exceeds 3,000 tokens, abort.”

# **The User Role**

represents the specific query, instruction, or context *provided directly by the human* interacting with your application, or by your application itself for a particular turn

**Purpose:**

- **Pose Questions:** "What is the status of sales order 12345?"
- **Provide New Data:** "Here is the customer's purchase history: [data]."
- **Issue Specific Commands:** "Draft an email to the client confirming shipment.

**Effectiveness:** Clearly *defines what the requestor* (human or application logic) is *asking* or providing in that specific turn. In multi-turn scenarios, previous user messages are included in the message list to remind the LLM of the conversation's progression.

```json
{
  "role": "user",
  "content": "Explain the concept of 'material master data' in simple terms for a new employee."
}

```

# **The Assistant Role**

maintaining conversational memory. It represents the *LLM's own prior responses* within an ongoing dialogue. It can also provide examples of the output expected from LLM. 

When building a multi-turn conversation, you send the entire history of alternating user and assistant messages back to the LLM with each new user prompt.

**Purpose:**

- **Maintain Context:** Allows the LLM to refer to what it or the user said previously, making conversations coherent.
- **Continue Dialogue:** Enables the LLM to follow up on its own previous answers or acknowledge earlier parts of the conversation.
- **Reinforce Learning (Subtly):** By including previous, well-formed assistant responses, you implicitly show the LLM examples of desirable output format, tone, and content within the specific conversation.

**Effectiveness:** Without the assistant role, the LLM would treat each user input as a brand new, isolated request, leading to repetitive or nonsensical dialogue. This role provides the necessary "*memory*" for sustained interaction.

```json
{
  "role": "assistant",
  "content": "Material master data in SAP refers to a central repository of information about materials that a company procures, produces, stores, and sells. It's essential for managing inventory, purchasing, production planning, and sales processes."
}

```

```json
[
  {
    "role": "system",
    "content": "You are a helpful assistant for SAP Logistics. Your goal is to provide accurate and concise information about supply chain processes. Always refer to official SAP terminology where possible. If you don't know the answer, state that you don't have enough information."
  },
  {
    "role": "user",
    "content": "Explain the concept of 'material master data' in simple terms for a new employee."
  },
  {
    "role": "assistant",
    "content": "Material master data in SAP refers to a central repository of information about materials that a company procures, produces, stores, and sells. It's essential for managing inventory, purchasing, production planning, and sales processes."
  },
  {
    "role": "user",
    "content": "That's clear, thank you. What about 'procurement process' in SAP?"
  }
]

```
# **System Prompt Architecture: The Constitutional Document Pattern**

### **Five-Layer Anatomy**

All major production agents converge on a common structure:

1. **Identity framing** — "You are X, built by Y" (~100 tokens). Sets the agent's role and capabilities.
2. **Behavioral rules** — Task execution principles, reversibility guidelines, tone, output format. The operational constitution.
3. **Typed tool APIs** — Explicit per-tool instruction sections beyond just JSON schemas. How to use each tool, when to prefer one over another, what side effects to expect.
4. **Safety layers** — Embedded as operational rules woven throughout the prompt, not a separate block. Security is architectural, not appended.
5. **Conditional sections** — Mode-specific instructions (plan mode, auto mode, minimal mode) assembled dynamically at runtime.

# **Instruction Hierarchy and Priority**

## **The Formal Model**

Without explicit hierarchy, LLMs treat all inputs equally — enabling injection through any channel.

```
Priority 0  — System messages (application developer)
Priority 10 — User messages (end user)
Priority 30 — Tool output (web results, API responses, third-party content)
```

OpenAI trained GPT-3.5 Turbo with two techniques for hierarchy enforcement: Context Synthesis (aligned responses) and Context Ignorance (teaching the model to answer "as if it never saw" conflicting lower-priority instructions). Results: +63% system prompt extraction defense, +30% jailbreak robustness, generalizing to unseen attack types.

## **Practical Layered Defense**

1. XML/delimiter separation between instruction blocks and user data
2. Filter known injection patterns: "ignore previous", "act as", DAN variants
3. LLM guardrails layer (Bedrock Guardrails, Azure AI Content Safety, NeMo)
4. RAG source authentication — 5 poisoned documents can manipulate 90% of RAG responses
5. Structured queries — treat user input as data structures, not natural language (USENIX Security 2025)

# **Understanding prompts**

There are three primary ways to structure the prompt: direct instructions, open-ended instructions and task-specific instructions: 

- **Direct instructions** are clear and specific commands that tell the AI *exactly what to do*. These prompts are ideal for straightforward tasks where the user has a clear expectation of the output. **Direct prompts** rely on the model’s ability to parse explicit instructions and generate responses that align closely with the command. 
”The more detailed the instruction, the more likely the output will meet expectations.”

```
Write a poem about nature.
```

- **Open-ended instructions** are less restrictive and encourage the AI to explore broader ideas or *provide creative and interpretive responses*. These prompts are useful for **brainstorming**, **storytelling** or **exploratory discussions** where the user values variety and originality in the output. **Open-ended prompts** tap into the model’s generative capabilities *without imposing constraints.* The model relies on its training data to infer the best approach to the prompt, which can produce diverse or unexpected results.

```
Tell me about the universe.
```

- **Task-specific instructions** are designed for precise, goal-oriented tasks, such as *translations*, *summarization* or *calculations*. **Task-specific prompts** leverage the model’s understanding of specialized tasks. They can incorporate advanced prompting techniques like few-shot prompting or zero-shot prompting.

```
Translate this text into French: ‘Hello.’
```

# **Key techniques in prompt engineering**

## **Zero-shot prompting**

asking the model to perform a task without providing any prior examples or guidance ⇒ relies entirely on the AI’s pretrained knowledge to interpret and respond to the prompt.

```
Explain the concept of climate change, its causes, and its effects in simple terms.
```

## **Few-shot prompting**

includes a small number of examples within the prompt to demonstrate the task to the model

```
Here are some examples of how to explain complex topics:

- Topic: Photosynthesis

- Explanation: Photosynthesis is the process by which plants convert sunlight, water, and carbon dioxide into energy and oxygen.

- Topic: Gravity

- Explanation: Gravity is the force that pulls objects toward each other, like how the Earth pulls us to its surface.

Now explain: Climate Change.
```

## **Chain of thought (CoT) prompting**

encourages the model to reason through a problem step by step, breaking it into smaller components to arrive at a logical conclusion

```
Step 1: Define what climate change is.

Step 2: Explain the causes of climate change.

Step 3: Describe its effects on the planet.

Now, follow these steps to explain climate change.
```

## **Meta prompting**

involves asking the model to *generate or refine its own prompts* to better perform the task. This technique can improve output quality by leveraging the model’s ability to self-direc

```
Create a prompt that will help you explain climate change, its causes, 
and its effects in simple terms
```

## **Self-consistency**

uses *multiple independent generations* from the model to identify the most coherent or accurate response. It’s particularly useful for tasks requiring reasoning or interpretation

```
Provide three different explanations of climate change, its causes, and its effects. 
Then identify the most coherent and clear explanation
```

## **Generate knowledge prompting**

 involves asking the model to *generate background knowledge* before addressing the main task, enhancing its ability to produce informed and accurate responses

```
Before explaining climate change, first list the key scientific principles related to it.
Once done, use these principles to explain the concept, its causes, and its effects.
```

## **Prompt chaining**

involves linking multiple prompts together, where the output of one prompt serves as the input for the next. This technique is ideal for multistep processes.

```
What is climate change? Provide a brief definition.

What are the primary causes of climate change?

What are the effects of climate change on the environment and human life?
```

## **Tree of thoughts prompting**

encourages the model to *explore multiple branches of reasoning* or ideas before arriving at a final output

```
List three possible ways to explain climate change to a general audience. 
For each method, describe its advantages and disadvantages. 
Then choose the best explanation and elaborate on it
```

## **Retrieval augmented generation (RAG)**

combines external information retrieval with generative AI to produce responses based on up-to-date or domain-specific knowledge

```
Using the global temperature datasets from NASA GISS (GISTEMP) dataset on 
climate science, explain climate change, its causes, and its effects in simple terms.
```

## **Automatic reasoning and tool-use**

Integrates *reasoning capabilities with external tools* or application programming interfaces (APIs), allowing the model to use resources like calculators or search engines

```
Use the provided climate data to calculate the global temperature rise over the last 
century, and then explain how this relates to climate change, its causes, and 
its effects.
```

## **Automatic prompt engineer**

## **Active-prompt**

 *adjusts* the prompt based on *intermediate outputs* from the model, refining the input for better results

```
Explain climate change, its causes, and its effects in simple terms.
```

Follow-up prompt

```
Add more detail about the causes of climate change, focusing on human activities.
```

## **ReAct**

combines *reasoning and acting prompts*, encouraging the model to think critically and act based on its reasoning

```
Analyze the following climate data and identify key trends. 
Based on your analysis, explain the concept of climate change, its causes, 
and its effects.
```

## **Multimodal chain of thought (multimodal CoT)**

 integrates chain of thought reasoning across multiple modalities, such as text, images or audio

```
Analyze this infographic on global warming trends, then explain climate change, 
its causes, and its effects step by step
```

# Token

tokens are the fundamental element in a language model’s input and output process. 

A token does not correspond to a single word. For instance, a simple word would be one token, but longer or more technical words may form two or three tokens. Even spaces, punctuation, and capitalization influence tokenization.

A reliable rule of thumb: 1 token is roughly 4 characters of English text, or about 75 words per 100 tokens.

# Context Window

context window is the maximum amount of information a model can process in a single interaction. It includes your system prompt, conversation history, and the current user message everything counts toward the limit.

template: [prompt-template-library](https://github.com/RayNguyent/prompt-template-library)