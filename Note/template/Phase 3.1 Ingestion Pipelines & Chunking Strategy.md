# Learning Objectives

- Understand the document ingestion pipeline
    - Load documents
    - Clean text
    - Chunk text
    - Attach metadata
    - Index for retrieval
- Learn chunking strategies
    - Fixed-size chunking
    - Recursive chunking
    - Semantic chunking
- Understand chunking parameters
    - Chunk size
    - Chunk overlap
    - Semantic boundaries
- Measure retrieval quality
    - Recall
    - Context preservation
    - Chunk count
    - Indexing speed
- Build an indexing pipeline
    - Generate document IDs
    - Store chunk metadata
    - Export chunks into a structured format

```python
pip install -U langchain-text-splitters
```

**Text splitters** break large docs into smaller chunks that will be retrievable individually and fit within model context window limit.

# **Text structure-based**

Text is naturally organized into hierarchical units such as paragraphs, sentences, and words

⇒ create split that maintain natural language *flow*, maintain *semantic coherence* within split, and adapts to varying levels of *text granularity* :

- The `RecursiveCharacterTextSplitter` attempts to **keep larger units** (e.g., paragraphs) **intact**.
- If a unit **exceeds** the chunk size, it moves to the **next level** (e.g., sentences).
- This process continues down to the word level if necessary.

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

text_splitter = RecursiveCharacterTextSplitter(chunk_size=100, chunk_overlap=0)
texts = text_splitter.split_text(document)
```

# **Length-based**

This simple yet effective approach ensures that each chunk doesn’t exceed a specified size limit.

Key benefits of length-based splitting:

- Straightforward implementation
- Consistent chunk sizes
- Easily adaptable to different model requirements

Types of length-based splitting:

- **Token-based**: Splits text based on the number of tokens, which is useful when working with language models.
- **Character-based:** Splits text based on the number of characters, which can be more consistent across different types of text.

## **Splitting by token - Text splitter integration guide**

When you count tokens in your text you should use the same tokenizer as used in the language model.

### tiktoken

We can use `tiktoken` to estimate tokens used. It will probably be more accurate for the OpenAI models.

3 methods:

#### 1. CharacterTextSplitter.from_tiktoken_encoder()

- split by **characters (paragraphs, sentences)**
- but measures chunk size using tiktoken
- Chunks might be slightly larger than the target size

```python

from langchain_text_splitters import CharacterTextSplitter

text_splitter = CharacterTextSplitter.from_tiktoken_encoder(
	encoding_name = "cl100k_base",
	chunk_size = 100,
	chunk_overlap = 0
	
chunks = text_splitter.split_text(doc)

```

#### 2. RecursiveCharacterTextSplitter.from_tiktoken_encoder()

- Splits recursively until each chunk is **stricly below** the token limit
- best when you need **hard** token constraints

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

text_splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(
											model_name = "gpt-4",
											chunk_size = 100,
											chunk_overlap = 0
											)
```

⇒ no chunk exceeds 100 tokens

#### 3. TokenTextSplitter

- splits **directly by tokens**
- guarantee each chunk is **smaller** than chunk_size

```python
from langchain_text_splitters import TokenTextSplitter

text_splitter = TokenTextSplitter(
								chunk_size = 100,
								chunk_overlap = 0
								)
```

⇒ no chars involved

*Flow: Langchain splits text by characters (not tokens) → For each piece, LangChain calls tiktoken_encode() to tokenize and return a list of token ids → LangChain merges pieces until token limit is reached (tiktoken keeps track of count)

### **spaCy**

1. the text is split: by `spaCy` tokenizer.
2. the chunk size is measured: by number of characters.

```python
pip install --upgrade --quiet  spacy

from langchain_text_splitters import SpacyTextSplitter

text_splitter = SpacyTextSplitter(chunk_size=1000)

texts = text_splitter.split_text(state_of_the_union)
```

### **SentenceTransformers**

specialized text splitter for use with the sentence-transformer models. The default behaviour is to split the text into chunks that *fit the token window* of the sentence transformer model that you would like to use

To split text and constrain token counts according to the sentence-transformers tokenizer, instantiate a `SentenceTransformersTokenTextSplitter`. You can optionally specify:

- `chunk_overlap`: integer count of token overlap;
- `model_name`: sentence-transformer model name, defaulting to `"sentence-transformers/all-mpnet-base-v2"`;
- `tokens_per_chunk`: desired token count per chunk.

```python
from langchain_text_splitters import SentenceTransformersTokenTextSplitter

splitter = SentenceTransformersTokenTextSplitter(chunk_overlap=0)
text = "Lorem "

count_start_and_stop_tokens = 2
text_token_count = splitter.count_tokens(text=text) - count_start_and_stop_tokens
print(text_token_count)

token_multiplier = splitter.maximum_tokens_per_chunk // text_token_count + 1

# `text_to_split` does not fit in a single chunk
text_to_split = text * token_multiplier

print(f"tokens in text to split: {splitter.count_tokens(text=text_to_split)}")
```

### **Hugging Face tokenizer**

We use Hugging Face tokenizer, the `GPT2TokenizerFast` to count the text length in tokens.

1. the text is split: by character passed in.
2. the chunk size is measured: by number of tokens calculated by the `Hugging Face` tokenizer.

```python
from transformers import GPT2TokenizerFast

tokenizer = GPT2TokenizerFast.from_pretrained("gpt2")

# This is a long document we can split up.
with open("state_of_the_union.txt") as f:
    state_of_the_union = f.read()
from langchain_text_splitters import CharacterTextSplitter

text_splitter = CharacterTextSplitter.from_huggingface_tokenizer(
    tokenizer, chunk_size=100, chunk_overlap=0
)
texts = text_splitter.split_text(state_of_the_union)
```

## **Splitting by character - Text splitter integration guide**

divides text using a specified character sequence (default: `"\n\n"`), with chunk length measured by the number of characters.

Key points:

- text is split: by a given character separator.
- chunk size is measured: by character count.

You can choose between:

- `.split_text` — returns plain string chunks.
- `.create_documents` — returns LangChain Document objects, useful when metadata needs to be preserved for downstream tasks.

```python
from langchain_text_splitters import CharacterTextSplitter

# Load an example document
with open("state_of_the_union.txt") as f:
    state_of_the_union = f.read()

text_splitter = CharacterTextSplitter(
    separator="\n\n",
    chunk_size=1000,
    chunk_overlap=200,
    length_function=len,
    is_separator_regex=False,
)
texts = text_splitter.create_documents([state_of_the_union])
print(texts[0])
```

Use `.create_documents` to propagate metadata associated with each document to the output chunks:

```python
metadatas = [{"document": 1}, {"document": 2}]
documents = text_splitter.create_documents(
    [state_of_the_union, state_of_the_union], metadatas=metadatas
)
print(documents[0])

#page_content='Madam Speaker, Madam Vice President, our First Lady and Second Gentleman. Members of Congress and the Cabinet. Justices of the Supreme Court. My fellow Americans.  \n\nLast year COVID-19 kept us apart. This year we are finally together again. \n\nTonight, we meet as Democrats Republicans and Independents. But most importantly as Americans. \n\nWith a duty to one another to the American people to the Constitution. \n\nAnd with an unwavering resolve that freedom will always triumph over tyranny. \n\nSix days ago, Russia’s Vladimir Putin sought to shake the foundations of the free world thinking he could make it bend to his menacing ways. But he badly miscalculated. \n\nHe thought he could roll into Ukraine and the world would roll over. Instead he met a wall of strength he never imagined. \n\nHe met the Ukrainian people. \n\nFrom President Zelenskyy to every Ukrainian, their fearlessness, their courage, their determination, inspires the world.' metadata={'document': 1}

```

Use `.split_text` to obtain the string content directly:
`text_splitter.split_text(state_of_the_union)[0]`
⇒ 'Madam Speaker, Madam Vice President, our First Lady and Second Gentleman…

# **Document structure-based**

Some documents have an inherent structure, such as HTML, Markdown, or JSON files.

⇒ split the document based on its structure, as it often naturally groups semantically related text. Key benefits of structure-based splitting:

- Preserves the logical organization of the document
- Maintains context within each chunk
- Can be more effective for downstream tasks like retrieval or summarization

Examples of structure-based splitting:

- Markdown: Split based on headers (e.g., `#`, `##`, `###`)
- HTML: Split using tags
- JSON: Split by object or array elements
- Code: Split by functions, classes, or logical blocks

**Available text splitters**

- Split Markdown
- Split JSON
- Split code
- Split HTML


In simple terms, **chunking** is the process of breaking down large documents into smaller, manageable pieces called *chunks*. 

The main reason is that LLMs have a limited **context window,** meaning they can only focus on a certain amount of text at once. If there is too much text within the context window, important details are lost, resulting in incomplete or inaccurate answers.

⇒ The **size, content, and semantic boundaries** of each chunk influence retrieval performance

# **Pre-Chunking vs Post-Chunking**

**Pre-chunking** is the most common method. It processes documents asynchronously by breaking them into smaller pieces **before embedding** and storing them in the vector database
⇒ fast retrieval at query time since all chunks are pre-computed and indexed.

!image.png

**Post-chunking** takes a different approach by embedding entire documents first, then performing chunking at query time *only on the documents that are actually retrieved.*
The chunked results can be cached, so the system becomes **faster** over time as frequently accessed documents build up cached chunks ⇒ avoids chunking documents that may never be queried while allowing for more dynamic, context-aware chunking strategies based on the specific query.

*However, it introduces **latency on first access** and requires additional infrastructure decisions.

!image.png

# **Chunking Strategies**

## **Fixed-Size Chunking (or Token Chunking)**

splits text into chunks of a **predetermined size**, often measured in tokens (pieces of text that the model processes) or characters. ⇒ can cut off in the middle of sentences ⇒ **chunk overlap**, where some tokens from the end of one chunk are repeated at the beginning of the next

**Key considerations:**

- **Chunk Size:** A common starting point is a chunk size that aligns with the context window of the embedding model. Smaller chunks can be better for capturing fine-grained details, while larger chunks might be more suitable for understanding broader themes.
- **Chunk Overlap:** A typical overlap is between 10% and 20% of the chunk size.

## **Recursive Chunking**

It splits text using a prioritized list of common separators, such as *double newlines* (for paragraphs) or *single newlines* (for sentences). It first tries to split the text by the highest-priority separator (paragraphs). If any resulting chunk is still too large, the algorithm *recursively* applies the next separator (sentences) to that specific chunk.

***Recommended for:** Unstructured text documents, such as articles, blog posts, and research papers. This is usually a solid default choice because it respects the natural organization of the text, rather than splitting it randomly.

## **Document-Based Chunking**

Document-based chunking uses the **intrinsic structure of a document**. Instead of relying on generic separators, it parses the document based on its format-specific elements

- **Markdown**: Split by headings (`#`, `##`) to capture sections or subsections.
- **HTML**: Split by tags (`<p>`, `<div>`) to preserve logical content blocks.
- **PDF**: After preprocessing (e.g., OCR or conversion to Markdown), split by headers, paragraphs, tables, or other structural elements.
- **Programming Code**: Split by functions or classes (e.g., `def` in Python) to maintain logical units of code.

## **Semantic Chunking (Context-Aware Chunking)**

divides text based on its semantic similarity. The process involves:

- **Sentence Segmentation**: Breaking the text into individual sentences
- **Embedding Generation**: Converting each sentence into a vector embedding
- **Similarity Analysis**: *Comparing the embeddings* to detect semantic breakpoints (places where the topic changes)
- **Chunk Formation**: Creating new chunks between these breakpoints

***Recommended for:** Dense, unstructured text to preserve the complete semantic context of an idea. This method works well for academic papers, legal documents, or long stories. These texts do not always use clear separators like paragraphs to show topic changes

## **LLM-Based Chunking**

LLM processes the document and generates *semantically coherent chunks*, often also adding additional context, summaries, or other information. This can be done by:

- **Identifying propositions** (breaking text into clear, logical statements)
- **Summarizing sections** into smaller, meaning-preserving chunks
- **Highlighting key points** to ensure the most relevant information is captured

***When to use:** High-value, complex documents where retrieval quality is critical and budget is less of a concern. Ideal for legal contracts, research papers, compliance documents, or enterprise knowledge bases.

## **Agentic Chunking**

an AI agent dynamically decides how to split your documents. 

It looks at the whole document, including its structure, density, and content ⇒ decides on the best chunking strategy or mix of strategies to use. For example, the agent might see that a document is a Markdown file ⇒ splits the file by headers.

 It might also find that a denser document needs a propositional approach. It can even enrich chunks with metadata tags for more advanced retrieval.

*In practice, agentic chunking is often used as part of a broader agent memory architecture, where the agent actively manages how information is stored and recalled.

***When to use:** High-stakes RAG systems where you need the best possible chunks and cost isn't a dealbreaker. Perfect when you need custom chunking strategies tailored to each document's unique characteristics.

## **Late Chunking**

In other chunking techniques, when you split a document first and then create embeddings, each chunk becomes *isolated*. This can result in ambiguous or lost context within the chunk that was explained or referenced earlier in a document.

Late chunking works **backwards**. Instead of splitting first, you start by feeding the entire document into a long-context embedding model ⇒ token-level embeddings that understand the full picture ⇒ split into chunks

***When to use:** Use this in RAG systems where retrieval quality depends on *understanding relationships between chunks and the whole document*. This is very useful for technical documents, research papers, or legal texts. These documents have sections that refer to ideas, methods, or definitions mentioned elsewhere.

## **Hierarchical Chunking**

You create *multiple layers of chunks* at different levels of detail.

- At the **top layer**, you create large chunks that *summarize broad sections* or themes, like the title and abstract.
- At the **next layers**, you split those sections into progressively smaller chunks that capture finer details such as arguments, examples, or definitions.

***When to use:** Very large and complex documents, such as textbooks, legal contracts, or extensive technical manuals. This strategy is ideal when you need to answer both high-level, summary-based questions and highly specific, detailed queries.

!image.png

## **Adaptive Chunking**

dynamically adjust key **parameters** (like chunk size and overlap) based on the document's content.
For example, it could automatically create smaller, more granular chunks for a complex, information-rich paragraph to capture fine-grained details, while using larger chunks for a more general, introductory section.

⇒ goal is to create chunks whose s*ize and boundaries are tailored* to the specific content they contain, leading to more precise and context-aware retrieval

***When to use:** Documents with varied and inconsistent internal structures. Think of a long report that contains dense, technical paragraphs alongside sparse, narrative sections. An adaptive strategy excels here because it avoids the "one-size-fits-all" problem.

# **How to Choose the Best Chunking Strategy**

> ***“Does my data need chunking at all?”***
> 

Chunking is designed to break down long, unstructured documents. If your data source already has small, complete pieces of information like FAQs, product descriptions, or social media posts, you usually do **NOT** need to chunk them.

Once you've confirmed that your documents are long enough to benefit from chunking, you can use the following questions to guide your choice of strategy

- **What is the nature of my documents?** Are they highly structured (like code or JSON), or are they unstructured narrative text?
- **What level of detail does my RAG system need?** Does it need to retrieve specific, granular facts or summarize broader concepts?
- **Which embedding model am I using?** What are the size of the output vectors (more dimensions increases the ability for more granular information to be stored)??
- **How complex will my user queries be?** Will they be simple questions that need small, targeted chunks, or complex ones that require more context?

| **Chunking Strategy** | **How It Works** | **Complexity** | **Best For** | **Examples** |
| --- | --- | --- | --- | --- |
| **Fixed-Size (or Token)** | Splits by token or character count. | Low | Small or simple docs, or when speed matters most | Meeting notes, short blog posts, emails, simple FAQs |
| **Recursive** | Splits text by repeatedly dividing it until it fits the desired chunk size, often preserving some structure. | Medium | Documents where some structure should be maintained but speed is still important | Research articles, product guides, short reports |
| **Document-Based** | Treats each document as a single chunk or splits only at document boundaries. | Low | Collections of short, standalone documents | News articles, customer support tickets, short contracts |
| **Semantic** | Splits text at natural meaning boundaries (topics, ideas). | Medium-High | Technical, academic, or narrative documents | Scientific papers, textbooks, novels, whitepapers |
| **LLM-Based** | Uses a language model to decide chunk boundaries based on context, meaning, or task needs. | High | Complex text where meaning-aware chunking improves downstream tasks like summarization or Q&A | Long reports, legal opinions, medical records |
| **Agentic** | Lets an AI agent decide how to split based on meaning and structure. | Very High | Complex, nuanced documents that require custom strategies | Regulatory filings, multi-section contracts, corporate policies |
| **Late** | Embeds the whole document first, then derives chunk embeddings from it. | High | Use cases where chunks need awareness of full document context | Case studies, comprehensive manuals, long-form analysis reports |
| **Hierarchical** | Breaks text into multiple levels (sections → paragraphs → sentences). Keeps structure intact. | Medium | Large, structured docs like manuals, reports, or contracts | Employee handbooks, government regulations, software documentation |
| **Adaptive** | Adjusts chunk size and overlap dynamically with ML or heuristics. | High | Mixed datasets with varying structures and lengths | Data from multiple sources: blogs, PDFs, emails, technical docs |
| **Code** | Splits by logical code blocks (functions, classes, modules) while preserving syntax. | Medium | Source code, scripts, or programming documentation | Python modules, JavaScript projects, API docs, Jupyter notebooks |

# **How to Optimize The Chunk Size for RAG in Production**

- Begin with a common baseline strategy, such as fixed-size chunking. A good place to start is a chunk size of *512 tokens* and a chunk overlap of *50-100 tokens*. This gives you a solid baseline that's easy to reproduce and compare other chunking strategies against.
- Experiment with different chunking approaches by *tweaking parameter*s like chunk size and overlap to find what works best for your data.
- Test how well your retrieval works by *running typical queries and checking metrics* like hit rate, precision, and recall to see which strategy delivers.
- *Involve humans* to review both the retrieved chunks and LLM-generated responses - their feedback will catch things metrics might miss.
- Continuously *monitor the performance* of your RAG system in production and be prepared to iterate on your chunking strategy as needed.
- In production LLM RAG pipelines, chunking is not just a preprocessing step — it defines the quality of retrieval and the effectiveness of agent memory.
- As AI systems evolve from simple RAG setups to long-running agents, chunking becomes a foundational decision for memory, reasoning, and cost efficiency.