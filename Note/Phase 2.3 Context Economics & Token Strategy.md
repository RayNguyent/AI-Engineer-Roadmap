# Phase 1 — Tokenization Fundamentals

Tokenization is the process of breaking text into smaller units (tokens) that an LLM can process.

LLMs see `[15496,1917]` but not `Help World`

Most modern models use variants of:

- BPE (Byte Pair Encoding)
- SentencePiece
- WordPiece

> Cost is usually token-based
Context windows are token-based
> 

```python
import tiktoken

encoding = tiktoken.encoding_for_model("gpt-4o")

text = "What is FastAPI?"

tokens = encoding.encode(text)

print(len(tokens))
print(tokens)
```

# Phase 2 — Token Budget Analysis

> Token Budget = Maximum tokens available for: System Prompt+ Conversation History + Retrieved Documents + Expected Output
> 

```python
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4o")

prompt = open("invoice.txt").read()

prompt_tokens = len(enc.encode(prompt))

expected_output = 1500

total = prompt_tokens + expected_output

print(f"Total Budget: {total}")
```

# Phase 3 — Context Window Analysis

> **Context Window** = the total number of tokens an LLM can handle in one request. It covers everything: system prompt, few-shot examples, retrieved docs, conversation history, and the model’s response.
> 

More context ≠ better

Instruction Loss (lost in the sauce)

Fact Loss

Long Chat Drift

*To address the limitations of fixed context windows, some models use a sliding window approach, where the context is updated as the model processes new tokens. This method ensures that the model always operates within the context window, but it may cause some loss of global information as tokens "fall out" of the window.

# Phase 4 — Pricing Analysis

Most providers charge: Input tokens + Output Tokens at diff rates

# Phase 5 — Latency Analysis

> Latency = Time required to produce a response.
> 

Bigger model = higher latency

More output = Higher latency

More calls = higher latency

## Metrics

TTFT = Time to 1st Token

TRT = Total Response Time

```python
import time

start = time.perf_counter()

response = client.responses.create(...)

elapsed = time.perf_counter() - start

print(elapsed)
```

## Optimization

### Process Tokens faster

Inference speed refers to the actual rate at which the LLM processes tokens, and is often measured in TPM (tokens per minute), or TPS (tokens per second)

Main factor that influences speed is **MODEL SIZE** - smaller models run faster. 

To maintain high quality performance with smaller models you can:

- longer, more detailed prompt
- adding few-show examples,
- fine-tuning/distillation

### Generate fewer Tokens

cutting 50% of your **output tokens** may cut ~50% your latency

- if generating natural language → ask the model to be more concise (”under 20 words” or “be very brief”). Also use few shot examples
- if generating structured output → minimize output syntax (shorten func names, omit named arguments, coalesce params)
- can also use `max_tokens` or `stop tokens` to end early

### Use fewer input tokens

cutting 50% of your **prompt** only cut 1-5% latency improvement.

but if you work with massive contexts, you can:

- **Fine-tuning the model**, to replace the need for lengthy instructions/examples
- **Filtering context input**, like pruning RAG results, cleaning HTML
- **Maximize shared prompt prefix,** by putting dynamic portions (Rag results, history) later in the prompt ⇒ your request more KV cache-friendly + fewer input tokens

## Make fewer requests

if you have **sequential steps** for the LLM to perform ⇒ put all into a single prompt

⇒ collecting your steps in an enumerated list in the combined prompt, and then requesting the model to return the results in named fields in a JSON

## Parallelize

if the steps are **NOT** strictly sequential ⇒ split into parallel cells

if the steps are stricly sequential ⇒ leverage speculative execution

1. Start step 1 and step 2 simultaneously (input moderation & story generation)
2. Verify the result of step 1
3. If result was not the expected, cancel step 2 (and retry if needed)

## Make your users wait less

- Streaming:
- **Chunking**: process in chunks instead of all at once ← streaming to backend then sending processed chunks to frontend
- **Show steps**: if u taking multiple steps or using tools, surface to user
- **Loading states**: spinners and progress bars go a long way

## Dont default to an LLM

Sometimes LLMs are abused in cases where a **faster classical method** would be more efficient. Consider:

- **Hard-coding:** if your output is **highly constrained (little variation)** ⇒ not need an LLM to generate it. *Action confirmations, refusal messages, and requests for standard input* are all great candidates to be hard-coded
- **Pre-computing:** if your input is constrained → generate multiple responses in advance and just make sure you never show the same one to a user twice
- **Leveraging UI:** summarized metrics, reports, or search results are repr with classical UI components than LLM-generated text
- Traditional optim. techniques: binary search, caching, hashmaps, and runtime complexity are codable

typical flow:

- User sends a message as part of an ongoin process
- last message is turned into self-contained query
- we determine whether or not additional (retrieved) info is required to respond to that query
- retrieval is performed ⇒ search results
- “assistant” reasons bout the user’s query and search results, and produces a response
- response is sent back to user

user → gpt-4: query contextualization → gpt-4: retrieval check → Retrieval → GPT-4: assistant prompt → assistant

optim.:

1. combined query contextualization and retrieval check steps to make fewer requests
2. for the new combined prompt, use GPT-3.5, smaller, fine-tuned to process tokens faster
3. split the assistant prompt into 2, switch to a smaller, fine-tuned GPT-3.5 for the reasoning
4. Parallelizaed the retrieval checks and the reasoning steps
5. Shortened reasoning field names and moved comments into the prompt, to generate fewer tokens

## Phase 6 — Temperature Experiments

Temperature controls randomness in token selection.

Low temp = deterministic

High temp = creative

LLMs generate text by predicting the next token according to a probability dist. Each token is assigned a logit ([0-1]) and total set sums to 1

⇒ Temp modifies this dist

### Configuring temp

- do_sample: control whether the model **samples** during text generation. when `=True` , model randomly samples from redacted token probabilities rather than always selecting the most probable word from a sequence in a dataset
- top_k: restricts model’s possible choices when random sampling to the **top k most likely tokens**
- top_p: consider tokens whose cumulative prob is ≥ a value (ex: ≥  0. 9) aka smallest set of tokens whose cumulative probs ≥ a value

### Controlling output

- maximum length (tokens):
- stop sequence
- Frequency penalty: discourages repetition in the generated text by penalizing tokens proportionally to how frequently they appear.
- Presence penalty: penalizes tokens based on whether they have occurred or not

## Cost Optim.

cost and latency are typically interconnected; reducing tokens and requests generally leads to faster processing. 

- **reduce requests**
- **minimize tokens**
- **select a smaller model**
- **Batch API (OpenAI):** there are cases where requests do not need immediate response or rate limits prevent u from executing a large num of queries. *Collect a set of requests into a file → kick off a batch processing job to execute these requests → query for the status of the batch while underlying requests execute → retrieve when batch is complete*. Best for running evals, classifying large datasets, embedding content repos, queuing large offline video-render jobs
- **Flex processing (OpenAPI):** (u dont pay for Nexus, still get thru but slower) but sometimes delayed and sometimes needs retry. Best for low priority (model evaluations, data enrichment, async workloads)