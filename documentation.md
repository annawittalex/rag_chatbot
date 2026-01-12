# DOCUMENTATION

## First Demo version

- Basic RAG chatbot setup for one document upload, using Ollama embedding model, Chroma DB and MultyqueryRetrieval.

    - Advanced Retrieval (Query Expansion): by using MultiQueryRetriever. Instead of searching for the exact user query, 3 semantic variations are being generated to catch documents that use different wording.

- **Local Privacy**: The entire stack (Ollama + Chroma + Local Embeddings) runs offline.

Source: GitHub Repository tonykipkemboi https://github.com/tonykipkemboi/ollama_pdf_rag/tree/main

- **Local database - ChromaDB**

    ![db](/screenshots/db.png)

    **How and when the database is created:**

    The ChromaDB database is automatically created when the RAG system is initialized. This happens:
    - On application startup (when `initialize_rag()` is first called)
    - After uploading new PDF files
    - After deleting PDF files (the system reinitializes with remaining files)

    The database is created using ChromaDB's `PersistentClient` which stores data locally in the `chroma_database/` directory. If no collection exists, it's created when documents are first embedded. If a collection already exists, only new or changed documents are added to avoid re-embedding existing content.

    **Files in chroma_database/:**
    After the program runs, the following folders, subfolders and files are created:

    - **chroma.sqlite3** = **Metadata & Index Layer** (the "catalog")
        - SQLite database storing
        - Stores structured information about collections, documents, and their properties
        - Acts as a lookup table that points to where actual vector data is stored (references to vector data files)
        - Contains references/pointers to the UUID folders
        - Fast to query for document IDs, metadata, and collection information
        - Relatively small file size (typically KB to MB)
    
    - **UUID folder/** = **Vector Data Storage** (the "storage")
        - Stores the actual high-dimensional vector embeddings (the numerical representations of your text)
        - Contains the HNSW (Hierarchical Navigable Small World) index for fast similarity search
        - Binary format optimized for vector operations and similarity calculations
        - Can be large (MB to GB depending on number of vectors)
        - The UUID identifies which collection or segment the vectors belong to
        - Contains binary files with the actual vector embeddings:
            - **data_level0.bin**: Vector embeddings data
            - **header.bin**: Header/metadata for the vectors
            - **length.bin**: Vector dimension/length info
            - **link_lists.bin**: HNSW index links for fast similarity search
        
    **Analogy:** Think of `chroma.sqlite3` as a library catalog card system that tells you which books exist and where to find them, while the UUID folders are the actual bookshelves containing the books (vectors) themselves.

        

## Features implemented

1. **Feature Update: Multiple documents upload.**

    **Reasoning**: To be able to work with multiple documents and in the future to edit(add/delete) document's stack for one's chatbot context the **data_folder** has been created to provide a single connection to all the documents.

2. **Feature Update: MultiQueryRetriever and Maximal Marginal Relevance (MMR)**

    **Reasoning**: to improve the retrieval and decrease the risk for hallucination

    How MultiQueryRetriever works:
    - Receives the original question.
    - Uses the LLM to generate 3 alternative questions:
    - Sends the original question to the LLM with QUERY_PROMPT
    - LLM returns 3 alternative rephrasings
    
    Now you have 4 questions total: original + 3 alternatives
    - Searches the vector database with each question separately:
    
    For each of the 4 questions, it calls base_retriever.invoke(question)
    base_retriever uses MMR (Maximal Marginal Relevance):
    Fetches 20 chunks from the vector database
    Filters to the best 8 diverse chunks (using lambda_mult=0.7 for balance)
    This happens 4 times (once per question)
    Combines the results:
    Merges/deduplicates chunks from all 4 searches
    Returns the final set of relevant chunks
    
    **Important points**:
    The LLM is only used to generate alternative questions, not to search the database.
    base_retriever searches the vector database (not the LLM) using semantic similarity.
    Each of the 4 questions triggers a separate vector search.
    The results are combined to improve coverage of different phrasings.
    So the flow is: Original Question → LLM generates 3 alternatives → 4 separate vector database searches → Combined results.

    Instead of just fetching the top 8 matches; now we are fetching 20 and selecting the 8 most diverse ones (lambda_mult=0.7). This prevents getting 8 identical chunks from the same paragraph.
    ```
    search_kwargs={
            "k": 8,           # Final number of chunks to give to the LLM
            "fetch_k": 20,    # Fetch 20 chunks first, then filter for the best 8 diverse ones
            "lambda_mult": 0.7 # 0.7 = balance (1.0 is pure similarity, 0.0 is pure diversity)
        }
    ```

    **Step-by-Step Process**:
    1. Fetch phase (fetch_k=20):
        - Retrieves the 20 chunks most similar to your query
        - Uses pure similarity search
    2. Selection phase (k=8):
        - Selects 8 chunks from those 20 using MMR
        - First chunk: the most similar one
        - Each subsequent chunk: balances similarity to the query and difference from already-selected chunks
    3. Balance parameter (lambda_mult=0.7):
        - lambda_mult=1.0: Pure similarity (top-8 most similar, ignores diversity)
        - lambda_mult=0.0: Pure diversity (ignores similarity, maximizes diversity)
        - lambda_mult=0.7: Balanced (about 70% similarity, 30% diversity)


3. **Feature update: Sustainable file embedding**
    **Reasoning**: To avoid re-embedding existing documents, embed only new, and delete the embeddings of deleted documents and therefore keep the database clean and up-to-date. 

    Implementing a synchronization engine that prevents re-embedding unchanged documents:
    - Uses file hashes (file_hash) to detect changes
    - Compares existing hashes in the database with current file hashes
    - Only adds chunks for new or changed files
    - Handles removed files by deleting their chunks

    ```
    # Synchronization Engine (make sure Vector Database = PDF folder)
    # Check if collection exists
    try:
        existing_collection = client.get_collection(name=COLLECTION_NAME)
        # Get existing document metadata to check what's already indexed
        existing_docs = existing_collection.get(include=["metadatas"]) # just send the labels
        existing_file_hashes = {
            meta.get("file_hash"): meta.get("file_path") #key: value
            for meta in existing_docs.get("metadatas", []) # The [] is a safety net: if the database returns nothing, it loops over an empty list instead of crashing.
            if meta and meta.get("file_hash") # if meta is not None and meta has a "file_hash" key
        }
        print(f"Found existing collection with {len(existing_file_hashes)} tracked files.")
    except Exception:
        # Collection doesn't exist yet -> create empty dictionary
        existing_file_hashes = {} 
        print("No existing collection found. Creating new one.")

    # Identify new or changed documents
    current_file_hashes = {
        chunk.metadata.get("file_hash"): chunk.metadata.get("file_path") #key: value
        for chunk in chunks
        if chunk.metadata.get("file_hash")
    }

    # Find chunks that need to be added (new or changed files)
    chunks_to_add = [
        chunk for chunk in chunks 
        if chunk.metadata.get("file_hash") not in existing_file_hashes
    ]

    # Find files that were removed (exist in DB but not in current files)
    removed_files = set(existing_file_hashes.keys()) - set(current_file_hashes.keys())

    # Handle removed files: delete their chunks from the vector database
    if removed_files:
        print(f"Detected {len(removed_files)} removed file(s).")
        try:
            existing_collection = client.get_collection(name=COLLECTION_NAME)
            
            # OPTIMIZED: Ask DB only for chunks where 'file_hash' is in our removed list
            # The "$in" operator checks if the field value exists in the provided list
            target_docs = existing_collection.get(
                where={"file_hash": {"$in": list(removed_files)}}
            )
            
            ids_to_delete = target_docs["ids"]
            
            if ids_to_delete:
                existing_collection.delete(ids=ids_to_delete)
                print(f"Deleted {len(ids_to_delete)} chunk(s).")
            else:
                print("Warning: Files were removed, but no matching chunks were found in DB.") # e.g. Manual deletion: chunks were deleted outside this code
                
        except Exception as e:
            print(f"Error removing chunks: {e}")

    # CREATE VECTOR DATABASE
    # Add new or changed chunks to vector database
    if chunks_to_add:
        # Load or create vector database
        try:
            vector_db = Chroma(
                collection_name=COLLECTION_NAME,
                embedding_function=embeddings,
                client=client
            )
            # Add only new/changed chunks
            vector_db.add_documents(chunks_to_add)
            print(f"Added {len(chunks_to_add)} new/changed chunks to vector database.")
        except Exception:
            # Collection doesn't exist, create it with all chunks
            vector_db = Chroma.from_documents(
                documents=chunks,
                embedding=embeddings,
                collection_name=COLLECTION_NAME,
                client=client
            )
            print(f"Created new vector database with {len(chunks)} chunks.")
    else:
        # All documents already indexed
        vector_db = Chroma(
            collection_name=COLLECTION_NAME,
            embedding_function=embeddings,
            client=client
        )
        print(f"No new or changed documents. Using existing vector database with {len(chunks)} current chunks.")


    ```

4. Feature Update: Source Attribution

    format_docs function specifically extracts filenames (os.path.basename) and prepends them to the context so the LLM can cite sources.

    ![format_docs function](/screenshots/formatdocs.png)


5. **14.12.2025** llama model downgrade from 3.2 to 3.1

    Reasoning: After the research llama model version has been downgraded from 3.2 to 3.1 to be able to use 8B parameters model (instead of 3B).

    Llama 3.1 is the current "flagship" text model for standard use. It comes in the robust 8B size (as well as 70B and 405B), making it the direct successor to the standard Llama 3. It handles complex reasoning and RAG tasks much better.

    Llama 3.2 was released specifically as a lightweight (1B and 3B parameters) or vision-specialized model. While it is newer, the text-only versions are significantly smaller and "dumber" than the 8B model, optimized for edge devices (like phones) rather than deep reasoning.

    Source: https://blog.getbind.co/2024/09/27/llama-3-2-overview-is-it-better-than-llama-3-1-and-gpt-4o/#:~:text=Local%20Processing%3A%20The%20lightweight%20models,processing%20and%20high%20data%20privacy.

    ```
    (venv_rag) alexandragalimurka@MacBookPro rag_chatbot % ollama list
    NAME                       ID              SIZE      MODIFIED    
    llama3.1:latest            46e0c10c039e    4.9 GB    2 weeks ago    
    nomic-embed-text:latest    0a109f422b47    274 MB    6 weeks ago    
    ```

6. **28.12.2025** Feature Update: Semantic Segmentation

    **Change**: Replaced RecursiveCharacterTextSplitter with SemanticChunker (via Ollama embeddings).

    **Reasoning**: To eliminate context fragmentation caused by fixed-size splitting. Semantic chunking dynamically groups sentences based on meaning (using nomic-embed-text), ensuring that retrieval chunks represent complete topics rather than arbitrary text fragments.

    **Parameter**: ```breakpoint_threshold_type="percentile"``` (default: 95th percentile) has been used. This method analyzes the relative distribution of semantic differences across the entire document and triggers a split only when a shift in meaning exceeds the top 5% of all observed changes. This ensures that splits occur exclusively at the most significant narrative breaks, preserving the semantic integrity of each chunk for the LLM.
    
    - Why not standard_deviation? Standard deviation assumes the changes in your text follow a "Normal Distribution" (Bell Curve). However, human writing is "bursty"—we stay on one topic for a long time (low difference) and then suddenly switch (massive difference). Standard deviation is often too sensitive to the small, noisy variations within a single paragraph.

    - Why not interquartile (IQR)? IQR is a statistical method designed to ignore outliers. In semantic chunking, the outliers are the goal. A massive spike in semantic difference is the topic change. Using IQR often effectively "filters out" the very splits you are trying to find.

    - Why percentile? (The Winner) Percentile is adaptive to the document. It doesn't care if the document is a dry legal text (where all sentences are similar) or a dramatic novel (where sentences are very different). It simply asks: "Relative to this specific document, where are the biggest 5% of changes?" This guarantees you get splits at the most logical points for that specific file.

    ![chunks](/screenshots/chunks.png)

    **Source**: Kamradt, G. (2024). 5 Levels Of Text Splitting. FullStackRetrieval.io. Available at: https://github.com/FullStackRetrieval-com/RetrievalTutorials


7. **28.12.2025** Looking for reasons of slow performance:

    by adding a timing/profiling version to measure each step:
    ```
    def chat_with_pdf(question, profile=False):
        """
        Chat with the PDF using the RAG chain.
        
        Args:
            question: The question to ask
            profile: If True, prints timing information for each step
        """
        if profile:
            print("=" * 60)
            print("PROFILING chat_with_pdf")
            print("=" * 60)
            
            # Time the retriever step (includes MultiQueryRetriever LLM call)
            print("\n[1] Retrieving documents...")
            start_retrieve = time.time()
            docs = retriever.invoke(question)
            time_retrieve = time.time() - start_retrieve
            print(f"    ✓ Retrieved {len(docs)} documents in {time_retrieve:.2f} seconds")
            
            # Time the formatting step
            print("\n[2] Formatting documents...")
            start_format = time.time()
            formatted_context = format_docs(docs)
            time_format = time.time() - start_format
            print(f"    ✓ Formatted context ({len(formatted_context)} chars) in {time_format:.3f} seconds")
            
            # Time the LLM answer generation
            print("\n[3] Generating answer with LLM...")
            start_llm = time.time()
            answer = llm.invoke(prompt.format(context=formatted_context, question=question))
            time_llm = time.time() - start_llm
            print(f"    ✓ Generated answer in {time_llm:.2f} seconds")
            
            # Time the parsing step
            print("\n[4] Parsing output...")
            start_parse = time.time()
            parsed_answer = StrOutputParser().invoke(answer)
            time_parse = time.time() - start_parse
            print(f"    ✓ Parsed output in {time_parse:.3f} seconds")
            
            total_time = time_retrieve + time_format + time_llm + time_parse
            print("\n" + "=" * 60)
            print("TIMING SUMMARY:")
            print(f"  Retrieval:     {time_retrieve:.2f}s ({time_retrieve/total_time*100:.1f}%)")
            print(f"  Formatting:    {time_format:.3f}s ({time_format/total_time*100:.1f}%)")
            print(f"  LLM Answer:    {time_llm:.2f}s ({time_llm/total_time*100:.1f}%)")
            print(f"  Parsing:       {time_parse:.3f}s ({time_parse/total_time*100:.1f}%)")
            print(f"  TOTAL TIME:    {total_time:.2f}s")
            print("=" * 60 + "\n")
            
            return display(Markdown(parsed_answer))
        else:
            return display(Markdown(chain.invoke(question)))

    ```
    and detailed profiling cell to break down the MultiQueryRetriever step
    ```
    # Detailed profiling function to understand MultiQueryRetriever performance
    def profile_retrieval_step(question):
        """
        Detailed profiling of the retrieval step to understand MultiQueryRetriever overhead.
        """
        print("=" * 60)
        print("DETAILED RETRIEVAL PROFILING")
        print("=" * 60)
        
        # Test 1: Direct base retriever (without MultiQueryRetriever)
        print("\n[Test 1] Direct base retriever (no MultiQueryRetriever)...")
        start_base = time.time()
        base_docs = base_retriever.invoke(question)
        time_base = time.time() - start_base
        print(f"    ✓ Base retriever: {len(base_docs)} docs in {time_base:.2f}s")
        
        # Test 2: MultiQueryRetriever (includes LLM call for query expansion)
        print("\n[Test 2] MultiQueryRetriever (includes LLM query expansion)...")
        start_multi = time.time()
        multi_docs = retriever.invoke(question)
        time_multi = time.time() - start_multi
        print(f"    ✓ MultiQueryRetriever: {len(multi_docs)} docs in {time_multi:.2f}s")
        print(f"    ⚠ MultiQueryRetriever overhead: {time_multi - time_base:.2f}s")
        
        print("\n" + "=" * 60)
        print("ANALYSIS:")
        print(f"  Base retrieval time:        {time_base:.2f}s")
        print(f"  MultiQueryRetriever time:   {time_multi:.2f}s")
        print(f"  Query expansion overhead:   {time_multi - time_base:.2f}s ({((time_multi - time_base) / time_multi * 100):.1f}%)")
        print("=" * 60 + "\n")
        
        return base_docs, multi_docs

    # Uncomment to run detailed retrieval profiling:
    # profile_retrieval_step("What are these documents about?")
    ```

**Results:**

After upgrading from llama 3.1 to llama 3.2 -> the answer generation is 2 x faster for example 1 and 3 x faster for example 2
### llama 3.1
**exercise 1 result llama 3.1**
![exercise 1 result llama 3.1](/screenshots/llama3.1_ex1.png)
**exercise 2 result llama 3.1** 
![exercise 2 result llama 3.1](/screenshots/llama3.1_ex2.png)
### llama 3.2
**exercise 1 result llama 3.2**
![exercise 1 result llama 3.2](/screenshots/llama3.2_ex1.png)
**exercise 2 result llama 3.2**
![exercise 2 result llama 3.2](/screenshots/llama3.2_ex2.png)