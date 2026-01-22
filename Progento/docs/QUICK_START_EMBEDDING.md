# Quick Start: Embedding Service

## Setup and Run (3 Steps)

### Step 1: Install Dependencies

```bash
cd /Users/stefano/Edgible/Progento

# Create and activate virtual environment
python3 -m venv venv_embedding
source venv_embedding/bin/activate

# Install dependencies
pip install --upgrade pip
pip install -r embedding_service/requirements.txt
```

This will install:
- FastAPI & Uvicorn (web framework)
- sentence-transformers (embedding models)
- PyTorch (with Metal/MPS support for M4 GPU)

### Step 2: Start the Embedding Service

**Option A: Using the startup script (easiest)**
```bash
cd /Users/stefano/Edgible/Progento
./scripts/start_embedding_service.sh
```

**Option B: Manual start**
```bash
cd /Users/stefano/Edgible/Progento
source venv_embedding/bin/activate
cd embedding_service
python main.py
```

The service will start on **http://localhost:8001**

### Step 3: Verify It's Working

In a new terminal:
```bash
curl http://localhost:8001/health
```

Expected response:
```json
{
  "status": "healthy",
  "model_loaded": true,
  "device": "mps"
}
```

If `device` is `"mps"` ✅ - It's using your M4 GPU!
If `device` is `"cpu"` ⚠️ - PyTorch MPS not available (but will still work)

### Step 4: Start Progento Services

```bash
cd /Users/stefano/Edgible/Progento/docker
docker compose up -d
```

The Progento API will automatically connect to the embedding service.

## Testing the Embedding Service

```bash
# Test single embedding
curl -X POST http://localhost:8001/embed \
  -H "Content-Type: application/json" \
  -d '{"text": "Hello world"}'

# Test batch embeddings
curl -X POST http://localhost:8001/embed_batch \
  -H "Content-Type: application/json" \
  -d '{"texts": ["Hello", "World"]}'
```

## Running in Background (Optional)

To run the embedding service in the background:

```bash
cd /Users/stefano/Edgible/Progento
source venv_embedding/bin/activate
cd embedding_service
nohup python main.py > embedding_service.log 2>&1 &
```

To stop it:
```bash
pkill -f "python.*embedding_service/main.py"
```

## Architecture

```
┌─────────────────────────────────────┐
│  macOS (Native) - Uses M4 GPU       │
│  ├── Ollama (Port 11434) ✅         │
│  └── Embedding Service (Port 8001) ✅│
└─────────────────────────────────────┘
              ↕ HTTP
┌─────────────────────────────────────┐
│  Docker                             │
│  ├── Progento API (Port 8000)       │
│  ├── Weaviate (Port 8080)           │
│  └── PostgreSQL (Port 5432)         │
└─────────────────────────────────────┘
```

