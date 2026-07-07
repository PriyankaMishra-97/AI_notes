

### Multi-Query Retrieval
Multi-Query Retrieval leverages an intermediate, lightweight LLM to automate the prompt tuning process. 
1. The user inputs a single query.
2. The intermediate model acts as a query expansion engine, generating 3 to 5 distinct semantic variations representing different perspectives.
3. The system executes independent, parallel vector lookups across the vector database for every variation.
4. The system aggregates all returned document chunks, executes a unique union (deduplication), and shoves the combined context into the main LLM.

* **Trade-off:** This maximize **Recall** (ensuring the necessary document is retrieved), but taking an unweighted union can flood the final LLM with redundant noise and duplicate statements, inflating token consumption.

### RAG-Fusion and Reciprocal Rank Fusion (RRF)
RAG-Fusion builds upon the Multi-Query architecture. Instead of executing a simple union of the retrieved document chunks, it passes the disjoint sets through an algorithmic ranking step known as **Reciprocal Rank Fusion (RRF)** before generating the final answer.

```
                  [ User Query ]
                        |
                        v
          [ LLM Generates 3-5 Queries ]
            /           |                [Query 1]      [Query 2]      [Query 3]
        |               |              |
     Vector          Vector          Vector
     Search          Search          Search
        |               |              |
   [Doc List 1]   [Doc List 2]    [Doc List 3]
            \           |           /
             v          v          v
          [ Reciprocal Rank Fusion (RRF) ]
                        |
                        v
          [ Re-ranked Top K Chunks ]
                        |
                        v
              [ Final LLM Answer ]
```

RRF is a zero-shot rank aggregation algorithm. It ignores raw distance scores (like cosine similarity or dot product), which can shift unpredictably between query variations. Instead, it aggregates based purely on the **relative position (rank)** of a document across all generated queries.

The mathematical scoring function for a document $d$ is defined as:

$$	ext{Score}(d \in D) = \sum_{m \in M} rac{1}{k + r_m(d)}$$

Where:
* $M$ represents the set of all query variant result lists.
* $r_m(d)$ is the explicit position/rank of document $d$ within the specific result list $m$ (e.g., 1 for first, 2 for second).
* $k$ is a constant smoothing factor. The industry standard default is **60**.

*Why is $k=60$?* The smoothing factor $k$ ensures that an outlier high-ranking document from a single low-quality query variant does not disproportionately dominate the global score. It penalizes variations while rewarding documents that consistently appear in top positions across multiple queries (global consensus).

---

## 4. Cross-Encoder Reranking: The 2-Stage Architecture

While Hybrid Search and RAG-Fusion provide excellent candidate pools, standard vector search uses **Bi-Encoder architectures**. In a Bi-Encoder framework, the user query and the database documents are embedded completely in isolation. This independence allows for highly scalable vector calculations, but prevents the model from capturing deep, token-level interactions between the query and the text chunks.

### The Bi-Encoder vs. Cross-Encoder Dichotomy

#### Bi-Encoder Architecture
* **Mechanism:** Document text chunks are embedded offline and indexed in a vector space. At query time, the user prompt is converted to a vector using the same embedding model. A similarity metric (e.g., cosine distance) is computed.
* **Attributes:** Extremely low latency, highly scalable to millions of records, but yields lower semantic precision because no cross-attention occurs between the query tokens and document tokens.

#### Cross-Encoder Architecture
* **Mechanism:** The user query and the candidate document are concatenated directly into a single string (e.g., `[CLS] User Query [SEP] Document Candidate [SEP]`) and fed into a deep Transformer simultaneously.
* **Attributes:** The model's internal self-attention mechanisms compute weights between *every single word in the query* and *every single word in the document*. This yields superior precision, but is computationally expensive and slow. Running a Cross-Encoder directly against an entire enterprise database of millions of documents is impossible at runtime.

### The Two-Stage Retrieval Architecture
To optimize for both speed and extreme accuracy, enterprise-grade AI applications implement a **Two-Stage Retrieval Pipeline**:

```
[ User Input ] 
     │
     ▼
┌────────────────────────────────────────┐
│ STAGE 1: High Recall Retrieval         │
│ (Bi-Encoder / Hybrid / RAG-Fusion)     │
│ Scans millions of documents rapidly    │
└────────────────────┬───────────────────┘
                     │ Returns Top 100 Candidates
                     ▼
┌────────────────────────────────────────┐
│ STAGE 2: High Precision Reranking      │
│ (Cross-Encoder / Cohere / BGE-Reranker)│
│ Computes deep token-to-token attention │
└────────────────────┬───────────────────┘
                     │ Filters down to Top 5 Chunks
                     ▼
             [ Final LLM Context ]
```

1.  **Stage 1 (High Recall):** The system uses a fast vector search, Hybrid Search, or RAG-Fusion to rapidly filter down the database from millions of records to the top 50–100 candidate documents. 
2.  **Stage 2 (High Precision):** These 100 documents are passed sequentially or in batches through a specialized **Cross-Encoder Rerank Model** (e.g., Cohere Rerank, BGE-Reranker). The Cross-Encoder assigns a deep attention-based relevance score to each chunk, ordering them precisely. 
3.  **Final Extraction:** The system isolates the top 5 highest-scoring documents from the reranker and passes them into the final LLM prompt.

### Benefits of the Two-Stage Architecture:
* **Mitigates "Lost in the Middle" Bias:** LLMs struggle to recall information buried in the middle of massive context windows. Trimming the input to the absolute most relevant chunks improves generation quality.
* **Optimizes Infrastructure Cost:** Eliminating irrelevant text chunks reduces token usage, cutting down overall cloud inference costs.
* **Maximizes Precision:** It ensures that even if the initial vector distance scoring was imprecise due to wording variations, the transformer's deep attention mechanism captures the exact contextual match before the final response is generated.


### Why should we add a reranker like Cohere or BGE to our RAG pipeline?"

## Give this structured 3-point answer:

**It Solves the "Embedding Bottleneck":** > "Bi-encoders lose granular token interactions because they compress text into a single vector. A Cross-Encoder like BGE or Cohere evaluates the query and document together, allowing the attention heads to map direct relationships between words."

**It Permits Bad Embedding Models:** > "Adding a reranker allows us to use a cheaper, faster, or older embedding model for Stage 1. The initial search just needs to ensure the right document is in the top 100 (high recall). The Cross-Encoder will do the heavy lifting to pull it to the absolute top (high precision)."

**It Directly Improves LLM Output:** > "LLMs suffer from 'Lost in the Middle' bias, meaning if the right answer is hidden in the 7th out of 10 chunks, the LLM might miss it. Cohere/BGE ensures that the most relevant answer is always in slot #1 or #2, drastically reducing hallucinations."
