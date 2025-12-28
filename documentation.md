# DOCUMENTATION

## Done
- Smart Ingestion (Differential Indexing): You have a custom synchronization engine that checks file hashes and modification times (mtime). This prevents re-embedding documents that haven't changed—a critical feature for scaling.

- Advanced Retrieval (Query Expansion): You are using MultiQueryRetriever. Instead of searching for the exact user query, you generate 3 semantic variations to catch documents that use different wording.

- Maximal Marginal Relevance (MMR): You aren't just fetching the top 8 matches; you are fetching 20 and selecting the 8 most diverse ones (lambda_mult=0.7). This prevents getting 8 identical chunks from the same paragraph.

- Source Attribution: Your format_docs function specifically extracts filenames (os.path.basename) and prepends them to the context so the LLM can cite sources.

- Local Privacy: The entire stack (Ollama + Chroma + Local Embeddings) runs offline.