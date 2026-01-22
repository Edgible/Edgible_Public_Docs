# Progento Embedding Service Setup (Native macOS)

The embedding service runs natively on macOS to leverage M4 chip GPU via Metal Performance Shaders (MPS) for better performance and lower memory usage.

## Quick Start

### 1. Install Dependencies

```bash
cd Progento

# Create virtual environment (if not exists)
python3 -m venv venv_embedding

# Activate virtual environment
source venv_embedding/bin/activate

# Install dependencies
pip install --upgrade pip
pip install -r embedding_service/requirements.txt
```

### 2. Start the Embedding Service

**Option A: Using the startup script (Recommended)**
```bash
cd Progento
./scripts/start_embedding_service.sh
```

**Option B: Manual start**
```bash
cd Progento
source venv_embedding/bin/activate
cd embedding_service
python main.py
```

The service will start on `http://localhost:8001`

### 3. Verify It's Running

```bash
# Check health
curl http://localhost:8001/health

# Should return:
# {"status":"healthy","model_loaded":true,"device":"mps"}
```

If `device` is `"mps"`, it's using your M4 GPU! If it's `"cpu"`, check PyTorch installation.

### 4. Start Progento Docker Services

```bash
cd Progento/docker
docker compose up -d
```

The Progento API will automatically connect to the embedding service via `host.docker.internal:8001`.

## Troubleshooting

### PyTorch MPS Not Available

If you see `device: "cpu"` instead of `"mps"`:

1. **Check PyTorch version:**
   ```bash
   python3 -c "import torch; print(torch.__version__)"
   ```
   Need PyTorch 1.12+ for MPS support

2. **Reinstall PyTorch:**
   ```bash
   pip install torch torchvision torchaudio
   ```

3. **Verify MPS:**
   ```bash
   python3 -c "import torch; print(torch.backends.mps.is_available())"
   ```
   Should print `True`

### Service Won't Start

- **Port 8001 already in use:**
  ```bash
  lsof -i :8001
  # Kill process if needed
  ```

- **Missing dependencies:**
  ```bash
  pip install -r embedding_service/requirements.txt
  ```

### Progento API Can't Connect

- **Check embedding service is running:**
  ```bash
  curl http://localhost:8001/health
  ```

- **Check Docker can reach host:**
  ```bash
  docker exec progento-api curl -s http://host.docker.internal:8001/health
  ```

- **Verify environment variable:**
  ```bash
  docker exec progento-api env | grep EMBEDDING
  ```
  Should show: `EMBEDDING_SERVICE_URL=http://host.docker.internal:8001`

## Architecture

```
Native macOS (M4 GPU):
  ├── Ollama (LLM) - Port 11434
  └── Embedding Service (NEW) - Port 8001

Docker:
  ├── Weaviate (Vector DB) - Port 8080
  ├── PostgreSQL (Metadata) - Port 5432
  └── Progento API - Port 8000
      └── Calls embedding service via host.docker.internal:8001
```

## Benefits

✅ **M4 GPU Acceleration** - Uses Metal Performance Shaders  
✅ **Lower Memory** - No embedding model in Docker container  
✅ **Better Performance** - Native macOS optimizations  
✅ **Reduced Crashes** - Memory isolated from Docker  
✅ **Consistent Pattern** - Same approach as Ollama migration  

