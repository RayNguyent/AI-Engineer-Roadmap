# **RAG Citation Prompt**

## **Description**

This prompt instructs a **Retrieval-Augmented Generation (RAG)** system (acting as a professional researcher) to search for necessary information in external documents and produce detailed explanations in Markdown format. The system must:

- Retrieve and consolidate content from a specified web source
- Process the retrieved content along with a user’s query using a LangChain-based language model (such as gpt-4o-mini).
- Embed **in-line citations** in footnote format (e.g., [^1], [^2], etc.) within the answer. Each citation must include the **complete source URL** or file reference along with any relevant metadata (such as document title or page number).
- Output the final answer as a fully rendered Markdown document with all major headings in level 2 (`##`) and a single consolidated **References** section at the end.

The prompt is designed to leverage various techniques—including document retrieval, text splitting, structured output (e.g., via Pydantic or XML), and post-processing annotation—to ensure that the answer is both comprehensive and properly cited.

## Relevant Document

- How to get a RAG application to add citations
- Build RAG with in-line citations
- **Context Document:** Attention Is All You Need (arXiv:1706.03762v7)

## **Input**

- **SYSTEM:**
    
    "You are a professional research assistant. Generate a detailed answer to the user's query based solely on the provided document.
    Your response must include in-line citations in footnote format. Use the content loaded from the provided web source and ensure every citation contains the full URL."
    
- **HUMAN:**
    
    "{input}" *(User's query)*
    

## **Output**

- "The output is a **Markdown** document with:
    1. **Detailed Body**:
        - Answers the user’s query thoroughly.
        - Embeds **footnote citations** (e.g., [^1], [^2]) to reference the source document or web links, including page numbers when citing local documents.
        - All headings must be formatted with `##` in the generated Markdown text.
    2. **References Section**:
        - A **single** list titled **References** at the end of the document.
        - Each citation in the body corresponds to an entry in this list, e.g.:
            
            ```markdown
            [^n]: Short description or document title
            ```
            
        - Make sure the link includes the **complete URL** with protocol if citing a web resource or the **filename/page** if citing a PDF or other local document.

## **Tools**

- **WebBaseLoader (doc retrieval)**
    - **Description:** Loads text and metadata from a specified URL (data example: `https://arxiv.org/html/1706.03762v7`[^3]) for context.
    - **Input:** The URL of the document to retrieve.
    - **Output:** Extracted text or relevant content along with any metadata.
- **Additional Processing Tools**
    - **Description:** Text splitters, embedding-based filters, or post-processing components that refine the retrieved text for better context alignment.

## **Additional Information**

- **Footnotes:**
    - “Citation” must specify either the **full URL** (including `https://`) or the **complete file reference** with page numbers if citing a PDF or local file.
    - The generated answer must use **level 2 (##) headers** for all headings to maintain a uniform structure.
- **Guidelines:**
    - Provide **in-line footnotes** (e.g., [^1], [^2], etc.) within the main text.
    - End the response with a consolidated **References** list, matching each footnote to its source.
    - Include only **one** **References** section per response.
    - The final Markdown must be **fully rendered** (ready to copy and paste).

## Examples

Below are five illustrative scenarios demonstrating different approaches to generate responses with in-line citations:

### Example 1: Basic RAG Chain

- **Process:**
    - **Document Retrieval:** Use a web-based loader to fetch the full content from the provided URL.
    - **Context Preparation:** Concatenate the entire document text into one block.
    - **Prompt Construction:** Instruct the model (gpt-4o-mini) to answer a question (e.g., "How does the Transformer model handle sequential data differently from traditional RNNs?") while embedding footnote citations.
- **Expected Outcome:**
The answer includes bullet-point explanations with citations in the format [^1], [^2], etc., followed by a single **References** section (e.g., [^1]: Vaswani et al., "Attention Is All You Need.").

### Example 2: Structured Output with Citation Identifiers

- **Process:**
    - **Pre-Processing:** Format the document content by assigning unique source IDs to each segment.
    - **Schema Enforcement:** Use a structured output approach (e.g., via a Pydantic model) so that the model returns an answer along with a list of citation IDs.
- **Expected Outcome:**
A structured response (e.g., JSON/dict) containing an "answer" field with in-line citation markers (e.g., "(Source ID: 0)") and a "citations" list that maps these IDs to specific sources.

### Example 3: XML-Based Output for Citations

- **Process:**
    - **XML Prompt:** Instruct the model to generate its response in XML format, with designated tags such as `<answer>` and `<citations>`.
    - **Parsing:** Convert the XML output into Markdown by extracting the answer text and citation details.
- **Expected Outcome:**
An XML structure that, when parsed, yields a Markdown document containing the answer with in-line citations and a consolidated **References** list.

### Example 4: Retrieval Post-Processing with Filtering

- **Process:**
    - **Chunking:** Split the document into smaller segments (using a text splitter).
    - **Filtering:** Apply embeddings-based filtering to select the most relevant segments for the query.
    - **Context Creation:** Combine the filtered segments into a final context for the prompt.
- **Expected Outcome:**
A concise answer that references only the most pertinent parts of the document, with in-line citations and a corresponding **References** section.

### Example 5: Annotation and Citation Enhancement

- **Process:**
    - **Two-Step Generation:**
        1. Generate an initial answer without citations.
        2. Use a follow-up annotation prompt to enrich the answer with precise citations (including source IDs and quotes).
- **Expected Outcome:**
A final annotated answer where the response text includes footnote markers and a JSON/structured list of citation annotations is produced. These annotations are then merged into the final **References** section.

high similarity scores dont always guarantee perfect relevance. In other words, the retriever may retrieve a text chunk that has a high similarity score but not useful ⇒ Reranking

## What about Reranking

text chunk retrieved solely based on a retrieval metric (raw retrieval) may not be useful:

- chunks may vary largely wit the selected number of top chunks k
- chunks are semantically close to what we looking for but still off-topic
- partial matches to specific words included in the user’s query, leading to chinks that include those specific words but are in fact irrelevant

ex: small k ⇒ chunks may not contain enuff info to answer a question

big k ⇒ irrelevant text chunks where keyword is just mentioned 

⇒ Reranking means re-evaluating the chunks that are retrieved based on **cosine similarity scores**

## Reranking with a Cross-Encoder

Unlike retriever functions used in the initial retrieval step (take into account the similarity scores of diff text chunks), cross-encoder **jointly embeds** a doc and user’s query and produces a similarity score

!image.png

So now let’s see how all these play out in the *‘War and Peace’* example by answering one more time my favorite question – ‘*Who is Anna Pávlovna*?’.

we will be using a cross-encoder model to re-rank the top-k retrieved documents before passing them to your LLM. More specifically, I will be utilizing the cross-encoder/ms-marco-TinyBERT-L2 cross-encoder, which is a simple, pre-trained cross-encoding model, running on top of PyTorch. 

```python
import torch
from sentence_transformers import CrossEncoder

# initialize cross-encoder model
cross_encoder = CrossEncoder('cross-encoder/ms-marco-TinyBERT-L-2', device='cuda' if torch.cuda.is_available() else 'cpu')

def rerank_with_cross_encoder(query, relevant_docs):
    
    pairs = [(query, doc.page_content) for doc in relevant_docs] # pairs of (query, document) for cross-encoder
    scores = cross_encoder.predict(pairs) # relevance scores from cross-encoder model
    
    ranked_indices = np.argsort(scores)[::-1] # sort documents based on cross-encoder score (the higher, the better)
    ranked_docs = [relevant_docs[i] for i in ranked_indices]
    ranked_scores = [scores[i] for i in ranked_indices]
    
    return ranked_docs, ranked_scores
    
import os
from langchain.chat_models import ChatOpenAI
from langchain.document_loaders import TextLoader
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import FAISS
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.docstore.document import Document

import faiss

api_key = "my_api_key"

# initialize LLM
llm = ChatOpenAI(openai_api_key=api_key, model="gpt-4o-mini", temperature=0.3)

# initialize embeddings model
embeddings = OpenAIEmbeddings(openai_api_key=api_key)

# loading documents to be used for RAG 
text_folder =  "RAG files"  

documents = []
for filename in os.listdir(text_folder):
    if filename.lower().endswith(".txt"):
        file_path = os.path.join(text_folder, filename)
        loader = TextLoader(file_path)
        documents.extend(loader.load())

splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=100)
split_docs = []
for doc in documents:
    chunks = splitter.split_text(doc.page_content)
    for chunk in chunks:
        split_docs.append(Document(page_content=chunk))
        
documents = split_docs

# normalize knowledge base embeddings
import numpy as np
def normalize(vectors):
    vectors = np.array(vectors)
    norms = np.linalg.norm(vectors, axis=1, keepdims=True)
    return vectors / norms

doc_texts = [doc.page_content for doc in documents]
doc_embeddings = embeddings.embed_documents(doc_texts)
doc_embeddings = normalize(doc_embeddings)

# faiss index with inner product
import faiss
dimension = doc_embeddings.shape[1]
index = faiss.IndexFlatIP(dimension)  # inner product index
index.add(doc_embeddings)

# create vector database w FAISS 
vector_store = FAISS(embedding_function=embeddings, index=index, docstore=None, index_to_docstore_id=None)
vector_store.docstore = {i: doc for i, doc in enumerate(documents)}

def main():
    print("Welcome to the RAG Assistant. Type 'exit' to quit.\n")
    
    while True:
        user_input = input("You: ").strip()
        if user_input.lower() == "exit":
            print("Exiting…")
            break

        # embedding + normalize query
        query_embedding = embeddings.embed_query(user_input)
        query_embedding = normalize([query_embedding]) 
        
        # search FAISS index
        D, I = index.search(query_embedding, k=6)
        
        # get relevant documents
        relevant_docs = [vector_store.docstore[i] for i in I[0]]
        
        # rerank with our function
        reranked_docs, reranked_scores = rerank_with_cross_encoder(user_input, relevant_docs)
        
        # get top reranked chunks
        retrieved_context = "\n\n".join([doc.page_content for doc in reranked_docs[:2]])
        
        # D contains inner product scores == cosine similarities (since normalized)
        print("\nTop 6 Retrieved Chunks:\n")
        for rank, (idx, score) in enumerate(zip(I[0], D[0]), start=1):
            print(f"Chunk {rank}:")
            print(f"Similarity: {score:.4f}")
            print(f"Content:\n{vector_store.docstore[idx].page_content}\n{'-'*40}")

        # display top reranked chunks
        print("\nTop 2 Re-ranked Chunks:\n")
        for rank, (doc, score) in enumerate(zip(reranked_docs[:2], reranked_scores[:2]), start=1):
            print(f"Rank {rank}:")
            print(f"Reranker Score: {score:.4f}") 
            print(f"Content:\n{doc.page_content}\n{'-'*40}")
        
        # system prompt
        system_prompt = (
            "You are a helpful assistant. "
            "Use ONLY the following knowledge base context to answer the user. "
            "If the answer is not in the context, say you don't know.\n\n"
            f"Context:\n{retrieved_context}"
        )

        # messages for LLM 
        messages = [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_input}
        ]

        # generate response
        response = llm.invoke(messages)
        assistant_message = response.content.strip()
        print(f"\nAssistant: {assistant_message}\n")

if __name__ == "__main__":
    main()

```

We use the vector search to narrow down the possibly relevant chunks, out of all the available documents in the knowledge base, but then use the reranking step to identify the most relevant chunks accurately.

!image.png