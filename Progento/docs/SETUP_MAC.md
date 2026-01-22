# Progento Setup on macOS (Apple Silicon)

Quick setup guide for running Progento on macOS with M4 chip GPU acceleration.

## Prerequisites

- **macOS** with Apple Silicon (M4 recommended)
- **32GB RAM** (or similar)
- **Docker Desktop** installed and running
- **Homebrew** (optional, for easy installation)

## Quick Installation

### 1. Install Ollama

```bash
# Download from https://ollama.ai/download
# Or using Homebrew:
brew install ollama
```

### 2. Install Docker Desktop

Download and install from [docker.com](https://www.docker.com/products/docker-desktop/)

### 3. Pull Required Models

```bash
# Pull the default model (phi - small, fast, ~1.6GB)
ollama pull phi

# Optional: Pull better models for code queries (requires more RAM)
# ollama pull codellama:7b    # ~3.8GB - Better for code
# ollama pull llama3:8b       # ~4.9GB - General purpose
```

### 4. Start All Services

The easiest way to start everything:

```bash
cd /path/to/Progento
./scripts/start_progento.sh
```

This script automatically:
- ✅ Starts Ollama (LLM service) on port 11434
- ✅ Starts Embedding Service (native, using M4 GPU) on port 8001
- ✅ Starts Docker services (Weaviate, PostgreSQL, Progento API)
- ✅ Waits for all services to be ready
- ✅ Verifies M4 GPU usage

**Expected output:**
```
=========================================
All Progento services started!
=========================================

Services:
  - Ollama (LLM):        http://localhost:11434
  - Embedding Service:   http://localhost:8001
  - Weaviate:            http://localhost:8080
  - PostgreSQL:          localhost:5432
  - Progento API:        http://localhost:8000
```

### 5. Verify Installation

```bash
# Check all services are healthy
./scripts/test_components.sh

# Or manually verify:
curl http://localhost:8000/api/health
curl http://localhost:8001/health  # Should show "device":"mps" for M4 GPU
```

### 6. Access the UI

Open your browser to:
```
http://localhost:3000
```

## Architecture

Progento uses a hybrid architecture optimized for Apple Silicon:

```
Native macOS (M4 GPU):
  ├── Ollama (LLM) - Port 11434
  └── Embedding Service - Port 8001

Docker:
  ├── Progento API (FastAPI) - Port 8000
  ├── Weaviate (Vector Database) - Port 8080
  └── PostgreSQL (Metadata) - Port 5432
```

**Why Native Services?**
- **M4 GPU Acceleration**: Native Ollama and embedding service use Metal Performance Shaders (MPS) for better performance
- **Lower Memory Overhead**: No Docker container overhead for ML models
- **Better Resource Management**: macOS optimizes memory and CPU for ML workloads

## Managing Services

**Stop all services:**
```bash
./scripts/stop_progento.sh
```

**Check service status:**
```bash
./scripts/status_progento.sh
```

**View logs:**
```bash
# Ollama and Embedding Service logs
tail -f .pids/ollama.log
tail -f .pids/embedding_service.log

# Docker services logs
cd docker
docker compose logs -f
```

## Troubleshooting

### Ollama Not Starting

```bash
# Check if Ollama is installed
which ollama

# Start Ollama manually
ollama serve

# Verify it's running
curl http://localhost:11434/api/tags
```

### Embedding Service Not Using GPU

Check the health endpoint:
```bash
curl http://localhost:8001/health
```

Should show `"device":"mps"`. If it shows `"device":"cpu"`, check:
- PyTorch MPS support is installed (handled automatically by start script)
- macOS version supports MPS (macOS 12.3+)

### Docker Services Not Starting

```bash
# Check Docker Desktop is running
docker info

# Restart Docker services
cd docker
docker compose down
docker compose up -d
```

### Port Conflicts

If ports are already in use:
- **11434**: Ollama - stop with `pkill ollama`
- **8001**: Embedding Service - stop with `pkill -f "python.*main.py"`
- **8000, 8080, 5432**: Docker services - stop with `docker compose down`

## Next Steps

1. **Add a Repository**: Use the UI or API to add a local repository path
2. **Scan Repository**: Click "Scan" to index the codebase
3. **Query**: Ask questions about your code using natural language

See the main [README.md](../README.md) for more details on using Progento.
