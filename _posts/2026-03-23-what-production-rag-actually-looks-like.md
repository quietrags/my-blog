---
layout: single
title: "What Production RAG Actually Looks Like"
date: 2026-03-23
categories: [AI]
tags: [rag, production, langchain, llamaindex, haystack, postgres, pgvector, observability, mcp, cohere]
toc: false
excerpt: "RAG is not a use case. It is a pattern — one that shows up in practically every LLM application you build. The interesting question is what it takes to make it work reliably."
---

Everyone talks about RAG. It is one of the most discussed patterns in the LLM world, and practically everyone can rattle off the three or four steps — chunk your documents, embed them, retrieve the relevant pieces, generate an answer. Simple enough to fit on a slide.

But I have been rethinking what RAG actually is. The common framing treats it as a use case — "build a chatbot over your documents." I think that undersells it. RAG is not a use case. It is a foundational pattern. Almost every LLM application you build, including agentic ones, will need to refer to some kind of knowledge base. There is almost always a corpus of information that the model does not have and needs to be given at the right moment. Once you see it that way, RAG stops being a feature you bolt on and becomes a core capability you need to get right.

And getting it right is the hard part. The three-step summary hides an enormous amount of engineering. How do you chunk reliably? How do you know your retrieval is actually returning what matters? How do you debug a bad answer when the failure could be in ingestion, retrieval, or synthesis? These questions require a deeper analysis than most tutorials offer.

What changed my thinking was encountering a treatment of RAG as a systems engineering problem rather than a demo. The discipline it applied — debuggability, observability, idempotent pipelines, explicit control flow — was the same discipline you would bring to any serious production system. That reframing is what this post is about.

## Frameworks as Utilities, Not Architects

There is a line from Paul Iusztin that captures something important: "AI frameworks are good utilities. They should not dictate the architecture or control flow of your system."

This is the tension I keep seeing. LangChain, LlamaIndex, Haystack — all of these offer out-of-the-box ways to build RAG. You call a chain, pass in your documents, and retrieval-augmented generation happens. It works for a demo. But production RAG is probably not at the level where you just invoke a framework primitive and trust the result.

The alternative is to treat these frameworks as libraries of primitives. Use LangChain's document loaders and embedding wrappers, but write the orchestration loop yourself. The ingestion pipeline becomes an explicit sequence:

```python
for file in discover_files(folder):
    raw    = load_document(file.path)       # LangChain loader
    clean  = normalize_text(raw)            # your preprocessing
    chunks = chunk_text(clean, size=512)    # LangChain splitter
    vecs   = embed_chunks(chunks)           # embedding API
    save_to_postgres(file, chunks, vecs)    # your persistence
```

Each step is a separate function. Each step is independently testable and replaceable. The loop itself is almost embarrassingly simple — and that is the point.

This matters because when something goes wrong — and it will — you need to know exactly where it went wrong. Consider a case where a PDF loader produces garbled text due to kerning issues, turning words into something like "P r e - C h u n k." When the loading step is its own isolated function, the bug is immediately visible and fixing it is a one-line swap to a different loader. In a framework chain where loading, chunking, and embedding are all wired together, the same bug would silently produce bad embeddings and hallucinated answers three steps later, with no obvious place to look.

The picture I now carry in my head looks something like this:

![RAG architecture: orchestration layer on top, interchangeable primitives below](/my-blog/assets/images/posts/production-rag/architecture.svg)

The bottom row is where frameworks earn their place — as interchangeable components. The top row is yours. You decide what runs in what order, what gets logged, what gets retried, and what gets skipped. That separation is what makes the system debuggable.

## Knowing the Primitives

Once you accept that the orchestration is yours, the next question is: which framework primitives are actually worth using? I spent some time going through the docs to understand which parts are leaf-node utilities versus "framework taking the wheel."

From **LangChain**, the useful primitives are narrow and well-defined. Document loaders like `PyMuPDFLoader` for complex PDFs or `UnstructuredHTMLLoader` for web pages give you standard Document objects, then your code takes over. `RecursiveCharacterTextSplitter` handles chunking by splitting on paragraph breaks, then sentences, then characters as a fallback. Embedding wrappers give you a consistent `.embed_documents()` interface across OpenAI, Cohere, and HuggingFace — swapping providers becomes a config change. Chat model wrappers (`ChatOpenAI`, `ChatAnthropic`, `ChatGoogleGenerativeAI`) all expose the same `.invoke(messages)` interface, which is what makes a factory pattern work cleanly. What to avoid: chains like `RetrievalQAChain` and LCEL's pipe `|` operator. These take the control flow away from you.

**LlamaIndex** goes deeper on ingestion and retrieval. Its `SimpleDirectoryReader` auto-detects and loads PDFs, Word documents, CSVs, and markdown from a folder — more capable out of the box than LangChain's individual loaders. More interesting is `SentenceWindowNodeParser`, which stores each sentence as a retrievable node but attaches surrounding sentences as context. Retrieval finds the right sentence; generation sees its neighbourhood. This improves answer quality without bloating what you retrieve. And `LlamaParse`, their cloud parser, is widely considered the best available option for complex documents with tables, figures, and multi-column layouts. People use it as a standalone primitive even when they are not using the rest of LlamaIndex.

**Haystack** is well-aligned with the explicit control flow philosophy. Every component is typed and testable with declared inputs and outputs. Its `BM25Retriever` provides classic keyword-based retrieval. Combining BM25 with vector search — hybrid retrieval — is a well-known production pattern that improves recall. Its evaluator components (`FaithfulnessEvaluator`, `ContextRelevanceEvaluator`) are drop-in steps for measuring retrieval and generation quality, which is genuinely underrated for production use.

**Cohere Rerank** is arguably the most widely used standalone primitive in the RAG ecosystem, used regardless of framework. The pattern is a two-stage approach: vector search returns the top 50 candidates quickly but imprecisely, then Cohere's Rerank model re-scores them with much higher accuracy and returns the best 5. Fast coarse retrieval followed by slow precise reranking. It is one of the most reliable quality improvements you can add to a RAG pipeline.

**Unstructured** is the most respected open-source library for parsing real-world messy documents. Its advantage over simpler loaders: it understands document structure. It identifies titles, tables, headers, narrative text, and list items as separate typed elements — not just one blob of text. Structure-aware parsing lets you write smarter chunking logic, like never splitting a table across two chunks.

The mental model: think of these as a layered toolkit where none of the layers need to know about each other. Parsing, chunking, embedding, storage, retrieval, reranking, generation — you pick the best tool for each layer independently, and your orchestration loop ties them together.

## One Database, Many Questions

Here is something I did not expect to find so interesting: the choice to use Postgres with pgvector instead of a dedicated vector database.

The instinct, and certainly my instinct, is to reach for Pinecone or Weaviate or one of the purpose-built vector stores. They are optimised for similarity search and they work well. But there is a compelling alternative: store everything — chunks, embeddings, and file metadata — in a single Postgres database. The practical benefits of this surprised me.

The most striking benefit is observability through plain SQL. If you want to know which chunks came from a particular file and how similar they are to a query, you can write this:

```sql
SELECT chunk_text, embedding <-> $query_vec AS distance
FROM chunks
JOIN files ON chunks.file_id = files.id
WHERE files.uploaded_at > '2026-01-01'
ORDER BY distance
LIMIT 5;
```

The `<->` operator is pgvector's distance function — smaller means more similar. The `JOIN` brings in file metadata. The `WHERE` filters by date. One query, combining semantic similarity with relational filters, run from any SQL client without starting the application, without an SDK, without a dashboard.

This is something you simply cannot do if your vectors live in Pinecone and your metadata lives in a separate database. The ability to inspect and reproduce retrieval outside the application, deterministically, using tools every engineer already knows — that is a genuine advantage.

I still do not know at what scale the choice between pgvector and a dedicated vector database begins to matter. Presumably at very large scale, the specialised databases offer performance that Postgres cannot match. But for most real systems, the observability and relational power seem worth more than performance headroom you may never need.

## The Residue Problem

There is a design concern in RAG pipelines that I think is underappreciated: idempotence.

When you are building a RAG system, you constantly re-chunk. You try different chunk sizes, different overlap strategies, different preprocessing. Each time you re-run the pipeline, it can leave residue — old chunks and embeddings from previous runs sitting in the database alongside the new ones. If you do not handle this carefully, you end up with stale data silently influencing your retrieval results, and no easy way to know it is there.

A clean solution is content hashing. Every file is hashed on ingestion. If the hash already exists, the file is skipped. If the content has changed, old chunks are deleted before new ones are written. The pipeline can be run any number of times and always converges to a consistent state.

This is not an AI-specific idea. It is the same principle that makes good ETL pipelines reliable. But it is rarely discussed in the RAG context, and I think that is a mistake. During development, idempotent ingestion means you can experiment freely without accumulating invisible residue. In production, it means a crashed pipeline can simply be restarted. Already-processed files are skipped, partially-processed files are cleaned up. No manual intervention, no corruption risk.

It is an unglamorous property. But its absence is the kind of thing that produces subtle, maddening bugs — answers that are sometimes wrong for no apparent reason, because the retrieval is pulling from chunks that should no longer exist.

## Where Quality Is Actually Decided

Retrieval is not one step. It is at least four, and each can fail independently:

```python
async def retrieve(query, top_k=5, rerank=False):
    processed = preprocess_query(query)          # clean the input
    embedding = embed_query(processed)           # convert to vector
    results   = search_similar(embedding, top_k) # find nearest chunks
    if rerank:
        results = rerank_results(query, results) # re-score with precision
    return results
```

Preprocessing cleans the raw query — typos, filler words, phrasing that does not match your documents. Embedding converts it to a vector using the same model used at ingestion. Similarity search finds the nearest chunk vectors in the database. And reranking, if enabled, re-scores the top candidates using a model like Cohere Rerank that reads actual text rather than just comparing vectors — much more accurate, but slower.

When an answer is wrong, this structure tells you where to look. Was preprocessing losing important terms? Did the embedding model fail to capture the domain? Did the right chunks exist but not make the top K? Each is a different diagnosis with a different fix. Without explicit steps, all of these look the same — you just get a wrong answer.

But the deeper idea I keep coming back to is the distinction between recall and precision at the chunk level.

When the system retrieves chunks in response to a query, two things can go wrong independently:

![Two retrieval failure modes: poor recall vs poor precision](/my-blog/assets/images/posts/production-rag/recall-precision.svg)

These are different failures with different fixes, and conflating them is a common mistake. If the relevant chunks are not being retrieved, no amount of prompt engineering or model improvement will help. The information is simply not there for the model to work with. That is a retrieval problem — you need better embeddings, better chunking, or better search.

If the relevant chunks are retrieved but the answer is still wrong, then the model had the information and failed to use it correctly. That is a synthesis problem — you might need a better prompt, a more capable model, or fewer distracting chunks.

The diagnostic rule that follows is simple but powerful:

![Diagnostic flowchart: is the answer in the retrieved chunks?](/my-blog/assets/images/posts/production-rag/diagnostic.svg)

Look at what was retrieved, and ask whether the answer is in there. This is the single most clarifying idea I have encountered about RAG debugging. It gives every bad answer a specific address. Without it, debugging a RAG system feels like guessing. With it, you at least know which half of the system to investigate.

## The Model Is Not a Reliable Service

I had been thinking about LLMs as a capability that just works. A more useful framing: treat the model the same way you would treat any unreliable external service — non-deterministic, occasionally unavailable, and capable of returning plausible-but-wrong outputs with no error signal. The engineering response is to isolate it, observe it, and make it swappable.

A factory pattern handles swappability. The rest of the code calls `get_llm()` and never references OpenAI or Anthropic directly. This is where LangChain's chat model wrappers genuinely earn their place — they normalise different provider APIs behind one consistent interface. Switching providers is a config change, not a code change.

For operational resilience, retries handle the transient failures — network timeouts, rate limiting, brief provider outages. A library like `tenacity` wraps the LLM call with exponential backoff: wait 1 second, then 2, then up to 10, and give up after 3 attempts. The important distinction is that retries handle operational failures, not semantic ones. A retry will not make the model smarter. It will not fix a hallucination or a reasoning error. It will make the system resilient to infrastructure blips. These are two very different problems, and conflating them leads to retry loops that waste money without improving answers.

## MCP as an Interface to RAG

An idea I had not seen before is exposing a RAG system through MCP — the Model Context Protocol.

The conventional approach is to serve RAG through a REST API. You build a FastAPI endpoint, accept a query, run retrieval and generation, and return the answer. That works well for applications that know when to call the RAG system and what to ask.

But MCP opens a different possibility. By wrapping the same core function as an MCP tool, you make it discoverable by AI agents. Claude Desktop, Cursor, or any MCP-compatible agent can find the tool, understand what it does, and invoke it autonomously when a question requires grounded knowledge. A user asks "what does our policy say about X?" — the agent decides on its own to call the RAG tool, gets an answer grounded in actual documents, and presents it.

The architectural insight is that the same RAG function serves both interfaces:

```python
# The core — knows nothing about HTTP or MCP
async def generate_answer(request) -> Response:
    chunks  = await retrieve(request.query)
    answer  = await synthesize(request.query, chunks)
    return Response(answer=answer, sources=chunks)

# Adapter 1: REST — for apps that call explicitly
@router.post("/query")
async def query(request: QueryRequest):
    return await generate_answer(request)

# Adapter 2: MCP — for agents that discover and call autonomously
@mcp.tool()
async def query_rag(query: str, top_k: int = 5):
    return await generate_answer(QueryRequest(query=query, top_k=top_k))
```

The core logic takes a query, runs retrieval, calls the model, and returns a response. The serving layer is just an adapter — one for software systems that call explicitly, one for AI agents that discover and call on their own. Serving is an adapter problem, not a RAG problem.

This was genuinely surprising to me. RAG and MCP are usually discussed as separate topics. Seeing them composed this way — MCP as a natural serving interface for a RAG capability — reframed both concepts for me.

## Making the Invisible Visible

The last piece that struck me was the idea of structured observability across the RAG pipeline, using tools like Opik — an open-source LLM observability platform.

I had not encountered Opik before, but the value was immediately obvious. In a traditional API, you measure response time and error rates. In a RAG system, you need to know much more: which phase was slow, how many tokens were consumed, what context the model actually received, what it returned, and what it cost. Without this, performance and cost are assumptions.

A good pattern is to wrap the observability tool behind your own decorator so your RAG functions never reference it directly. If you switch from Opik to LangSmith or Arize Phoenix tomorrow, one wrapper changes. The rest of the codebase is untouched. What you get in return is a trace tree per request that looks something like this:

```
Query: "What is the expense policy?"
├── [retrieval]  preprocess_query        12ms
├── [retrieval]  embed_query             45ms
├── [retrieval]  search_similar_chunks   38ms
├── [retrieval]  rerank_results         210ms
└── [generation] invoke_llm           1,240ms
                 tokens_in:  842
                 tokens_out: 156
                 model: gpt-4o
Total: 1,545ms | Cost: $0.014
```

This turns performance and cost from assumptions into measurable signals. Is reranking worth the 210ms it adds? Is the LLM call dominating cost? Did a prompt change improve token efficiency? These become answerable questions rather than speculation.

The same approach extends to data lineage: preserve chunk IDs through the generation step, so when an answer is wrong, you can trace it deterministically back to which chunks the model used, and from there back to the source file. That kind of lineage turns debugging from speculation into investigation.

I have not used Opik myself, but the pattern it represents — structured observability across the entire RAG pipeline — seems like something that should be standard practice rather than an afterthought.

## What Stays With Me

The thing I find most surprising about production RAG is that the hardest parts have almost nothing to do with AI. Idempotent pipelines, explicit control flow, observability, data lineage — these are the same problems that make any production data system hard. The AI-specific parts — choosing an embedding model, tuning chunk sizes, writing prompts — are almost the easy part by comparison.

Perhaps that is the real insight. RAG is a systems engineering problem that happens to involve a language model. The engineers who will build it well are the ones who bring production discipline to it, not the ones who know the most about transformers.

---

*Much of my thinking here was shaped by Priya Mishra's article "Why Most RAG Tutorials Fail You" in Decoding AI Magazine, and her [rag-101](https://github.com/CalvHobbes/rag-101) implementation on GitHub. Both are worth reading if you want to go deeper.*
