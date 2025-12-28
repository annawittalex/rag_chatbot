# DOCUMENTATION

## First Demo version

- Basic RAG chatbot setup for one document upload, using Ollama embedding model, Chroma DB and MultyqueryRetrieval.

    - Advanced Retrieval (Query Expansion): by using MultiQueryRetriever. Instead of searching for the exact user query, 3 semantic variations are being generated to catch documents that use different wording.

- Local Privacy: The entire stack (Ollama + Chroma + Local Embeddings) runs offline.

## Features implemented

1. **Feature Update: Multiple documents upload.**

    Reasoning: To be able to work with multiple documents and in the future to edit(add/delete) document's stack for one's chatbot context the **data_folder** has been created to provide a single connection to all the documents. 

2. **Feature Update: Maximal Marginal Relevance (MMR)** - **DO I NEED IT AFTER IMPLEMENTING SEMANTICCHUNKING?????**

    Instead of just fetching the top 8 matches; now we are fetching 20 and selecting the 8 most diverse ones (lambda_mult=0.7). This prevents getting 8 identical chunks from the same paragraph.
    ```
    search_kwargs={
            "k": 8,           # Final number of chunks to give to the LLM
            "fetch_k": 20,    # Fetch 20 chunks first, then filter for the best 8 diverse ones
            "lambda_mult": 0.7 # 0.7 = balance (1.0 is pure similarity, 0.0 is pure diversity)
        }
    ```

3. **Feature update: Sustainable file embedding**
    Reasoning: To avoid re-embedding existing documents, embed only new, and delete the embeddings of deleted documents and therefore keep the database clean and up-to-date. 

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

    Change: Replaced RecursiveCharacterTextSplitter with SemanticChunker (via Ollama embeddings).

    Reasoning: To eliminate context fragmentation caused by fixed-size splitting. Semantic chunking dynamically groups sentences based on meaning, ensuring that retrieval chunks represent complete topics rather than arbitrary text fragments.

    Parameter: breakpoint_threshold_type="percentile" has been used. This setting was chosen over standard_deviation or interquartile because it offers the most consistent performance across heterogeneous document types, triggering splits only at the highest relative peaks of semantic difference (natural topic shifts).

    ![chunks](/screenshots/chunks.png)


