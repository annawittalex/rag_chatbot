# DOCUMENTATION

## Done

- Basic RAG chatbot setup using Ollama embedding model, Chroma DB and MultyqueryRetrieval.

    - Advanced Retrieval (Query Expansion): by using MultiQueryRetriever. Instead of searching for the exact user query, 3 semantic variations are being generated to catch documents that use different wording.

- Local Privacy: The entire stack (Ollama + Chroma + Local Embeddings) runs offline.


Feature Update: Maximal Marginal Relevance (MMR) - **DO I NEED IT AFTER IMPLEMENTING SEMANTICCHUNKING?????**

Intead of just fetching the top 8 matches; now we are fetching 20 and selecting the 8 most diverse ones (lambda_mult=0.7). This prevents getting 8 identical chunks from the same paragraph.


Feature Update: Smart Ingestion (Differential Indexing)

Implementing a custom synchronization engine that checks file hashes and modification times (mtime). This prevents re-embedding documents that haven't changed—a critical feature for scaling.


Feature Update: Source Attribution

format_docs function specifically extracts filenames (os.path.basename) and prepends them to the context so the LLM can cite sources.

**28.12.2025**
Feature Update: Semantic Segmentation

Change: Replaced RecursiveCharacterTextSplitter with SemanticChunker (via Ollama embeddings).

Reasoning: To eliminate context fragmentation caused by fixed-size splitting. Semantic chunking dynamically groups sentences based on meaning, ensuring that retrieval chunks represent complete topics rather than arbitrary text fragments.

Parameter: Uses breakpoint_threshold_type="percentile". This setting was chosen over standard_deviation or interquartile because it offers the most consistent performance across heterogeneous document types, triggering splits only at the highest relative peaks of semantic difference (natural topic shifts).