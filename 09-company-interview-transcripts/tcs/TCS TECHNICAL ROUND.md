## Great work cracking round 1! Let me research what TCS asks in technical + managerial rounds for Python + GenAI + Azure AI roles, then I'll give you deep answers to every round 1 question you struggled with. Good research done. Now I'll give you everything — deep answers to round 1 gaps, then the full technical + managerial interview simulation.

## Round 1 Gaps — Deep Answers First

### 1. Retrievers in LangChain

A retriever is any component that accepts a query string and returns a list of Document objects. It wraps a vector store or any search backend behind a standard interface so the rest of your chain doesn't care how documents are fetched.

Types you must know:

- **VectorStoreRetriever** — the default. Calls `similarity_search()` on a vector store.
  
  ```python
  retriever = vectorstore.as_retriever(search_type="similarity", search_kwargs={"k": 5})
  ```

- **MMR Retriever (Maximum Marginal Relevance)** — balances relevance AND diversity. Prevents returning 5 chunks that all say the same thing.
  
  ```python
  retriever = vectorstore.as_retriever(search_type="mmr", search_kwargs={"k": 5, "fetch_k": 20})
  ```

- **ContextualCompressionRetriever** — retrieves chunks then compresses them, keeping only sentences relevant to the query. Reduces token usage.
  
  ```python
  from langchain.retrievers import ContextualCompressionRetriever
  from langchain.retrievers.document_compressors import LLMChainExtractor

  compressor = LLMChainExtractor.from_llm(llm)
  retriever = ContextualCompressionRetriever(base_compressor=compressor, base_retriever=base_retriever)
  ```

- **MultiQueryRetriever** — generates 3-5 variants of your query using an LLM, retrieves for each, merges results. Handles vocabulary mismatch.
  
  ```python
  from langchain.retrievers import MultiQueryRetriever

  retriever = MultiQueryRetriever.from_llm(retriever=base_retriever, llm=llm)
  ```

- **SelfQueryRetriever** — LLM parses natural language query into a structured filter + semantic search. "Show me Nutreco documents from 2023" → metadata filter + vector search.

- **EnsembleRetriever** — combines multiple retrievers with weighted scores. This is what your project does — vector + BM25 combined.
  
  ```python
  from langchain.retrievers import EnsembleRetriever

  retriever = EnsembleRetriever(retrievers=[bm25, vector], weights=[0.4, 0.6])
  ```

How to answer in interview: "In my project I implemented an EnsembleRetriever combining Qdrant dense vector search with BM25 sparse retrieval, plus a cross-encoder reranker — this is called hybrid retrieval. I also applied filename and modality boosting as a custom scoring layer on top."

### 2. Embedding Cache — How You Implemented It

The concept: Embedding the same text twice is wasteful — it costs API calls and time. Cache the embedding vector and reuse it if the same text appears again.

LangChain's built-in approach:

```python
from langchain.embeddings import CacheBackedEmbeddings
from langchain.storage import LocalFileStore

underlying_embedder = OpenAIEmbeddings()
store = LocalFileStore("./embedding_cache/")

cached_embedder = CacheBackedEmbeddings.from_bytes_store(
    underlying_embedder, store, namespace=underlying_embedder.model
)

# First call: hits OpenAI API, saves to cache
# Second call for same text: reads from local file, zero API cost
```

How to answer for YOUR project: "In my project we use all-MiniLM-L6-v2 as a local embedding model running on CPU — it costs nothing per call so API-level caching wasn't critical. However, delta-based ingestion is my caching strategy — SharePoint files are hashed on download. If the hash matches what's stored, we skip re-embedding entirely. Only changed or new files trigger the embedding pipeline. This is file-level embedding caching. For production I would add CacheBackedEmbeddings with a Redis backend so repeated queries don't re-embed the same query text."

### 3. LangSmith for Cost Saving

LangSmith is an observability and evaluation platform for LLM apps. How it saves cost:

- **Token visibility** — every LLM call shows exact input/output tokens. You can see which prompts are bloated and trim them. In your project this would have caught the 337K token spike immediately.

- **Trace analysis** — shows which chain steps take most time and cost. If your retriever is fetching 10 chunks but the LLM only uses 2, you cut top_k and save tokens.

- **Prompt optimization** — A/B test shorter prompts. LangSmith shows cost per run so you can measure exact savings.

- **Caching identification** — shows repeated identical queries. You cache those responses and skip the LLM call entirely.

How to answer: "LangSmith gives me full token-level visibility on every LLM call. In my SharePoint RAG project I had a production issue where context was hitting 337K tokens and failing with a rate limit error — LangSmith would have caught that immediately by showing the token count per trace. I use it to identify which prompt templates are most expensive and trim them, which queries repeat frequently so I can cache responses, and to monitor latency regressions when I update the pipeline."

### 4. Correct Chain for the Scenario

This is an LCEL (LangChain Expression Language) question. The interviewer gave you:

```text
retriever | chat_prompt_template | llm | output_parser
```

The problem is that retriever returns List[Document] but ChatPromptTemplate expects a dict with keys matching its variables ({context} and {question}).

Correct answer:

```python
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

chain = (
    {
        "context": retriever,           # retriever output → context variable
        "question": RunnablePassthrough() # original query passes through unchanged
    }
    | chat_prompt_template
    | llm
    | output_parser
)

# Invoke
response = chain.invoke("What is the SBC IP for Nutreco?")
```

Why: The dict `{"context": retriever, "question": RunnablePassthrough()}` is a RunnableParallel. It runs both branches simultaneously — retriever fetches docs for context, RunnablePassthrough passes the original question unchanged. Both outputs are combined into a dict that maps to the template's {context} and {question} variables.

The interviewer's chain was wrong because the retriever's output (List of Documents) can't directly pipe into ChatPromptTemplate — you need the mapping step.

### 5. Chain Retrieval (Retrieval Chain)

This refers to RetrievalQA or create_retrieval_chain — a pre-built chain that combines retriever + prompt + LLM:

```python
from langchain.chains import create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain

# Step 1: chain that stuffs documents into prompt
document_chain = create_stuff_documents_chain(llm, prompt)

# Step 2: retrieval chain wraps it with retriever
retrieval_chain = create_retrieval_chain(retriever, document_chain)

# Invoke
result = retrieval_chain.invoke({"input": "What is the SBC IP?"})

print(result["answer"])
```

- **"Stuff" strategy** — all docs stuffed into one prompt. Simple, works for small context.

- **"Map-Reduce"** — each doc processed separately, answers combined. For large doc sets.

- **"Refine"** — first doc answers, each subsequent doc refines the answer iteratively.

### 6 & 7. Chunking Strategy + Types

Why chunking matters: LLMs have context limits. Vector search works better on focused chunks. Large chunks = more noise, small chunks = lost context.

Types of chunking:

| Type                     | How it works                                          | Best for                                      |
|--------------------------|------------------------------------------------------|-----------------------------------------------|
| Fixed size               | Split every N characters                              | Simple text, quick start                      |
| Recursive Character       | Split by \n\n, then \n, then . , then — tries to keep paragraphs together | General purpose — most common                 |
| Token-based              | Split by token count not char count                  | When you need exact token budget control      |
| Sentence                 | Split at sentence boundaries using NLTK/spaCy        | Q&A where each sentence is a unit             |
| Semantic                 | Embed sentences, group similar ones together         | Dense technical docs, research papers         |
| Markdown/HTML            | Split at headers, code blocks                         | Documentation, wikis                          |
| Section-aware (yours)    | Detect headings by font size, group body under heading | Large PDFs — Sonus guide, SBC handbook        |
| Page-based               | One chunk per PDF page                               | Scanned PDFs, image-heavy docs                |
| Sheet-based (yours)      | One chunk per Excel sheet                            | Structured operational runbooks                |

Overlap: Every chunk strategy includes chunk_overlap — last N characters of chunk N are repeated at start of chunk N+1. Prevents answers from being split across chunk boundaries.

### 8. Embedding Methods — The Full Answer

For a single PDF document, how do you embed it:

```python
# Step 1: Load
from langchain.document_loaders import PyPDFLoader

loader = PyPDFLoader("SBC_Guide.pdf")
pages = loader.load()  # List[Document], one per page

# Step 2: Chunk
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
chunks = splitter.split_documents(pages)

# Step 3: Embed + Store
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import Qdrant

embeddings = OpenAIEmbeddings()
vectorstore = Qdrant.from_documents(chunks, embeddings, collection_name="sbc_guide")
```

Types of embedding models:

- **Dense embeddings** — every dimension has a value, captures semantic meaning:
  - `text-embedding-ada-002 (OpenAI)` — 1536 dimensions, general purpose
  - `text-embedding-3-small (OpenAI)` — cheaper, nearly same quality
  - `all-MiniLM-L6-v2 (Sentence Transformers)` — 384 dimensions, free, runs locally — what your project uses
  - `BAAI/bge-large-en` — top open-source performer

- **Sparse embeddings** — most dimensions are zero, keyword-focused:
  - `BM25` — classic TF-IDF based, no neural network — what your project uses for hybrid
  - `SPLADE` — learned sparse model, better than BM25

- **Hybrid** — combine dense + sparse — exactly what your project does with Qdrant + BM25

- **Multimodal embeddings**:
  - `CLIP` — joint image + text embedding space
  - `ImageBind` — 6 modalities in one embedding space

For your project specifically: "I use all-MiniLM-L6-v2 for dense embeddings running locally on CPU — zero cost per call. For PDFs I use section-aware chunking so each chunk includes its heading context. For Excel I do sheet-level chunking — one embedding per sheet. For images I use Tesseract OCR first then embed the extracted text. For videos I use Whisper to transcribe then chunk at 2-minute windows with timestamp labels."

## Technical Round — Expected Questions

Now I'm the TCS Technical Architect interviewing you. These are the real questions based on the research:

### RAG Architecture Deep Dives:

1. Your RAG system is returning wrong answers even though the right document exists. Walk me through how you debug this.
2. What is the difference between naive RAG, advanced RAG, and modular RAG?
3. How do you evaluate a RAG system? What metrics do you use?
4. When would you use fine-tuning instead of RAG?
5. What is HyDE (Hypothetical Document Embedding) and when is it useful?
6. Your vector search returns semantically similar but factually wrong chunks. How do you fix this?
7. Explain parent-child chunking and when you'd use it over regular chunking.
8. What is GraphRAG? How is it different from standard RAG?

### LangGraph / Agentic:

9. Why did you use LangGraph instead of LangChain chains?
10. How do you handle state persistence in LangGraph across multiple turns?
11. What is the difference between an agent and a chain?
12. How do you implement Human-in-the-Loop in LangGraph?
13. Your agent is going into infinite loops. How do you prevent this?

### Azure AI Stack:

14. What is Azure AI Search and how does it compare to Qdrant?
15. How would you use Azure OpenAI vs OpenAI API — when to choose which?
16. What is Azure AI Studio and what can you do with it?
17. How does Azure Cognitive Services integrate with a RAG pipeline?
18. Explain Azure Managed Identity and how it replaces API keys in production.

### Production & Scalability:

19. Your RAG pipeline works in dev but is slow in production with 100 concurrent users. How do you scale it?
20. How do you handle hallucination detection in production?
21. What is RAGAS and how do you use it to evaluate your pipeline?
22. You have 1 million documents. How do you keep the vector index updated in real time?
23. How do you implement multi-tenancy in a RAG system — different clients, different data, same deployment?

### Python + System Design:

24. Write a function that chunks a PDF into sections based on headings.
25. Implement a simple retry wrapper for OpenAI API calls with exponential backoff.
26. Design a document Q&A system for 10 enterprise clients with isolated data.

## Managerial Round — Expected Questions

1. Tell me about a production AI system you built and what went wrong.
2. How do you estimate effort for a GenAI project?
3. A client wants to use ChatGPT for their sensitive HR data. How do you advise them?
4. Your team is split between RAG and fine-tuning. How do you make the decision?
5. How do you explain hallucination risk to a non-technical stakeholder?
6. What is your approach to prompt versioning and change management in production?
7. A junior engineer on your team deployed a new prompt that caused the chatbot to expose credentials. How do you handle this?
8. How do you measure ROI of a GenAI project for a client?
9. Your SharePoint RAG project — what would you do differently if you started today?

Want me to do a mock interview now? I'll ask you questions one by one, you answer, and I'll give you feedback exactly like a TCS Technical Architect would. Start with whichever topic you feel weakest on.