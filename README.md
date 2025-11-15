# Prerequisites

1. Install Ollama

Visit Ollama's website to download and install
Pull required models:

```
ollama pull llama3.2  # or your preferred model
ollama pull nomic-embed-text 

```
nomic-embed-text is used for Turning text into vectors (for search, RAG, clustering, etc.)

2. Set Up Environment

```
python3 -m venv venv_rag
source venv_rag/bin/activate  # On Windows: .\venv\Scripts\activate
pip install -r requirements.txt
```

for reboot
deactivate
rm -rf venv_rag
python3 -m venv venv_rag
source venv_rag/bin/activate
pip install -r requirements.txt

then 

# Install the kernel tool
(venv) $ pip install ipykernel