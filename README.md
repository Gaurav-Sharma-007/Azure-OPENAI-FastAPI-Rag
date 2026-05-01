# Azure RAG Practice

A small FastAPI backend for retrieval-augmented generation using:

- Azure OpenAI for answer generation
- Azure Cosmos DB for storing document chunks and embeddings
- SentenceTransformers for local text embeddings
- FastAPI and Uvicorn for the HTTP API

The app accepts `.txt` files, splits them into chunks, embeds each chunk locally, stores the chunks in Cosmos DB, and answers questions using the most relevant stored chunks as context.

## Project Files

- `azure-openai.py` - main FastAPI app
- `config.ini` - local runtime configuration, ignored by git
- `requirements.txt` - Python dependencies
- `sentence-transformer-sample.py` - standalone embedding sample
- `uploads/` - uploaded text files

## Setup

Create and activate a virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Install dependencies:

```bash
pip install -r requirements.txt
```

## Configuration

Create or edit `config.ini` in the project root:

```ini
[azure_openai]
endpoint = https://your-azure-openai-resource.openai.azure.com
key = your-azure-openai-key
deployment = your-deployment-name
api_version = 2025-04-01-preview
timeout = 60

[cosmos]
endpoint = https://your-cosmos-account.documents.azure.com:443/
key = your-cosmos-key
database = rag_db
container = rag_chunks

[rag]
st_model = all-MiniLM-L6-v2
collection = Beatles
chunk_size = 500
chunk_overlap = 50
top_k = 3
upload_dir = ./uploads

[debug]
print_embeddings = true
embedding_print_limit = 3
embedding_print_dims = 12
```

`config.ini` is ignored by git because it contains keys. Environment variables can still override config values if needed.

## Run

```bash
uvicorn azure-openai:app --reload
```

Open the Swagger UI:

```text
http://127.0.0.1:8000/docs
```

## API Usage

Health check:

```bash
curl http://127.0.0.1:8000/health
```

Ingest a text file:

```bash
curl -X POST "http://127.0.0.1:8000/ingest?collection=Beatles" \
  -F "file=@The-Beatles.txt"
```

Ask a question:

```bash
curl -X POST "http://127.0.0.1:8000/ask" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "Who were all the members of the Beatles?",
    "top_k": 3,
    "collection": "Beatles"
  }'
```

List logical collections:

```bash
curl http://127.0.0.1:8000/collections
```

Get collection stats:

```bash
curl http://127.0.0.1:8000/collections/Beatles/stats
```

Delete chunks from one source file:

```bash
curl -X DELETE "http://127.0.0.1:8000/documents?source=The-Beatles.txt&collection=Beatles"
```

Delete a full logical collection:

```bash
curl -X DELETE http://127.0.0.1:8000/collections/Beatles
```

## How Retrieval Works

During ingestion, each text file is split into overlapping chunks. The app generates a local embedding for each chunk using SentenceTransformers and stores the chunk text, source filename, chunk index, logical collection name, and embedding vector in Cosmos DB.

During question answering, the app embeds the user query, loads chunks from the selected logical collection, ranks them by cosine similarity, and sends the top chunks to Azure OpenAI as context.

## Troubleshooting

If startup fails with a missing config error, check `config.ini` exists in the project root and that placeholder values have been replaced.

If `/ask` returns an Azure OpenAI authentication error, confirm the endpoint and key come from the same Azure OpenAI resource, and that `deployment` is the Azure deployment name.

If `/ask` says the collection is empty, ingest a `.txt` file into the same `collection` value you are querying.

If Cosmos DB calls fail, confirm the Cosmos endpoint, key, database, and container in `config.ini`. The app creates the database and container if they do not already exist.
