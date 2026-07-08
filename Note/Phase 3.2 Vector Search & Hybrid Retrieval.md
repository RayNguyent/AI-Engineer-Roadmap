# Learning Objectives

- Understand embeddings and vector search
- Learn how BM25 keyword retrieval works
- Understand hybrid retrieval (BM25 + dense vectors)
- Learn metadata filtering
- Implement tenant isolation
- Prevent cross-tenant document exposure
- Build a retrieval engine that only returns authorized documents

We create vector embeddings, which are just lists of numbers, for data like this to perform various operations with them. A whole paragraph of text or any other object can be reduced to a vector. Even numerical data can be turned into vectors for easier operations.

!image.png

⇒ makes it possible to translate semantic similarity as perceived by humans to proximity in a vector space.

!image.png

# **SentenceTransformers**

 can be used to compute embeddings from text, images, audio, or video using Sentence Transformer models (quickstart), to calculate similarity scores using Cross-Encoder (a.k.a. reranker) models (quickstart), or to generate sparse embeddings using Sparse Encoder models (quickstart).

```python
from sentence_transformers import SentenceTransformer

# 1. Load a pretrained Sentence Transformer model
model = SentenceTransformer("sentence-transformers/all-MiniLM-L6-v2")

# The sentences to encode
sentences = [
    "The weather is lovely today.",
    "It's so sunny outside!",
    "He drove to the stadium.",
]

# 2. Calculate embeddings by calling model.encode()
embeddings = model.encode(sentences)
print(embeddings.shape)
# [3, 384]

# 3. Calculate the embedding similarities
similarities = model.similarity(embeddings, embeddings)
print(similarities)
# tensor([[1.0000, 0.6660, 0.1046],
#         [0.6660, 1.0000, 0.1411],
#         [0.1046, 0.1411, 1.0000]])
```

## **Semantic Search**

seeks to improve search accuracy by understanding the semantic meaning of the search query and the corpus to search over

<aside>
💡

The idea behind semantic search is to **embed all entries** in your corpus, whether they be sentences, paragraphs, or documents, into a vector space. At search time, the **query is embedded into the same vector space** and the **closest** embeddings from your corpus are found. These entries should have a high semantic similarity with the query.

</aside>

!image.png

```
sentence_transformers.util.semantic_search(query_embeddings: ~torch.Tensor, 
	corpus_embeddings: ~torch.Tensor, 
	query_chunk_size: int = 100, 
	corpus_chunk_size: int = 500000, 
	top_k: int = 10, 
	score_function: ~collections.abc.Callable[[~torch.Tensor, ~torch.Tensor], 
	~torch.Tensor] = <function cos_sim>)→ list[list[dict[str, int | float]]][source]

This function performs by default a cosine similarity search between a list of 
query embeddings and a list of corpus embeddings. It can be used for Information Retrieval / Semantic Search for corpora up to about 1 Million entries.

Parameters:
- query_embeddings (Tensor) – A 2 dimensional tensor with the query embeddings. Can be a sparse tensor.

- corpus_embeddings (Tensor) – A 2 dimensional tensor with the corpus embeddings. Can be a sparse tensor.

- query_chunk_size (int, optional) – Process 100 queries simultaneously. Increasing that value increases the speed, but requires more memory. Defaults to 100.

- corpus_chunk_size (int, optional) – Scans the corpus 100k entries at a time. Increasing that value increases the speed, but requires more memory. Defaults to 500000.

- top_k (int, optional) – Retrieve top k matching entries. Defaults to 10.

- score_function (Callable[[Tensor, Tensor], Tensor], optional) – Function for computing scores. By default, cosine similarity.

Returns:
A list with one entry for each query. Each entry is a list of dictionaries with the keys ‘corpus_id’ and ‘score’, sorted by decreasing cosine similarity scores.

Return type:
List[List[Dict[str, Union[int, float]]]]
```

# OpenAI’s Embedding

```python
from openai import openai

client = OpenAI()

response = client.embeddings.create(
	input="Your text string goes here",
	model = "text-embeddings-3-small"
	)
	
print(reponse.data[0].embedding)
```

## Reducing embedding dimensions

using larger embeddings (storing in a vector for retrieval), generally costs more and consumes more compute, memory, storage than using smaller embeddings.

developers can shorten embeddings w/o the embedding losing its concept-representing properties by passing in the `dimensions` param.

## Recommendations using embeddings

Because shorter distances between embedding vectors represent greater similarity ⇒ embeddings

```python
def rec_from_strings(
	strings: List[str],
	index_of_source_string: int,
	model = "text-embedding-3-small",) -> List[int]:
	"""Return nearest neighbors of a given string"""
	
	# get embeddings for all strings
	embeddings = [embedding_from_string(string, model=model) for string in strings]
	
	# get embeddings of the source string
	query_embedding = embedding[index_of_source_string]
	
	# get distances between the source embedding and other embeddings
	distances = distances_from_embeddings(query_embedding, embeddings, distance_metric = "cosine")
	
	# get indices of nearest neighbors
	indices_of_nearest_neighbors = indices_of_nearst_neighbors_from_distances(distances)
	return indices_of_nearest_neighbors
```

Semantic search uses dense vectors. Each number in a dense vector corresponds to a point in a multidimensional space. Vectors that are closer together in that space are semantically similar.

1. **Query Understanding:** The search engine uses NLP to analyze the structure of the query, its words and their relationships which helps it to grasp the user's intent and meaning.
2. **Entity Recognition:** It identifies key entities such as people, places and concepts in the query which allows the search engine to focus on specific topics the user is interested in.
3. **Semantic Matching:** The search engine compares the meaning of the query with the content of documents in its index helps in evaluating the relationships between words and phrases for relevance.
4. **Contextual Analysis:** Factors like location, search history and user preferences are considered to personalize search results which ensures that they are useful and tailored to the individual.
5. **Ranking and Relevance:** Results are then ranked according to their semantic relevance, source authority and user preferences which ensures the most important content is displayed.

Combine the results of a vector search and a keyword search. The search uses a single query string.

Ironically, as useful as this similarity search is, it sometimes fails to retrieve parts of text that are exact maTches to the user’s promp (when searching in a large knowledge base, some keywords may get lost, and relevant chunks may not be retrieved even if the user’s query contains the **exact words**) ⇒ BM25

BM25 is a ranking function used to evaluate how relevant a doc is to a search query.  BM25 is bag-of-words model (it doesn’t take into account the order of the words in a doc but rather the freq with which each word appears in the doc

!image.png

BM25 computes a relevance score between a query ***q*** and a document ***d*** using three main components: Term Frequency (TF), Inverse Document Frequency (IDF) and Document Length Normalization.

**1. Term Frequency (TF)**

measures how often a query term appears in a document. Intuitively, a document containing a query term multiple times is more likely to be relevant. However, BM25 introduces a *saturation effect* i.e beyond a certain point, a*dditional occurrences of a term contribute less to the score*. This prevents overly long documents from being unfairly favored.

!image.png

where:

- t: Query term
- d: Document
- freq(t,d): Number of times term *t* appears in document *d*
- ∣d∣: Length of document *d*
- avg dl: average document length in corpus
- k1: controls term frequency scaling
- b: controls document length normalization

**2. Inverse Document Frequency (IDF)**

Inverse document frequency measures the importance of a term across the entire corpus. Rare terms are considered more informative than common ones

The IDF component is calculated as:

!image.png

where:

- N: Total number of documents in the corpus
- n_t: Number of documents containing term *t*

**3. Document Length Normalization**

BM25 accounts for document length by normalizing scores to prevent *longer documents from dominating the rankings*. This is controlled by the parameter b which adjusts the influence of document length relative to the average document length (avgdl).

The final BM25 score for a document d with respect to a query q is computed as:

!image.png

!image.png

A multitenant solution is used by multiple customers. Each customer, or tenant, consists of multiple users from the same organization, company, or group. In multitenant scenarios, you need to make sure that tenants, or individuals within tenants, are **only able to incorporate grounding data that they're authorized to access.**

## **Single-tenant RAG architecture with an orchestrator**

!image.png

The following steps describe a high-level workflow.

1. A user *issues a request* to the intelligent web application.
2. An identity provider *authenticates the requestor.*
3. The intelligent application calls the orchestrator API with the user's query and the authorization token for the user.
4. The orchestration logic *extracts the user's query* from the request and calls the appropriate data store to f*etch relevant grounding data* for the query. The grounding data is added to the prompt that's sent to the foundation model, like a model that's exposed in Azure OpenAI, in the next step.
5. The orchestration logic connects to the foundation model's inferencing API and sends the prompt that includes the retrieved grounding data. The results are returned to the intelligent application.

## **Single-tenant RAG architecture with direct data access**

In this architecture, you either **DONT** have your own orchestrator, or your orchestrator has fewer responsibilities. The Azure OpenAI API calls into the data store to fetch the grounding data and passes that data to the language model

!image.png

1. *A user issues a request to the intelligent web application.*
2. *An identity provider authenticates the requestor.*
3. The intelligent application calls Azure OpenAI with the user's query.
4. Azure OpenAI connects to supported data stores, such as AI Search and Azure Blob Storage, to fetch the grounding data. The grounding data is used as part of the context when Azure OpenAI calls the OpenAI language model. The results are returned to the intelligent application.

## **Multitenancy in RAG architecture**

tenant data might exist in a *tenant-specific store* or coexist with other tenants in a multitenant store. Data might also be in a store that's shared across tenants. 

> Only data that the user is authorized to access should be used as grounding data
> 

!image.png

1. *A user issues a request to the intelligent web application.*
2. *An identity provider authenticates the requestor.*
3. *The intelligent application calls the orchestrator API with the user's query and the authorization token for the user.*
4. The orchestration logic extracts the user's query from the request and calls the appropriate data stores to fetch tenant-authorized, relevant grounding data for the query. The grounding data is added to the prompt that's sent to Azure OpenAI in the next step. Some or all of the following steps are included:
    1. The orchestration logic fetches grounding data from the appropriate *tenant-specific data store instance* and potentially applies security filtering rules to return only the data that the user is authorized to access.
    2. The orchestration logic fetches the appropriate tenant's grounding data from the multitenant data store and potentially applies security filtering rules to return only the data that the user is authorized to access.
    3. The orchestration logic fetches data from a data store that's shared across tenants.
5. *The orchestration logic connects to the foundation model's inferencing API and sends the prompt that includes the retrieved grounding data. The results are returned to the intelligent application.*

## **Design considerations for multitenant data in RAG**

### **Choose a store isolation model**

The two main architectural approaches for storage and data in multitenant scenarios are store-per-tenant and multitenant stores. 

#### **Store-per-tenant stores**

each tenant has its own store. This approach also simplifies cost allocation because the entire cost of a store deployment can be attributed to a single tenant.

might present challenges such as increased management and operational overhead and higher costs

<aside>
🚫

 You shouldn't use this approach if you have a l**arge number of small tenants**, like in business-to-consumer scenarios. This approach might also reach or exceed service limits.

</aside>

#### **Multitenant stores**

, multiple tenants' data coexists in the same store

Advantages of this approach include the potential for cost optimization, the ability to handle a higher number of tenants than the store-per-tenant model, and lower management overhead because of the lower number of store instances.

The challenges of using shared stores include the need for data isolation and management, the potential for the noisy neighbor antipattern, and more complex cost allocation to tenants.

Data isolation is the most important concern when you use this approach. You need to implement secure approaches to help ensure that tenants can only access their data. Data management can also be challenging if tenants have different data lifecycles that require operations such as building indexes on different schedules.

### **Identity**

Identity is a key aspect of multitenant solutions, including multitenant RAG solutions. The intelligent application should integrate with an identity provider to authenticate the identity of the user. The multitenant RAG solution needs an identity directory that stores authoritative identities or references to identities. This identity needs to flow through the request chain and allow downstream services, such as the orchestrator or the data store itself, to identify the user.

Essentially, what we are trying to find out here is “*Are the right documents somewhere in the top-k retrieved set*?”, or does our vector search return complete garbage? 

## **Some order-unaware, binary measures**

Some common and useful binary, order-unaware measures are **HitRate@k,** **Recall@k**, **Precision@k, and F1@k**

### **HitRate@K**

indicating whether there is *at least one* relevant result in the top k retrieved chunks, or not. 

⇒ 1 (if at least one relevant doc exists in the retrieved set), or 0 (if none of the retrieved documents is in reality relevant). 

we can calculate different Hit Rates for all queries and retrieved results in a test set, and finally, calculate the average HitRate@K of the entire test set.

!image.png

### **Recall@K**

expresses how often the *relevant* documents appear within the top k retrieved documents. In essence, it evaluates how good we did with avoiding False Negatives.

!image.png

⇒ *Out of all the items that existed, how many did we get?*

### **Precision@k**

how many of the top k retrieved documents are indeed relevant

!image.png

⇒ *Out of the items that we retrieved, how many are correct*

### F1@K

But what if we need both correct and complete results — what if we need the retrieved set to score high both in Recall and Precision

!image.png

```python
# Function to normalize text
def normalize_text(text):
    return " ".join(text.lower().split())

# Hit Rate @ K 
def hit_rate_at_k(retrieved_docs, ground_truth_texts, k):
    for doc in retrieved_docs[:k]:
        doc_norm = normalize_text(doc.page_content)
        if any(normalize_text(gt) in doc_norm or doc_norm in normalize_text(gt) for gt in ground_truth_texts):
            return True
    return False

# Precision @ k 
def precision_at_k(retrieved_docs, ground_truth_texts, k):
    hits = 0
    for doc in retrieved_docs[:k]:
        doc_norm = normalize_text(doc.page_content)
        if any(normalize_text(gt) in doc_norm or doc_norm in normalize_text(gt) for gt in ground_truth_texts):
            hits += 1
    return hits / k

# Recall @ k
def recall_at_k(retrieved_docs, ground_truth_texts, k):
    matched = set()
    for i, gt in enumerate(ground_truth_texts):
        gt_norm = normalize_text(gt)
        for doc in retrieved_docs[:k]:
            doc_norm = normalize_text(doc.page_content)
            if gt_norm in doc_norm or doc_norm in gt_norm:
                matched.add(i)
                break
    return len(matched) / len(ground_truth_texts) if ground_truth_texts else 0

# F1 @ K
def f1_at_k(precision, recall):
    return 2 * precision * recall / (precision + recall) if (precision + recall) > 0 else 0
```

In order to calculate any of those evaluation metrics, we need to first define a set of queries and the respective truly relevant chunks. This is a rather copious exercise; thus, I will be only demonstrating the process for one query — ‘*Who is Anna Pávlovna*? — and the respective relevant text chunks that should ideally be retrieved. In any case, this information — either for just one query or for a real-life evaluation set — can be defined in the form of a ***ground truth dictionary***, allowing us to map various test queries to the expected relevant text chunks.

```python
query = "Who is Anna Pávlovna?"

ground_truth_texts = [
    "It was in July, 1805, and the speaker was the well-known Anna Pávlovna Schérer, maid of honor and favorite of the Empress Márya Fëdorovna. With these words she greeted Prince Vasíli Kurágin, a man of high rank and importance, who was the first to arrive at her reception. Anna Pávlovna had had a cough for some days. She was, as she said, suffering from la grippe; grippe being then a new word in St. Petersburg, used only by the elite. All her invitations without exception, written in French, and delivered by a scarlet-liveried footman that morning, ran as follows: “If you have nothing better to do, Count (or Prince), and if the prospect of spending an evening with a poor invalid is not too terrible, I shall be very charmed to see you tonight between 7 and 10—Annette Schérer.”",

    "Anna Pávlovna’s “At Home” was like the former one, only the novelty she offered her guests this time was not Mortemart, but a diplomatist fresh from Berlin with the very latest details of the Emperor Alexander’s visit to Potsdam, and of how the two august friends had pledged themselves in an indissoluble alliance to uphold the cause of justice against the enemy of the human race. Anna Pávlovna received Pierre with a shade of melancholy, evidently relating to the young man’s recent loss by the death of Count Bezúkhov (everyone constantly considered it a duty to assure Pierre that he was greatly afflicted by the death of the father he had hardly known), and her melancholy was just like the august melancholy she showed at the mention of her most august Majesty the Empress Márya Fëdorovna. Pierre felt flattered by this. Anna Pávlovna arranged the different groups in her drawing room with her habitual skill. The large group, in which were",

    "drawing room with her habitual skill. The large group, in which were Prince Vasíli and the generals, had the benefit of the diplomat. Another group was at the tea table. Pierre wished to join the former, but Anna Pávlovna—who was in the excited condition of a commander on a battlefield to whom thousands of new and brilliant ideas occur which there is hardly time to put in action—seeing Pierre, touched his sleeve with her finger, saying:"
]

# Evaluate reranked docs using metrics
        top_k_docs = reranked_docs[:k_]  # or change `k` as needed
        precision = precision_at_k(top_k_docs, ground_truth_texts, k=k_)
        recall = recall_at_k(top_k_docs, ground_truth_texts, k=k_)
        f1 = f1_at_k(precision, recall)
        hit = hit_rate_at_k(top_k_docs, ground_truth_texts, k=k_)
```

Notice that we run the evaluation **after reranking**. Since the metrics we calculate— like Precision@K, Recall@K, and F1@K — **are order-unaware, evaluating them on the top k retrieved chunks before or after reranking yields the same results, as long as the set of top k items remains the same.**