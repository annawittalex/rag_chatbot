# Motivation

Managing complex projects often means searching through massive PDF documents to find a single fact. While cloud AI tools (like ChatGPT) can help, uploading sensitive files to them creates serious privacy risks. This Local RAG Chatbot solves that problem:

- **100% Data Privacy**: Your documents never leave your computer. The entire system runs offline, ensuring confidential data (legal, medical, or project specs) remains secure.

- **Instant Information**: Stop manually re-reading 30+ page reports. Instantly extract specific details like deadlines, tasks, and requirements.

- **Trusted Accuracy**: Reduces AI "hallucinations" (lying) by forcing the chatbot to answer only using facts found in your provided documents.

- **Zero Cost**: No monthly subscriptions or API fees. Because it uses your own hardware, you can process unlimited documents for free.

# Laptop Requirements

- MacOS System Requirements

    - MacOS Sonoma (v14) or newer
    - Apple M series (CPU and GPU support) or x86 (CPU only)
​
- Windows System Requirements

    - Windows 10 22H2 or newer, Home or Pro
    - NVIDIA 452.39 or newer Drivers if you have an NVIDIA card
    - AMD Radeon Driver https://www.amd.com/en/support if you have a Radeon card

*Source: Based on Ollama system requirements and general recommendations for running local LLMs (e.g., llama3.2) as documented in [Ollama's official documentation](https://ollama.ai/).*

**General Requirenments:**
- **RAM:** Likely the most important resource. The RAM required depends on the model size:
    - 8GB: minimum recommended for small models (1B, 3B, 7B), though performance may suffer.
    - 16GB: recommended for smooth performance with 7B and 13B models.
    - 32GB or more: optimal for larger models (30B, 40B, 70B).
- **GPU:** Crucial for accelerating model performance by handling massive parallel computations. GPU acceleration options are available via drivers compatible with each OS and for Docker-based setups using NVIDIA or AMD.
- **Disk space:** While Ollama’s codebase is relatively small, model files take up significant space depending on their size. Estimates:
    - Small quantized models → 2GB required
    - Medium quantized models → 5GB required
    - Large quantized models → 40GB required
    - Very large models → 200GB required (some can reach 1.3TB!)
- **CPU:** Most modern processors should suffice, though a minimum of 4 cores is recommended, and 8+ cores is ideal.
- **Operating system:** Latest versions of major OSes (Windows, Linux, macOS) should be compatible, though some may require additional configuration.
- **Network:** Temporary internet access to download models; offline usage afterward

*Source: Simón Rodríguez, 18/11/2025, [Running LLMs Locally: Getting Started with Ollama.](https://en.paradigmadigital.com/dev/running-llms-locally-getting-started-ollama/#:~:text=GPU:%20Crucial%20for%20accelerating%20model,Use%20the%20Ollama%20Docker%20image)*

# Architecture

RAG chatbot is a functional Python prototype of a chatbot based on the LangChain architecture and local open-source models (Ollama). Its goal is to deliver precise, context-specific answers to questions about an uploaded document.

The prototype is intended to significantly improve answer quality and relevance compared to standard Large Language Models (LLMs) by ensuring that responses rely exclusively on information found in the document. This minimizes the risk of hallucinations (invented facts) and ensures that all information is traceable and grounded in the source.

![RAG flow](/RAG_flow.png)
*Icons taken from [Flaticon](https://www.flaticon.com)*

# Installation

### Phase 1: Installation (Internet Required) 

You need an active internet connection to: 
1. **Download Software**: Install Python and Ollama. 
2. **Install Libraries**: Run pip install -r requirements.txt to get LangChain, ChromaDB, etc. 
3. **Pull Models**: You must download the specific AI models into Ollama before going offline. 
    - Run in terminal: 
```ollama pull llama3.2 ``` (LLM)
    - Run in terminal: ```ollama pull nomic-embed-text``` (embedding model)
 
4. **First Run (Recommended)**: It is best to run the script once while online. Some document loaders (like unstructured) may need to auto-download small helper files (like NLTK tokenizers) the first time they process text. 

    - Set Up Environment (on Mac)

        ```
        python3 -m venv venv_rag

        source venv_rag/bin/activate  # On Windows: \venv\Scripts\activate

        pip install -r requirements.txt
        ```
    - For reboot
        ```
        deactivate

        rm -rf venv_rag

        python3 -m venv venv_rag

        source venv_rag/bin/activate

        pip install -r requirements.txt
        
        ```

### Phase 2: Daily Usage (100% Offline) 
Once Phase 1 is done, you can disconnect the internet. The following features work without any connection: 
- **Loading Documents**: Processing new PDFs works locally. 
- **Creating Database**: The Vector Database (Chroma) is created and stored on your hard drive. 
- **Chatting**: All questions are answered by the local Llama model. 
- **Data Privacy**: No data is ever sent to the cloud.

## Result Example

![result](/screenshots/result_chat.png)

# Experiment 

## Benchmarking Framework: A 2×3 Factorial Approach

| Configuration id | Embedding Model | Chunking Strategy |
| :--- | :--- | :--- |
| C1 | Nomic-embed-text | Fixed Size |
| C2 | Nomic-embed-text | Recursive |
| C3 | Nomic-embed-text | Semantic |
| C4 | BGE-M3 | Fixed Size |
| C5 | BGE-M3 | Recursive |
| C6 | BGE-M3 | Semantic |

**Table:** Experiment setup: 2×3 Factorial Approach

The benchmarking setup follows a 2x3 factorial design in which two embedding models are crossed with three chunking strategies, resulting in six configurations (C1–C6). The embedding factor compares BGE-M3 and Nomic-embed-text, while the chunking factor compares Fixed Size, Recursive, and Semantic segmentation. This structure isolates the main effects of representation quality (embedding choice) and context construction (chunking), and also reveals interaction effects where a chunking strategy may work differently depending on the embedding space.

All configurations share the same downstream retrieval-and-generation pipeline so that only the factors of interest vary. In practical terms, each run uses identical retriever and LLM settings, while chunk creation changes according to strategy: fixed and recursive chunkers operate with size/overlap control, whereas semantic chunking uses embedding-aware breakpoints. This controlled design provides a fair basis for comparing speed, retrieval quality, and answer quality across the six experimental conditions.


## Performance and Accuracy Metrics

Evaluation combines efficiency metrics (time in seconds) with accuracy metrics (manual hit rate and RAGAS). Runtime is decomposed into staged measurements: ingestion time (ingestion_seconds), retrieval-only time (retrieval_only_seconds), generation-only time (generation_only_seconds), query-to-response latency (query_to_response_seconds), and total end-to-end runtime (end_to_end_seconds). This decomposition is important because a configuration may improve answer quality while shifting cost from one stage to another.

To assess retrieval accuracy, a manual hit rate is computed from exported retrieved chunks. For each benchmark query, top-k retrieved chunks are audited against expected evidence passages; a hit is counted when at least one retrieved chunk contains the required supporting information. Aggregating this over all queries yields an interpretable grounding measure that is independent of the final generation model’s wording.

For answer-level quality, RAGAS metrics (notably faithfulness and answer relevancy) are used when benchmark question-reference pairs are available. RAGAS complements manual retrieval auditing by quantifying whether generated responses are both relevant to the query and supported by the provided context. Together, staged timing, manual hit rate, and RAGAS provide a balanced view of system behavior across speed, retrieval robustness, and final answer quality.


### Step-by-Step Guideline for Running the Experiment

To ensure your 2×3 factorial design is scientifically valid, follow this exact order:

**Phase 1: The "Cold Start" Cleanup**

Before running the main loop, manually delete the following folders if they exist:

chroma_database/

runs/ (This ensures metrics.csv starts with a fresh header).