# Motivation

This project aims to keep sensitive information private by avoiding dependence on large, hosted LLMs. While those models offer strong performance, they also require sending data to external services, which can introduce confidentiality and compliance risks.

It also addresses the challenge of working through lengthy documents, where valuable details are easy to overlook. With this tool, users get a focused assistant that helps extract, organize, and retrieve relevant information without repeatedly re-reading source material. And this all happens locally, on your laptop.

# Laptop Requirements

- CPU: Recent 8+ core processor (Apple M1/M2 or modern Intel/AMD)
- RAM: 16 GB minimum (32 GB recommended for larger models/documents)
- Disk: At least 10 GB free for model weights, embeddings, and virtualenv
- GPU: Optional; Apple Silicon runs well on CPU. On Windows/Linux, a CUDA-capable GPU speeds up inference.
- OS: macOS, Linux, or Windows with WSL2; Python 3.10+ installed
- Network: Temporary internet access to download models; offline usage afterward


# Introduction

RAG chatbot is a functional Python prototype of a chatbot based on the LangChain architecture and local open-source models (Ollama). Its goal is to deliver precise, context-specific answers to questions about an uploaded document (TaskFlow.pdf).

The prototype is intended to significantly improve answer quality and relevance compared to standard Large Language Models (LLMs) by ensuring that responses rely exclusively on information found in the document. This minimizes the risk of hallucinations (invented facts) and ensures that all information is traceable and grounded in the source.

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