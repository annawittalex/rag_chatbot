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

        (venv) $ pip install ipykernel
        ```

### Phase 2: Daily Usage (100% Offline) 
Once Phase 1 is done, you can disconnect the internet. The following features work without any connection: 
- **Loading Documents**: Processing new PDFs works locally. 
- **Creating Database**: The Vector Database (Chroma) is created and stored on your hard drive. 
- **Chatting**: All questions are answered by the local Llama model. 
- **Data Privacy**: No data is ever sent to the cloud.

## Result Example

![result](/screenshots/result_chat.png)