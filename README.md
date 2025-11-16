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