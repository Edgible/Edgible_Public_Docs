# Migrating Ollama from Docker to Native macOS

This guide helps you migrate your Ollama models from Docker to native macOS installation.

## Current Status

You currently have:
- **Docker Ollama** running on port 11434 with models: phi, llama2:7b, codellama:13b, llama2:13b
- **Native Ollama** installed but no models yet

## Migration Steps

### Step 1: Install Native Ollama (if not already)

```bash
# Check if Ollama is installed
which ollama

# If not, install it:
# Option A: Download from https://ollama.ai/download
# Option B: Homebrew
brew install ollama
```

### Step 2: Stop Docker Ollama (optional, can keep running during migration)

```bash
cd Progento/docker
docker compose stop ollama
```

### Step 3: Start Native Ollama

```bash
# Start Ollama service
ollama serve

# Or if using Homebrew service:
brew services start ollama
```

### Step 4: Pull Models in Native Ollama

**Quick Option (Recommended):**
Just pull the models you need. Ollama will download them efficiently:

```bash
# Pull the models you're using
ollama pull phi              # Default model for Progento (1.5 GB)
ollama pull llama2:7b        # If you want this model (3.6 GB)
ollama pull codellama:13b    # If you want this model (6.9 GB)
ollama pull llama2:13b       # If you want this model (6.9 GB)
```

**Time estimate:** 
- phi: ~1-2 minutes
- llama2:7b: ~3-5 minutes  
- 13B models: ~5-10 minutes each

### Step 5: Verify Native Ollama Has Models

```bash
# List models
ollama list

# Should show:
# phi              latest    ...
# llama2:7b        latest    ...
# etc.
```

### Step 6: Update Progento Configuration

The configuration is already set to use `host.docker.internal:11434`, which will connect to native Ollama once Docker Ollama is stopped.

### Step 7: Restart Progento Services

```bash
cd Progento/docker
docker compose restart progento
```

### Step 8: Test Connection

```bash
# From inside the container, test connection to native Ollama
docker exec progento-api curl -s http://host.docker.internal:11434/api/tags

# Should return list of models
```

### Step 9: Remove Docker Ollama (optional)

Once everything is working with native Ollama, you can remove the Docker service:

```bash
cd Progento/docker
docker compose stop ollama
docker compose rm ollama
```

## Advanced: Copy Models from Docker (Not Recommended)

If you have limited bandwidth and want to avoid re-downloading, you could try copying models from Docker, but this is complex and may cause compatibility issues:

1. Docker models are in: `~/.ollama/models` inside the container
2. Native Ollama stores in: `~/.ollama/models` on your Mac
3. The formats should be compatible, but copying manually is error-prone

**Recommendation:** Just pull the models fresh - it's simpler and ensures compatibility.

## Troubleshooting

### Native Ollama won't start

```bash
# Check if port 11434 is already in use
lsof -i :11434

# If Docker Ollama is still using it, stop it first
docker compose stop ollama
```

### Models not appearing

```bash
# Check Ollama is running
ollama list

# Check logs
tail -f ~/.ollama/logs/server.log
```

### Can't connect from Docker container

```bash
# Test connectivity
docker exec progento-api ping -c 1 host.docker.internal

# Should respond with IP address
```

## Benefits After Migration

✅ **Better Performance**: Native Ollama uses M4 chip GPU acceleration  
✅ **Lower Memory**: No Docker overhead  
✅ **Faster Inference**: Direct access to system resources  
✅ **Easier Updates**: Update Ollama independently from Docker  

