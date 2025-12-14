# Motivation

This project aims to keep sensitive information private by avoiding dependence on large, hosted LLMs. While those models offer strong performance, they also require sending data to external services, which can introduce confidentiality and compliance risks.

It also addresses the challenge of working through lengthy documents, where valuable details are easy to overlook. With this tool, users get a focused assistant that helps extract, organize, and retrieve relevant information without repeatedly re-reading source material. And this all happens locally, on your laptop.

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

# Introduction

RAG chatbot is a functional Python prototype of a chatbot based on the LangChain architecture and local open-source models (Ollama). Its goal is to deliver precise, context-specific answers to questions about an uploaded document.

The prototype is intended to significantly improve answer quality and relevance compared to standard Large Language Models (LLMs) by ensuring that responses rely exclusively on information found in the document. This minimizes the risk of hallucinations (invented facts) and ensures that all information is traceable and grounded in the source.

![RAG flow](/RAG_flow.png)
*Icons taken from [Flaticon](https://www.flaticon.com)*

# Prerequisites

1. Install Ollama

Visit Ollama's website to download and install
Pull required models:

```
ollama pull llama3.2  # or your preferred model
ollama pull nomic-embed-text 

```


2. Set Up Environment (on Mac)

```
python3 -m venv venv_rag

source venv_rag/bin/activate  # On Windows: \venv\Scripts\activate

pip install -r requirements.txt
```

For reboot
```
deactivate

rm -rf venv_rag

python3 -m venv venv_rag

source venv_rag/bin/activate

pip install -r requirements.txt

(venv) $ pip install ipykernel
```