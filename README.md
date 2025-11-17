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